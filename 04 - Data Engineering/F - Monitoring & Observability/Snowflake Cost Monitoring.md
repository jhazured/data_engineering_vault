# Snowflake Cost Monitoring

## Key Data Sources

All monitoring views source from `SNOWFLAKE.ACCOUNT_USAGE` — note the ingestion lag:

| View | Lag | Purpose |
|------|-----|---------|
| `WAREHOUSE_METERING_HISTORY` | ~3 hours | Credit consumption per warehouse |
| `QUERY_HISTORY` | ~45 minutes | Query execution details |
| `TABLES` | ~3 hours | Table metadata, row counts, sizes |
| `TABLE_STORAGE_METRICS` | ~3 hours | Active/time-travel/failsafe bytes |
| `ACCESS_HISTORY` | ~3 hours | Object-level access tracking |
| `COLUMNS` | ~3 hours | Column metadata across all databases |

**Requires:** `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE SYSADMIN`

## Monthly Cost Forecasting

Project end-of-month spend from month-to-date consumption:

```sql
WITH daily_usage AS (
  SELECT
    DATE_TRUNC('day', START_TIME) AS usage_date,
    SUM(CREDITS_USED) AS daily_credits,
    SUM(CREDITS_USED) * 2.0 AS daily_cost_usd  -- Adjust per contract rate
  FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
  WHERE START_TIME >= DATE_TRUNC('month', CURRENT_DATE())
  GROUP BY 1
)

SELECT
  AVG(daily_credits) * DAY(LAST_DAY(CURRENT_DATE())) AS projected_monthly_credits,
  AVG(daily_cost_usd) * DAY(LAST_DAY(CURRENT_DATE())) AS projected_monthly_cost,
  CASE
    WHEN AVG(daily_cost_usd) * DAY(LAST_DAY(CURRENT_DATE())) > 10000 THEN 'HIGH_COST'
    WHEN AVG(daily_cost_usd) * DAY(LAST_DAY(CURRENT_DATE())) > 5000 THEN 'MEDIUM_COST'
    ELSE 'LOW_COST'
  END AS cost_category
FROM daily_usage
```

## Warehouse Cost Analysis

Daily credit breakdown per warehouse over the last 30 days:

```sql
SELECT
  WAREHOUSE_NAME,
  DATE_TRUNC('day', START_TIME) AS usage_date,
  SUM(CREDITS_USED) AS daily_credits,
  ROUND(SUM(CREDITS_USED) * 2.0, 2) AS estimated_daily_cost_usd,
  COUNT(*) AS session_count,
  AVG(CREDITS_USED) AS avg_credits_per_session
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= CURRENT_DATE() - 30
GROUP BY 1, 2
ORDER BY usage_date DESC, daily_credits DESC
```

## Resource Monitor Usage

Classify warehouses by usage pattern and variance:

```sql
SELECT
  WAREHOUSE_NAME,
  SUM(daily_credits) AS total_credits_30d,
  AVG(daily_credits) AS avg_daily_credits,
  MAX(daily_credits) AS max_daily_credits,
  CASE
    WHEN AVG(daily_credits) > 100 THEN 'HIGH_USAGE'
    WHEN AVG(daily_credits) > 50 THEN 'MEDIUM_USAGE'
    WHEN AVG(daily_credits) > 10 THEN 'LOW_USAGE'
    ELSE 'MINIMAL_USAGE'
  END AS usage_category,
  CASE
    WHEN MAX(daily_credits) / NULLIF(AVG(daily_credits), 0) > 5 THEN 'HIGH_VARIANCE'
    WHEN MAX(daily_credits) / NULLIF(AVG(daily_credits), 0) > 2 THEN 'MEDIUM_VARIANCE'
    ELSE 'LOW_VARIANCE'
  END AS usage_variance
FROM (
  SELECT
    WAREHOUSE_NAME,
    DATE_TRUNC('day', START_TIME) AS usage_date,
    SUM(CREDITS_USED) AS daily_credits
  FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
  WHERE START_TIME >= CURRENT_DATE() - 30
  GROUP BY 1, 2
)
GROUP BY 1
ORDER BY total_credits_30d DESC
```

## Query Cost Analysis

Identify expensive queries by scan volume and execution time:

```sql
SELECT
  DATE_TRUNC('hour', START_TIME) AS query_hour,
  WAREHOUSE_NAME,
  USER_NAME,
  COUNT(*) AS query_count,
  AVG(TOTAL_ELAPSED_TIME) / 1000 AS avg_execution_seconds,
  SUM(BYTES_SCANNED) / POWER(1024, 3) AS total_gb_scanned,
  COUNT(CASE WHEN TOTAL_ELAPSED_TIME > 30000 THEN 1 END) AS long_running_queries,
  COUNT(CASE WHEN BYTES_SCANNED > 1000000000 THEN 1 END) AS high_scan_queries
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= CURRENT_DATE() - 7
  AND EXECUTION_STATUS = 'SUCCESS'
GROUP BY 1, 2, 3
ORDER BY total_gb_scanned DESC
```

## Data Freshness Monitoring

Flag schemas with stale data:

```sql
SELECT
  TABLE_SCHEMA,
  COUNT(*) AS total_tables,
  COUNT(CASE WHEN LAST_ALTERED >= CURRENT_TIMESTAMP() - INTERVAL '24 hours' THEN 1 END) AS fresh_24h,
  COUNT(CASE WHEN LAST_ALTERED >= CURRENT_TIMESTAMP() - INTERVAL '7 days' THEN 1 END) AS fresh_7d,
  ROUND(
    COUNT(CASE WHEN LAST_ALTERED >= CURRENT_TIMESTAMP() - INTERVAL '24 hours' THEN 1 END)
    / NULLIF(COUNT(*), 0) * 100, 2
  ) AS freshness_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE DELETED IS NULL AND TABLE_CATALOG = 'MY_DATABASE'
GROUP BY 1
```

## Data Skew & Storage Efficiency

Identify tables needing optimization:

```sql
SELECT
  TABLE_NAME,
  ROW_COUNT,
  ROUND(BYTES / 1024 / 1024, 2) AS size_mb,
  CASE WHEN BYTES > 0 THEN BYTES / NULLIF(ROW_COUNT, 0) ELSE 0 END AS avg_row_bytes,
  CASE
    WHEN avg_row_bytes > 10000 THEN 'CONSIDER_PARTITIONING_AND_CLUSTERING'
    WHEN size_mb > 100 AND LAST_ALTERED < CURRENT_TIMESTAMP() - INTERVAL '90 days'
      THEN 'CONSIDER_ARCHIVAL'
    ELSE 'NO_ACTION_NEEDED'
  END AS recommendation
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE DELETED IS NULL AND ROW_COUNT > 0
ORDER BY BYTES DESC
```

## Resource Monitor Setup

```sql
CREATE OR REPLACE RESOURCE MONITOR RM_ACCOUNT
  WITH CREDIT_QUOTA = 435          -- Monthly budget
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 90 PERCENT DO NOTIFY        -- Email warning
    ON 100 PERCENT DO NOTIFY       -- Email alert
    ON 110 PERCENT DO SUSPEND;     -- Hard stop

ALTER ACCOUNT SET RESOURCE_MONITOR = RM_ACCOUNT;
```

## Pipeline Run Monitoring

dbt `on-run-end` hooks can log every model execution to an audit table:

| Column | Source |
|--------|--------|
| `pipeline_name` | `result.node.name` |
| `run_status` | `result.status` (success/error/fail) |
| `layer` | Derived from model tags |
| `execution_time_secs` | `result.execution_time` |
| `rows_inserted/updated` | `result.adapter_response` |
| `invocation_id` | Groups all models in one run |

Query for failed models: `SELECT * FROM TBL_PIPELINE_RUN_LOG WHERE run_status IN ('error', 'fail')`

## Email Alerting

Snowflake supports native email alerts via Tasks + Stored Procedures:

```sql
-- Create notification integration
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS email_alerts
  TYPE = EMAIL
  ENABLED = TRUE
  ALLOWED_RECIPIENTS = ('oncall@example.com');

-- Schedule nightly quality check
CREATE TASK nightly_quality_alert
  WAREHOUSE = TRN_PROD_CENTRAL_WH
  SCHEDULE = 'USING CRON 0 6 * * * Australia/Sydney'
AS
  CALL check_data_quality_and_alert();
```
