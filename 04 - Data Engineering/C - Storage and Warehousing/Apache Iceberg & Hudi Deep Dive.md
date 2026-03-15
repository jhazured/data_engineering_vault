#iceberg #hudi #open-table-formats #data-lakehouse #data-engineering

# Apache Iceberg & Hudi Deep Dive

A detailed technical exploration of Apache Iceberg and Apache Hudi architectures, internals, and operational patterns. This note goes deeper than the overview in [[Open Table Formats - Iceberg Hudi & Delta]] — covering metadata layer mechanics, catalog options, engine compatibility, and migration strategies.

See also: [[Open Table Formats - Iceberg Hudi & Delta]], [[Delta Lake Operations & Patterns]], [[Databricks & Delta Lake]], [[Data Engineering Lifecycle]]

---

## 1 — Iceberg Architecture In Depth

### Metadata Layer

Iceberg's power comes from its layered metadata tree. Each layer serves a distinct purpose:

```
Catalog
  └── Table Metadata File (JSON)
        ├── current-snapshot-id
        ├── schemas (versioned list)
        ├── partition-specs (versioned list)
        ├── sort-orders
        ├── properties
        └── snapshot-log
              └── Snapshot
                    └── Manifest List (Avro)
                          ├── Manifest File 1 (Avro)
                          │     ├── data_file: s3://bucket/data/part-00001.parquet
                          │     │     ├── partition values
                          │     │     ├── record_count: 50000
                          │     │     ├── file_size_in_bytes: 12345678
                          │     │     └── column_sizes, value_counts, null_counts,
                          │     │         lower_bounds, upper_bounds
                          │     └── data_file: s3://bucket/data/part-00002.parquet
                          │           └── ...
                          └── Manifest File 2 (Avro)
                                └── ...
```

**Table Metadata File (JSON):**
- Points to the current snapshot
- Stores all historical schema versions (by ID, not by name)
- Stores all partition spec versions
- Contains table properties (format version, default file format, etc.)
- The catalog stores a pointer to the current metadata file

**Manifest List (Avro):**
- One per snapshot — lists all manifest files that constitute the snapshot
- Each entry records the manifest file path, partition spec ID, added/deleted file counts, and partition field summaries
- Partition field summaries enable manifest-level pruning (skip entire manifests if no partition values match the query)

**Manifest File (Avro):**
- Tracks a subset of data files (typically one per partition or per write operation)
- Stores per-file statistics: record count, file size, column sizes, null counts, distinct counts, lower/upper bounds per column
- These statistics enable **file-level pruning** — the query engine skips files whose column bounds do not match the query predicate

**Data Files (Parquet/ORC/Avro):**
- The actual row data in an open columnar format
- Decoupled from metadata — the same file can be referenced by multiple snapshots

### Snapshot Isolation

Every write operation creates a new snapshot. The sequence is:

1. Writer reads the current table metadata
2. Writer creates new data files
3. Writer creates new manifest files referencing the new data files
4. Writer creates a new manifest list referencing existing + new manifest files
5. Writer atomically updates the catalog pointer to the new metadata file

**Optimistic concurrency:** if two writers attempt to commit simultaneously, one succeeds and the other detects the conflict (metadata file has changed). The failing writer retries against the new table state. Conflicts only occur when writers modify the same partitions.

```
Time ──►

Writer A:  read metadata v1 → write files → commit metadata v2 ✓
Writer B:  read metadata v1 → write files → commit fails (v1 stale)
                                          → retry against v2 → commit v3 ✓
```

### Hidden Partitioning

Traditional Hive-style partitioning requires queries to know the partition column layout:

```sql
-- Hive-style: user must know the partition structure
WHERE year = 2026 AND month = 3 AND day = 15

-- Iceberg hidden partitioning: filter on source column
WHERE event_timestamp > '2026-03-15'
-- Iceberg automatically applies the partition transform (e.g., days(event_timestamp))
```

Available partition transforms:

| Transform | Input | Partition Value |
|-----------|-------|----------------|
| `identity` | Any | Exact value |
| `year` | Timestamp | Year integer |
| `month` | Timestamp | Year-month integer |
| `day` | Timestamp | Date integer |
| `hour` | Timestamp | Date-hour integer |
| `bucket(N)` | Any | `hash(value) % N` |
| `truncate(W)` | String/Integer | Truncated to width W |
| `void` | Any | Always null (unpartitioned) |

