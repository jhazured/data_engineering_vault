# Open Table Formats — Iceberg, Hudi & Delta

Open table formats bring warehouse-grade capabilities (ACID, schema evolution, time travel) to files on object storage. They add a metadata layer between compute engines and physical data files, turning Parquet collections into proper tables. This note compares the three dominant formats and provides guidance on when to use each.

See also: [[Delta Lake Operations & Patterns]], [[Databricks & Delta Lake]], [[Distributed Systems Fundamentals]], [[Data Engineering Lifecycle]]

---

## 1 — Why Open Table Formats

Traditional data lakes (raw Parquet/ORC on S3/ADLS/GCS) suffer from several limitations:

- **No ACID transactions** — concurrent writers can produce corrupt or partial reads
- **No time travel** — once overwritten, previous data is gone
- **Schema rigidity** — changing column types or adding columns requires full rewrites and downstream coordination
- **Partition coupling** — repartitioning means rewriting every data file
- **Vendor lock-in** — proprietary formats tie you to a single engine or platform

Open table formats solve these by introducing a **metadata layer** between engines and data files. Data stays in open formats (Parquet, ORC, Avro) on object storage while metadata manages transactions, versioning, and schema. This **decouples storage from compute** — any engine that understands the format can read and write the same table.

---

## 2 — Apache Iceberg

Iceberg was created at Netflix to solve petabyte-scale table management problems. It is now an Apache top-level project with broad engine adoption.

### Architecture

Iceberg uses a tree-structured metadata model with four levels:

```
Catalog (Hive Metastore / Glue / Nessie / REST)
  └── Metadata File (JSON) — current schema, partition spec, properties
        └── Manifest List (Avro) — one per snapshot, lists all manifest files
              └── Manifest File (Avro) — tracks a subset of data files + column stats
                    └── Data Files (Parquet/ORC/Avro) — actual row data
```

**Key insight:** the manifest list is the snapshot. Creating a new snapshot means writing a new manifest list that may reuse most existing manifest files unchanged. This makes commits O(number of changed partitions), not O(total table size).

### Snapshot Isolation

Every write produces a new **snapshot**. Readers always see a complete, consistent snapshot — no partial writes visible. Writers use optimistic concurrency: if two writers conflict, one retries against the new table state.

### Hidden Partitioning

Iceberg applies **partition transforms** (year, month, day, hour, bucket, truncate) at write time. Queries filter on the source column and pruning happens automatically — no need for `WHERE year = 2025 AND month = 3`.

```sql
ALTER TABLE events ADD PARTITION FIELD hours(event_ts);
-- Queries just filter on source column — pruning is automatic
SELECT * FROM events WHERE event_ts > TIMESTAMP '2025-12-01';
```

### Partition Evolution

Changing a partition scheme does **not** require rewriting existing data. Iceberg tracks which partition spec each data file was written with. Old files keep their old layout; new files use the new spec.

### Schema Evolution

Iceberg supports full schema evolution without rewriting data:

- **Add columns** — new columns read as NULL for existing files
- **Drop columns** — metadata marks them deleted; physical files untouched until rewritten
- **Rename columns** — tracked by internal column IDs, not names
- **Reorder columns** — column IDs make physical order irrelevant
- **Widen types** — e.g., int → long, float → double

### Time Travel & Snapshot Expiration

```sql
SELECT * FROM catalog.db.events VERSION AS OF 12345678901234;
SELECT * FROM catalog.db.events TIMESTAMP AS OF '2025-12-01 00:00:00';
-- Expire old snapshots to reclaim storage
CALL catalog.system.expire_snapshots('db.events', TIMESTAMP '2025-11-01 00:00:00');
```

---

## 3 — Apache Hudi

Hudi (Hadoop Upserts Deletes and Incrementals) was created at Uber for near-real-time ingestion of change data. It excels at **record-level upserts** and **incremental processing**.

### Table Types

