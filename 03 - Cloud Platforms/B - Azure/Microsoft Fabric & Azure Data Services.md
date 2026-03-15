# Microsoft Fabric & Azure Data Services

## 1. Microsoft Fabric Overview

Microsoft Fabric is a unified analytics platform built on **OneLake** -- a single logical data lake that serves as the storage layer for all Fabric workloads. Every Fabric item (Lakehouse, Warehouse, semantic model) stores its data in OneLake as Delta/Parquet files.

**Core components:**
- **OneLake** -- unified storage layer; all data lives here as Delta/Parquet regardless of compute engine
- **Lakehouse** -- Spark engine on top of OneLake; supports Delta tables, VARIANT columns, materialized views
- **Warehouse** -- T-SQL engine on top of OneLake; supports stored procedures, views, zero-copy clones
- **Workspaces** -- logical containers that group related items (lakehouses, warehouses, pipelines, semantic models)
- **Capacities** -- compute resources billed by CU (Capacity Units)
  - **F8** (16 CU) -- suitable for development and small-to-medium workloads
  - **F64** -- recommended for production; better performance and scalability
  - Power BI Pro license required for semantic model and report creation

**Lakehouse vs Warehouse:**

| Aspect | Lakehouse | Warehouse |
|--------|-----------|-----------|
| Engine | Spark + SQL endpoint | T-SQL |
| Strengths | VARIANT columns, streaming, notebooks | Stored procedures, MERGE, views, clones |
| Primary use | T1 raw landing, data science | T2 SCD2, T3 storage, T5 presentation |
| Storage | Delta/Parquet in OneLake | Delta/Parquet in OneLake |

Both access the same underlying OneLake storage. Direct Lake reads the Parquet files directly regardless of whether they sit behind a Lakehouse or Warehouse.

---

## 2. T0-T5 Layered Architecture

The T0-T5 pattern is an enterprise data warehouse architecture for Fabric with clear separation of concerns at each layer.

```
T0: Control Layer (Warehouse - T-SQL tables)
  |
T1: Raw Landing (Lakehouse - VARIANT tables via Data Factory)
  | (shortcuts)
T2: Historical Record (Warehouse - SCD2 MERGE stored procedures)
  | (Dataflows Gen2)
T3: Transformations (Warehouse - star schema via Power Query M)
  | (zero-copy clone)
T3._FINAL: Validated Snapshots (Warehouse - clone tables)
  |
T5: Presentation (Warehouse - SQL views only)
  |
Semantic Layer: Direct Lake on OneLake + DirectQuery fallback
  |
Power BI Reports
```

**T0 -- Control Layer.** Pipeline orchestration metadata, watermark tracking (`t0.watermark`), logging (`t0.pipeline_log`), and configuration. Implemented as T-SQL tables in the Warehouse.