### Schema Evolution

Iceberg tracks columns by **integer IDs**, not by name or position. This enables:

| Operation | Mechanism | Data Rewrite? |
|-----------|-----------|--------------|
| **Add column** | New ID assigned, existing files return NULL | No |
| **Drop column** | ID marked as deleted in metadata | No |
| **Rename column** | Same ID, new name in metadata | No |
| **Reorder columns** | IDs unchanged, new ordering in metadata | No |
| **Widen type** | int to long, float to double | No |
| **Make nullable** | Required to optional | No |
| **Nested schema changes** | Struct, list, map fields also tracked by ID | No |

**Contrast with Delta Lake:** Delta tracks columns by name and position. Renaming requires column mapping mode. Iceberg's ID-based approach is fundamentally more robust for schema evolution.

### Partition Evolution

Changing a partition scheme does **not** rewrite existing data:

```sql
-- Original: partitioned by day
ALTER TABLE events ADD PARTITION FIELD days(event_ts);

-- Later: change to hourly partitioning
ALTER TABLE events DROP PARTITION FIELD days(event_ts);
ALTER TABLE events ADD PARTITION FIELD hours(event_ts);
```

After evolution:
- Old data files retain their daily partition layout
- New data files use hourly partitioning
- Iceberg tracks which partition spec each manifest was written with
- Queries work correctly across both layouts — the planner handles the translation

### Time Travel

```sql
-- Query a specific snapshot
SELECT * FROM events VERSION AS OF 7654321098765;

-- Query as of a timestamp
SELECT * FROM events TIMESTAMP AS OF '2026-03-01 00:00:00';

-- List all snapshots
SELECT * FROM events.snapshots;

-- Roll back to a previous snapshot
CALL catalog.system.rollback_to_snapshot('db.events', 7654321098765);

-- Cherry-pick a snapshot (apply its changes to current state)
CALL catalog.system.cherrypick_snapshot('db.events', 7654321098765);
```

### Branching and Tagging (Iceberg v2)

```sql
-- Create a branch for testing
ALTER TABLE events CREATE BRANCH test_branch;

-- Write to the branch
INSERT INTO events.branch_test_branch VALUES (...);

-- Tag a snapshot for retention
ALTER TABLE events CREATE TAG release_2026q1 AS OF VERSION 12345;

-- Branches and tags have independent retention policies
ALTER TABLE events CREATE BRANCH audit_branch
    RETAIN 365 DAYS
    WITH SNAPSHOT RETENTION 90 SNAPSHOTS;
```

---

## 2 — Hudi Architecture In Depth

### Copy-on-Write vs Merge-on-Read

**Copy-on-Write (CoW):**

```
Write operation (upsert):
1. Read the existing Parquet file containing the record
2. Merge the update into the existing records
3. Write a completely new Parquet file
4. Mark the old file as replaced in the timeline

Result: every read is a simple Parquet scan — no merge needed at read time
```

**Merge-on-Read (MoR):**

```
Write operation (upsert):
1. Write only the changed records to a log file (Avro)
2. Log file references the base Parquet file it modifies

Read operation:
1. Read the base Parquet file
2. Read associated log files
3. Merge log records into base records at read time

Compaction (async):
1. Read base file + all log files
2. Merge into a new base Parquet file
3. Delete old log files
```

### Timeline Architecture

The timeline is Hudi's transaction log — every operation is an **instant** on the timeline:

```
.hoodie/
├── 20260315080000.commit              # completed commit
├── 20260315090000.deltacommit         # completed MoR delta commit
├── 20260315100000.compaction.requested # compaction plan created
├── 20260315100000.compaction.inflight  # compaction in progress
├── 20260315100500.compaction          # compaction completed
├── 20260315110000.clean.requested     # cleanup plan
├── 20260315110000.clean               # cleanup completed
├── 20260315120000.rollback            # rolled back a failed commit
└── hoodie.properties                  # table configuration
```

| Instant Type | Description |
|-------------|-------------|
| **commit** | A CoW write operation completed |
| **deltacommit** | A MoR write operation completed (log files written) |
| **compaction** | Log files merged into base files (MoR only) |
| **clean** | Old file versions removed based on retention |
| **rollback** | A failed or aborted commit undone |
| **savepoint** | A snapshot pinned for disaster recovery |
| **replace** | Clustering or insert-overwrite completed |