| Aspect | Copy-on-Write (CoW) | Merge-on-Read (MoR) |
|--------|---------------------|---------------------|
| Write | Rewrites entire file on update | Writes delta log files |
| Read | Fast (no merge needed) | Slower (merges base + log at read) |
| Write latency | Higher | Lower |
| Read latency | Lower | Higher (until compaction) |
| Best for | Read-heavy, batch workloads | Write-heavy, near-real-time |

### Timeline & Instants

Hudi tracks all operations on a **timeline** as **instants** — each with a timestamp, action type (commit, deltacommit, compaction, clean, rollback), and state (requested, inflight, completed). The timeline is the source of truth for table state.

### File Groups & Record-Level Indexing

Data is organized into **file groups** — each containing a base file (Parquet) plus zero or more log files (Avro, MoR only). A **file slice** is a base file plus its associated log files at a given instant.

Hudi maintains indexes (bloom filter, simple, HBase, bucket) that map **record keys** to file groups. This enables efficient upserts — the engine knows exactly which file group contains a given record, avoiding full-table scans during writes. See also: [[Incremental Loading Strategies]]

### Incremental Queries

```python
hudi_options = {
    "hoodie.datasource.query.type": "incremental",
    "hoodie.datasource.read.begin.instanttime": "20251201000000"
}
df = spark.read.format("hudi").options(**hudi_options).load(path)
```

This powers **incremental ETL** — downstream jobs process only changed records instead of full snapshots.

### Compaction

For MoR tables, compaction merges log files into base files (synchronous or asynchronous). Strategy balances write amplification against read performance.

---

## 4 — Delta Lake

Delta Lake was created by Databricks and open-sourced under the Linux Foundation. It is the native storage format for the [[Databricks & Delta Lake|Databricks lakehouse]].

### Transaction Log & Checkpoints

Every Delta table has a `_delta_log/` directory containing JSON files (one per commit) recording file additions/removals, schema changes, and metadata. Every 10 commits, a Parquet **checkpoint file** summarizes the full table state so readers skip replaying old logs.

Writers use **optimistic concurrency** — each writes the next sequentially numbered JSON commit file. Conflicting writes on the same files cause a `ConcurrentModificationException`; compatible changes retry automatically.

See [[Delta Lake Operations & Patterns]] for detailed MERGE patterns, maintenance commands, and Change Data Feed.

### Schema Enforcement & Evolution

- **Enforcement** — rejects writes with mismatched schemas by default (safe guardrail)
- **Evolution** — enabled with `.option("mergeSchema", "true")` for additive changes

### Key Maintenance Operations

```sql
-- Compact small files into larger ones
OPTIMIZE catalog.schema.table;

-- Co-locate data for multi-dimensional filters
OPTIMIZE catalog.schema.table ZORDER BY (region, event_date);

-- Remove files older than retention threshold
VACUUM catalog.schema.table RETAIN 168 HOURS;
```

For comprehensive coverage of OPTIMIZE, VACUUM, Z-ORDER, MERGE, and Change Data Feed, see [[Delta Lake Operations & Patterns]].

---

## 5 — Feature Comparison

| Feature | Apache Iceberg | Apache Hudi | Delta Lake |
|---------|---------------|-------------|------------|
| **ACID transactions** | Yes (snapshot isolation) | Yes (timeline-based) | Yes (optimistic concurrency) |
| **Time travel** | Yes (snapshot-based) | Yes (timeline-based) | Yes (version-based) |
| **Schema evolution** | Full (add/drop/rename/reorder) | Full (add/drop/rename) | Add columns; limited rename/drop |
| **Partition evolution** | Yes (no rewrite) | Limited (requires bootstrap) | Overwrite required |
| **Hidden partitioning** | Yes (transforms) | No (Hive-style) | No (Hive-style) |
| **Record-level indexing** | No (file-level stats) | Yes (bloom/bucket/HBase) | Liquid Clustering (recent) |
| **Streaming ingestion** | Flink, Spark Structured Streaming | Flink, Spark, DeltaStreamer | Spark Structured Streaming |
| **Incremental queries** | Incremental scan via snapshots | Native incremental query API | Change Data Feed |
| **Engine support** | Broad (Spark, Trino, Flink, Dremio, Snowflake, Starrocks) | Moderate (Spark, Flink, Trino, Presto) | Strong (Spark, Trino, Flink, Databricks) |
| **Governance** | Nessie (Git-like branching), REST catalog | Metadata table | Unity Catalog, Delta Sharing |
| **Community/Governance** | Apache Foundation (vendor-neutral) | Apache Foundation (Uber-driven) | Linux Foundation (Databricks-driven) |
| **Primary strength** | Multi-engine, partition evolution | CDC/upserts, incremental processing | Databricks ecosystem integration |

