# Databricks Certified Data Engineer Associate Study Guide

## Exam Overview

**Certification:** Databricks Certified Data Engineer Associate
**Target audience:** Data engineers with at least 6 months of experience using Databricks, Delta Lake, and Apache Spark. Candidates should understand ELT patterns, lakehouse architecture, and basic data governance.

**Format and scoring:**
- 45 questions (multiple-choice, multiple-select)
- 90 minutes
- Passing score: approximately 70%
- Proctored online via Kryterion

**Skills measured domains:**

| Domain | Weight |
|--------|--------|
| Databricks Lakehouse Platform | 24% |
| ELT with Spark SQL and Python | 29% |
| Incremental Data Processing | 22% |
| Production Pipelines | 16% |
| Data Governance | 9% |

> The ELT domain carries the largest weight. Prioritise Spark SQL, Delta Lake MERGE operations, and transformation patterns.

---

## Domain 1: Databricks Lakehouse Platform (24%)

### Key Concepts

**Lakehouse architecture** combines the flexibility of data lakes (schema-on-read, raw file storage) with the reliability of data warehouses (ACID transactions, schema enforcement). Delta Lake is the storage layer that enables this. See [[Databricks & Delta Lake]].

**Cluster types -- know the differences:**

| Cluster Type | Purpose | Key Behaviour |
|-------------|---------|---------------|
| **All-Purpose** | Interactive development, notebook exploration | Persistent, shared, manually started/stopped |
| **Job Cluster** | Automated pipeline execution | Created per job run, auto-terminates on completion |
| **SQL Warehouse** | BI queries, dashboards, SQL analytics | Serverless or classic, auto-scales and auto-stops |

**Databricks Repos** provide Git integration for version-controlled notebooks and project files. Supports GitHub, Azure DevOps, GitLab, and Bitbucket.

**Photon Engine** is a C++ vectorised query engine that accelerates SQL and DataFrame workloads. It is most effective for large scans, aggregations, and join-heavy queries. Enabled at the cluster level via `runtime_engine = "PHOTON"`.

### Practice Questions

**Q1.** A data engineer needs to run a nightly ETL pipeline that processes 500 GB of data. The pipeline runs for approximately 2 hours. Which cluster type should they use?

**A:** A **Job Cluster**. Job clusters are created specifically for automated workloads, scale independently per run, and auto-terminate on completion -- reducing cost compared to an always-on all-purpose cluster.

---

**Q2.** What distinguishes a lakehouse architecture from a traditional data lake?

**A:** A lakehouse adds **ACID transactions**, **schema enforcement**, and **governance** (via Unity Catalog) to the raw file storage of a data lake. The key enabler is Delta Lake, which provides a transaction log on top of Parquet files. A plain data lake has no transactional guarantees and cannot enforce schema consistency.

---

**Q3.** A data analyst needs to run ad-hoc SQL queries against production tables and build dashboards. Which compute resource is most appropriate?

**A:** A **SQL Warehouse**. SQL warehouses are optimised for BI workloads, support auto-scaling and auto-stop, and integrate with dashboarding tools. All-purpose clusters would work but are costlier for pure SQL workloads.

---

## Domain 2: ELT With Spark SQL And Python (29%)

### Key Concepts

**Delta Lake fundamentals** -- see [[Databricks & Delta Lake]] and [[Delta Lake Operations & Patterns]] for detailed coverage.

| Operation | Syntax | Purpose |
|-----------|--------|---------|
| **Time Travel** | `SELECT * FROM t VERSION AS OF 5` | Query historical snapshots |
| **RESTORE** | `RESTORE TABLE t TO VERSION AS OF 5` | Roll back to a previous version |
| **MERGE** | `MERGE INTO target USING source ON ...` | Upsert (update + insert) |
| **Schema Evolution** | `.option("mergeSchema", "true")` | Allow new columns on write |
| **Schema Enforcement** | Default behaviour | Reject writes that violate schema |
| **OPTIMIZE** | `OPTIMIZE t ZORDER BY (col)` | Compact small files, co-locate data |
| **VACUUM** | `VACUUM t RETAIN 168 HOURS` | Remove unreferenced data files |

**MERGE is heavily tested.** Understand the full syntax:

