# SCD Type 2 Patterns

Slowly Changing Dimension Type 2 preserves the **full history** of dimension changes by creating a new row for each version of a record, with validity timestamps.

## Core Concept

```
customer_sk | customer_id | name       | tier      | valid_from | valid_to   | is_current
------------|-------------|------------|-----------|------------|------------|----------
sk_001      | CUST_001    | Acme Corp  | BASIC     | 2023-01-01 | 2024-03-15 | false
sk_002      | CUST_001    | Acme Corp  | STANDARD  | 2024-03-15 | NULL       | true
```

**Key fields:**
- **Surrogate key** (`customer_sk`): Unique per version — safe for joins
- **Natural key** (`customer_id`): Business identifier — NOT unique in an SCD2 table
- **valid_from / valid_to**: Time range this version was active
- **is_current**: Convenience flag for the active version

## dbt Snapshot Implementation

### Timestamp Strategy

Use when the source has a reliable `updated_at` column:

```sql
{% snapshot customers_snapshot %}
{{ config(
    target_database=env_var('SF_ENV', 'DEV') ~ '_T2_PERSISTENT_STAGING',
    target_schema='LOGISTICS',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
) }}
SELECT * FROM {{ source('RAW', 'CUSTOMERS') }}
{% endsnapshot %}
```

### Check Strategy

Use when no reliable timestamp exists — dbt compares column values directly:

```sql
{% snapshot traffic_conditions_snapshot %}
{{ config(
    target_database=env_var('SF_ENV', 'DEV') ~ '_T2_PERSISTENT_STAGING',
    target_schema='LOGISTICS',
    unique_key='traffic_id',
    strategy='check',
    check_cols=['traffic_level', 'congestion_delay_minutes', 'average_speed_kmh'],
) }}
SELECT * FROM {{ ref('tbl_stg_traffic_conditions') }}
{% endsnapshot %}
```

**Trade-off:** Check strategy is slower (compares every row every run) but works when `updated_at` is unreliable or derived from `created_at`.

### Columns Added by dbt Snapshots

| Column | Purpose |
|--------|---------|
| `dbt_scd_id` | Surrogate key (hash of unique_key + valid_from) |
| `dbt_valid_from` | When this version became active |
| `dbt_valid_to` | When this version was superseded (NULL = current) |
| `dbt_updated_at` | When dbt last processed this row |

## Point-in-Time Joins

The main reason for SCD2: join facts to the dimension version that was active when the fact occurred.

### Joining Facts to SCD2 Dimensions

```sql
SELECT
  f.shipment_id,
  f.shipment_date,
  d.customer_name,
  d.customer_type
FROM {{ ref('tbl_fact_shipments') }} f
LEFT JOIN {{ ref('tbl_dim_customer') }} d
  ON f.customer_id = d.customer_id
  AND f.shipment_date >= d.effective_from
  AND (f.shipment_date < d.effective_to OR d.effective_to IS NULL)
```

**WARNING:** Never join on `customer_id` alone — this causes fan-out duplicates because multiple versions exist per customer.

### Current-State Views

For BI tools that don't need history, create a filtered presentation view:

```sql
-- T4 Presentation
{{ config(materialized='view') }}

SELECT *
FROM {{ ref('tbl_dim_customer') }}
WHERE is_current = TRUE
```

## Building the Dimension from Snapshots

The T3 dimension model reads from the snapshot and adds business logic:

```sql
{{ config(
    materialized='incremental',
    unique_key='customer_sk'
) }}

SELECT
  snap.dbt_scd_id AS customer_sk,
  snap.customer_id,
  snap.customer_name,
  -- Business logic applied in T3 (not in staging)
  CASE
    WHEN snap.customer_type IN ('ENTERPRISE') THEN 'ENTERPRISE'
    WHEN snap.customer_type IN ('STANDARD', 'SME') THEN 'STANDARD'
    ELSE 'BASIC'
  END AS customer_type,
  snap.credit_limit AS credit_limit_usd,
  UPPER(COALESCE(snap.status, 'ACTIVE')) = 'ACTIVE' AS is_active,
  snap.dbt_valid_from AS effective_from,
  snap.dbt_valid_to AS effective_to,
  snap.dbt_valid_to IS NULL AS is_current,
  ROW_NUMBER() OVER (
    PARTITION BY snap.customer_id ORDER BY snap.dbt_valid_from
  ) AS version
FROM {{ ref('customers_snapshot') }} snap
{% if is_incremental() %}
  WHERE snap.dbt_updated_at > (
    SELECT COALESCE(MAX(effective_from), '1900-01-01') FROM {{ this }}
  )
{% endif %}
```

## SCD Type Comparison

| Type | What Changes | History | Storage | Complexity |
|------|-------------|---------|---------|------------|
| **Type 0** | Nothing | None | Minimal | None |
| **Type 1** | Overwrite in place | Lost | Minimal | Low |
| **Type 2** | New row per change | Full | Grows over time | Medium |
| **Type 3** | Previous + current columns | Last change only | Fixed | Low |
| **Type 6** | Hybrid (1+2+3) | Full + current flag | Grows | High |

## When to Use SCD2

- Customer attributes that affect historical reporting (tier, segment, region)
- Vehicle status changes that impact fleet utilisation calculations
- Location classifications that change over time
- Any dimension where "what was the value at that point in time?" matters

## When NOT to Use SCD2

- High-velocity data (telemetry, logs) — use append-only facts instead
- Dimensions that never change (date, static reference data)
- When only the current state matters and history is irrelevant
