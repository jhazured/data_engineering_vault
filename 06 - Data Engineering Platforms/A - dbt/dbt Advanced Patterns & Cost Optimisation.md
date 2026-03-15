# dbt Advanced Patterns & Cost Optimisation

Advanced dbt patterns from the logistics-analytics-platform, covering incremental cost reduction, macro architecture, ML feature engineering, snapshots, and environment config. Builds on [[Core dbt Fundamentals]], [[dbt Incremental Loading Patterns]], and [[dbt Macro Patterns]].

---

## 1. Incremental Loading for Cost Reduction

**70-90% Fivetran savings**, **50-70% Snowflake compute reduction** via merge-based incremental loading.

### Merge Strategy with Update Column Control

```sql
{{ config(
    materialized='incremental', unique_key='shipment_id',
    incremental_strategy='merge',
    merge_update_columns=['delivery_status', 'actual_delivery_date', 'updated_at'],
    on_schema_change='sync_all_columns',
    tags=['staging', 'shipments', 'incremental']
) }}

WITH src AS (
  SELECT * FROM {{ source('RAW_LOGISTICS', 'SHIPMENTS') }}
  {% if is_incremental() %}
    WHERE "_LOADED_AT" > (SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }})
  {% endif %}
)
SELECT TRIM("SHIPMENT_ID") AS shipment_id, ..., CURRENT_TIMESTAMP() AS _ingested_at
FROM src
```

### Data Retention Windows

| Source | Retention | Frequency |
|--------|-----------|-----------|
| Customers, Shipments, Vehicles | All history | Daily |
| Weather, Traffic | 30-day rolling | Daily |
| Telematics (IoT) | 7-day rolling | Hourly |

### Reusable Filter & Late-Arriving Data

```sql
{% macro get_incremental_filter(timestamp_column) %}
  {% if is_incremental() %}
    WHERE {{ timestamp_column }} > (SELECT COALESCE(MAX({{ timestamp_column }}), '1900-01-01') FROM {{ this }})
  {% endif %}
{% endmacro %}

-- Late-arriving data: reprocess trailing window alongside watermark
{% macro logistics_incremental_strategy() %}
  {% if is_incremental() %}
    WHERE updated_at > (SELECT COALESCE(MAX(updated_at), '1900-01-01'::timestamp) FROM {{ this }})
       OR shipment_date >= CURRENT_DATE - 7
  {% endif %}
{% endmacro %}
```

---

## 2. Advanced Macro Patterns

### Rolling Window Aggregations

```sql
{% macro rolling_metrics(column, partition_by, order_by, windows=[7, 30, 90]) %}
  {% for window in windows %}
    AVG({{ column }}) OVER (PARTITION BY {{ partition_by }} ORDER BY {{ order_by }}
      ROWS BETWEEN {{ window - 1 }} PRECEDING AND CURRENT ROW
    ) AS {{ column }}_{{ window }}d_avg,
    SUM({{ column }}) OVER (...) AS {{ column }}_{{ window }}d_sum
    {%- if not loop.last -%},{%- endif %}
  {% endfor %}
{% endmacro %}
-- Usage: {{ rolling_metrics('revenue', 'customer_id', 'shipment_date') }}
```

### Business Logic Macros

```sql
{% macro classify_delivery_performance(on_time_rate) %}
  CASE WHEN {{ on_time_rate }} >= 0.95 THEN 'excellent'
       WHEN {{ on_time_rate }} >= 0.90 THEN 'good'
       WHEN {{ on_time_rate }} >= 0.80 THEN 'acceptable'
       ELSE 'needs_improvement' END
{% endmacro %}

{% macro calculate_performance_score(on_time_rate, efficiency_rate, satisfaction_score) %}
  ROUND(({{ on_time_rate }} * 0.4 + {{ efficiency_rate }} * 0.3 +
         {{ satisfaction_score }} / 10 * 0.3) * 10, 2)
{% endmacro %}
```

### Error Handling & Data Quality Validation

```sql
{% macro validate_data_quality(table_name, column_name, validation_rule) %}
  {% if execute %}
    {% set query %}SELECT COUNT(*) FROM {{ table_name }} WHERE NOT ({{ validation_rule }}){% endset %}
    {% set results = run_query(query) %}
    {% if results[0][0] > 0 %}
      {% do log("DQ issue: " ~ table_name ~ "." ~ column_name ~ ": " ~ results[0][0] ~ " failures", info=true) %}
    {% endif %}
  {% endif %}
{% endmacro %}
```

