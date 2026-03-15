# Snowflake RBAC & Data Security

## Role Hierarchy

A production Snowflake environment uses purpose-specific roles with least-privilege access:

```
ACCOUNTADMIN
    └── SYSADMIN
        ├── ENGINEER         — dbt development, model building, staging/integration writes
        ├── CHANGE_CONTROL   — production deployments only (CI/CD service account)
        ├── ANALYST          — read access to T3 Integration + T4 Presentation
        ├── CONSUMER         — read access to T4 Presentation only
        ├── SCIENTIST        — read access to T3 + T4, plus ML workspace
        └── TASKADMIN        — owns Snowflake Tasks and scheduled jobs
    └── SECURITYADMIN
        └── manages role grants and user assignments
```

### Role-to-Warehouse Mapping

Isolate workloads by assigning specific warehouses per role:

| Prefix | Purpose | Roles |
|--------|---------|-------|
| `IN_` | Ingestion (Fivetran, data loaders) | Service accounts |
| `TRN_` | Transformation (dbt runs) | ENGINEER, CHANGE_CONTROL |
| `OUT_` | Consumption (BI queries, dashboards) | ANALYST, CONSUMER, SCIENTIST |

```sql
-- Warehouse creation pattern
CREATE WAREHOUSE IF NOT EXISTS TRN_DEV_CENTRAL_WH
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

GRANT USAGE ON WAREHOUSE TRN_DEV_CENTRAL_WH TO ROLE ENGINEER;
```

### Role Creation

```sql
CREATE ROLE IF NOT EXISTS ENGINEER;
CREATE ROLE IF NOT EXISTS CHANGE_CONTROL;
CREATE ROLE IF NOT EXISTS ANALYST;
CREATE ROLE IF NOT EXISTS CONSUMER;

-- Hierarchy
GRANT ROLE ENGINEER TO ROLE SYSADMIN;
GRANT ROLE CHANGE_CONTROL TO ROLE SYSADMIN;
GRANT ROLE ANALYST TO ROLE SYSADMIN;
GRANT ROLE CONSUMER TO ROLE ANALYST;  -- Analyst inherits Consumer privileges
```

## Multi-Tier Database Grants

In a tiered architecture (T0–T4), each role gets access only to the tiers it needs:

```sql
-- ENGINEER: full access to dev databases
GRANT ALL ON DATABASE DEV_T2_PERSISTENT_STAGING TO ROLE ENGINEER;
GRANT ALL ON DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;

-- CHANGE_CONTROL: prod write access (CI/CD only)
GRANT ALL ON DATABASE PROD_T2_PERSISTENT_STAGING TO ROLE CHANGE_CONTROL;
GRANT ALL ON DATABASE PROD_T3_INTEGRATION TO ROLE CHANGE_CONTROL;

-- ANALYST: read-only on integration + presentation
GRANT USAGE ON DATABASE PROD_T3_INTEGRATION TO ROLE ANALYST;
GRANT SELECT ON ALL TABLES IN SCHEMA PROD_T3_INTEGRATION.LOGISTICS TO ROLE ANALYST;
GRANT USAGE ON DATABASE PROD_T4_PRESENTATION TO ROLE ANALYST;
GRANT SELECT ON ALL VIEWS IN SCHEMA PROD_T4_PRESENTATION.LOGISTICS TO ROLE ANALYST;

-- CONSUMER: presentation only
GRANT USAGE ON DATABASE PROD_T4_PRESENTATION TO ROLE CONSUMER;
GRANT SELECT ON ALL VIEWS IN SCHEMA PROD_T4_PRESENTATION.LOGISTICS TO ROLE CONSUMER;
```

## Data Classification

Use a seed-driven approach to classify columns without hardcoding in views:

### Classification Seed (`data_classification_tags.csv`)