```sql
MERGE INTO silver.customers AS target
USING bronze.customers_raw AS source
ON target.customer_id = source.customer_id
WHEN MATCHED AND source.updated_at > target.updated_at
  THEN UPDATE SET *
WHEN NOT MATCHED
  THEN INSERT *
WHEN NOT MATCHED BY SOURCE
  THEN DELETE;
```

**VACUUM and Time Travel interaction:** VACUUM removes files older than the retention threshold (default 168 hours / 7 days). After vacuuming, time travel to versions that relied on those files will fail. This is a common exam trap.

**Schema enforcement vs schema evolution:**
- **Enforcement** (default): writes that add or change column types are rejected
- **Evolution** (opt-in): `.option("mergeSchema", "true")` or `ALTER TABLE ADD COLUMNS` allows schema changes
- In MERGE statements, use `SET * ` with `.option("spark.databricks.delta.schema.autoMerge.enabled", "true")` to enable automatic evolution

### Practice Questions

**Q4.** A data engineer runs `VACUUM my_table RETAIN 24 HOURS` and then attempts `SELECT * FROM my_table VERSION AS OF 3`, which was created 48 hours ago. What happens?

**A:** The query **fails** with a `FileNotFoundException`. VACUUM deleted the data files backing version 3 because they exceeded the 24-hour retention window. Time travel only works for versions whose underlying files have not been vacuumed.

---

**Q5.** Which SQL command compacts small files in a Delta table and co-locates data by specific columns for faster query performance?

**A:** `OPTIMIZE table_name ZORDER BY (column_name)`. OPTIMIZE compacts small files into larger ones (solving the "small file problem"). Z-ordering physically co-locates related data to enable data skipping during query execution.

---

**Q6.** A data engineer has a Bronze table and needs to upsert records into Silver -- inserting new records and updating existing ones based on a business key. Which operation should they use?

**A:** `MERGE INTO`. The MERGE statement supports `WHEN MATCHED THEN UPDATE` and `WHEN NOT MATCHED THEN INSERT` clauses, making it the standard pattern for upserts in Delta Lake. See [[Delta Lake Operations & Patterns]].

---

**Q7.** What is the difference between schema enforcement and schema evolution in Delta Lake?

**A:** **Schema enforcement** is the default behaviour: Delta Lake rejects any write that introduces new columns or changes column types, protecting downstream consumers. **Schema evolution** is opt-in (via `mergeSchema` option or `autoMerge` configuration) and allows the schema to grow by accepting new columns during writes.

---

## Domain 3: Incremental Data Processing (22%)

### Key Concepts

**Structured Streaming** is Spark's stream processing engine, treating a stream as an unbounded table. Key concepts:
- **Trigger modes:** `availableNow` (process all available, then stop), `processingTime` (micro-batch interval), continuous (experimental)
- **Output modes:** `append` (new rows only), `complete` (full result), `update` (changed rows)
- **Checkpointing:** stores stream state and progress; enables exactly-once processing and fault recovery
- **Watermarking:** `withWatermark("event_time", "10 minutes")` defines how late data can arrive before being dropped