**T1 -- Raw Landing.** Schema-agnostic ingestion using VARIANT columns in Lakehouse. Data is transient; truncated after successful T2 processing. See [[#5. Lakehouse Patterns]].

**T2 -- Historical Record.** SCD2 dimensions via T-SQL MERGE stored procedures. Tracks effective_date/expiry_date/is_current with surrogate keys. See [[#6. Warehouse Patterns]].

**T3 -- Transformations.** Business logic exclusively through Dataflows Gen2 (Power Query M). Append-only writes; data already versioned in T2. Sub-patterns: T3.ref (lookups), T3.table_01 (base), T3.table_02 (joins), T3.agg_01 (aggregations).

**T3._FINAL -- Validated Snapshots.** Zero-copy clones of T3 tables isolating the semantic layer from pipeline failures. Drop-and-recreate after successful T3 completion.

**T5 -- Presentation.** Views only, no base tables. Business-friendly names, light formatting, RLS-ready. Version-controlled and deployed via CI/CD.

**Naming conventions:**
- Schemas: `t0`, `t2`, `t3`, `t5`
- Tables: `t2.dim_*`, `t2.fact_*`, `t3.ref_*`, `t3.*_FINAL`
- Views: `t5.vw_*`
- Procedures: `t2.usp_merge_dim_*`, `t3.usp_refresh_final_clones`
- Pipelines: `PL_T1_*`, `PL_T2_*`, `PL_MASTER_*`

---

## 3. Data Factory Pipelines

Data Factory serves two roles: **T1 ingestion** (primary) and **pipeline orchestration** (secondary). It does not perform business logic transformations -- that belongs to [[#4. Dataflows Gen2]].

**Master pipeline orchestration pattern:**

```
PL_MASTER_[Domain]
  |-- PL_T1_Master_Ingest      (Data Factory copy activities)
  |-- PL_T2_Process_SCD2       (execute T-SQL stored procedures)
  |-- PL_T3_Transform          (trigger Dataflows Gen2)
  |-- PL_T5_Clone_Refresh      (execute clone refresh procedure)
```

Each step depends on the previous step succeeding. Independent tables within a step run in parallel.

**T1 copy activity** uses `JsonSource` (or CSV/Parquet source) with `LakehouseSink` in append mode. Sources include ADLS Gen2, SQL Server, and REST APIs.

**Watermark-based incremental loading:**
1. Lookup watermark from `t0.watermark` (last processed timestamp)
2. Filter source data with `WHERE timestamp > @watermark`
3. Copy to T1, process through T2
4. Update watermark only after successful processing

**Error handling:** Configure retry with exponential backoff at the activity level. Log all pipeline executions to `t0.pipeline_log`. T1 truncation happens only after confirmed T2 success, followed by a watermark MERGE into `t0.watermark`.

---

## 4. Dataflows Gen2

Dataflows Gen2 is the exclusive tool for T3 transformations. It uses Power Query M language for no-code/low-code data preparation.

**Key distinction from Data Factory:**
- **Data Factory** = data movement and ingestion (T1)
- **Dataflows Gen2** = data transformation and business logic (T3)

**T3 transformation example -- data enrichment with joins:**

```m
let
    EmployeeSource = t3_employee_base,
    JobLevelRef = t3_ref_job_level,
    JoinTables = Table.NestedJoin(
        EmployeeSource, {"job_title"},
        JobLevelRef, {"job_title"},
        "JobLevel", JoinKind.LeftOuter
    ),
    ExpandColumns = Table.ExpandTableColumn(
        JoinTables, "JobLevel",
        {"job_level", "job_category"},
        {"job_level", "job_category"}
    )
in
    ExpandColumns
```

**Aggregation pattern (T3.agg):**

```m
let
    Source = t2_fact_payroll,
    GroupBy = Table.Group(Source,
        {"year", "month", "department_id"},
        {
            {"TotalGrossPay", each List.Sum([gross_pay]), type number},
            {"EmployeeCount", each List.Count(List.Distinct([employee_id])), Int64.Type}
        }
    )
in
    GroupBy
```

**Incremental refresh** is supported for large tables (over 1M rows). Filter on date/timestamp columns. Schedule periodic full refreshes alongside incremental ones.

**Query folding:** Push operations to the source engine where possible. Select columns early, filter before joins, use indexed columns.

**Destination configuration:**
- T3 transformation tables: **Append** mode (data already versioned in T2)
- T3.ref reference tables: **Replace** mode (full refresh of lookups)

---

## 5. Lakehouse Patterns

The Lakehouse serves as the T1 raw landing zone. Its primary pattern is VARIANT-based schema-agnostic ingestion.

**VARIANT table creation:**

```sql
CREATE TABLE raw_department (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_file STRING,
    payload VARIANT
);
```

**Querying VARIANT data:**

```sql
SELECT
    payload:dept_id::STRING AS dept_id,
    payload:dept_name::STRING AS dept_name,
    payload:division_id::STRING AS division_id
FROM raw_department;
```

**Materialized views** flatten VARIANT data for downstream consumption. Always create a materialized view for every VARIANT table; Warehouse shortcuts should point to materialized views, not raw VARIANT tables.

```sql
CREATE MATERIALIZED VIEW mv_department AS
SELECT id, ingested_at,
    payload:dept_id::STRING AS dept_id, payload:dept_name::STRING AS dept_name
FROM raw_department;

-- Zero-copy shortcut from Warehouse to Lakehouse materialized view
CREATE SHORTCUT t1_department IN SCHEMA dbo
FROM LAKEHOUSE 'T1_DATA_LAKE' TABLE 'mv_department';
```

**Schema evolution:** VARIANT columns absorb source changes automatically. New JSON fields appear without table alterations. Update materialized views to expose new fields.

---

## 6. Warehouse Patterns

The Warehouse handles T2 (SCD2 history), T3 storage, T3._FINAL clones, and T5 presentation views.

**SCD2 MERGE stored procedure** -- expire changed records, then insert new versions:

```sql
CREATE PROCEDURE t2.usp_merge_dim_department AS
BEGIN
    SET NOCOUNT ON;
    -- Step 1: Expire changed records
    UPDATE t2.dim_department
    SET expiry_date = GETDATE(), is_current = 0
    WHERE is_current = 1 AND dept_id IN (
        SELECT s.dept_id FROM t1_department s
        JOIN t2.dim_department t ON s.dept_id = t.dept_id AND t.is_current = 1
        WHERE s.dept_name <> t.dept_name OR s.division_id <> t.division_id);
    -- Step 2: Insert new/changed records
    INSERT INTO t2.dim_department (dept_id, dept_name, division_id, effective_date, is_current)
    SELECT s.dept_id, s.dept_name, s.division_id, GETDATE(), 1
    FROM t1_department s
    WHERE NOT EXISTS (SELECT 1 FROM t2.dim_department t WHERE t.dept_id = s.dept_id AND t.is_current = 1);
END;
```

**Zero-copy clone refresh** and **T5 views:**

```sql
-- Clone refresh: drop views first, then drop/recreate clones
CREATE TABLE t3.dim_employee_FINAL AS CLONE OF t3.dim_employee;

-- Presentation view with business-friendly names
CREATE VIEW t5.vw_employee AS
SELECT employee_key AS [Employee Key],
    first_name + ' ' + last_name AS [Employee Name], job_title AS [Job Title]
FROM t3.dim_employee_FINAL;
```

**Batch processing** for large datasets (over 1M rows) uses temp tables, configurable batch sizes, and watermark-based iteration with TRY-CATCH error handling and T0 logging.

---

## 6b. Batch Processing with Pagination & Hash Merge

A production pattern for REST API sources where the API does not return total page count and the source `updated_at` timestamp cannot be trusted for SCD2 change detection. Implemented entirely in T-SQL (Warehouse SQL scripts, not notebooks).

### Problem

1. **Unreliable pagination** — the source API does not return `total_pages` or `total_records` in its response. The only signal that you have reached the end is when the payload returns fewer records than the requested page size (or an empty set).
2. **Untrusted timestamps** — the source system's `updated_at` column is unreliable (backdated, missing, or overwritten). You cannot use watermark-based incremental loading to detect changes.

### Solution: Payload-Size Pagination + Hash Merge

**Step 1 — Paginated Ingestion (Data Factory)**

Use a Data Factory pipeline with an `Until` loop that reads pages until the payload is smaller than the page size:

```
PL_T1_Ingest_[Entity]
  |-- Set variable: @page = 1, @hasMore = true
  |-- Until: @hasMore = false
  |   |-- Web activity: GET /api/entity?page=@page&page_size=500
  |   |-- If: length(output.value) < 500
  |   |   |-- Set @hasMore = false
  |   |-- Copy activity: append response to T1 Lakehouse (VARIANT)
  |   |-- Set variable: @page = @page + 1
```

Key design decisions:
- **Page size** — match the source API's maximum (e.g. 500). Larger pages = fewer API calls.
- **Empty set guard** — also check `length(output.value) = 0` for APIs that return exact multiples.
- **Rate limiting** — add a `Wait` activity (e.g. 1 second) between iterations if the API throttles.
- **Idempotency** — truncate the T1 staging table before each full extraction run. The full dataset is re-ingested each time because watermarks are unreliable.

**Step 2 — Hash Column Generation (Warehouse SQL Script)**

Since `updated_at` cannot be trusted, generate a deterministic hash of all business columns to detect actual data changes:

```sql
-- Create a staging view with hash over T1 data
CREATE OR ALTER VIEW t2.vw_staging_entity AS
SELECT
    entity_id,
    col_a,
    col_b,
    col_c,
    HASHBYTES('SHA2_256',
        CONCAT_WS('|',
            ISNULL(CAST(col_a AS NVARCHAR(MAX)), ''),
            ISNULL(CAST(col_b AS NVARCHAR(MAX)), ''),
            ISNULL(CAST(col_c AS NVARCHAR(MAX)), '')
        )
    ) AS row_hash
FROM t1_entity;
```

**Step 3 — Hash Merge SCD2 (Warehouse Stored Procedure)**

Compare hash values instead of individual columns or timestamps:

```sql
CREATE PROCEDURE t2.usp_hash_merge_dim_entity AS
BEGIN
    SET NOCOUNT ON;

    -- Step 1: Expire records where hash has changed
    UPDATE t2.dim_entity
    SET expiry_date = GETDATE(),
        is_current = 0
    WHERE is_current = 1
      AND entity_id IN (
          SELECT s.entity_id
          FROM t2.vw_staging_entity s
          JOIN t2.dim_entity t
            ON s.entity_id = t.entity_id
           AND t.is_current = 1
          WHERE s.row_hash <> t.row_hash
      );

    -- Step 2: Insert new versions (changed + genuinely new)
    INSERT INTO t2.dim_entity (
        entity_id, col_a, col_b, col_c,
        row_hash, effective_date, expiry_date, is_current
    )
    SELECT
        s.entity_id, s.col_a, s.col_b, s.col_c,
        s.row_hash, GETDATE(), NULL, 1
    FROM t2.vw_staging_entity s
    WHERE NOT EXISTS (
        SELECT 1 FROM t2.dim_entity t
        WHERE t.entity_id = s.entity_id
          AND t.is_current = 1
          AND t.row_hash = s.row_hash
    );

    -- Step 3: Log to T0
    INSERT INTO t0.pipeline_log (procedure_name, rows_expired, rows_inserted, executed_at)
    SELECT 'usp_hash_merge_dim_entity', @@ROWCOUNT, @@ROWCOUNT, GETDATE();
END;
```

### Why Hash Merge

| Approach | When to Use | When to Avoid |
|----------|-------------|---------------|
| **Watermark** (`updated_at > @last_run`) | Source timestamps are reliable | Timestamps are backdated or missing |
| **Column-by-column compare** | Few columns, simple types | Many columns (verbose SQL, maintenance burden) |
| **Hash merge** | Untrusted timestamps, many columns | Very high row counts where hash compute is expensive |

Hash merge advantages:
- **Single comparison** — one `WHERE s.row_hash <> t.row_hash` replaces N column comparisons
- **Immune to timestamp manipulation** — detects actual data changes regardless of what `updated_at` says
- **Maintainable** — adding a new column means updating the hash definition in one place (the staging view)

### Why Payload-Size Pagination

| Signal | Reliable? | Notes |
|--------|-----------|-------|
| `total_pages` header | Yes, if provided | Preferred — use `ForEach` with range. Not always available. |
| `next_page` link | Yes, if provided | Follow links until null. Common in well-designed APIs. |
| **Payload size < page_size** | Always works | Universal fallback. Only signal available when API returns no metadata. |
| Empty response body | Always works | Final guard — catch exact multiples of page_size. |

---

## 7. Delta Lake on Fabric

Both the Lakehouse (Spark) and Warehouse (T-SQL) store data as Delta Lake tables backed by Parquet files in OneLake. The aged-care-lakehouse project demonstrates PySpark-based Delta patterns.

**MERGE upsert helper** (from `utils/delta_helpers.py`):

```python
def merge_upsert(spark, target_table: str, source_df, key_columns: list):
    source_df.createOrReplaceTempView("_merge_source")
    key_conds = " AND ".join(f"t.{c} = s.{c}" for c in key_columns)
    spark.sql(
        f"MERGE INTO {target_table} AS t USING _merge_source AS s ON {key_conds} "
        "WHEN MATCHED THEN UPDATE SET * WHEN NOT MATCHED THEN INSERT *"
    )
```

**SCD2 close/insert pattern** (from `utils/scd2_helpers.py`) -- two-step approach: close current rows via MERGE, then append new versions with `valid_from`, `valid_to=null`, `is_current=true`:

```python
# Step 1: Close current rows
spark.sql(f'''MERGE INTO {table} AS t USING _keys AS k
    ON t.{key_column} = k.{key_column} AND t.is_current = true
    WHEN MATCHED THEN UPDATE SET t.valid_to = '{valid_to_ts}', t.is_current = false''')
# Step 2: Insert new versions
df.withColumn("valid_from", F.lit(valid_from_ts)) \
  .withColumn("valid_to", F.lit(None).cast("timestamp")) \
  .withColumn("is_current", F.lit(True)) \
  .write.format("delta").mode("append").saveAsTable(table)
```

**OPTIMIZE and VACUUM:**

```sql
OPTIMIZE raw_department;
VACUUM raw_department RETAIN 168 HOURS;
```

Partitioning by date and Z-ordering on frequently filtered columns improve query performance. Use `DESCRIBE HISTORY` for time travel and auditing.

---

## 8. Eventstreams

Fabric Eventstream captures real-time events and lands them in Raw layer tables. The aged-care-lakehouse project demonstrates this for visit status tracking.

**Flow:**
1. Simulator or application emits events to Eventstream
2. Eventstream writes to Raw table (`raw_streaming_visits`) in the Lakehouse
3. Batch or micro-batch notebook processes Raw to Silver (`silver_streaming_visits`)
4. Analytics layer consumes as `fact_realtime_metrics`

**Simulators for development:** `visit_event_simulator.py` (visit status events with `--rate`, `--max-events`) and `iot_device_simulator.py` (IoT device events). Event schemas defined in `streaming/config/event_schemas.json`.

Streaming ingestion uses **append-only semantics**. Eventstream consumers use offsets managed by the runtime (not custom watermark tables). Batch/API sources continue to use T0 watermark tables.

---

## 9. Direct Lake Semantic Models

Direct Lake is Fabric's high-performance analytics mode. It reads OneLake Parquet files directly into an in-memory cache without importing data.

**Direct Lake vs DirectQuery:**

| Aspect | Direct Lake | DirectQuery |
|--------|-------------|-------------|
| Data source | OneLake Parquet files | SQL endpoint (views, queries) |
| Performance | In-memory cache, very fast | SQL pushdown, real-time |
| Refresh | Automatic (reads latest Parquet) | No refresh needed |
| Best for | T3._FINAL tables (facts, dims) | T5 views, complex aggregations |

**Dual-mode operation:** Fabric handles this automatically. T3._FINAL tables use Direct Lake; T5 views and complex aggregations fall back to DirectQuery with SQL pushdown. No manual configuration required.

**Semantic model setup:**
1. Connect to OneLake (Lakehouse) or Warehouse
2. Select T3._FINAL tables for Direct Lake (primary)
3. T5 views automatically route through DirectQuery
4. Define star schema relationships (many-to-one, single-direction cross-filter)
5. Add DAX measures in the semantic layer

**Aggregation design:** Pre-compute common summaries as T3 aggregation tables (e.g., monthly payroll by department) or as T5 DirectQuery views for complex aggregations needing SQL pushdown.

**OneLake optimization:** Partition large tables by date, Z-order on frequently filtered columns, use zstd compression. Monitor with DAX Studio (server timings, cache hit rate, SE/FE query split).

---

## 10. Azure Supporting Services

**ADLS Gen2 (Azure Data Lake Storage Gen2):** External storage for source files. Data Factory copy activities pull from ADLS Gen2 into T1 Lakehouse. Requires Storage Blob Data Contributor role. OneLake can also create shortcuts to ADLS Gen2 containers for zero-copy access to external data.

**Azure Key Vault:** Stores connection strings, API keys, and secrets used by Data Factory linked services and pipeline parameters. Integrate via Fabric workspace managed identity or service principal.

**Azure Monitor:** Tracks Fabric capacity utilization, pipeline execution metrics, and query performance. Set up alerts for pipeline failures and capacity threshold breaches. Complement with T0 pipeline_log tables for application-level observability.

**Azure DevOps / Azure Pipelines:** Version control for T5 view definitions, stored procedures, and pipeline configurations. CI/CD deployment pipeline pattern:
- Git repo stores SQL scripts and Dataflow definitions
- Azure Pipelines deploys to Dev/Test/Prod workspaces
- Deployment rules in semantic models handle environment mapping
- T5 views are deployed from version-controlled scripts after clone refresh

**Required permissions across services:**
- Workspace Admin or Contributor for Fabric workspace
- SQL Database Contributor for Warehouse operations
- Storage Blob Data Contributor for ADLS Gen2 access
- Power BI Service Administrator for semantic model deployment

---

## Cross-References

- [[Delta Lake Fundamentals]] -- Delta table format, time travel, OPTIMIZE/VACUUM
- [[Data Warehouse Design Patterns]] -- star schema, SCD types, fact table design
- [[Power Query M Language]] -- M language reference for Dataflows Gen2
- [[ETL & ELT Patterns]] -- incremental loading, watermark patterns, CDC
- [[Apache Spark Fundamentals]] -- PySpark patterns used in Fabric notebooks
