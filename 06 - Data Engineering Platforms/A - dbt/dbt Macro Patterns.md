# dbt Macro Patterns

## Rolling Window Aggregations

Reusable macros for rolling metrics across multiple time windows:

```sql
{% macro rolling_window_days(days) %}
  ROWS BETWEEN {{ days - 1 }} PRECEDING AND CURRENT ROW
{% endmacro %}

{% macro rolling_metrics(column, partition_by, order_by, windows=[7, 30, 90]) %}
  {% for window in windows %}
    AVG({{ column }}) OVER (
      PARTITION BY {{ partition_by }}
      ORDER BY {{ order_by }}
      {{ rolling_window_days(window) }}
    ) AS {{ column }}_{{ window }}d_avg,
    SUM({{ column }}) OVER (
      PARTITION BY {{ partition_by }}
      ORDER BY {{ order_by }}
      {{ rolling_window_days(window) }}
    ) AS {{ column }}_{{ window }}d_sum
    {%- if not loop.last -%},{%- endif %}
  {% endfor %}
{% endmacro %}
```

**Usage in a model:**
```sql
SELECT
  date_key,
  customer_id,
  revenue,
  {{ rolling_metrics('revenue', 'customer_id', 'date_key') }}
FROM {{ ref('tbl_fact_shipments') }}
```

Generates: `revenue_7d_avg`, `revenue_7d_sum`, `revenue_30d_avg`, `revenue_30d_sum`, `revenue_90d_avg`, `revenue_90d_sum`.

## Safe Division

Prevent divide-by-zero errors across the project:

```sql
{% macro safe_divide(numerator, denominator, default_value=0) %}
  CASE
    WHEN CAST({{ denominator }} AS FLOAT) = 0 OR {{ denominator }} IS NULL
    THEN {{ default_value }}
    ELSE CAST({{ numerator }} AS FLOAT) / CAST({{ denominator }} AS FLOAT)
  END
{% endmacro %}
```

**Usage:** `{{ safe_divide('revenue - total_cost', 'revenue', 0) }} AS profit_margin`

## Unit Conversion Macros

Centralise conversions to avoid magic numbers scattered through models:

```sql
{% macro safe_decimal(column, precision=10, scale=2) %}
  TRY_TO_DECIMAL({{ column }}::VARCHAR, {{ precision }}, {{ scale }})
{% endmacro %}

{% macro lbs_to_kg(lbs_column) %}
  {{ safe_decimal(lbs_column) }} / 2.20462
{% endmacro %}

{% macro miles_to_km(miles_column) %}
  {{ safe_decimal(miles_column) }} * 1.60934
{% endmacro %}

{% macro fahrenheit_to_celsius(f_column) %}
  ({{ safe_decimal(f_column) }} - 32) * 5/9
{% endmacro %}

{% macro mpg_to_l_per_100km(mpg_column) %}
  235.214 / NULLIF({{ safe_decimal(mpg_column) }}, 0)
{% endmacro %}
```

**Usage in T3 dimension:**
```sql
SELECT
  vehicle_id,
  {{ lbs_to_kg('capacity_lbs') }} AS capacity_kg,
  {{ miles_to_km('current_mileage') }} AS odometer_km,
  {{ mpg_to_l_per_100km('fuel_efficiency_mpg') }} AS fuel_efficiency_l_100km
FROM {{ ref('tbl_stg_vehicles') }}
```

## Deterministic Key Generation

Generate surrogate keys from composite natural keys:

```sql
{% macro generate_route_id(origin_col, destination_col) %}
  MD5(CONCAT(
    COALESCE(CAST({{ origin_col }} AS VARCHAR), ''),
    '_',
    COALESCE(CAST({{ destination_col }} AS VARCHAR), '')
  ))
{% endmacro %}
```

**Usage:** `{{ generate_route_id('origin_location_id', 'destination_location_id') }} AS route_id`

## Business Classification Macros

Encapsulate classification logic for reuse:

```sql
{% macro classify_delivery_performance(on_time_rate) %}
  CASE
    WHEN {{ on_time_rate }} >= 0.95 THEN 'excellent'
    WHEN {{ on_time_rate }} >= 0.90 THEN 'good'
    WHEN {{ on_time_rate }} >= 0.80 THEN 'acceptable'
    ELSE 'needs_improvement'
  END
{% endmacro %}

{% macro classify_haul_type(distance_km) %}
  CASE
    WHEN {{ distance_km }} <= 50 THEN 'short_haul'
    WHEN {{ distance_km }} <= 200 THEN 'medium_haul'
    ELSE 'long_haul'
  END
{% endmacro %}
```

## SQL Injection Prevention

When building dynamic SQL in macros (e.g., `on-run-end` hooks), sanitise all string values:

```sql
{% macro _safe_sql_string(value, max_length=none) %}
    {%- set s = value | default('') | string -%}
    {%- if max_length is not none -%}
        {%- set s = s[:max_length] -%}
    {%- endif -%}
    {{- s | replace("'", "''") -}}
{% endmacro %}
```

**Critical:** Truncate FIRST, then escape. Escaping before truncation can split a `''` pair at the boundary, reopening a SQL string literal.

## Pipeline Run Logging (on-run-end)

Log every model result to an audit table:

```sql
{% macro log_pipeline_run_results() %}
  {% set log_db = env_var('SF_ENV', 'DEV') | upper ~ '_T0_CONTROL' %}

  {% for result in results %}
    {% if result.node.resource_type == 'test' %}{% continue %}{% endif %}

    {# Derive layer from tags using namespace (persists outside loop) #}
    {% set ns = namespace(layer='unknown') %}
    {% for tag in result.node.tags %}
      {% if tag in ['t0','staging','dimensions','facts','presentation'] and ns.layer == 'unknown' %}
        {% set ns.layer = tag %}
      {% endif %}
    {% endfor %}

    {% set insert_sql %}
      INSERT INTO {{ log_db }}.JOB_CONTROL.TBL_PIPELINE_RUN_LOG (
        pipeline_name, run_status, invocation_id, layer,
        execution_time_secs, logged_at
      ) VALUES (
        '{{ _safe_sql_string(result.node.name) }}',
        '{{ _safe_sql_string(result.status) }}',
        '{{ _safe_sql_string(invocation_id) }}',
        '{{ _safe_sql_string(ns.layer) }}',
        {{ result.execution_time | round(2) if result.execution_time else 'NULL' }},
        CURRENT_TIMESTAMP()
      )
    {% endset %}
    {% do run_query(insert_sql) %}
  {% endfor %}
{% endmacro %}
```

## Jinja Tips

- **`namespace()`**: Required to set variables inside a `for` loop that persist outside it
- **`loop.last`**: Check if current iteration is the last (useful for trailing commas)
- **`{%- -%}`**: Whitespace-trimming delimiters prevent extra newlines in compiled SQL
- **`| default('')`**: Safe fallback for potentially-null variables
- **`dispatch`**: Override package macros with project-specific implementations

```yaml
# dbt_project.yml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ["my_project", "dbt_utils"]
```

---

## Extended Aggregation Macros

Beyond the basic `rolling_metrics` macro above, a full macro library provides specialised aggregation patterns.

### Daily Metrics (Pre-Aggregated)

Encapsulate standard daily rollups so fact tables remain consistent:

```sql
{% macro daily_metrics(partition_by, order_by) %}
  COUNT(*) AS daily_count,
  SUM(weight_kg) AS daily_weight,
  SUM(revenue) AS daily_revenue,
  SUM(distance_km) AS daily_distance,
  AVG(customer_rating) AS daily_avg_rating,
  {{ calculate_on_time_rate('is_on_time') }} AS daily_on_time_rate,
  COUNT(DISTINCT destination_location_id) AS daily_unique_destinations
{% endmacro %}
```

### Rolling Averages, Sums, and Counts (Standalone)

When only one aggregation type is needed, use the targeted variant to avoid generating unused columns:

```sql
{% macro rolling_averages(column, partition_by, order_by, windows=[7, 30, 90]) %}
  {% for window in windows %}
    AVG({{ column }}) OVER (
      PARTITION BY {{ partition_by }}
      ORDER BY {{ order_by }}
      {{ rolling_window_days(window) }}
    ) AS {{ column }}_{{ window }}d_avg
    {%- if not loop.last -%},{%- endif %}
  {% endfor %}
{% endmacro %}
```

`rolling_sums` and `rolling_counts` follow the same pattern with `SUM` and `COUNT` respectively.

### Volatility (Coefficient of Variation)

Measures how volatile a metric is over a rolling window -- useful for demand forecasting and anomaly detection:

```sql
{% macro calculate_volatility(column, partition_by, order_by, window_days=30) %}
  STDDEV({{ column }}) OVER (
    PARTITION BY {{ partition_by }}
    ORDER BY {{ order_by }}
    {{ rolling_window_days(window_days) }}
  ) / NULLIF(AVG({{ column }}) OVER (
    PARTITION BY {{ partition_by }}
    ORDER BY {{ order_by }}
    {{ rolling_window_days(window_days) }}
  ), 0)
{% endmacro %}
```

### Percentile Distributions

Calculate multiple percentiles in a single macro call:

```sql
{% macro calculate_percentiles(column, partition_by, percentiles=[25, 50, 75, 90]) %}
  {% for percentile in percentiles %}
    PERCENTILE_CONT({{ percentile / 100 }}) WITHIN GROUP (ORDER BY {{ column }}) OVER (
      PARTITION BY {{ partition_by }}
    ) AS {{ column }}_p{{ percentile }}
    {%- if not loop.last -%},{%- endif %}
  {% endfor %}
{% endmacro %}
```

**Usage:** `{{ calculate_percentiles('delivery_duration_mins', 'region_id') }}` generates `delivery_duration_mins_p25`, `_p50`, `_p75`, `_p90`.

### Moving Averages (Single Window)

When a specific window is needed rather than a multi-window sweep:

```sql
{% macro moving_average(column, partition_by, order_by, window_days) %}
  AVG({{ column }}) OVER (
    PARTITION BY {{ partition_by }}
    ORDER BY {{ order_by }}
    {{ rolling_window_days(window_days) }}
  )
{% endmacro %}
```

### Trend Calculation

Compare a recent metric against a historical baseline to express percentage change:

```sql
{% macro calculate_trend(recent_metric, historical_metric) %}
  CASE
    WHEN {{ historical_metric }} > 0
    THEN ({{ recent_metric }} / {{ historical_metric }} - 1) * 100
    ELSE NULL
  END
{% endmacro %}
```

---

## Extended Business Logic Macros

### On-Time Rate from Boolean

```sql
{% macro calculate_on_time_rate(boolean_column) %}
  AVG(CASE WHEN CAST({{ boolean_column }} AS BOOLEAN) THEN 1.0 ELSE 0.0 END)
{% endmacro %}
```

### Enumeration-to-Numeric Conversion

Encode categorical values as numerics for scoring or ML feature pipelines:

```sql
{% macro customer_tier_to_numeric(customer_tier) %}
  CASE
    WHEN {{ customer_tier }} = 'PREMIUM' THEN 3
    WHEN {{ customer_tier }} = 'STANDARD' THEN 2
    WHEN {{ customer_tier }} = 'BASIC' THEN 1
    ELSE 0
  END
{% endmacro %}
```

### Performance Scoring (Weighted Composite)

Combine multiple KPIs into a single normalised score:

```sql
{% macro calculate_performance_score(on_time_rate, efficiency_rate, satisfaction_score) %}
  CASE
    WHEN {{ on_time_rate }} IS NULL OR {{ efficiency_rate }} IS NULL
      OR {{ satisfaction_score }} IS NULL THEN NULL
    ELSE ROUND(
      ({{ on_time_rate }} * 0.4 +
       {{ efficiency_rate }} * 0.3 +
       {{ satisfaction_score }} / 10 * 0.3) * 10, 2
    )
  END
{% endmacro %}
```

### Delivery Window Classification

Classify shipments by the gap between planned and actual delivery:

```sql
{% macro classify_delivery_window(planned_date, actual_date) %}
  CASE
    WHEN DATEDIFF(day, {{ planned_date }}, {{ actual_date }}) = 0 THEN 'same_day'
    WHEN DATEDIFF(day, {{ planned_date }}, {{ actual_date }}) = 1 THEN 'next_day'
    WHEN DATEDIFF(day, {{ planned_date }}, {{ actual_date }}) > 1 THEN 'multi_day'
    ELSE 'unknown'
  END
{% endmacro %}
```

### Customer Value Classification

```sql
{% macro classify_customer_value(annual_revenue) %}
  CASE
    WHEN {{ annual_revenue }} >= 1000000 THEN 'high_value'
    WHEN {{ annual_revenue }} >= 100000 THEN 'medium_value'
    WHEN {{ annual_revenue }} >= 10000 THEN 'low_value'
    ELSE 'minimal_value'
  END
{% endmacro %}
```

### Cost Per Unit

Generic cost-per-unit calculation with safe division:

```sql
{% macro calculate_cost_per_unit(total_cost, volume, unit='km') %}
  CASE
    WHEN {{ volume }} > 0 THEN {{ total_cost }} / {{ volume }}
    ELSE NULL
  END
{% endmacro %}
```

---

## Extended Data Type Macros

### Safe Casting with TRY_CAST / TRY_TO_*

Snowflake's `TRY_CAST` and `TRY_TO_*` functions return `NULL` instead of raising errors on invalid input. Wrap these in macros to standardise usage:

```sql
{% macro safe_cast(column, target_type) %}
  TRY_CAST({{ column }} AS {{ target_type }})
{% endmacro %}

{% macro safe_number(column) %}
  TRY_TO_NUMBER({{ column }})
{% endmacro %}

{% macro safe_date(column) %}
  TRY_TO_DATE({{ column }})
{% endmacro %}

{% macro safe_timestamp(column) %}
  TRY_TO_TIMESTAMP({{ column }})
{% endmacro %}

{% macro safe_varchar(column, length=255) %}
  TRY_CAST({{ column }} AS VARCHAR({{ length }}))
{% endmacro %}
```

### Safe Cast with Default Fallback

When a `NULL` result from `TRY_CAST` is unacceptable, wrap with `COALESCE`:

```sql
{% macro safe_cast_with_error_handling(column, target_type, default_value=NULL) %}
  {% if default_value is not none %}
    COALESCE(TRY_CAST({{ column }} AS {{ target_type }}), {{ default_value }})
  {% else %}
    TRY_CAST({{ column }} AS {{ target_type }})
  {% endif %}
{% endmacro %}
```

### Extended Unit Conversions

Beyond the core conversions (lbs/kg, miles/km), the full library includes:

| Macro | Conversion |
|---|---|
| `kg_to_lbs(column)` | Kilograms to pounds |
| `km_to_miles(column)` | Kilometres to miles |
| `cubic_feet_to_cubic_meters(column)` | ft3 to m3 (`/ 35.3147`) |
| `cubic_meters_to_cubic_feet(column)` | m3 to ft3 (`* 35.3147`) |
| `mph_to_kmh(column)` | Miles/hour to km/hour |
| `kmh_to_mph(column)` | Km/hour to miles/hour |
| `hours_to_minutes(column)` | Hours to minutes |
| `minutes_to_hours(column)` | Minutes to hours |
| `celsius_to_fahrenheit(column)` | Celsius to Fahrenheit |

All conversions pipe through `safe_decimal()` or `safe_number()` to handle non-numeric input gracefully.

---

## Extended Date/Time Macros

### Rolling Window Clause

The `rolling_window_days` macro generates Snowflake window frame clauses and is used by all aggregation macros:

```sql
{% macro rolling_window_days(days) %}
  ROWS BETWEEN {{ days - 1 }} PRECEDING AND CURRENT ROW
{% endmacro %}
```

### Date Part Extraction

```sql
{% macro extract_date_part(date_column, part) %}
  EXTRACT({{ part }} FROM {{ date_column }})
{% endmacro %}
```

### Date Arithmetic Helpers