**Auto Loader** (`cloudFiles` format) is Databricks' recommended approach for incremental file ingestion:

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    .option("cloudFiles.inferColumnTypes", "true")
    .load("/data/landing/")
)
```

See [[Databricks & Delta Lake]] for the full Auto Loader and streaming patterns.

**Auto Loader vs COPY INTO:**

| Feature | Auto Loader | COPY INTO |
|---------|-------------|-----------|
| Architecture | Streaming (readStream) | Batch SQL command |
| Schema inference | Automatic, evolves over time | Manual schema definition |
| File tracking | Automatic via cloud notification or directory listing | Tracks processed files in table metadata |
| Scalability | Millions of files efficiently | Better for smaller, predictable file sets |
| Recommended use | Default choice for production | Simple one-off or periodic batch loads |

**Change Data Feed (CDF)** tracks row-level changes (`INSERT`, `UPDATE_PREIMAGE`, `UPDATE_POSTIMAGE`, `DELETE`) for downstream CDC processing. Enabled with `ALTER TABLE t SET TBLPROPERTIES (delta.enableChangeDataFeed = true)`.

### Practice Questions

**Q8.** A data engineer needs to incrementally ingest CSV files that land in a cloud storage directory. New files arrive every few minutes, and the schema may evolve over time. Which approach should they use?

**A:** **Auto Loader** (`cloudFiles` format). It automatically detects new files, infers and evolves schema, and tracks which files have been processed. COPY INTO would work for simpler scenarios but does not handle schema evolution or scale as well to millions of files.

---

**Q9.** What is the purpose of a checkpoint location in Structured Streaming?

**A:** The checkpoint stores the **stream's processing state and progress metadata**. It enables exactly-once processing guarantees and allows the stream to resume from where it left off after a failure or restart, without reprocessing data.

---

**Q10.** A streaming query uses `outputMode("append")`. What does this mean?

**A:** In append mode, only **new rows** that have not been seen before are written to the sink. Previously output rows are never modified or retracted. This is the most common mode for streaming ingestion into Delta tables.

---

## Domain 4: Production Pipelines (16%)

### Key Concepts

**Delta Live Tables (DLT)** is a declarative framework for building reliable ETL pipelines. Key features:
- Define tables as SQL or Python functions decorated with `@dlt.table`
- Built-in quality constraints: `@dlt.expect` (warn), `@dlt.expect_or_drop` (filter), `@dlt.expect_or_fail` (halt)
- Automatic dependency resolution and incremental processing
- Manages the medallion architecture (Bronze -> Silver -> Gold) declaratively

```python
@dlt.table(comment="Cleaned shipments")
@dlt.expect("valid_weight", "weight_kg > 0")
@dlt.expect_or_drop("valid_status", "status IN ('PENDING','DELIVERED')")
@dlt.expect_or_fail("has_id", "shipment_id IS NOT NULL")
def silver_shipments():
    return dlt.read("bronze_shipments").select(...)
```

See [[Databricks & Delta Lake]] for additional DLT examples.

**DLT expectation behaviour -- this is frequently tested:**

| Decorator | On Violation | Use Case |
|-----------|-------------|----------|
| `@dlt.expect` | Log warning, **keep** the row | Monitor quality without blocking |
| `@dlt.expect_or_drop` | **Drop** the invalid row silently | Filter out bad data |
| `@dlt.expect_or_fail` | **Fail** the entire pipeline | Critical data integrity checks |

**Databricks Workflows** orchestrate multi-task jobs:
- Tasks can be notebooks, DLT pipelines, SQL queries, Python scripts, or JAR files
- Support linear and fan-out/fan-in dependency graphs
- **Job clusters** are created per run and auto-terminate
- Schedule with cron expressions or trigger on file arrival
- Retry policies, timeout settings, and email/webhook notifications

**Medallion architecture** in production:

| Layer | Purpose | Typical Pattern |
|-------|---------|----------------|
| **Bronze** | Raw ingestion, append-only | Auto Loader or COPY INTO, minimal transformation |
| **Silver** | Cleaned, deduplicated, conformed | MERGE for dedup, schema standardisation, quality checks |
| **Gold** | Business-level aggregates and KPIs | Aggregations, joins, materialised views for BI |

### Practice Questions

**Q11.** A data engineer wants to ensure that any shipment record without a `shipment_id` causes the entire DLT pipeline to halt. Which expectation decorator should they use?

**A:** `@dlt.expect_or_fail("has_id", "shipment_id IS NOT NULL")`. The `expect_or_fail` decorator stops the pipeline immediately when a violation is detected, preventing invalid data from propagating.

---

**Q12.** What is the difference between `@dlt.expect` and `@dlt.expect_or_drop`?

**A:** `@dlt.expect` logs a warning metric but **keeps the violating row** in the output. `@dlt.expect_or_drop` **removes the violating row** from the output entirely. Use `expect` for monitoring and `expect_or_drop` for filtering.

---

**Q13.** A team needs to run three tasks in sequence: ingest raw data, clean and deduplicate, then build aggregations. The pipeline should run every hour and use cost-efficient compute. How should this be configured?

**A:** Create a **Databricks Workflow** with three tasks in a linear dependency chain, each running on a **job cluster**. Set a cron schedule for hourly execution. Job clusters are created per run and auto-terminate, making them the most cost-efficient option for scheduled production workloads.

---

## Domain 5: Data Governance (9%)

### Key Concepts

**Unity Catalog** provides centralised governance with a three-level namespace. See [[Databricks & Delta Lake]].

```
Metastore (organisation-level)
+-- Catalog ("prod", "dev")
|   +-- Schema ("bronze", "silver", "gold")
|   |   +-- Table / View / Function / Volume
```

**Key governance capabilities:**
- **Fine-grained access control:** GRANT and REVOKE at catalog, schema, table, column, and row levels
- **Data lineage:** Automatic tracking of upstream/downstream dependencies across tables, notebooks, and jobs
- **Audit logging:** Records who accessed what data and when
- **Delta Sharing:** Open protocol for secure cross-organisation data sharing without data copying
- **Volumes:** Managed or external storage for non-tabular data (files, images, models)

**Identity and access:**
- Unity Catalog uses identity federation -- workspace users inherit from the account level
- **Groups** simplify permission management; prefer group-level grants over individual grants
- Service principals for automated workloads (CI/CD, scheduled jobs)
- `INFORMATION_SCHEMA` views expose metadata about grants, tables, columns, and lineage

**Dynamic views** for fine-grained access:
```sql
CREATE VIEW secure_customers AS
SELECT
  customer_id,
  CASE WHEN is_member('pii_readers') THEN email ELSE '***' END AS email,
  region