### Post-Hooks (Production Only)

```sql
{% macro optimize_clustering(table) %}
  {% if target.name == 'prod' %}
    {% do run_query("ALTER TABLE " ~ table ~ " RECLUSTER") %}
  {% endif %}
{% endmacro %}
```

Other post-hooks: `update_feature_store_metadata` (updates row count + timestamp in metadata table), `log_table_stats` (logs row counts per model run).

### Date/Time Utilities

`rolling_window_days(days)` generates `ROWS BETWEEN N-1 PRECEDING AND CURRENT ROW`. Also: `days_between`, `hours_between`, `is_weekend`, `start_of_month`, `end_of_month`.

---

## 3. Tag Strategy & DAG Organisation

### Load Priority Tags

```
Staging (1) --> Dimensions (2a) --> Facts (2b) --> ML Features (2c) --> Consumption (3)
```

| Priority | Tag | Materialisation |
|----------|-----|----------------|
| 1 | `load_priority_1` | incremental |
| 2a | `load_priority_2a` | incremental |
| 2b | `load_priority_2b` | incremental |
| 2c | `load_priority_2c` | table |
| 3 | `load_priority_3` | view |

**Business context tags:** `core_business`, `business_intelligence`, `predictive`, `operational`
**Technical tags:** `scd_type_2`, `ml_optimized`, `real_time`

### Selective Execution

```bash
dbt run --select tag:load_priority_1      # Staging only
dbt run --select tag:ml_features          # All ML models
dbt run --select tag:core_business        # Critical business models
dbt run --select tag:incremental          # All incremental models
```

---

## 4. ML Feature Engineering in dbt

### Consolidated Feature Store

`tbl_ml_consolidated_feature_store` unifies shipment and telemetry features with entity-type tagging (`entity_id`, `entity_type`, `feature_date`). Derives `on_time_flag`, `profit_margin`, `cost_per_km`, `haul_type`, `delivery_performance` from fact/dimension joins. Filtered to last 90 days.

### Customer Segmentation (RFM + Churn)

```sql
-- RFM scoring in tbl_ml_customer_behavior_segments
CASE WHEN days_since_last_shipment <= 30 THEN 5 WHEN ... END AS recency_score,
NTILE(5) OVER (ORDER BY avg_monthly_shipments) AS frequency_score,
NTILE(5) OVER (ORDER BY total_revenue) AS monetary_score,
-- Churn prediction features
days_since_last_shipment AS churn_risk_days,
(avg_monthly_shipments * 30 - days_since_last_shipment) / 30.0 AS expected_vs_actual_frequency
```

### Predictive Maintenance Features

Clustered by `[vehicle_id, feature_date]`, combines rolling 7d/30d/90d telemetry (engine health, harsh braking, maintenance alerts) with maintenance history to produce a predictive maintenance score (0-100) and engine health trend (degrading/improving/stable).

### ML Feature Scaling Macro

```sql
{% macro scale_feature(column_name, method='standardize') %}
  {% if method == 'standardize' %}
    ({{ column_name }} - AVG({{ column_name }}) OVER ()) / NULLIF(STDDEV({{ column_name }}) OVER (), 0)
  {% elif method == 'normalize' %}
    ({{ column_name }} - MIN({{ column_name }}) OVER ()) / NULLIF(MAX({{ column_name }}) OVER () - MIN({{ column_name }}) OVER (), 0)
  {% endif %}
{% endmacro %}
```

### Model Registry

Tracks ML model lifecycle as a dbt table with `PARSE_JSON` for performance metrics, feature columns, and hyperparameters. Status tracking: TRAINING / DEPLOYED / RETIRED.

---

## 5. Consumption Layer Design

All consumption models use `materialized='view'` for zero storage cost and always-fresh reads.

- **`vw_consolidated_dashboard`** -- Daily KPIs with 7d/30d/90d rolling context, WoW/YoY comparisons, trend classification (increasing/stable/decreasing), alert indicators (performance/satisfaction/revenue alerts)
- **`vw_ml_real_time_customer_features`** -- Latest features per customer via `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY feature_date DESC)` for ML inference serving
- **`vw_ai_recommendations`** -- Route, vehicle, maintenance, and customer retention recommendations with priority and estimated cost impact
- **`vw_sustainability_metrics`** -- ESG: CO2 per km/delivery, fleet electrification maturity, YoY carbon reduction

---

## 6. Snapshot Patterns ([[SCD Type 2 Patterns]])

