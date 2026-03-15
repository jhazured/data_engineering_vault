# Incremental Loading Strategies

Full-refresh pipelines are simple but wasteful -- processing millions of unchanged rows every run burns compute and inflates latency. Incremental loading processes only new or changed data. The trick is choosing the right strategy for each table's change profile.

See also: [[Data Ingestion Patterns]], [[SCD Type 2 Patterns]], [[dbt Advanced Patterns & Cost Optimisation]]

---

## Strategy Decision Matrix

| Strategy | Latency | Complexity | Data Volume | Cost | Best Use Case |
|----------|---------|-----------|-------------|------|---------------|
| **Watermark** | Medium (batch) | Low | Low (delta only) | Low | Append-only facts, event logs |
| **CDC** | Low (near real-time) | Medium-High | Low (changes only) | Medium | High-frequency OLTP changes |
| **SCD2 Incremental** | Medium (batch) | Medium | Medium (history grows) | Medium | Dimension tables with history |
| **Snapshot** | High (full copy) | Low | High (full each run) | High | Small dims, validation phase |
| **Event-Driven** | Very Low (sub-second) | High | Low (per-event) | Variable | IoT telemetry, clickstream |

---

## Watermark-Based Loading

A control table tracks the last successfully processed value per source. Each run: lookup, filter, copy, advance.

### Watermark Table Design

From the aged-care lakehouse (`raw_ingestion_watermarks.sql`):

```sql
CREATE TABLE IF NOT EXISTS raw_ingestion_watermarks (
    source_name       STRING NOT NULL,      -- logical key, one row per source
    last_watermark_value STRING,            -- flexible: timestamp, ID, or LSN
    last_watermark_ts TIMESTAMP,
    last_updated_utc  TIMESTAMP NOT NULL,
    last_run_id       STRING                -- correlate with pipeline run
) USING DELTA;
```

`source_name` as logical key (not table name), `STRING` for value (handles timestamps and IDs), `last_run_id` for auditability.

### Lookup-Filter-Copy-Update Pattern

```
1. Lookup  → SELECT last_watermark_ts FROM watermarks WHERE source_name = 'X'
2. Filter  → SELECT * FROM source WHERE modified_date > @watermark
3. Copy    → Append to raw layer
4. Update  → SET last_watermark_ts = MAX(modified_date) from loaded batch
```

Watermark update via MERGE (from the Fabric wiki):

```sql
MERGE INTO t0.watermark AS target
USING (
    SELECT 'fact_payroll' AS table_name,
           MAX(modified_date) AS last_processed_timestamp
    FROM T1_DATA_LAKE.raw_payroll
) AS source ON target.table_name = source.table_name
WHEN MATCHED THEN UPDATE SET last_processed_timestamp = source.last_processed_timestamp
WHEN NOT MATCHED THEN INSERT (table_name, last_processed_timestamp)
    VALUES (source.table_name, source.last_processed_timestamp);
```

Critical rule: update the watermark only **after** the load succeeds. Advancing it before processing and failing means silently losing that batch.

---

## Change Data Capture (CDC)

Captures row-level inserts, updates, and deletes from the source transaction log.

| Platform | Mechanism | Offset Tracking |
|----------|-----------|-----------------|
| **Snowflake** | Streams (append-only or standard) | Managed per stream |
| **SQL Server** | `sys.sp_cdc_enable_table` | LSN (Log Sequence Number) |
| **PostgreSQL** | Debezium + WAL | Kafka offsets |
| **Azure SQL** | Change Tracking (`CHANGETABLE`) | Sync version number |

**Append-only** -- every change is a new row (audit logs, event-sourced):

```sql
INSERT INTO silver.events
SELECT *, METADATA$ACTION, METADATA$ISUPDATE FROM raw.events_stream;
```

**Upsert** -- MERGE changes into the target. Process DELETE/INSERT/UPDATE_AFTER in LSN order. Never skip deletes. Track offsets in a control table (`last_processed_lsn BINARY(10)`). Clean up change tables before retention expires.