---

## 6 — Catalog Integration

### Iceberg Catalogs

Iceberg requires a **catalog** to map table names to metadata file locations:

| Catalog | Use Case |
|---------|----------|
| **Hive Metastore** | Legacy compatibility; pointer to current metadata file |
| **AWS Glue** | Serverless on AWS; native Athena/EMR integration |
| **Nessie** | Git-like branching, tagging, merge — data-as-code workflows |
| **REST Catalog** | Vendor-neutral HTTP API (Polaris, Tabular) — emerging standard |
| **JDBC Catalog** | Catalog state in any JDBC-compatible database |

### Unity Catalog (Delta)

Databricks [[Databricks & Delta Lake|Unity Catalog]] provides a three-level namespace (`catalog.schema.table`) with centralized access control, lineage tracking, and cross-workspace governance. It now also supports Iceberg read interoperability via UniForm.

### Hudi Metadata Table

Hudi stores its own metadata (file listings, column stats, bloom filters) in an internal metadata table co-located with the data. This avoids expensive file listing operations on object storage at scale.

---

## 7 — Engine Compatibility

| Engine | Iceberg | Hudi | Delta Lake |
|--------|---------|------|------------|
| **Spark** | Full read/write | Full read/write | Full read/write (native on Databricks) |
| **Trino / Presto** | Full read/write | Read/write (connector) | Read/write (connector) |
| **Flink** | Read/write + streaming | Read/write + streaming | Read/write (limited) |
| **Dremio** | Native (Iceberg-first) | Limited | Read only |
| **Snowflake** | Native Iceberg Tables | Not supported | External tables (read) |
| **Databricks** | Read via UniForm | Read (connector) | Native (first-class) |
| **BigQuery** | BigLake (read/write) | Not supported | Not supported |
| **Athena** | Full read/write (v3) | Read (limited) | Read only |

**Takeaway:** Iceberg has the broadest engine support. Delta has the deepest Databricks integration. Hudi is strongest for Spark/Flink streaming upserts.

---

## 8 — Migration Patterns

### Hive to Iceberg (In-Place Migration)

Iceberg can migrate Hive tables without rewriting data — existing Parquet files become Iceberg data files:

```sql
CALL catalog.system.snapshot('hive_db.events', 'iceberg_db.events');  -- read-only copy
CALL catalog.system.migrate('hive_db.events');                        -- full in-place migration
```

### Delta to Iceberg (UniForm)

Databricks **UniForm** generates Iceberg metadata alongside Delta metadata at write time, making a single table readable as both formats:

```sql
ALTER TABLE catalog.schema.my_table
SET TBLPROPERTIES ('delta.universalFormat.enabledFormats' = 'iceberg');
```

### Choosing for Greenfield Projects

- **Databricks-first?** --> Delta Lake (native, Unity Catalog, Photon)
- **Multi-engine (Snowflake + Trino + Spark)?** --> Iceberg (broadest compatibility, REST catalog)
- **CDC-heavy with record-level upserts?** --> Hudi (record-level indexing, incremental queries)
- **No strong preference?** --> Iceberg (safest default, strongest community momentum)