### Timestamp Strategy (6 snapshots)

```sql
{% snapshot customers_snapshot %}
{{ config(target_schema='snapshots', unique_key='customer_id',
          strategy='timestamp', updated_at='updated_at',
          tags=['snapshots', 'scd_type_2', 'customer']) }}
SELECT * FROM {{ ref('tbl_stg_customers') }}
{% endsnapshot %}
```

Used for: customers, vehicles, locations, traffic, weather, maintenance.

### Check Strategy (ML-derived tables)

```sql
{% snapshot customer_behavior_segments_snapshot %}
{{ config(target_schema='snapshots', unique_key='CUSTOMER_ID',
          strategy='check',
          check_cols=['TOTAL_SHIPMENTS', 'TOTAL_REVENUE', 'ON_TIME_RATE', 'AVG_ORDER_VALUE']) }}
SELECT * FROM {{ ref('tbl_ml_customer_behavior_segments') }}
{% endsnapshot %}
```

### Point-in-Time SCD Joins

Facts join snapshots using `dbt_valid_from`/`dbt_valid_to` for as-of-date dimension values:

```sql
LEFT JOIN {{ ref('customers_snapshot') }} c
  ON s.customer_id = c.customer_id
  AND s.shipment_date >= TO_TIMESTAMP_NTZ(c.dbt_valid_from)
  AND (s.shipment_date < TO_TIMESTAMP_NTZ(c.dbt_valid_to) OR c.dbt_valid_to IS NULL)

-- Reusable macro version
{% macro join_scd_snapshot(snapshot_ref, alias, business_key, date_field) %}
  LEFT JOIN {{ ref(snapshot_ref) }} {{ alias }}
    ON {{ business_key }} = {{ alias }}.{{ business_key.split('.')[-1] }}
    AND {{ date_field }} >= {{ alias }}.dbt_valid_from
    AND ({{ date_field }} < {{ alias }}.dbt_valid_to OR {{ alias }}.dbt_valid_to IS NULL)
{% endmacro %}
```

---

## 7. Custom Tests

### Business Rule Tests

Return rows violating the rule (test passes when zero rows returned):

```sql
-- test_route_efficiency_bounds.sql
SELECT shipment_id FROM {{ ref('tbl_fact_shipments') }}
WHERE route_efficiency_score NOT BETWEEN 0 AND 100
   OR (ABS(actual_duration_minutes - planned_duration_minutes) < 5 AND route_efficiency_score < 95)
```

### Referential Integrity Tests

`UNION ALL` pattern: LEFT JOIN each FK to its dimension, return rows where the dimension key IS NULL. Covers customer, route, vehicle, and date relationships in `test_fact_dimension_relationships.sql`.

### Test Severity & Source Tests

Default severity `warn` in `dbt_project.yml`; source-level tests use `dbt_expectations` for range/set validation (`expect_column_values_to_be_between`, `expect_column_values_to_be_in_set`).

---

## 8. Environment-Specific Configuration

```yaml
vars:
  dev:
    max_partition_days: 7
    materialized: 'view'            # Views in dev to minimise storage
    enable_real_time_features: false
  staging:
    max_partition_days: 90
    enable_real_time_features: true
  prod:
    max_partition_days: 365
    enable_real_time_features: true
```

### Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `dbt-labs/dbt_utils` | 1.3.1 | Surrogate keys, pivots, `generate_surrogate_key` |
| `metaplane/dbt_expectations` | 0.10.9 | Great Expectations-style source tests |
| `dbt-labs/codegen` | 0.13.1 | Model/source YAML generation |
| `dbt-labs/audit_helper` | 0.12.1 | `compare_relations` for validation |
| `godatadriven/dbt_date` | 0.10.1 | Date spine generation |

### Dispatch Override

```yaml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ["smart_logistics", "dbt_utils"]
```

---

## 9. Fivetran Integration

### Source Freshness

Sources declare `freshness: warn_after: 12h, error_after: 24h` with `loaded_at_field: _LOADED_AT`. Each table column uses `dbt_expectations` tests for uniqueness, not-null, range, and set membership validation.

### Incremental Sync Chain

`_LOADED_AT` bridges Fivetran and dbt incremental logic:

```
Fivetran sync (_LOADED_AT) --> Staging watermark (_ingested_at) --> Mart watermark (_last_updated)
```

Run `dbt source freshness` to verify Fivetran is delivering within SLA windows. Combined with merge strategy, only new/changed rows flow through the pipeline.