```csv
table_name,column_name,classification,description,permitted_roles,masked_roles,masking_policy_applied
CUSTOMERS,CUSTOMER_NAME,PII_HIGH,Direct identifier,ENGINEER|ANALYST,CONSUMER,true
CUSTOMERS,CONTACT_EMAIL,PII_HIGH,Direct identifier,ENGINEER,ANALYST|CONSUMER,true
CUSTOMERS,CONTACT_PHONE,PII_HIGH,Direct identifier,ENGINEER,ANALYST|CONSUMER,true
CUSTOMERS,CREDIT_LIMIT,FINANCIAL,Monetary value,ENGINEER|ANALYST,CONSUMER,false
SHIPMENTS,REVENUE,OPERATIONAL,Commercially sensitive,ENGINEER|ANALYST,CONSUMER,false
```

### Classification Tiers

| Tier | Examples | Treatment |
|------|----------|-----------|
| `PII_HIGH` | Name, email, phone | Masking policy required |
| `PII_LOW` | Account manager name | Masking recommended |
| `FINANCIAL` | Credit limit, revenue | Restricted by role |
| `OPERATIONAL` | Cost, margin | Restricted by role |
| `UNCLASSIFIED` | Default for all other columns | No restriction |

### Classification Catalogue View

A generic view joins `SNOWFLAKE.ACCOUNT_USAGE.COLUMNS` against the seed:
- Classified columns show their tier, permitted/masked roles, and masking status
- Unclassified columns appear with `classification = 'UNCLASSIFIED'`
- Orphaned seed rows (column renamed/dropped) surface with `tag_orphaned = true`

To classify a new column: add a row to the seed, run `dbt seed --select data_classification_tags`.

## Access History Auditing

Monitor who accessed sensitive data using `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY`:

```sql
WITH raw_access AS (
  SELECT
    ah.query_id,
    ah.query_start_time,
    ah.user_name,
    obj.value:objectName::VARCHAR AS object_name,
    obj.value:columns AS columns_accessed
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
    LATERAL FLATTEN(input => ah.direct_objects_accessed) obj
  WHERE ah.query_start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
)

SELECT
  *,
  -- Off-hours detection (outside 06:00–22:00 AEST)
  CASE
    WHEN EXTRACT(HOUR FROM query_start_time) NOT BETWEEN 20 AND 12
    THEN TRUE ELSE FALSE
  END AS is_off_hours,
  -- Sensitive column detection
  TO_JSON(columns_accessed) ILIKE '%CUSTOMER_NAME%'
    OR TO_JSON(columns_accessed) ILIKE '%CONTACT_EMAIL%'
  AS accessed_pii,
  -- Risk scoring
  CASE
    WHEN is_off_hours AND accessed_pii THEN 'HIGH'
    WHEN is_off_hours THEN 'MEDIUM'
    WHEN accessed_pii THEN 'LOW'
    ELSE 'NORMAL'
  END AS access_risk_level
FROM raw_access
```

**Note:** ACCESS_HISTORY has up to 3-hour ingestion lag — use for daily compliance review, not real-time alerting.

## Resource Monitors

Prevent runaway costs with account-level credit limits:

```sql
CREATE OR REPLACE RESOURCE MONITOR RM_ACCOUNT
  WITH CREDIT_QUOTA = 435
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO NOTIFY
    ON 110 PERCENT DO SUSPEND;

ALTER ACCOUNT SET RESOURCE_MONITOR = RM_ACCOUNT;
```

## SQL Injection Prevention in Dynamic SQL

### Parameterized Queries

Always use parameter binding — never string interpolation for user input:

```python
# SAFE — parameterized
sql = "SELECT SNOWFLAKE.CORTEX.COMPLETE(%s, %s) AS response"
run_sql(sql, params=(model_name, prompt))

# DANGEROUS — string interpolation
sql = f"SELECT SNOWFLAKE.CORTEX.COMPLETE('{model_name}', '{prompt}')"
```

### Identifier Validation

Database, schema, and warehouse names can't be parameterized in DDL. Validate with a strict regex:

```python
import re

def safe_id(name: str) -> str:
    """Only allow alphanumeric + underscore identifiers."""
    if not re.match(r'^[A-Za-z_][A-Za-z0-9_]*$', name):
        raise ValueError(f"Invalid Snowflake identifier: {name}")
    return name
```

### Model Name Allowlisting

For `CORTEX.COMPLETE(model, prompt)`, validate the model against a known set:

```python
ALLOWED_MODELS = frozenset({'mistral-large2', 'mixtral-8x7b', 'llama3.1-70b', 'llama3.1-8b'})

def safe_model_name(model: str) -> str:
    model = model.strip().lower()
    if model not in ALLOWED_MODELS:
        raise ValueError(f"Unknown model: {model}")
    return model
```

## Cortex AI Grants

Grant access to Snowflake Cortex functions:

```sql
USE ROLE ACCOUNTADMIN;

-- Database role for text generation
CREATE DATABASE ROLE IF NOT EXISTS CORTEX_USER;
GRANT USAGE ON FUNCTION SNOWFLAKE.CORTEX.COMPLETE(VARCHAR, VARCHAR) TO DATABASE ROLE CORTEX_USER;
GRANT DATABASE ROLE CORTEX_USER TO ROLE ENGINEER;
GRANT DATABASE ROLE CORTEX_USER TO ROLE ANALYST;

-- Database role for embedding generation (ingestion)
CREATE DATABASE ROLE IF NOT EXISTS CORTEX_EMBED_USER;
GRANT USAGE ON FUNCTION SNOWFLAKE.CORTEX.AI_EMBED(VARCHAR, VARCHAR) TO DATABASE ROLE CORTEX_EMBED_USER;
GRANT DATABASE ROLE CORTEX_EMBED_USER TO ROLE ENGINEER;
```

## Network Policies

Restrict access by IP range:

```sql
CREATE NETWORK POLICY IF NOT EXISTS OFFICE_POLICY
  ALLOWED_IP_LIST = ('203.0.113.0/24', '198.51.100.0/24')
  BLOCKED_IP_LIST = ();

ALTER ACCOUNT SET NETWORK_POLICY = OFFICE_POLICY;
```

---

## Extended Role Hierarchy (Logistics Analytics Platform)

The logistics-analytics-platform introduces a deeper, domain-specific role hierarchy that extends the base pattern above with specialised roles for ML engineering, data stewardship, and dbt environment isolation.

### Full Role Set

```
ACCOUNTADMIN
    └── SYSADMIN
        ├── DATA_ENGINEER      — full pipeline access, all schema writes
        │   ├── DATA_ANALYST   — marts and analytics read access
        │   │   └── BUSINESS_USER — read-only analytics views
        │   ├── DATA_SCIENTIST — ML features, ML objects, feature store
        │   └── ML_ENGINEER    — ML features, ML serving, model deployment
        ├── DBT_DEV_ROLE       — inherits DATA_ENGINEER, full dev environment
        ├── DBT_STAGING_ROLE   — inherits DATA_ENGINEER, full staging environment
        └── DBT_PROD_ROLE      — inherits DATA_ENGINEER, full production environment
    └── SECURITYADMIN
        ├── SECURITY_ADMIN     — security schema only, access control policies
        └── DATA_STEWARD       — governance schema, data quality and compliance
```

Key design decisions:
- **DATA_ANALYST inherits BUSINESS_USER** -- analysts can see everything business users can, plus marts-level detail
- **DATA_SCIENTIST and ML_ENGINEER are peers** under DATA_ENGINEER, not hierarchically related to each other
- **dbt roles inherit DATA_ENGINEER** rather than being separate -- this ensures dbt runs have the same permissions as manual engineering work
- **DATA_STEWARD sits under SECURITYADMIN**, not SYSADMIN, to enforce separation of duties between data operations and governance oversight

### Role Descriptions