---

## Incremental SCD2

Combine watermark filtering with [[SCD Type 2 Patterns]] merge logic -- compare only rows that arrived since the last load, expire/insert only those that actually changed.

### Hash Comparison for Change Detection

```sql
SELECT *, MD5(CONCAT_WS('|', col_a, col_b, col_c)) AS row_hash
FROM source_dimension WHERE _loaded_at > @last_watermark;

-- In MERGE: compare hash instead of N columns
WHEN MATCHED AND target.row_hash <> source.row_hash THEN
    UPDATE SET expiry_date = GETDATE(), is_current = 0
```

### ROW_NUMBER Dedup

From `VW_FLIGHTS_SILVER.sql` -- when the same key arrives multiple times, keep only the latest:

```sql
ROW_NUMBER() OVER (
    PARTITION BY r."TRANSACTIONID"
    ORDER BY r."UPDATED_AT" DESC, r."LOAD_BATCH_ID" DESC
) AS "RN"
-- downstream: WHERE "RN" = 1
```

### LAG for Change Detection

Also from the flights silver view -- flag actual changes vs re-ingested identical rows:

```sql
LAG(r."UPDATED_AT") OVER (
    PARTITION BY r."TRANSACTIONID" ORDER BY r."UPDATED_AT"
) AS "PREVIOUS_UPDATED_AT"

-- Derive:
CASE WHEN "PREVIOUS_UPDATED_AT" IS NULL THEN TRUE          -- new record
     WHEN "UPDATED_AT" != "PREVIOUS_UPDATED_AT" THEN TRUE  -- modified
     ELSE FALSE
END AS "HAS_CHANGED"
```

Avoids unnecessary SCD2 version creation when a row is re-ingested unchanged.

---

## Snapshot Loading

Full-copy-every-run. Use for small reference tables, validation phase, or sources with no change indicator.

### Diff-Based Loading

Add `snapshot_date` partition for point-in-time queries. Combine with MERGE to detect actual changes:

```sql
MERGE t2.dim_department AS target USING t1_department AS source
ON target.dept_id = source.dept_id AND target.is_current = 1
WHEN MATCHED AND (target.dept_name <> source.dept_name OR target.division_id <> source.division_id)
THEN UPDATE SET expiry_date = @SnapshotDate, is_current = 0
WHEN NOT MATCHED BY TARGET THEN INSERT (...) VALUES (...);
```

### Compaction

```sql
-- Keep last 90 days; beyond that, keep only 1st-of-month snapshots
DELETE FROM snapshots.dim_department
WHERE snapshot_date < DATEADD(DAY, -90, CURRENT_DATE) AND DAY(snapshot_date) != 1;
```

---

## Event-Driven Ingestion

Push-based ingestion triggered by source events rather than scheduled polling.

```
S3 PUT event  → SNS/SQS → Lambda       → COPY INTO warehouse
GCS finalise  → Pub/Sub → Cloud Function → BigQuery load
ADLS created  → Event Grid → Azure Func → Fabric pipeline
```

Snowpipe (native Snowflake):

```sql
CREATE PIPE raw.auto_ingest_pipe AUTO_INGEST = TRUE
AS COPY INTO raw.events FROM @s3_stage;
```

For real-time append (IoT, clickstream): `Source → Event Hub / Kafka → Eventstream → Lakehouse`. Event-driven shifts complexity from scheduling to exactly-once delivery -- use idempotent writes to handle at-least-once duplicates.

---

## Late-Arriving Data

Data that arrives after the watermark has already advanced past its business timestamp.

### Reprocessing Windows

```sql
CREATE PROCEDURE load_with_lookback @LookbackDays INT = 7 AS
BEGIN
    DECLARE @CutoffDate DATETIME2 = DATEADD(DAY, -@LookbackDays, GETDATE());
    INSERT INTO target (...) SELECT ... FROM source
    WHERE ingested_at >= @CutoffDate
    AND NOT EXISTS (SELECT 1 FROM target WHERE target.id = source.id);
END;
```

