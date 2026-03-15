# Delta Lake Operations & Patterns

Delta Lake adds [[ACID Transactions]] to [[Apache Spark]], turning a data lake into a lakehouse. It stores data as [[Parquet]] files plus a transaction log (`_delta_log/`) that tracks every change as an atomic commit. This note covers the core operations, write/read patterns, SCD2 handling, table maintenance, and platform considerations drawn from production projects.

---

## 1 — Fundamentals

### ACID Transactions
Every write to a Delta table is an atomic commit recorded in the transaction log. Concurrent readers always see a consistent snapshot, and failed writes leave no partial data behind. This eliminates the "dirty read" and "partial file" problems common with plain Parquet lakes.

### Schema Enforcement & Evolution
Delta rejects writes whose schema does not match the target table (schema enforcement). To allow additive changes, enable **schema evolution**:

```python
df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .saveAsTable("silver_clients")
```

Schema evolution permits adding new columns but will not silently drop or rename existing ones.

### Time Travel
Every commit creates a new version. You can query any previous state:

```sql
-- By version number
SELECT * FROM silver_clients VERSION AS OF 12;

-- By timestamp
SELECT * FROM silver_clients TIMESTAMP AS OF '2026-03-01 08:00:00';
```

Time travel works as long as the underlying files have not been removed by [[#VACUUM]].

---

## 2 — MERGE Upsert Pattern

The `MERGE` statement is the workhorse for incremental loads. It atomically matches source rows against the target table and applies updates or inserts in a single transaction.

### Helper function (Fabric lakehouse)

```python
def merge_upsert(spark, target_table: str, source_df, key_columns: list):
    """Delta MERGE (upsert): merge source_df into target_table on key_columns."""
    source_df.createOrReplaceTempView("_merge_source")
    key_conds = " AND ".join(f"t.{c} = s.{c}" for c in key_columns)
    spark.sql(
        f"MERGE INTO {target_table} AS t USING _merge_source AS s ON {key_conds} "
        "WHEN MATCHED THEN UPDATE SET * WHEN NOT MATCHED THEN INSERT *"
    )
```

### Equivalent DeltaTable API

```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forName(spark, target_table).alias("t")
delta_table.merge(source_df.alias("s"), key_conds) \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()
```

### Composite Key Matching
For tables with multi-column business keys, pass all key columns to build the `ON` clause:

```python
merge_upsert(spark, "silver_service_visits", visits_df,
             key_columns=["visit_id", "client_id"])
```

The helper joins them with `AND`, producing `t.visit_id = s.visit_id AND t.client_id = s.client_id`.

---

## 3 — SCD Type 2 with Delta

[[Slowly Changing Dimensions]] Type 2 tracks the full history of dimension changes. The Silver layer uses `valid_from`, `valid_to`, and `is_current` columns.

### Table structure

```sql
CREATE TABLE IF NOT EXISTS silver_clients (
    client_id       STRING,
    first_name      STRING,
    last_name       STRING,
    date_of_birth   DATE,
    care_package_level INT,
    funding_amount  DOUBLE,
    region          STRING,
    enrollment_date DATE,
    -- ... additional business columns ...
    valid_from      TIMESTAMP NOT NULL,
    valid_to        TIMESTAMP,
    is_current      BOOLEAN NOT NULL
) USING DELTA
LOCATION 'Tables/silver_clients';
```

### Step 1 — Close the current row

```python
def close_current_row(spark, table, key_column, valid_to_ts):
    """Set valid_to and is_current = false for existing current row(s)."""
    keys_df.createOrReplaceTempView("_keys")
    spark.sql(f'''
        MERGE INTO {table} AS t
        USING _keys AS k
        ON t.{key_column} = k.{key_column} AND t.is_current = true
        WHEN MATCHED THEN UPDATE
            SET t.valid_to = cast('{valid_to_ts}' as timestamp),
                t.is_current = false
    ''')
```

### Step 2 — Insert the new row

```python
def insert_new_row(spark, table, df, valid_from_ts, is_current=True):
    """Insert new SCD2 row(s) with valid_from, valid_to = null, is_current = true."""
    from pyspark.sql import functions as F
    df_with_scd = (df
        .withColumn("valid_from", F.lit(valid_from_ts))
        .withColumn("valid_to", F.lit(None).cast("timestamp"))
        .withColumn("is_current", F.lit(True)))
    df_with_scd.write.format("delta").mode("append").saveAsTable(table)
```

### Temporal query patterns

```sql
-- Current state of all clients
SELECT * FROM silver_clients WHERE is_current = true;

-- State as of a specific business date
SELECT * FROM silver_clients
WHERE valid_from <= '2026-01-15' AND (valid_to IS NULL OR valid_to > '2026-01-15');
```

---

## 4 — Table Maintenance

Delta tables accumulate small files over time. Three maintenance operations keep things healthy.

### OPTIMIZE (bin-packing & Z-ORDER)

`OPTIMIZE` compacts small files into larger ones (~1 GB target). This dramatically improves scan performance.

```sql
-- Basic compaction
OPTIMIZE silver_clients;
OPTIMIZE silver_service_visits;
OPTIMIZE fact_service_delivery_daily;

-- Z-ORDER for multi-dimensional queries
OPTIMIZE silver_service_visits ZORDER BY (region, visit_date);
```

[[#Z-ORDER Clustering]] reorders data within files so predicates on the Z-ORDER columns skip more files.

### VACUUM

`VACUUM` deletes data files no longer referenced by the transaction log. The `RETAIN` hours parameter sets a safety window — files newer than that threshold are kept for time travel.

```sql
-- Remove files older than 7 days (168 hours)
VACUUM silver_clients RETAIN 168 HOURS;
```

> **Warning:** Once vacuumed, you cannot time-travel to versions older than the retain window. The default retain threshold is 7 days. Setting it lower requires `spark.databricks.delta.retentionDurationCheck.enabled = false`.

### ANALYZE TABLE (statistics)

Statistics help the [[Catalyst Optimizer]] choose better join strategies and scan plans. Run after large loads:

```sql
ANALYZE TABLE silver_clients COMPUTE STATISTICS;
ANALYZE TABLE silver_service_visits COMPUTE STATISTICS;
ANALYZE TABLE fact_service_delivery_daily COMPUTE STATISTICS;
```

### Scheduling Maintenance

Run OPTIMIZE then ANALYZE together after bulk ETL jobs. A typical maintenance script covers all layers:

```sql
-- Post-load maintenance for high-write tables
OPTIMIZE raw_service_visits;
OPTIMIZE silver_service_visits;
OPTIMIZE fact_service_delivery_daily;

ANALYZE TABLE raw_service_visits COMPUTE STATISTICS;
ANALYZE TABLE silver_service_visits COMPUTE STATISTICS;
ANALYZE TABLE fact_service_delivery_daily COMPUTE STATISTICS;
```

Schedule this as a downstream step in your [[Orchestration]] pipeline or as a periodic job.

---

## 5 — Watermark Pattern

The watermark pattern tracks how far each source has been ingested, enabling idempotent incremental loads.

### Watermark table design

```sql
CREATE TABLE IF NOT EXISTS raw_ingestion_watermarks (
    source_name          STRING NOT NULL,   -- logical key: one row per source
    last_watermark_value STRING,            -- opaque bookmark (e.g., max ID)
    last_watermark_ts    TIMESTAMP,         -- timestamp-based bookmark
    last_updated_utc     TIMESTAMP NOT NULL,
    last_run_id          STRING
) USING DELTA
LOCATION 'Tables/raw_ingestion_watermarks';
```

### Incremental load flow

```
1. Read watermark   → SELECT last_watermark_ts FROM raw_ingestion_watermarks
                       WHERE source_name = 'batch_clients'
2. Extract new data → Copy only rows WHERE LastModified > watermark
3. Load into raw    → df.write.format("delta").mode("append")...
4. Update watermark → UPDATE raw_ingestion_watermarks
                       SET last_watermark_ts = <new_max>, last_updated_utc = current_timestamp()
                       WHERE source_name = 'batch_clients'
```

This pattern is idempotent: re-running with the same watermark re-extracts the same window, and the MERGE in downstream layers deduplicates.

---

## 6 — Write Patterns

### Append vs Overwrite vs Merge

| Mode | Use case | Command |
|------|----------|---------|
| **append** | Raw ingestion, event/fact tables | `.mode("append")` |
| **overwrite** | Full snapshot refresh, dimension reload | `.mode("overwrite")` |
| **merge** | Incremental upsert, SCD2 | `MERGE INTO ... USING ...` |

### Partitioning Strategies

```python
# Partition by date for time-series data
df.write.format("delta") \
    .partitionBy("visit_date") \
    .mode("append") \
    .saveAsTable("raw_service_visits")
```

Keep partition cardinality reasonable (date is good, timestamp is not). Over-partitioning creates too many small files.

### Partition Overwrite Mode

Overwrite only the partitions touched by the current write, leaving other partitions intact:

```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

df.write.format("delta") \
    .partitionBy("region") \
    .mode("overwrite") \
    .saveAsTable("silver_clients")
```

---

## 7 — Read Patterns

### Time Travel Queries

```sql
-- Query a specific version
SELECT * FROM silver_clients VERSION AS OF 5;

-- Query as of a timestamp
SELECT * FROM silver_clients TIMESTAMP AS OF '2026-03-10 00:00:00';
```

### Version History

```sql
-- View full commit history
DESCRIBE HISTORY silver_clients;

-- Limit to recent versions
DESCRIBE HISTORY silver_clients LIMIT 20;
```

Each row shows the version number, timestamp, operation type, and metrics (rows affected, files added/removed).

### Restore to Version

```sql
-- Roll back to a previous version
RESTORE TABLE silver_clients TO VERSION AS OF 5;

-- Roll back to a timestamp
RESTORE TABLE silver_clients TO TIMESTAMP AS OF '2026-03-10 00:00:00';
```

`RESTORE` creates a new commit that makes the table look like the target version — it does not delete intermediate history.

---

## 8 — Performance

### Z-ORDER Clustering
Z-ORDER co-locates related data within files based on the columns you specify. It works best on high-cardinality columns used in `WHERE` and `JOIN` predicates:

```sql
OPTIMIZE silver_service_visits ZORDER BY (client_id, visit_date);
```

Z-ORDER is not a substitute for partitioning — use partitioning for coarse-grained splits (date, region) and Z-ORDER for fine-grained access patterns within partitions.

### Partition Pruning
When the query predicate matches a partition column, Spark skips entire directories of files:

```sql
-- Only reads the visit_date = '2026-03-15' partition
SELECT * FROM raw_service_visits WHERE visit_date = '2026-03-15';
```

### File Compaction
Small files degrade performance. `OPTIMIZE` compacts them. For streaming tables that produce many tiny files, run OPTIMIZE on a schedule (hourly or after each micro-batch window).

### Predicate Pushdown
Delta stores min/max statistics per column per file in the transaction log. The engine uses these to skip files that cannot contain matching rows — no need to open them at all. `ANALYZE TABLE` refreshes these statistics.

### Delta Cache
On [[Databricks]], the Delta cache automatically caches remote data on local SSDs. It is transparent to the user and dramatically reduces repeated-read latency. Enable with:

```python
spark.conf.set("spark.databricks.io.cache.enabled", "true")
```

---

## 9 — Delta Lake on Different Platforms

### Databricks (native)
Delta Lake is the default storage format. All Spark tables are Delta tables. OPTIMIZE, VACUUM, Z-ORDER, and liquid clustering are fully supported. Additional features include Delta Sharing and Unity Catalog integration.

### Microsoft Fabric (OneLake)
Fabric lakehouses use Delta as the native table format. Tables are stored under `Tables/` in the lakehouse and are accessible from [[Spark Notebooks]], [[SQL Analytics Endpoint]], and [[Power BI]] direct lake mode.

```sql
CREATE TABLE IF NOT EXISTS silver_clients (...)
USING DELTA
LOCATION 'Tables/silver_clients';
```

Key differences: Fabric manages compute separately, VACUUM behaviour may differ, and some Databricks-specific features (liquid clustering, deletion vectors) are not yet available.

### Spark OSS (delta-spark JAR)
For open-source Spark, add the Delta Lake dependency:

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
    .config("spark.jars.packages", "io.delta:delta-spark_2.12:3.1.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate())
```

Core features (MERGE, time travel, OPTIMIZE, VACUUM) work the same. Platform-specific features like Delta cache and Unity Catalog are Databricks-only.

---

## Related Notes

- [[Medallion Architecture]] — Bronze / Silver / Gold layering
- [[Slowly Changing Dimensions]] — SCD patterns beyond Type 2
- [[PySpark DataFrame Operations]] — Core Spark operations
- [[Data Quality Patterns]] — Validation and quality scoring
- [[Apache Spark Architecture]] — Catalyst, Tungsten, execution model