| Role | Responsibility | Schema Access |
|------|---------------|---------------|
| `DATA_ENGINEER` | Pipeline development, all data layers | RAW, STAGING, MARTS, ML_FEATURES, SNAPSHOTS, MONITORING, PERFORMANCE |
| `DATA_ANALYST` | Business analysis, reporting | MARTS (core + analytics) |
| `DATA_SCIENTIST` | Model development, experimentation | ML_FEATURES, ML_OBJECTS, ML_SERVING |
| `ML_ENGINEER` | Model deployment, feature engineering | ML_FEATURES, ML_OBJECTS, ML_SERVING, ANALYTICS_ML_FEATURES |
| `BUSINESS_USER` | Stakeholder read-only access | Analytics views only |
| `DATA_STEWARD` | Data quality, compliance | GOVERNANCE |
| `SECURITY_ADMIN` | Access control, security policies | SECURITY |

### Role Creation

```sql
CREATE ROLE IF NOT EXISTS DATA_ENGINEER;
CREATE ROLE IF NOT EXISTS DATA_ANALYST;
CREATE ROLE IF NOT EXISTS DATA_SCIENTIST;
CREATE ROLE IF NOT EXISTS ML_ENGINEER;
CREATE ROLE IF NOT EXISTS BUSINESS_USER;
CREATE ROLE IF NOT EXISTS DATA_STEWARD;
CREATE ROLE IF NOT EXISTS SECURITY_ADMIN;

-- Hierarchy
GRANT ROLE DATA_ANALYST TO ROLE DATA_ENGINEER;
GRANT ROLE BUSINESS_USER TO ROLE DATA_ANALYST;
GRANT ROLE DATA_SCIENTIST TO ROLE DATA_ENGINEER;
GRANT ROLE ML_ENGINEER TO ROLE DATA_ENGINEER;
```

## Multi-Environment dbt Role Isolation

The logistics-analytics-platform uses environment-specific dbt roles with dedicated warehouse sizing:

### dbt Role Pattern

```sql
CREATE ROLE IF NOT EXISTS DBT_DEV_ROLE;
CREATE ROLE IF NOT EXISTS DBT_STAGING_ROLE;
CREATE ROLE IF NOT EXISTS DBT_PROD_ROLE;

-- Each inherits DATA_ENGINEER
GRANT ROLE DATA_ENGINEER TO ROLE DBT_DEV_ROLE;
GRANT ROLE DATA_ENGINEER TO ROLE DBT_STAGING_ROLE;
GRANT ROLE DATA_ENGINEER TO ROLE DBT_PROD_ROLE;
```

### Environment-Specific Database Naming

Use an environment variable to drive the target database, keeping the SQL scripts reusable across DEV, STAGING, and PROD:

```sql
SET DATABASE_NAME = IFNULL($SF_DATABASE, 'LOGISTICS_DW_DEV');

-- All grants reference IDENTIFIER($DATABASE_NAME)
GRANT USAGE ON DATABASE IDENTIFIER($DATABASE_NAME) TO ROLE DATA_ENGINEER;
GRANT USAGE ON DATABASE IDENTIFIER($DATABASE_NAME) TO ROLE DBT_DEV_ROLE;
```

This pattern allows the same permission script to run in any environment by changing one variable:
- `SF_DATABASE=LOGISTICS_DW_DEV` for development
- `SF_DATABASE=LOGISTICS_DW_STAGING` for staging
- `SF_DATABASE=LOGISTICS_DW_PROD` for production

### Warehouse-to-Role Mapping (Logistics Platform)

| Warehouse | Size | Assigned Role | Purpose |
|-----------|------|--------------|---------|
| `COMPUTE_WH_XS` | X-Small | DATA_ENGINEER, DBT_DEV_ROLE | Development, light transforms |
| `COMPUTE_WH_SMALL` | Small | DATA_ANALYST, DBT_STAGING_ROLE | Reporting, staging dbt runs |
| `COMPUTE_WH_MEDIUM` | Medium | ML_ENGINEER, DBT_PROD_ROLE | Production dbt, ML workloads |
| `COMPUTE_WH_LARGE` | Large | DATA_SCIENTIST | Heavy ML training, feature engineering |

