# Snowflake & dbt Troubleshooting

Quick-reference troubleshooting guide for common issues in a Snowflake + dbt platform.

## Privilege Errors

### `Insufficient privileges to operate on schema`

**Cause:** Snowflake Workspace runs as `DBT_RUNTIME_ROLE`, not the role in `profiles.yml`. This role doesn't inherit `ENGINEER` unless explicitly granted.

```sql
USE ROLE ACCOUNTADMIN;
GRANT ROLE ENGINEER TO ROLE DBT_RUNTIME_ROLE;
-- Verify:
SHOW GRANTS TO ROLE DBT_RUNTIME_ROLE;
```

**If running locally:** Check `SF_ROLE` in `.env` is set to `ENGINEER`.

### FUTURE SCHEMA grants not applying

New schemas created after `GRANT ALL ON ALL SCHEMAS` was run are not accessible.

```sql
USE ROLE SYSADMIN;
-- Apply to future objects
GRANT ALL ON FUTURE SCHEMAS IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;
GRANT ALL ON FUTURE TABLES IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;
GRANT ALL ON FUTURE VIEWS IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;

-- Re-apply to catch already-created schemas
GRANT ALL ON ALL SCHEMAS IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;
GRANT ALL ON ALL TABLES IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;
GRANT ALL ON ALL VIEWS IN DATABASE DEV_T3_INTEGRATION TO ROLE ENGINEER;
```

Repeat for T4_PRESENTATION and any TEST/PROD equivalents.

## Environment & Connection Issues

### `SF_ACCOUNT not set` or variables not found

**Cause:** Plain `source .env` does not export variables to subprocesses.

```bash
# Correct way — exports all variables
set -a && source .env && set +a

# Or use the wrapper which handles this
./dbt_cmd.sh run --select tag:load_priority_1
```

### Connection fails — hostname error

**Cause:** `SF_ACCOUNT` contains `.snowflakecomputing.com` suffix or is uppercase.

```bash
SF_ACCOUNT=xxxxxxx-xxxxxxx           # correct — lowercase, no suffix
SF_ACCOUNT=XXXXXXX.snowflakecomputing.com  # wrong
```

### dbt models writing to wrong database

**Cause:** `DBT_TARGET` and `SF_ENV` are out of sync.

```bash
# Diagnosis
./dbt_cmd.sh debug

# Fix — ensure both match in .env
SF_ENV=DEV
DBT_TARGET=dev
```

### dbt connects to wrong warehouse

**Cause:** `SF_WAREHOUSE` is set to an `IN_` (ingestion) warehouse instead of `TRN_` (transformation).

```bash
SF_WAREHOUSE=TRN_DEV_CENTRAL_WH    # correct for dbt
SF_WAREHOUSE=IN_DEV_CENTRAL_WH     # wrong — reserved for Fivetran
```

## dbt Command Issues

### `dbt docs generate` fails via dbt_cmd.sh

**Cause:** dbt-fusion (used by `dbt_cmd.sh`) doesn't support `docs generate`.

```bash
# Use dbt-core directly
source .venv/bin/activate
set -a && source .env && set +a
dbt docs generate --project-dir dbt --profiles-dir dbt
dbt docs serve --project-dir dbt --profiles-dir dbt --port 8080
```

### `env_var() is not defined` in Snowflake Workspace

**Cause:** The Workspace has no shell — `env_var()` calls in `profiles.yml` fail at compile time.

**Fix:** Replace with the Workspace profile that uses `authenticator: oauth` and hardcoded values instead of `env_var()`.

### `DATA_CLASSIFICATION_TAGS` table not found

**Cause:** The seed hasn't been loaded yet.

```bash
./dbt_cmd.sh seed --select data_classification_tags
./dbt_cmd.sh run --select tag:t0
```

### `dbt deps` fails — version conflict

```bash
./dbt_cmd.sh deps --debug
# Ensure dbt-snowflake~=1.8 is installed
pip install -r requirements.txt
```

## Deployment Issues

### Deploy script skips phases that need to re-run

**Cause:** `.deployment_status` file tracks completed phases.

