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