### File Group Organisation

```
table/
├── partition=2026-03-15/
│   ├── file_group_1/
│   │   ├── base_file_001.parquet     (base — written by commit or compaction)
│   │   ├── .log.1_0-0-0             (log — delta records, MoR only)
│   │   └── .log.1_0-0-1             (log — more delta records)
│   └── file_group_2/
│       └── base_file_002.parquet
└── partition=2026-03-14/
    └── ...
```

A **file slice** = base file + associated log files at a given instant. Readers see either:
- **Read-optimised view** — base files only (fast, potentially stale)
- **Real-time view** — base + log files merged (current, slower)

### Record-Level Indexing

Hudi's indexing maps record keys to file groups, enabling O(1) record location:

| Index Type | Mechanism | Trade-off |
|-----------|-----------|-----------|
| **Bloom filter** | In-memory bloom filter per file group | Fast for point lookups; false positives cause extra reads |
| **Simple** | In-memory join of incoming records with file metadata | Lower memory but slower for large tables |
| **Bucket** | Hash-based assignment to fixed number of buckets | Predictable, no index lookup needed; inflexible bucket count |
| **HBase** | External HBase table stores (record key → file group) mapping | Scalable for very large tables; operational overhead of HBase |
| **Record-level** (0.14+) | Metadata table with record-to-file mapping | Best performance for upserts; higher write overhead |

### Compaction Strategies

For MoR tables, compaction is critical for read performance:

```python
# Inline compaction (during writes)
hoodie_options = {
    "hoodie.compact.inline": "true",
    "hoodie.compact.inline.max.delta.commits": "5",
}

# Async compaction (separate job)
hoodie_options = {
    "hoodie.compact.inline": "false",
    "hoodie.compact.schedule.inline": "true",  # schedule during writes
    # Run compaction as a separate Spark job
}
```

| Strategy | Behaviour |
|----------|-----------|
| **BoundedIOCompactionStrategy** | Compact file groups with most log data first (minimise I/O) |
| **LogFileSizeBasedCompactionStrategy** | Compact when log files exceed a size threshold |
| **UnBoundedCompactionStrategy** | Compact all eligible file groups |
| **DayBasedCompactionStrategy** | Compact file groups by partition date |

### Incremental Queries

```python
# Read only changes since a specific instant
hudi_options = {
    "hoodie.datasource.query.type": "incremental",
    "hoodie.datasource.read.begin.instanttime": "20260315080000",
    "hoodie.datasource.read.end.instanttime": "20260315120000",  # optional
}

df = spark.read.format("hudi").options(**hudi_options).load(path)

# CDC mode — includes before/after images
hudi_options["hoodie.datasource.query.incremental.format"] = "cdc"
```

This powers **incremental ETL** — downstream jobs process only changed records, drastically reducing compute compared to full-snapshot reprocessing. See [[Incremental Loading Strategies]].

---

## 3 — Iceberg vs Delta Lake vs Hudi Comparison (Detailed)