FROM silver.customers;
```

### Practice Questions

**Q14.** A data engineer needs to grant a team read access to all tables in the `silver` schema of the `prod` catalog. What is the correct SQL?

**A:**
```sql
GRANT SELECT ON SCHEMA prod.silver TO `analytics_team`;
```
This grants SELECT on all current and future tables within the schema. Unity Catalog supports inheritance -- schema-level grants cascade to all objects within.

---

**Q15.** What is Delta Sharing, and how does it differ from traditional data sharing approaches?

**A:** Delta Sharing is an **open protocol** for securely sharing data across organisations without copying it. Recipients access live data via a sharing server, meaning they always see the latest version. Unlike traditional approaches (data exports, SFTP transfers, API calls), Delta Sharing avoids data duplication and works across platforms -- recipients can use Spark, pandas, Power BI, or any compatible client.

---

**Q16.** A compliance officer asks a data engineer to identify which downstream tables and dashboards depend on the `silver.customers` table. Which Unity Catalog feature provides this information?

**A:** **Data lineage**. Unity Catalog automatically tracks column-level lineage across tables, views, notebooks, and workflows. The lineage graph shows both upstream sources and downstream consumers, enabling impact analysis before schema changes.

---

## Exam Tips And Strategies

### Time Management
- 45 questions in 90 minutes gives 2 minutes per question
- Flag difficult questions and return to them -- do not spend more than 3 minutes on any single question
- The exam does not penalise for wrong answers, so never leave a question blank

### High-Frequency Topics
These topics appear repeatedly on the exam -- ensure thorough understanding:

1. **MERGE syntax and behaviour** -- upsert patterns, matched/not-matched clauses
2. **VACUUM and time travel** -- retention periods, interaction between the two
3. **DLT expectations** -- the three decorators and their violation behaviours
4. **Auto Loader vs COPY INTO** -- when to use each
5. **Cluster types** -- all-purpose vs job vs SQL warehouse
6. **Unity Catalog namespace** -- metastore > catalog > schema > object hierarchy
7. **Structured Streaming** -- checkpointing, output modes, trigger modes
8. **Schema enforcement vs evolution** -- default behaviour and opt-in mechanisms

### Common Exam Traps

**VACUUM destroys time travel.** If a question combines VACUUM with a short retention period and then queries a historical version, the query will fail. The default retention is 7 days (168 hours).

**Schema enforcement is the default.** Delta Lake rejects schema-mismatched writes by default. Schema evolution must be explicitly enabled -- do not assume it is automatic.

**`@dlt.expect` keeps the row.** The plain `expect` decorator only logs a quality metric; it does not drop or fail. Questions often test whether you know that rows pass through despite violations.

**Auto Loader is streaming, not batch.** Auto Loader uses `readStream` and is fundamentally a streaming operation, even when using `trigger(availableNow=True)` for batch-like behaviour. COPY INTO is the batch SQL equivalent.

**Job clusters vs all-purpose clusters for cost.** Production pipelines should always use job clusters. Questions about cost optimisation almost always have "job cluster" as the correct answer.

**OPTIMIZE does not delete data.** It compacts small files into larger ones but does not remove data. VACUUM removes old, unreferenced files. These are complementary but distinct operations.

### What To Study Further

For deeper coverage of the underlying technologies, see these vault notes:
- [[PySpark Core Concepts]] -- Spark fundamentals, RDDs, DataFrames, and transformations
- [[Delta Lake Operations & Patterns]] -- advanced Delta patterns including CDC and SCD2
- [[Databricks & Delta Lake]] -- platform architecture, API, and infrastructure as code

---

## Key Concepts Quick Reference

| Term | Definition |
|------|-----------|
| **Delta Lake** | Open-source storage layer providing ACID transactions on top of Parquet files |
| **Transaction Log** | JSON-based log (`_delta_log/`) recording every change to a Delta table |
| **Time Travel** | Querying historical versions of a table using `VERSION AS OF` or `TIMESTAMP AS OF` |
| **VACUUM** | Removes unreferenced data files older than the retention threshold (default 7 days) |
| **OPTIMIZE** | Compacts small files into larger ones; Z-ordering co-locates data by column values |
| **MERGE** | SQL command for upserts -- combines INSERT, UPDATE, and DELETE in one atomic operation |
| **Schema Enforcement** | Default Delta behaviour rejecting writes with mismatched schemas |
| **Schema Evolution** | Opt-in setting allowing new columns to be added during writes |
| **Auto Loader** | Streaming file ingestion using `cloudFiles` format with automatic schema inference |
| **COPY INTO** | Batch SQL command for loading files into a Delta table with idempotent file tracking |
| **DLT** | Delta Live Tables -- declarative ETL framework with built-in quality expectations |
| **Structured Streaming** | Spark's stream processing engine treating data streams as unbounded tables |
| **Checkpoint** | Persistent metadata enabling exactly-once processing and stream recovery |
| **Change Data Feed** | Feature tracking row-level changes (insert/update/delete) for downstream CDC |
| **Unity Catalog** | Centralised governance layer with three-level namespace and fine-grained access control |
| **Delta Sharing** | Open protocol for cross-organisation data sharing without data copying |
| **Medallion Architecture** | Bronze (raw) -> Silver (cleaned) -> Gold (aggregated) layered data pattern |
| **Photon** | C++ vectorised query engine for accelerated SQL and DataFrame workloads |
| **AQE** | Adaptive Query Execution -- dynamically optimises shuffle partitions, joins, and skew handling |
| **Z-Ordering** | Data co-location technique enabling file-level data skipping for filtered queries |

---

## Study Resources

**Official preparation:**
- [Databricks Certified Data Engineer Associate](https://www.databricks.com/learn/certification/data-engineer-associate) -- exam registration and syllabus
- [Databricks Academy](https://www.databricks.com/learn/training) -- free self-paced courses aligned with exam domains
- Databricks documentation on [Delta Lake](https://docs.databricks.com/en/delta/index.html), [Auto Loader](https://docs.databricks.com/en/ingestion/auto-loader/index.html), and [Unity Catalog](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)

**Recommended approach:**
1. Complete the Databricks Academy "Data Engineer Associate" learning path
2. Review the exam guide and map each objective to your knowledge gaps
3. Build a small end-to-end pipeline (Auto Loader -> Bronze -> Silver -> Gold) in the Community Edition or a trial workspace
4. Focus extra time on the ELT domain (29%) and incremental processing domain (22%)
5. Take practice exams under timed conditions, then review every incorrect answer

**Hands-on practice:**
- Databricks Community Edition (free) for notebook-based Spark and Delta Lake experimentation
- Build a medallion pipeline with Auto Loader ingestion, MERGE-based deduplication, and DLT quality checks
- Practise Unity Catalog GRANT/REVOKE statements and data lineage exploration

---

*See also: [[Databricks & Delta Lake]] | [[Delta Lake Operations & Patterns]] | [[PySpark Core Concepts]]*