```sql
{% macro days_ago(days) %}
  DATEADD(day, -{{ days }}, CURRENT_DATE())
{% endmacro %}

{% macro days_between(start_date, end_date) %}
  DATEDIFF(day, {{ start_date }}, {{ end_date }})
{% endmacro %}

{% macro hours_between(start_timestamp, end_timestamp) %}
  DATEDIFF(hour, {{ start_timestamp }}, {{ end_timestamp }})
{% endmacro %}

{% macro minutes_between(start_timestamp, end_timestamp) %}
  DATEDIFF(minute, {{ start_timestamp }}, {{ end_timestamp }})
{% endmacro %}
```

### Period Boundary Helpers

```sql
{% macro start_of_month(date_column) %}
  DATE_TRUNC('month', {{ date_column }})
{% endmacro %}

{% macro end_of_month(date_column) %}
  LAST_DAY({{ date_column }})
{% endmacro %}

{% macro start_of_week(date_column) %}
  DATE_TRUNC('week', {{ date_column }})
{% endmacro %}
```

### Weekend/Weekday Detection

```sql
{% macro is_weekend(date_column) %}
  EXTRACT(dayofweek FROM {{ date_column }}) IN (0, 6)
{% endmacro %}

{% macro is_weekday(date_column) %}
  EXTRACT(dayofweek FROM {{ date_column }}) BETWEEN 1 AND 5
{% endmacro %}
```

---

## Snapshot Strategy Selection

dbt snapshots implement [[SCD Type 2]] change tracking. The choice of strategy depends on the source system's change-detection capabilities.

### Timestamp Strategy

Use when the source table has a reliable `updated_at` column that changes whenever any column value is modified.

```sql
{% snapshot customers_snapshot %}
{{
  config(
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    tags=['snapshots', 'scd_type_2', 'customer']
  )
}}
select * from {{ ref('tbl_stg_customers') }}
{% endsnapshot %}
```

**When to use:** Most dimension tables where the source system maintains an `updated_at` timestamp -- customers, vehicles, locations, maintenance records. This is the preferred strategy because it is more performant: dbt only compares a single timestamp column rather than hashing multiple columns.

### Check Strategy

Use when the source has **no reliable `updated_at` column** -- e.g. the `updated_at` is derived from `created_at` and would never trigger change detection.

```sql
{% snapshot traffic_conditions_snapshot %}
{{
  config(
    unique_key='traffic_id',
    strategy='check',
    check_cols=['traffic_level', 'congestion_delay_minutes',
                'average_speed_kmh', 'incident_count'],
    tags=['snapshots', 'scd_type_2', 'traffic']
  )
}}
select * from {{ ref('tbl_stg_traffic_conditions') }}
{% endsnapshot %}
```

**When to use:** Event-like tables or reference data where `updated_at` is unreliable. Specify only the columns that represent meaningful business changes in `check_cols` -- exclude audit columns, timestamps, and derived fields to avoid unnecessary snapshot rows.

### Strategy Selection Decision Tree

| Condition | Strategy | Example |
|---|---|---|
| Source has a reliable, independently-maintained `updated_at` | `timestamp` | Customers, vehicles, locations |
| Source has no `updated_at`, or it mirrors `created_at` | `check` | Traffic conditions, weather conditions |
| You need to detect changes in specific columns only | `check` with selective `check_cols` | Price changes on a product catalogue |
| `check_cols='all'` | `check` (all columns) | Use sparingly -- hashes every column on every run |

### Snapshot Schema Conventions

All snapshots generate dbt-managed columns:

| Column | Description |
|---|---|
| `dbt_scd_id` | Surrogate key for the snapshot row (unique across all versions) |
| `dbt_updated_at` | Timestamp when dbt detected the change |
| `dbt_valid_from` | Start of the validity period |
| `dbt_valid_to` | End of the validity period (`NULL` for the current record) |

Test `dbt_scd_id` for `unique` and `not_null`. Test `dbt_valid_from` and `dbt_updated_at` for `not_null`. Leave `dbt_valid_to` without a `not_null` test since current records will have `NULL`.
