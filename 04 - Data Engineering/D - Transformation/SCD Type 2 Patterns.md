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

## Null-Safe Hash Pattern for Schema Evolution

When using hash-based SCD2 change detection, adding a new column to the hash input can invalidate all existing hashes — causing a mass false-positive expire-and-reinsert across the entire table. The null-safe key-value concatenation pattern avoids this by only including non-NULL columns in the hash input:

```sql
CONVERT(
    VARCHAR(64),
    HASHBYTES(
        'SHA2_256',
        CONCAT(
            CASE WHEN col1 IS NOT NULL THEN 'col1=' + CAST(col1 AS VARCHAR(MAX)) + '|' ELSE '' END,
            CASE WHEN col2 IS NOT NULL THEN 'col2=' + CAST(col2 AS VARCHAR(MAX)) + '|' ELSE '' END,
            CASE WHEN col3 IS NOT NULL THEN 'col3=' + CAST(col3 AS VARCHAR(MAX)) + '|' ELSE '' END
            -- all mapped columns excluding primary key
        )
    ),
    2
) AS etl_hash
```

**Why this works:** When a new column is added to the mapping and existing records have NULL for that column, the `CASE WHEN ... IS NOT NULL` skips it entirely — so the hash input is identical to what it was before the column was added. Only records where the new column has a non-NULL value will produce a different hash.

**Trade-offs:**
- A genuine change from NULL to a non-NULL value will be detected (correct)
- A change from a non-NULL value to NULL will also be detected (correct — the key-value pair disappears from the hash input)
- Column order in the CONCAT matters — must be consistent between runs (derive from a canonical source like `column_mapping_json`)

**Contrast with CONCAT_WS approach:**

```sql
-- Standard approach — breaks on schema evolution
MD5(CONCAT_WS('|', col_a, col_b, col_c)) AS row_hash
-- Adding col_d changes every hash even when col_d is NULL
-- because CONCAT_WS still includes the separator position
```

Use the null-safe key-value pattern when the column list may evolve over time and you cannot afford to re-version the entire history table.

---

## Microsoft Fabric T-SQL SCD2 (Warehouse Pattern)

When using Microsoft Fabric Warehouse (not Lakehouse), SCD2 is implemented with T-SQL stored procedures rather than dbt snapshots. This approach uses dynamic SQL to make the merge procedure **generic** — a single SP handles any table, driven by metadata from a control table.

### ETL Tracking Columns

Every T2 persistent table includes these columns alongside the business columns:

```sql
[etl_is_current]  BIT          NOT NULL,   -- 1 = active version, 0 = historical/expired
[etl_hash]        VARCHAR(MAX) NOT NULL,   -- SHA2_256 hash of all business columns
[etl_start_time]  DATETIME2(6) NOT NULL,   -- when this version became current
[etl_end_time]    DATETIME2(6) NULL,       -- when expired (NULL = still current)
[etl_is_deleted]  BIT          NOT NULL    -- 1 = soft-deleted (snapshot merge only)
```

**Design decisions:**
- `etl_hash` uses `HASHBYTES('SHA2_256', ...)` — change detection without comparing every column
- `etl_is_deleted` only set by the snapshot merge SP (full-reload tables where absence = deletion)
- `etl_end_time` is NULL for current rows — enables simple range joins for point-in-time queries
- All business columns land as `VARCHAR(MAX)` in T2 (type casting deferred to T3 integration layer)

### Incremental Merge (Windowed / Watermark Tables)

Used for tables loaded via rolling window or watermark — **no soft deletes** because the source is a partial dataset.

**Two-step process within a single transaction:**

