# dbt Incremental Loading Patterns

## The Watermark Pattern

The most reliable incremental strategy uses a processing timestamp (`_ingested_at`) as a high-water mark:

```sql
{{ config(
    materialized='incremental',
    unique_key='shipment_id',
    incremental_strategy='merge',
    on_schema_change='sync_all_columns'
) }}

WITH src AS (
  SELECT * FROM {{ source('RAW', 'SHIPMENTS') }}
  {% if is_incremental() %}
    WHERE _loaded_at > (
      SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }}
    )
  {% endif %}
)

SELECT
  TRIM(shipment_id) AS shipment_id,
  ...
  CURRENT_TIMESTAMP() AS _ingested_at
FROM src
```

**How it works:**
1. Source system writes `_loaded_at` on every row it lands
2. Staging model tracks its own `_ingested_at` (when it processed the row)
3. On incremental runs, only rows where `_loaded_at > MAX(_ingested_at)` are selected
4. Fallback to `'1900-01-01'` handles the first run (full load)

## merge_update_columns

Restrict which columns are updated during a merge to protect immutable audit fields:

```sql
{{ config(
    materialized='incremental',
    unique_key='shipment_id',
    merge_update_columns=[
        'shipment_status',
        'actual_delivery_date',
        'delivery_time_hours',
        'updated_at'
    ]
) }}
```

**Why:** Without this, every column is overwritten on merge — including `created_at` and other audit fields that should never change after initial insert.

## on_schema_change

Controls what happens when source schema evolves:

| Option | Behaviour |
|--------|-----------|
| `ignore` (default) | New columns silently dropped |
| `append_new_columns` | New columns added, no existing columns modified |
| `sync_all_columns` | New columns added, removed columns dropped |
| `fail` | Pipeline fails on schema mismatch |

**Recommendation:** Use `sync_all_columns` for staging models so schema drift is handled automatically.

## Incremental Strategy Comparison

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **merge** | `MERGE INTO` with unique_key matching | Dimension/fact tables with updates |
| **append** | `INSERT INTO` only, no dedup | Immutable event logs, telemetry |
| **delete+insert** | Delete matching rows, then insert | When merge is expensive or unsupported |

## Staging vs Downstream Incremental

**Staging (T2):** Filters on source `_loaded_at` vs staging `_ingested_at`

```sql
WHERE _loaded_at > (SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }})
```

**Downstream (T3):** Filters on upstream model's `_ingested_at`

```sql
WITH src AS (
  SELECT * FROM {{ ref('tbl_stg_shipments') }}
  {% if is_incremental() %}
    WHERE _ingested_at > (SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }})
  {% endif %}
)
```

This creates a chain: source `_loaded_at` → staging `_ingested_at` → mart `_ingested_at`.

## Full Refresh

When data drift accumulates or schema changes break incremental logic:

```bash
dbt run --full-refresh --select tbl_stg_shipments
```

This drops and recreates the table, running the model without the `{% if is_incremental() %}` filter.

## High-Volume Considerations

For telemetry/IoT data, tag models for separate execution and monitoring:

```sql
{{ config(
    materialized='incremental',
    unique_key='telemetry_id',
    tags=['staging', 'telemetry', 'incremental', 'high_volume']
) }}
```

This allows selective runs: `dbt run --select tag:high_volume` and cost tracking per data category.

## Clean Pass-Through Pattern

Staging models should be clean pass-throughs — no business logic, just type casting and null handling:

```sql
SELECT
  CAST(customer_id AS VARCHAR) AS customer_id,
  COALESCE(TRIM(customer_name), 'UNKNOWN') AS customer_name,
  CAST(credit_limit AS DECIMAL(15, 2)) AS credit_limit,
  COALESCE(is_deleted, FALSE) AS is_deleted,
  CURRENT_TIMESTAMP() AS _ingested_at
FROM src
```

Business logic (segment mapping, unit conversions, derived fields) belongs in T3 integration models.