| Dimension | Apache Iceberg | Delta Lake | Apache Hudi |
|-----------|---------------|------------|-------------|
| **Metadata format** | JSON metadata + Avro manifests | JSON transaction log + Parquet checkpoints | Timeline of instants (.commit files) |
| **Metadata location** | Separate catalog pointer | `_delta_log/` co-located with data | `.hoodie/` co-located with data |
| **Transaction model** | Snapshot isolation, optimistic concurrency | Optimistic concurrency (serial log commits) | Timeline-based MVCC |
| **Schema tracking** | Column IDs (integer-based) | Column names | Column names + IDs (recent versions) |
| **Schema evolution** | Full (add/drop/rename/reorder/widen) | Add, limited rename/drop (needs mapping mode) | Full (add/drop/rename) |
| **Partition evolution** | Yes — no data rewrite | No — requires full rewrite | Limited — needs bootstrap |
| **Hidden partitioning** | Yes (transforms: year, month, bucket, truncate) | No (Hive-style, explicit partition columns) | No (Hive-style) |
| **Record-level ops** | File-level (position/equality deletes in v2) | File-level (MERGE rewrites files) | Native record-level indexing |
| **Delete mechanism** | Position delete files + equality delete files | Copy-on-write (rewrite affected files) | CoW: rewrite file, MoR: log delete marker |
| **Streaming support** | Flink, Spark Structured Streaming | Spark Structured Streaming | Flink, Spark, DeltaStreamer |
| **Incremental reads** | Incremental scan via snapshot range | Change Data Feed (CDF) | Native incremental query API |
| **Compaction** | `rewrite_data_files` (manual/scheduled) | `OPTIMIZE` (manual/Predictive Optimisation) | Inline or async compaction (configurable) |
| **Small file handling** | Manifest-based rewrite, bin-packing | OPTIMIZE with bin-packing | Clustering, compaction |
| **Governance** | Nessie (Git-like), REST catalog, Polaris | Unity Catalog, Delta Sharing | Metadata table (self-contained) |
| **Engine breadth** | Broadest (Spark, Trino, Flink, Dremio, Snowflake, Athena, StarRocks) | Strong (Spark, Trino, Flink, Databricks) | Moderate (Spark, Flink, Trino, Presto) |
| **Primary strength** | Multi-engine, partition evolution, vendor-neutral | Databricks ecosystem, Photon performance | CDC/upserts, incremental processing |
| **Community governance** | Apache Foundation (vendor-neutral) | Linux Foundation (Databricks-driven) | Apache Foundation (formerly Uber-driven) |
| **Interoperability** | REST catalog standard emerging | UniForm (generates Iceberg metadata) | Apache XTable (cross-format metadata) |
| **Format version** | v2 (row-level deletes, branching) | Reader/Writer protocol versioning | 0.x → 1.x transition ongoing |

---

## 4 — Iceberg Catalog Options

The catalog is the entry point for all table operations. It maps table identifiers to metadata file locations and handles atomic commits.

### Catalog Comparison

| Catalog | Description | Strengths | Limitations |
|---------|-------------|-----------|-------------|
| **Hive Metastore** | Traditional Hadoop catalog; stores pointer to current metadata file in HMS database | Legacy compatibility, widely deployed | Single point of failure, no branching, HMS dependency |
| **AWS Glue** | Serverless catalog on AWS | Zero ops, native Athena/EMR/Redshift Spectrum integration | AWS-only, eventual consistency on listing |
| **Nessie** | Git-like catalog with branching, tagging, and merge | Data-as-code workflows, branch isolation for testing | Additional infrastructure, smaller community |
| **REST Catalog** | HTTP-based catalog API (Polaris, Tabular, custom) | Vendor-neutral, emerging standard, any backend | Newer, fewer battle-tested implementations |
| **JDBC Catalog** | Stores catalog state in any JDBC database (Postgres, MySQL) | Simple, self-hosted, no additional infrastructure | Single-database bottleneck at scale |
| **Unity Catalog** | Databricks' unified governance catalog | Deep Databricks integration, fine-grained access control | Databricks-centric (OSS version limited) |
| **Hadoop Catalog** | File-system based (metadata in HDFS/S3 directories) | No external dependency | No atomic rename on S3 (use with caution) |

### Catalog Configuration Examples

```python
# Spark with Glue Catalog
spark.conf.set("spark.sql.catalog.glue_catalog", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.glue_catalog.catalog-impl",
               "org.apache.iceberg.aws.glue.GlueCatalog")
spark.conf.set("spark.sql.catalog.glue_catalog.warehouse",
               "s3://my-warehouse/iceberg/")

# Spark with REST Catalog
spark.conf.set("spark.sql.catalog.rest_catalog", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.rest_catalog.type", "rest")
spark.conf.set("spark.sql.catalog.rest_catalog.uri", "https://catalog.example.com")
spark.conf.set("spark.sql.catalog.rest_catalog.warehouse", "my_warehouse")

# Spark with Nessie
spark.conf.set("spark.sql.catalog.nessie", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.nessie.catalog-impl",
               "org.apache.iceberg.nessie.NessieCatalog")
spark.conf.set("spark.sql.catalog.nessie.uri", "http://nessie:19120/api/v1")
spark.conf.set("spark.sql.catalog.nessie.ref", "main")
```

### Nessie Git-Like Workflows