---

## 9 — Maintenance Operations

All three formats require regular maintenance to keep query performance high and storage costs low.

| Operation | Iceberg | Hudi | Delta Lake |
|-----------|---------|------|------------|
| **Snapshot/version expiration** | `expire_snapshots` | `clean` (timeline archival) | `VACUUM` (removes old versions) |
| **Orphan file cleanup** | `remove_orphan_files` | `clean` | `VACUUM` |
| **Small file compaction** | `rewrite_data_files` | Compaction (inline/async) | `OPTIMIZE` |
| **Sort/cluster data** | `rewrite_data_files` with sort order | Clustering | `ZORDER` / Liquid Clustering |
| **Manifest rewriting** | `rewrite_manifests` | N/A (metadata table) | Checkpoint files (automatic) |
| **Statistics collection** | Manifest-level column stats (automatic) | Metadata table stats | `ANALYZE TABLE` / file-level stats |

### Iceberg Maintenance Example

```sql
CALL catalog.system.expire_snapshots('db.events', TIMESTAMP '2026-03-08 00:00:00');
CALL catalog.system.remove_orphan_files('db.events');
CALL catalog.system.rewrite_data_files(table => 'db.events', strategy => 'sort',
    sort_order => 'event_ts ASC NULLS LAST');
CALL catalog.system.rewrite_manifests('db.events');
```

For Delta maintenance (OPTIMIZE, VACUUM, ZORDER), see [[Delta Lake Operations & Patterns]].

---

## 10 — When to Use What

**Delta Lake** — primary compute is Databricks, Unity Catalog governance, tightest [[Databricks & Delta Lake|Databricks integration]] (Liquid Clustering, Predictive Optimization, Delta Sharing). Use UniForm for Iceberg interop.

**Apache Iceberg** — multi-engine access (Snowflake + Trino + Spark + Flink), partition evolution needed, vendor-neutral governance, REST catalog. Safest default for greenfield projects.

**Apache Hudi** — CDC-heavy ingestion with frequent upserts/deletes, record-level indexing, incremental queries for downstream ETL (see [[Incremental Loading Strategies]]), near-real-time via DeltaStreamer or Flink.

### Decision Summary

| Factor | Choose Delta | Choose Iceberg | Choose Hudi |
|--------|-------------|----------------|-------------|
| Platform | Databricks | Multi-engine | Spark/Flink |
| Governance | Unity Catalog | REST/Nessie catalog | Metadata table |
| Write pattern | Batch + streaming | Batch + streaming | CDC/upsert-heavy |
| Partition needs | Static | Evolving | Static |
| Community trajectory | Databricks-driven | Broadest adoption | Niche but strong |
| Record-level operations | MERGE (file-level) | MERGE (file-level) | Native indexing |

---

## Key Takeaways

1. **All three solve the same core problem** — ACID, time travel, and schema evolution on object storage. Differences are in architecture, engine support, and operational characteristics.
2. **Iceberg is winning the multi-engine war** — Snowflake, Trino, Dremio, Flink, and Databricks (via UniForm) are converging on Iceberg as the interoperability standard.
3. **Delta remains the best choice on Databricks** — tightest integration, Photon performance, Unity Catalog. Use UniForm for cross-engine reads.
4. **Hudi occupies a valuable niche** — record-level indexing and incremental queries for CDC-heavy and near-real-time upsert workloads.
5. **Maintenance is non-negotiable** — compaction, snapshot expiration, and orphan cleanup are required or performance degrades and costs grow.
6. **The formats are converging** — UniForm, REST catalog adoption, and Apache XTable are reducing the cost of switching.

---

*Related:* [[Delta Lake Operations & Patterns]] | [[Databricks & Delta Lake]] | [[Distributed Systems Fundamentals]] | [[Incremental Loading Strategies]] | [[SCD Type 2 Patterns]] | [[Data Ingestion Patterns]] | [[Data Engineering Lifecycle]]