```sql
-- Step 1: Expire changed rows
UPDATE T2
SET T2.etl_is_current = 0,
    T2.etl_end_time   = SYSUTCDATETIME()
FROM [T2_persistent].[schema].[table] AS T2
INNER JOIN (
    SELECT [pk],
           CONVERT(VARCHAR(64),
               HASHBYTES('SHA2_256',
                   ISNULL(CAST(T1.[col1] AS NVARCHAR(MAX)), '')
                 + ISNULL(CAST(T1.[col2] AS NVARCHAR(MAX)), '')
                 + ...
               ), 2) AS new_hash
    FROM [T1_transient].[schema].[table] AS T1
) AS T1 ON T1.[pk] = T2.[pk]
WHERE T2.etl_is_current = 1
  AND T1.new_hash <> T2.etl_hash;

-- Step 2: Insert new + changed rows as current versions
INSERT INTO [T2_persistent].[schema].[table]
    ([pk], [col1], [col2], ...,
     etl_is_current, etl_hash, etl_start_time, etl_end_time, etl_is_deleted)
SELECT T1.[pk], T1.[col1], T1.[col2], ...,
    1,                                              -- is_current
    CONVERT(VARCHAR(64), HASHBYTES('SHA2_256',
        ISNULL(CAST(T1.[col1] AS NVARCHAR(MAX)), '')
      + ISNULL(CAST(T1.[col2] AS NVARCHAR(MAX)), '')
      + ...
    ), 2),                                          -- etl_hash
    SYSUTCDATETIME(),                               -- etl_start_time
    NULL,                                           -- etl_end_time (current)
    0                                               -- not deleted
FROM [T1_transient].[schema].[table] AS T1
LEFT JOIN [T2_persistent].[schema].[table] AS T2
    ON T2.[pk] = T1.[pk] AND T2.etl_is_current = 1
WHERE T2.[pk] IS NULL                               -- brand new row
   OR CONVERT(VARCHAR(64), HASHBYTES('SHA2_256',
        ISNULL(CAST(T1.[col1] AS NVARCHAR(MAX)), '')
      + ISNULL(CAST(T1.[col2] AS NVARCHAR(MAX)), '')
      + ...
      ), 2) <> T2.etl_hash;                         -- changed row
```

**Why no MERGE statement?** Fabric Warehouse has limited `MERGE` support. The two-step UPDATE→INSERT pattern achieves the same result and is easier to audit (separate row counts per step).

### Snapshot Merge (Full-Reload Tables)

Used for tables where the API returns the **complete dataset** every run. Adds a soft-delete step before expire + insert.

**Three-step process within a single transaction:**

```sql
-- Step 1: Soft-delete rows absent from source
UPDATE T2
SET T2.etl_is_current = 0,
    T2.etl_is_deleted = 1,
    T2.etl_end_time   = SYSUTCDATETIME()
FROM [T2_persistent].[schema].[table] AS T2
LEFT JOIN [T1_transient].[schema].[table] AS T1
    ON T1.[pk] = T2.[pk]
WHERE T2.etl_is_current = 1
  AND T1.[pk] IS NULL;

-- Step 2: Expire changed rows (same as incremental)
-- Step 3: Insert new + changed rows (same as incremental)
```

**Why soft deletes only for snapshots?** When you receive a partial dataset (windowed/watermark), a missing record doesn't mean it was deleted — it just wasn't in the window. Soft deletes are only safe when the source guarantees completeness.

### Making It Generic with Dynamic SQL

Both procedures use `INFORMATION_SCHEMA.COLUMNS` to dynamically build the hash, insert, and select column lists — no hardcoded column names:

```sql
CREATE PROCEDURE [schema].[usp_scd2_merge]
    @schema    VARCHAR(100),   -- e.g. 'source_schema'
    @table     VARCHAR(100),   -- e.g. 'branch'
    @pk        VARCHAR(200),   -- e.g. 'id'
    @raw_wh    VARCHAR(200),   -- e.g. 'T1_transient'
    @pers_wh   VARCHAR(200),   -- e.g. 'T2_persistent'
    @ctrl_wh   VARCHAR(200),   -- e.g. 'T0_control'
    @job_id    VARCHAR(100),   -- correlation ID from master pipeline
    @pipe_name VARCHAR(100)    -- for audit logging
AS
BEGIN
    DECLARE @hash_cols NVARCHAR(MAX) = NULL;

    -- Build column lists dynamically, excluding PK and ETL columns
    SELECT
        @hash_cols = COALESCE(
            @hash_cols + ' + ISNULL(CAST(T1.' + QUOTENAME(COLUMN_NAME)
                + ' AS NVARCHAR(MAX)), '''')',
            'ISNULL(CAST(T1.' + QUOTENAME(COLUMN_NAME)
                + ' AS NVARCHAR(MAX)), '''')')
    FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_SCHEMA = @schema AND TABLE_NAME = @table
      AND COLUMN_NAME NOT IN (
          @pk, 'etl_is_current', 'etl_hash',
          'etl_start_time', 'etl_end_time', 'etl_is_deleted')
    ORDER BY ORDINAL_POSITION;

    -- Build and execute UPDATE + INSERT as dynamic SQL
    -- Wrapped in BEGIN TRY / BEGIN TRAN for atomicity
END;
```

**Key design points:**
- `QUOTENAME()` prevents SQL injection on column/table names
- `ORDINAL_POSITION` ordering ensures hash consistency across runs
- ETL columns are excluded from the hash — only business columns are compared
- `@job_id` flows through to the audit log for end-to-end pipeline tracing
- Both steps run in a single transaction — if the insert fails, the expire is rolled back

### Choosing Merge Type via Control Table

The pipeline doesn't hardcode which SP to call. A `merge_type` column in the control table drives the choice:

| `load_type` | `merge_type` | SP Called | Soft Deletes? |
|-------------|-------------|-----------|:-------------:|
| `snapshot` | `snapshot` | `usp_scd2_merge_snapshot` | Yes |
| `windowed` | `incremental` | `usp_scd2_merge` | No |
| `incremental` | `incremental` | `usp_scd2_merge` | No |

The T2 child pipeline reads `merge_type` from the control table and passes it as a parameter to select the correct stored procedure.

### SCD2 Health Check View

Monitor SCD2 integrity with a dedicated view across all tables:

```sql
CREATE VIEW [schema].[vw_scd2_health] AS
SELECT 'branch' AS table_name,
    -- Historical rows with no current version (orphans)
    SUM(CASE WHEN h.id IS NOT NULL AND c.id IS NULL
        THEN 1 ELSE 0 END) AS orphaned_historical,
    -- Rows where PK is NULL (data quality issue)
    SUM(CASE WHEN h.id IS NULL
        THEN 1 ELSE 0 END) AS null_pk_rows,
    -- Percentage of rows marked as soft-deleted
    CAST(SUM(CASE WHEN h.etl_is_deleted = 1 THEN 1.0 ELSE 0 END)
        / NULLIF(COUNT(*), 0) * 100 AS DECIMAL(5,2)) AS deleted_pct,
    -- Most recent merge timestamp
    MAX(h.etl_start_time) AS latest_merge
FROM {schema}.{table} h
LEFT JOIN {schema}.{table} c
    ON h.id = c.id
    AND c.etl_is_current = 1
    AND c.etl_is_deleted = 0
WHERE h.etl_is_current = 0
  AND h.etl_is_deleted = 0
UNION ALL
-- ... repeat per table
```

**What to watch for:**
- `orphaned_historical > 0` — expired rows with no current version; investigate if the delete step ran out of order
- `null_pk_rows > 0` — source data quality issue; should never happen with a well-defined API
- `deleted_pct` climbing steadily — normal for snapshot tables as records are deactivated in the source system
- `latest_merge` stale — pipeline may have failed; cross-reference with `T0_control.audit.pipeline_run_log`

---

## When NOT to Use SCD2

- High-velocity data (telemetry, logs) — use append-only facts instead
- Dimensions that never change (date, static reference data)
- When only the current state matters and history is irrelevant