```sql
-- Create a branch for development
CREATE BRANCH dev_branch IN nessie;

-- Switch to the branch
USE REFERENCE dev_branch IN nessie;

-- Make changes on the branch (isolated from main)
INSERT INTO nessie.db.events VALUES (...);
ALTER TABLE nessie.db.events ADD COLUMN new_col STRING;

-- Verify changes before merging
SELECT COUNT(*) FROM nessie.db.events AT BRANCH dev_branch;

-- Merge back to main
MERGE BRANCH dev_branch INTO main IN nessie;
```

---

## 5 — Query Engine Compatibility Matrix

| Engine | Iceberg Read | Iceberg Write | Hudi Read | Hudi Write | Notes |
|--------|-------------|---------------|-----------|------------|-------|
| **Spark** | Full (v2) | Full (v2) | Full (CoW + MoR) | Full | Best support across all formats |
| **Trino** | Full | Full | Read (connector) | Limited | Iceberg is first-class in Trino |
| **Flink** | Full | Full (streaming + batch) | Full (streaming) | Full (streaming) | Both formats excel in Flink streaming |
| **Dremio** | Native (Iceberg-first) | Full | Limited | No | Dremio built around Iceberg |
| **Snowflake** | Native Iceberg Tables | Write (managed) | No | No | Snowflake converging on Iceberg |
| **Athena** | Full (v3) | Full | Read (limited) | No | Iceberg via Glue Catalog |
| **Presto** | Full | Full | Read | Limited | Similar to Trino |
| **StarRocks** | Full | External | No | No | Iceberg for external tables |
| **BigQuery** | BigLake (read/write) | BigLake | No | No | Via BigLake Metastore |
| **Databricks** | Read (UniForm) | Via UniForm | Read (connector) | No | Delta native; Iceberg via interop |
| **Redshift** | Spectrum (read) | No | No | No | Iceberg via Spectrum external tables |
| **DuckDB** | Full | Full | Read | No | Growing Iceberg support |
| **Doris** | Full | Limited | Read | No | Iceberg for lakehouse queries |

**Key insight:** Iceberg has the broadest engine compatibility. If multi-engine access is a requirement, Iceberg is the clear choice.

---

## 6 — Migration Patterns

### Delta Lake to Iceberg

**Option 1: UniForm (Dual-Write)**

Databricks UniForm generates Iceberg metadata alongside Delta metadata at write time:

```sql
ALTER TABLE catalog.schema.my_table
SET TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

- Same physical Parquet files, dual metadata
- Non-Databricks engines read the Iceberg metadata
- No data duplication, near-zero overhead
- Limitation: Iceberg metadata is read-only for non-Databricks engines

**Option 2: Apache XTable (Metadata Translation)**

XTable translates metadata between formats without copying data:

```bash
# Convert Delta metadata to Iceberg metadata
java -jar xtable.jar \
    --sourceFormat DELTA \
    --targetFormats ICEBERG \
    --tableBasePath s3://bucket/delta_table/ \
    --icebergCatalogConfig catalog.properties
```

- Open-source, vendor-neutral
- Supports Delta → Iceberg, Hudi → Iceberg, and other combinations
- Must be re-run after each write to keep metadata in sync

**Option 3: Full Data Migration**

```python
# Read from Delta
df = spark.read.format("delta").load("s3://bucket/delta_table/")

# Write as Iceberg
df.writeTo("iceberg_catalog.db.table").createOrReplace()
```

- Complete format migration, clean break
- Requires downtime or dual-write period
- Use for tables where format independence is worth the migration cost

### Hive to Iceberg (In-Place)

```sql
-- Snapshot: creates Iceberg metadata pointing to existing Parquet files (read-only copy)
CALL catalog.system.snapshot('hive_db.events', 'iceberg_db.events');

-- Migrate: full in-place migration (converts table to Iceberg)
CALL catalog.system.migrate('hive_db.events');
```

- Existing Parquet files become Iceberg data files — no data copying
- New writes go through Iceberg's transaction model
- Schema and partition information transferred from Hive Metastore

### Hudi to Iceberg

```bash
# Via Apache XTable
java -jar xtable.jar \
    --sourceFormat HUDI \
    --targetFormats ICEBERG \
    --tableBasePath s3://bucket/hudi_table/ \
    --icebergCatalogConfig catalog.properties