The `NOT EXISTS` makes it idempotent. Set lookback based on observed patterns (3-7 days batch, 1-24 hours streaming).

### FORCE_RELOAD Flag

From `SP_LOAD_FLIGHTS_DATA_ETL.sql` -- normal runs skip already-processed files, but `FORCE_RELOAD = TRUE` bypasses the check for backfills:

```javascript
if (!FORCE_RELOAD) {
    var existing_check = snowflake.execute({
        sqlText: 'SELECT COUNT(*) FROM "TBL_FLIGHTS_RAW" WHERE "SOURCE_FILE" = ?',
        binds: [filename]
    });
    if (existing_check.getColumnValue(1) > 0) skip_raw_load = true;
}
```

Downstream `NOT EXISTS` checks prevent duplicates even on forced reloads.

---

## Cost Optimisation

From the logistics platform benchmarks -- switching to incremental:

| Metric | Full Refresh | Incremental | Savings |
|--------|-------------|-------------|---------|
| **Fivetran rows/sync** | All rows every run | Changed rows only | 70-90% |
| **Snowflake credits** | Full table scan | Delta scan | 50-70% |
| **dbt build time** | All models | Incremental only | 60-80% |

**Retention windows** -- not all data needs equal retention:

| Source | Retention | Rationale |
|--------|-----------|-----------|
| Core (customers, shipments) | All historical | Business-critical |
| Weather API | 30-day rolling | Re-fetchable reference data |
| Telematics API | 7-day rolling | High volume, summarised downstream |

**Selective column sync** -- exclude large BLOB/TEXT columns from high-frequency Fivetran syncs.

---

## Idempotency Patterns

Every incremental pipeline must be safe to re-run without duplicates.

### File-Level Dedup

From the flights ETL -- track which source files have been processed:

```sql
INSERT INTO "TBL_FLIGHTS_RAW" (...)
SELECT ... FROM temp_stage_data temp
WHERE NOT EXISTS (
    SELECT 1 FROM "TBL_FLIGHTS_RAW" existing
    WHERE existing."TRANSACTIONID" = temp.col1 AND existing."SOURCE_FILE" = ?
);
```

### Batch ID Tracking

Every run gets a monotonically increasing batch ID -- enables audit (`which batch loaded this?`), rollback (`DELETE WHERE LOAD_BATCH_ID = @bad`), and gap detection in downstream layers.

### MERGE with Unique Keys

```sql
MERGE INTO target USING source ON target.business_key = source.business_key
WHEN MATCHED AND target.row_hash <> source.row_hash THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...) VALUES (...);
```

dbt equivalent with ROW_NUMBER dedup (from the logistics platform):

```sql
{{ config(materialized='incremental', unique_key='business_key',
          merge_update_columns=['col_a', 'col_b', 'updated_at']) }}

{% if is_incremental() %}
WITH incremental_data AS (
    SELECT sd.*, ROW_NUMBER() OVER (PARTITION BY sd.unique_key
        ORDER BY sd._loaded_at DESC) AS rn
    FROM source_data sd
    WHERE sd._loaded_at > (SELECT MAX(_loaded_at) FROM {{ this }})
)
SELECT * FROM incremental_data WHERE rn = 1
{% else %}
SELECT * FROM source_data
{% endif %}
```

---

## Key Takeaways

1. **Start with watermark** -- covers 80% of use cases with minimal complexity
2. **Watermark table is infrastructure** -- design once, reuse across all sources
3. **Always dedup** -- `ROW_NUMBER` or `NOT EXISTS` on every incremental load
4. **Never advance watermark before confirming success** -- most common bug
5. **Build in lookback** -- late-arriving data is inevitable, not exceptional
6. **Track batch IDs** -- makes debugging and rollback possible
7. **Transition snapshot to incremental** -- use snapshot for validation, then migrate
