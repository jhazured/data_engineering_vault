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

## Warehouse Auto-Suspend and Resume Patterns

All warehouses should be created with `INITIALLY_SUSPENDED = TRUE` so they consume zero credits until first use. The `AUTO_RESUME = TRUE` setting ensures transparent start-up when a query arrives.

**Suspend timing by workload type:**

| Warehouse Purpose | Prefix | `AUTO_SUSPEND` | Rationale |
|-------------------|--------|----------------|-----------|
| Ingestion (source to Snowflake) | `IN_` | 60s | Short ETL bursts; no idle benefit |
| Transformation (dbt runs) | `TRN_` | 60s | Batch workloads with defined start/end |
| Analytics / BI consumption | `OUT_` | 120s | Longer suspend to avoid repeated cold-starts from interactive queries |
| Administration / ad-hoc | `WORK_` | 60s | Infrequent, low-volume tasks |

```sql
CREATE OR REPLACE WAREHOUSE "TRN_PROD_CENTRAL_WH"
    WITH WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 5
    SCALING_POLICY = 'STANDARD'
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Transforming data in PROD';
```

The naming convention `{PURPOSE}_{ENV}_{SCOPE}_WH` makes it straightforward to attribute costs per environment and function in [[Snowflake Cost Monitoring#Warehouse Cost Analysis|warehouse cost queries]].

## Resource Monitor Configuration

Resource monitors operate at the account level and reset on a monthly cadence. The `RM_ACCOUNT` monitor uses a three-tier trigger strategy:

| Threshold | Action | Purpose |
|-----------|--------|---------|
| 90% | `NOTIFY` | Early warning to ACCOUNTADMIN users |
| 100% | `NOTIFY` | Budget reached; investigate immediately |
| 110% | `NOTIFY` | Overspend alert (alternatively use `SUSPEND` for a hard stop) |

The production setup uses `NOTIFY` at all three thresholds rather than `SUSPEND` to avoid breaking production pipelines. For non-production accounts, consider `ON 110 PERCENT DO SUSPEND` as a hard guardrail.

**Prerequisites:** ACCOUNTADMIN users must have email notifications enabled in their Snowflake profile preferences, otherwise trigger alerts are silently dropped.

**Credit quota** is set to the monthly budget in credits (e.g., 435). To convert to dollars, multiply by the contract credit price.

## Cost Per Credit Tracking With dbt

The dbt project uses `var("snowflake_credit_price")` (defaulting to `2.0`) to convert raw credits into estimated USD cost. This keeps the credit price configurable across environments without hardcoding:

```sql
-- In vw_warehouse_cost_analysis.sql
ROUND(SUM(CREDITS_USED) * {{ var("snowflake_credit_price") }}, 2)
    AS ESTIMATED_DAILY_COST_USD

-- In vw_resource_monitor_usage.sql
ROUND(TOTAL_CREDITS_30D * {{ var('snowflake_credit_price', 2.0) }}, 2)
    AS ESTIMATED_COST_30D
```

Set the variable in `dbt_project.yml`:

```yaml
vars:
  snowflake_credit_price: 2.0  # Update when contract rate changes
```

This pattern centralises the credit price so that all cost views — warehouse analysis, monthly forecast, resource monitor — stay consistent when the contract rate is renegotiated.

## Daily Credit Tracking and Budget Utilisation

The `vw_monthly_cost_forecast` model extends the basic projection (see [[Snowflake Cost Monitoring#Monthly Cost Forecasting|Monthly Cost Forecasting]]) with budget utilisation tracking:

```sql
-- Derived columns from the monthly projection CTE
MONTH_TO_DATE_COST / DAYS_ELAPSED          AS COST_PER_DAY,
MONTH_TO_DATE_COST / MONTH_TO_DATE_CREDITS AS COST_PER_CREDIT,

-- Percentage of projected monthly spend already consumed
(MONTH_TO_DATE_COST / PROJECTED_MONTHLY_COST) * 100
    AS BUDGET_UTILIZATION_PCT
```

This gives a single-row daily dashboard showing month-to-date spend, average daily burn rate, projected end-of-month cost, and remaining budget headroom.

## Cost Categorisation Thresholds

Projected monthly cost is classified into tiers:

| Category | Projected Monthly Cost | Suggested Action |
|----------|----------------------|------------------|
| `HIGH_COST` | > $10,000 | Immediate review; consider warehouse downsizing or query optimisation |
| `MEDIUM_COST` | > $5,000 | Monitor weekly; check for unused warehouses |
| `LOW_COST` | <= $5,000 | Within normal operating range |

These thresholds should be adjusted to reflect the organisation's actual budget. They are implemented as `CASE` expressions in the forecast model so they appear inline with every daily refresh.

## Budget Status Flags

The forecast model also derives a budget status flag based on how much of the projected monthly spend has already been consumed:

| Flag | Condition | Interpretation |
|------|-----------|----------------|
| `OVER_BUDGET_RISK` | Utilisation > 80% of projected spend | On track to exceed budget; escalate |
| `BUDGET_WARNING` | Utilisation > 60% of projected spend | Trending high; review top consumers |
| `WITHIN_BUDGET` | Utilisation <= 60% | Normal trajectory |

These flags can be consumed by downstream alerting (e.g., a [[Snowflake Cost Monitoring#Email Alerting|Snowflake email task]]) or surfaced in a BI dashboard.

## Query Cost Analysis by Dimension

The `vw_query_cost_analysis` model groups query activity across multiple dimensions — user, warehouse, database, schema, and query type — over a rolling 7-day window:

| Metric | What It Reveals |
|--------|-----------------|
| `QUERY_COUNT` | Volume per user/warehouse combination |
| `AVG_EXECUTION_SECONDS` | Typical query duration |
| `TOTAL_GB_SCANNED` | Data volume driving compute cost |
| `LONG_RUNNING_QUERIES` | Count where elapsed time > 30 seconds |
| `HIGH_SCAN_QUERIES` | Count where bytes scanned > 1 GB |
| `FAILED_QUERIES` | Count where `ERROR_CODE IS NOT NULL` |

Sorting by `TOTAL_GB_SCANNED DESC` surfaces the most expensive user/warehouse pairings first. Pair this with the [[Snowflake Cost Monitoring#Warehouse Cost Analysis|warehouse cost view]] to correlate scan volume with credit consumption.

## Warehouse Usage Variance Detection

The `vw_resource_monitor_usage` model computes a variance ratio (`MAX_DAILY_CREDITS / AVG_DAILY_CREDITS`) to flag unpredictable consumption patterns:

| Variance Category | Ratio | Typical Cause |
|-------------------|-------|---------------|
| `HIGH_VARIANCE` | > 5x | Ad-hoc large queries, one-off backfills |
| `MEDIUM_VARIANCE` | > 2x | Periodic batch spikes (e.g., weekly full refreshes) |
| `LOW_VARIANCE` | <= 2x | Steady, predictable workloads |

High-variance warehouses are prime candidates for:
- Moving ad-hoc workloads to a dedicated sandbox warehouse
- Setting warehouse-level resource monitors
- Reviewing query history for the spike days

The model also tracks `ACTIVE_DAYS` out of 30 and `AVG_SESSIONS_PER_DAY` to distinguish genuinely busy warehouses from those with sporadic but heavy usage.

## Table Storage Analysis

The `vw_data_skew_analysis` model joins `ACCOUNT_USAGE.TABLES` with `TABLE_STORAGE_METRICS` to surface storage inefficiencies:

### Storage Ratios

| Ratio | Formula | Red Flag Threshold |
|-------|---------|-------------------|
| Time travel ratio | `TIME_TRAVEL_BYTES / ACTIVE_BYTES` | > 2.0 — time travel storage exceeds twice the active data |
| Failsafe ratio | `FAILSAFE_BYTES / ACTIVE_BYTES` | > 1.0 — failsafe overhead is disproportionate |

### Optimisation Recommendations

The model generates actionable recommendations based on combined signals:

| Condition | Recommendation |
|-----------|---------------|
| Large table (>100 MB) with large rows (>1 KB avg) | `CONSIDER_PARTITIONING_AND_CLUSTERING` |
| Time travel ratio > 2.0 | `REDUCE_TIME_TRAVEL_RETENTION` |
| Large table, not altered in 90+ days | `CONSIDER_ARCHIVAL_OR_PURGING` |
| Average row size > 10 KB | `OPTIMIZE_ROW_STRUCTURE` |
| Large table, recently active | `CONSIDER_CLUSTERING` |

To reduce time travel costs on staging tables, set shorter retention:

```sql
ALTER TABLE my_staging_table SET DATA_RETENTION_TIME_IN_DAYS = 1;
```

Transient staging databases (`T1_TRANSIENT_STAGING`) already use `DATA_RETENTION_TIME_IN_DAYS = 5`, whilst persistent tiers use 10 (dev/test) or 45 (prod).

## ACCOUNT_USAGE Lag Awareness

When building monitoring dashboards or alerts, account for the ingestion delay in `SNOWFLAKE.ACCOUNT_USAGE` views:

| Lag | Affected Views | Implication |
|-----|---------------|-------------|
| ~45 minutes | `QUERY_HISTORY` | Near-real-time query monitoring is not possible via ACCOUNT_USAGE |
| ~3 hours | `WAREHOUSE_METERING_HISTORY`, `TABLES`, `TABLE_STORAGE_METRICS` | Cost and storage metrics reflect state from several hours ago |

For real-time query monitoring, use `INFORMATION_SCHEMA.QUERY_HISTORY()` instead — it returns results with no lag but is scoped to the current session or recent history only. The ACCOUNT_USAGE views are better suited for trend analysis, daily/weekly reporting, and cost dashboards where a few hours of delay is acceptable.