```

- Metadata translation only — no data copying
- Hudi CoW tables translate cleanly (base Parquet files)
- MoR tables require compaction first (log files are Hudi-specific)

### Migration Decision Framework

| Scenario | Recommended Approach |
|----------|---------------------|
| Databricks shop adding non-Databricks readers | UniForm (lowest friction) |
| Full platform migration away from Databricks | Data migration to Iceberg with REST catalog |
| Gradual migration, mixed workloads | XTable for metadata sync during transition |
| Hive-based data lake modernisation | In-place Iceberg migration (`migrate` procedure) |
| Hudi CDC tables moving to Iceberg | Compact MoR tables, then XTable or data migration |
| Greenfield project, no existing data | Start with Iceberg + REST catalog |

---

## 7 — Iceberg Maintenance Operations

### Essential Maintenance Tasks

```sql
-- 1. Expire old snapshots (reclaim metadata space, enable data file deletion)
CALL catalog.system.expire_snapshots('db.events', TIMESTAMP '2026-03-01 00:00:00');

-- 2. Remove orphan files (data files not referenced by any snapshot)
CALL catalog.system.remove_orphan_files(
    table => 'db.events',
    older_than => TIMESTAMP '2026-03-08 00:00:00',
    dry_run => true  -- preview first
);

-- 3. Rewrite data files (compaction / sort optimisation)
CALL catalog.system.rewrite_data_files(
    table => 'db.events',
    strategy => 'sort',
    sort_order => 'event_ts ASC NULLS LAST',
    options => map('target-file-size-bytes', '134217728')  -- 128 MB target
);

-- 4. Rewrite manifests (reduce manifest file count)
CALL catalog.system.rewrite_manifests('db.events');
```

### Maintenance Schedule

| Task | Frequency | Purpose |
|------|-----------|---------|
| **expire_snapshots** | Daily | Remove old snapshots, enable orphan cleanup |
| **remove_orphan_files** | Weekly | Delete unreferenced data files from storage |
| **rewrite_data_files** | Weekly/as needed | Compact small files, optimise sort order |
| **rewrite_manifests** | Monthly | Reduce manifest overhead for large tables |

---

## 8 — Hudi Maintenance Operations

```python
# Cleaning (remove old file versions)
hoodie_options = {
    "hoodie.clean.automatic": "true",
    "hoodie.cleaner.policy": "KEEP_LATEST_COMMITS",
    "hoodie.cleaner.commits.retained": "10",
}

# Clustering (reorganise data layout for better read performance)
hoodie_options = {
    "hoodie.clustering.inline": "true",
    "hoodie.clustering.inline.max.commits": "4",
    "hoodie.clustering.plan.strategy.sort.columns": "event_date,region",
    "hoodie.clustering.plan.strategy.target.file.max.bytes": "134217728",
}

# Archival (archive old timeline instants to reduce metadata overhead)
hoodie_options = {
    "hoodie.archive.automatic": "true",
    "hoodie.keep.min.commits": "20",
    "hoodie.keep.max.commits": "30",
}
```

---

## Key Takeaways

1. **Iceberg's metadata tree is the key differentiator** — manifest-level partition summaries and file-level column statistics enable aggressive pruning without listing files on object storage.
2. **Hudi's record-level indexing is unmatched** — for CDC-heavy workloads with frequent upserts, Hudi locates target file groups in O(1) rather than scanning.
3. **Schema evolution is safest in Iceberg** — integer column IDs make renames, drops, and reorders metadata-only operations with zero ambiguity.
4. **Partition evolution is Iceberg-exclusive** — neither Delta nor Hudi can change partition schemes without data rewrites.
5. **Catalog choice matters for Iceberg** — REST catalog is the emerging standard for vendor-neutral deployments; Glue for AWS-native; Nessie for Git-like workflows.
6. **Migration paths are maturing** — UniForm and XTable reduce the cost of format migration, but a greenfield project should default to Iceberg unless Databricks-native or CDC-heavy.
7. **Maintenance is non-negotiable** — both formats require regular compaction, snapshot/version cleanup, and orphan file removal to maintain performance and control costs.

---

*Related:* [[Open Table Formats - Iceberg Hudi & Delta]] | [[Delta Lake Operations & Patterns]] | [[Databricks & Delta Lake]] | [[Incremental Loading Strategies]] | [[Data Engineering Lifecycle]] | [[SCD Type 2 Patterns]]
