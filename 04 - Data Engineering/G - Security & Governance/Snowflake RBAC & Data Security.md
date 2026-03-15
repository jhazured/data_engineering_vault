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