## Schema-Level Least Privilege

The logistics platform applies granular schema-level grants rather than database-wide access. The principle: each role only gets `USAGE` and `CREATE` on the schemas it needs.

### Schema Access Matrix

| Schema | DATA_ENGINEER | DATA_ANALYST | ML_ENGINEER | DATA_SCIENTIST | DATA_STEWARD | SECURITY_ADMIN |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|
| RAW | RW | - | - | - | - | - |
| STAGING | RW | - | - | - | - | - |
| MARTS | RW | R | - | - | - | - |
| ML_FEATURES | RW | - | RW | R | - | - |
| ML_OBJECTS | RW | - | RW | R | - | - |
| ML_SERVING | RW | - | RW | R | - | - |
| SNAPSHOTS | RW | - | - | - | - | - |
| MONITORING | RW | - | - | - | - | - |
| GOVERNANCE | RW | - | - | - | RW | - |
| SECURITY | - | - | - | - | - | RW |
| PERFORMANCE | RW | - | - | - | - | - |

**R** = USAGE + SELECT, **RW** = USAGE + CREATE TABLE + CREATE VIEW

### Future Grants

Future grants ensure new objects automatically inherit the correct permissions:

```sql
-- Analysts automatically get SELECT on new marts tables
GRANT SELECT ON FUTURE TABLES IN SCHEMA $MARTS_SCHEMA TO ROLE DATA_ANALYST;

-- ML roles automatically get SELECT on new feature tables
GRANT SELECT ON FUTURE TABLES IN SCHEMA $ML_FEATURES_SCHEMA TO ROLE ML_ENGINEER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA $ML_FEATURES_SCHEMA TO ROLE DATA_SCIENTIST;

-- Data stewards automatically get SELECT on governance tables
GRANT SELECT ON FUTURE TABLES IN SCHEMA $GOVERNANCE_SCHEMA TO ROLE DATA_STEWARD;

-- Security admin automatically gets SELECT on security views
GRANT SELECT ON FUTURE VIEWS IN SCHEMA $SECURITY_SCHEMA TO ROLE SECURITY_ADMIN;
```

## Security Gaps and Recommendations

Observations from the logistics-analytics-platform implementation:

1. **No row-level security** -- the platform relies entirely on role-based schema access; no RLS predicates are defined for multi-tenant or regional filtering. Consider adding [[Cross-Platform IAM & Data Governance#Row-Level Security Across Platforms|RLS patterns]] if the platform serves multiple business units.

2. **dbt roles are overly broad** -- `DBT_DEV_ROLE`, `DBT_STAGING_ROLE`, and `DBT_PROD_ROLE` all receive `USAGE + CREATE TABLE + CREATE VIEW ON ALL SCHEMAS`. In production, the dbt role should be restricted to only the schemas dbt actually writes to (e.g., MARTS, STAGING) and denied access to SECURITY and GOVERNANCE schemas.

3. **No dynamic data masking** -- the data classification seed pattern (above) identifies PII columns, but the logistics platform does not apply masking policies. Adding `CREATE MASKING POLICY` for PII_HIGH columns would close this gap.

4. **Missing SECURITY_ADMIN to SECURITYADMIN grant** -- the custom SECURITY_ADMIN role is not explicitly granted to the built-in SECURITYADMIN role, which could create orphaned permissions.

5. **No network policy applied** -- the logistics platform scripts do not set a network policy. Apply the OFFICE_POLICY pattern documented above.

## Related Notes

- [[Cross-Platform IAM & Data Governance]] -- GCP IAM, Azure/Fabric RLS, and cross-platform comparison
- [[Snowflake Pipeline Patterns]] -- dbt model configuration and incremental strategies
- [[Data Contracts & Schema Evolution]] -- governance patterns for schema changes