```bash
# Reset and re-run all phases
./scripts/02_deployment/handlers/deploy_all.sh --reset

# Or delete manually
rm .deployment_status
```

### Infrastructure setup fails with permission error

**Cause:** `SF_ROLE` is not `ACCOUNTADMIN` during Phase 1-2 setup.

```bash
# Set in .env before infrastructure setup
SF_ROLE=ACCOUNTADMIN

# Switch back after setup completes
SF_ROLE=ENGINEER
```

### Warehouse GRANT fails during setup

**Cause:** `SECURITYADMIN` cannot grant on `SYSADMIN`-owned warehouses — only `ACCOUNTADMIN` can.

**Fix:** Ensure `SF_ROLE=ACCOUNTADMIN` when running Phase 2.

## Snapshot Issues

### Timestamp type mismatch warning

```
Data type of snapshot table timestamp columns (TIMESTAMP_NTZ)
doesn't match derived column 'updated_at' (TEXT)
```

**Impact:** Non-blocking — snapshots complete. Safe to ignore.

**Fix (if needed):** Cast in the source staging model:
```sql
TRY_TO_TIMESTAMP_NTZ(updated_at) AS updated_at
```

## GitLab / Workspace Connectivity

### `Secret does not exist or not authorized`

**Cause:** GitLab PAT has expired.

```sql
USE ROLE ACCOUNTADMIN;
ALTER SECRET DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS
    SET USERNAME = '<gitlab-username>'
        PASSWORD = '<new-glpat-token>';
```

### `Git repository does not exist`

```sql
-- Check where it actually is
SHOW GIT REPOSITORIES IN ACCOUNT;
-- Should be: DEV_T0_CONTROL.SECURITY.APAC_DBT_REPO
```

## Performance & Cost

### Diagnosing slow queries or unexpected cost

```sql
-- Top queries by elapsed time (last 24h)
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_QUERY_PERFORMANCE_SUMMARY
ORDER BY total_elapsed_time_seconds DESC LIMIT 20;

-- Warehouse credit consumption (last 7 days)
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_WAREHOUSE_COST_ANALYSIS
ORDER BY cost_date DESC;

-- Monthly cost forecast
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_MONTHLY_COST_FORECAST;
```

**Note:** T0 views query `SNOWFLAKE.ACCOUNT_USAGE` with 45-min to 3-hour lag.

### Warehouse not starting / queries queuing

```sql
SHOW WAREHOUSES;
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_WAREHOUSE_STATUS;
```

Common causes: auto-suspended (60/120s), resource monitor quota hit, wrong warehouse for workload.

### Large table scans on fact tables

```sql
ALTER TABLE DEV_T3_INTEGRATION.LOGISTICS.TBL_FACT_SHIPMENTS
    CLUSTER BY (shipment_date, customer_id);

-- Check clustering status
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_DATA_SKEW_ANALYSIS
WHERE table_name = 'TBL_FACT_SHIPMENTS';
```

## Access & Security

### Permission denied on T4 views for ANALYST/CONSUMER

```sql
USE ROLE ACCOUNTADMIN;
GRANT USAGE ON WAREHOUSE OUT_DEV_CENTRAL_WH TO ROLE ANALYST;
GRANT USAGE ON DATABASE DEV_T4_PRESENTATION TO ROLE ANALYST;
GRANT USAGE ON ALL SCHEMAS IN DATABASE DEV_T4_PRESENTATION TO ROLE ANALYST;
GRANT SELECT ON ALL VIEWS IN DATABASE DEV_T4_PRESENTATION TO ROLE ANALYST;
GRANT SELECT ON FUTURE VIEWS IN DATABASE DEV_T4_PRESENTATION TO ROLE ANALYST;
```

### Data classification not showing for a column

```bash
# Refresh the seed and rebuild security views
./dbt_cmd.sh seed --select data_classification_tags --full-refresh
./dbt_cmd.sh run --select tag:t0,tag:security
```

```sql
-- Check coverage
SELECT * FROM DEV_T0_CONTROL.SECURITY.VW_DATA_CLASSIFICATION
WHERE table_name = 'TBL_FACT_SHIPMENTS'
ORDER BY column_name;
```
