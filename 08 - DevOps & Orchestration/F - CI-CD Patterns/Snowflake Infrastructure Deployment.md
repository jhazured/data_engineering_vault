# Snowflake Infrastructure Deployment

Patterns for automated Snowflake infrastructure provisioning using SQL scripts orchestrated by bash and Python.

## Deployment Phases

A production Snowflake platform deploys in four phases:

| Phase | Scripts | What It Creates | Role Required |
|-------|---------|-----------------|---------------|
| **1. Environment Setup** | Python venv, pip install, .env validation | Local tooling | N/A |
| **2. Infrastructure** | SQL scripts via Python executor | Warehouses, roles, databases, schemas, users, resource monitors, network policies | ACCOUNTADMIN |
| **3. Data Generation** | Python (Faker + pandas) | Sample CSV data or direct Snowflake load | ENGINEER |
| **4. Data Loading & dbt** | data_loader.py + dbt run | T1 raw tables, T2-T4 dbt models | ENGINEER |

## SQL Script Execution Order

Infrastructure scripts run in numbered order:

```
0.0 Admin Creation.sql          → ACCOUNTADMIN users with MFA
1.0 Account Parameters.sql      → Network policies, session params
2.0 Warehouse Creation.sql      → 10 warehouses (IN/TRN/OUT × DEV/PROD)
3.0 Create Base Roles.sql       → ENGINEER, CHANGE_CONTROL, ANALYST, CONSUMER, SCIENTIST, TASKADMIN
4.0 Database and Schema Creation.sql → T0-T4 databases (DEV/TEST/PROD), LOGISTICS schema
5.0 Create Users.sql            → Service accounts and human users
6.0 Create Resource Monitor.sql → Account-level credit limits
8.0 Create Network Policy.sql   → IP allowlisting
9.0 AD Integration.sql          → Active Directory federation
```

## Parameterised SQL Execution

Scripts use environment variables for multi-environment support. A Python executor handles variable substitution:

### Variable Substitution Patterns

```sql
-- IFNULL pattern: use env var with fallback default
SET SF_ENV = IFNULL($SF_ENV, 'DEV');
SET SF_SCHEMA = IFNULL($SF_SCHEMA, 'LOGISTICS');

-- IDENTIFIER() for dynamic object names
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T2_PERSISTENT_STAGING');
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T2_PERSISTENT_STAGING' || '.' || $SF_SCHEMA);
```

### Python SQL Executor

The executor does more than just run SQL — it:
1. Loads dbt project variables from `dbt_project.yml`
2. Extracts `SET` statements and resolves them from env vars
3. Handles `IDENTIFIER()` for safe dynamic DDL
4. Splits multi-statement files and executes sequentially
5. Logs each statement with timing

```python
# Simplified flow
def execute_sql_file(sql_file):
    sql_content = Path(sql_file).read_text()
    session_vars = extract_session_vars(sql_content)

    # Resolve from env vars
    for var, default in session_vars.items():
        session_vars[var] = os.environ.get(var, default)

    # Substitute and execute
    for statement in split_statements(sql_content):
        statement = substitute_vars(statement, session_vars)
        cursor.execute(statement)
```

## Warehouse Creation Pattern

Production creates warehouses per workload type and environment:

```sql
-- Naming convention: {PURPOSE}_{ENV}_CENTRAL_WH
-- IN = Ingestion, TRN = Transformation, OUT = Consumption

CREATE WAREHOUSE IF NOT EXISTS IN_DEV_CENTRAL_WH
  WITH WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60           -- Dev: aggressive suspend
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Ingestion warehouse for development environment';

CREATE WAREHOUSE IF NOT EXISTS TRN_PROD_CENTRAL_WH
  WITH WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 120          -- Prod: slightly longer for batch jobs
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;
```

## Database & Schema Creation

Tier databases are created per environment with `IF NOT EXISTS` for idempotency:

```sql
-- For each environment (DEV, TEST, PROD):
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T0_CONTROL');
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T1_TRANSIENT_STAGING');
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T2_PERSISTENT_STAGING');
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T3_INTEGRATION');
CREATE DATABASE IF NOT EXISTS IDENTIFIER($SF_ENV || '_T4_PRESENTATION');

-- Create LOGISTICS schema in each
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T1_TRANSIENT_STAGING.LOGISTICS');
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T2_PERSISTENT_STAGING.LOGISTICS');
-- etc.

-- T0 has additional schemas
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T0_CONTROL.AUDIT');
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T0_CONTROL.SECURITY');
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($SF_ENV || '_T0_CONTROL.JOB_CONTROL');
```

## Validation Scripts

After infrastructure deployment, validate that everything was created:

```python
# validate_schemas.py
tier_databases = [
    f"{env}_T1_TRANSIENT_STAGING",
    f"{env}_T2_PERSISTENT_STAGING",
    f"{env}_T3_INTEGRATION",
    f"{env}_T4_PRESENTATION",
]

for db in tier_databases:
    cursor.execute(f"SHOW SCHEMAS IN DATABASE {db}")
    schemas = [row[1] for row in cursor.fetchall()]
    if 'LOGISTICS' not in schemas:
        print(f"MISSING: {db}.LOGISTICS")
```

```bash
# Validate raw tables exist before dbt run
validate_raw_tables_exist() {
    local required_tables=("CUSTOMERS" "VEHICLES" "SHIPMENTS" "MAINTENANCE" "TELEMATICS" "TRAFFIC" "WEATHER")
    # Query INFORMATION_SCHEMA.TABLES and compare
}
```

## Teardown

Controlled teardown for dev/test environments:

```bash
#!/bin/bash
# teardown.sh — destroys all platform objects for an environment

DATABASES=(
    "${SF_ENV}_T0_CONTROL"
    "${SF_ENV}_T1_TRANSIENT_STAGING"
    "${SF_ENV}_T2_PERSISTENT_STAGING"
    "${SF_ENV}_T3_INTEGRATION"
    "${SF_ENV}_T4_PRESENTATION"
)

for db in "${DATABASES[@]}"; do
    echo "Dropping database: $db"
    python3 execute_sql_python.py <(echo "DROP DATABASE IF EXISTS IDENTIFIER('$db');")
done

# Reset deployment status
rm -f "$PROJECT_ROOT/.deployment_status"
```

## GitLab Integration Setup

Automated setup for Snowflake Native Workspace (dbt-in-Snowflake):

```bash
#!/bin/bash
# Creates: SECRET, API_INTEGRATION, GIT_REPOSITORY

# Validate credentials
for var in SF_ACCOUNT SF_USER SF_PASSWORD GITLAB_USERNAME GITLAB_PAT; do
    if [[ -z "${!var}" ]]; then
        echo "ERROR: $var is not set"
        exit 1
    fi
done

# Substitute placeholders in SQL template
sed \
    -e "s/GITLAB_USERNAME_PLACEHOLDER/$GITLAB_USERNAME/g" \
    -e "s|GITLAB_PAT_PLACEHOLDER|$GITLAB_PAT|g" \
    "$SQL_TEMPLATE" > "$TEMP_SQL"

trap 'rm -f "$TEMP_SQL"' EXIT

SF_ROLE=ACCOUNTADMIN python3 execute_sql_python.py "$TEMP_SQL"
```

## Email Alerting Setup

Automated monitoring configuration:

```sql
-- Create notification integration
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS email_int
  TYPE = EMAIL ENABLED = TRUE
  ALLOWED_RECIPIENTS = ('oncall@example.com');

-- Create stored procedure for quality checks
CREATE OR REPLACE PROCEDURE check_data_freshness()
RETURNS VARCHAR
LANGUAGE SQL
AS $$
  -- Check for stale tables and send alert
$$;

-- Schedule as a Snowflake Task
CREATE OR REPLACE TASK nightly_freshness_check
  WAREHOUSE = TRN_PROD_CENTRAL_WH
  SCHEDULE = 'USING CRON 0 6 * * * Australia/Sydney'
AS CALL check_data_freshness();

ALTER TASK nightly_freshness_check RESUME;
```
