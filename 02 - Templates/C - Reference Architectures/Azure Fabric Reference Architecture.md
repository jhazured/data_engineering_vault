---
tags: [template, reference-architecture, azure, fabric, data-warehouse]
---

# Azure Fabric Reference Architecture

> A reusable reference architecture template for enterprise data platforms on [[Microsoft Fabric & Azure Data Services|Microsoft Fabric]], based on the T0-T5 layered pattern. Adapt the domain-specific examples to your own business context.

---

## 1. Architecture Overview

The T0-T5 pattern separates an enterprise data warehouse into six layers with clear ownership of concerns. All layers share [[Microsoft Fabric & Azure Data Services#1. Microsoft Fabric Overview|OneLake]] as the unified storage substrate, with data persisted as Delta/Parquet regardless of the compute engine used.

```
T0: Control Layer         (Warehouse -- T-SQL tables)
  |
T1: Raw Landing           (Lakehouse -- VARIANT tables via Data Factory)
  | (shortcuts)
T2: Historical Record     (Warehouse -- SCD2 MERGE stored procedures)
  | (Dataflows Gen2)
T3: Transformations       (Warehouse -- star schema via Power Query M)
  | (zero-copy clone)
T3._FINAL: Validated      (Warehouse -- clone tables for semantic isolation)
  |
T5: Presentation          (Warehouse -- SQL views only)
  |
Semantic Layer            (Direct Lake on OneLake + DirectQuery fallback)
  |
Power BI Reports
```

### Design Principles

- **Single source of truth** -- OneLake stores every artefact as Delta/Parquet. No data leaves the platform.
- **Engine-appropriate processing** -- Spark (Lakehouse) for schema-agnostic ingestion; T-SQL (Warehouse) for relational operations; Power Query M (Dataflows Gen2) for business logic.
- **Immutability by default** -- raw data is preserved in T1, historised in T2, and cloned before presentation. Reprocessing is always possible.
- **Semantic isolation** -- the reporting layer reads from cloned snapshots (T3._FINAL), never from in-flight pipeline tables.

---

## 2. Project Structure

Organise Fabric items into workspaces aligned with environments. Each workspace contains the full set of artefacts for one environment.

### Workspace Layout

```
[Project]-Dev
  |-- LH_T1_RAW                     # Lakehouse -- raw landing zone
  |-- WH_MAIN                       # Warehouse -- T0, T2, T3, T5 schemas
  |-- DF_T3_*                       # Dataflows Gen2 -- transformation logic
  |-- PL_MASTER_*                   # Data Factory pipelines
  |-- SM_[Project]                   # Semantic model (Direct Lake)
  |-- RPT_*                         # Power BI reports

[Project]-Test
  |-- (same structure)

[Project]-Prod
  |-- (same structure)
```

### Source Control Structure

```
repo/
  pipelines/
    PL_MASTER_[Domain].json
    PL_T1_Master_Ingest.json
    PL_T2_Process_SCD2.json
    PL_T3_Transform.json
    PL_T5_Clone_Refresh.json
  warehouse/
    schemas/
      t0_create_schema.sql
      t2_create_schema.sql
      t3_create_schema.sql
      t5_create_schema.sql
    tables/
      t0.watermark.sql
      t0.pipeline_log.sql
      t2.dim_*.sql
      t2.fact_*.sql
    procedures/
      t2.usp_merge_dim_*.sql
      t2.usp_merge_fact_*.sql
      t3.usp_refresh_final_clones.sql
    views/
      t5.vw_*.sql
  lakehouse/
    tables/
      raw_*.sql
    materialized_views/
      mv_*.sql
  dataflows/
    DF_T3_*.json
  tests/
    validation_queries/
  docs/
    architecture_decision_records/
```

---

## 3. Layer Responsibilities

### T0 -- Control Layer

**Purpose:** Pipeline orchestration metadata. Tracks what has been processed, when, and whether it succeeded.

**Key tables:**

| Table | Purpose |
|-------|---------|
| `t0.watermark` | Stores last-processed timestamp per source entity |
| `t0.pipeline_log` | Records pipeline execution: procedure name, row counts, timestamps, status |
| `t0.config` | Optional runtime configuration (batch sizes, feature flags) |

**Implementation:** T-SQL tables in the Warehouse. Updated by stored procedures at the end of each pipeline step.

### T1 -- Raw Landing

**Purpose:** Schema-agnostic ingestion. Land source data with minimal transformation.

**Engine:** Lakehouse (Spark).

**Pattern:** Use VARIANT columns to absorb any JSON structure without schema definition.

```sql
CREATE TABLE raw_[entity] (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_file STRING,
    payload     VARIANT
);
```

**Rules:**
- Data is transient -- truncate after confirmed T2 processing.
- Always create a materialised view to flatten VARIANT data.
- Warehouse accesses T1 via shortcuts to materialised views, never raw VARIANT tables.
- For APIs without reliable pagination metadata, use payload-size pagination (loop until response size < page size).

### T2 -- Historical Record

**Purpose:** Slowly Changing Dimension Type 2 (SCD2) historisation.

**Engine:** Warehouse (T-SQL).

**Pattern:** MERGE stored procedures that expire changed records and insert new versions.

**Standard columns:**

| Column | Type | Purpose |
|--------|------|---------|
| `[entity]_key` | INT IDENTITY | Surrogate key |
| `[business_key]` | VARCHAR | Natural key from source |
| `effective_date` | DATETIME | Row version start |
| `expiry_date` | DATETIME NULL | Row version end (NULL = current) |
| `is_current` | BIT | Active record flag |
| `row_hash` | VARBINARY(32) | Optional -- for hash-based change detection |

**Change detection strategies:**
- **Watermark** -- when source timestamps are reliable. Filter `WHERE updated_at > @last_watermark`.
- **Hash merge** -- when source timestamps are unreliable. Compute `HASHBYTES('SHA2_256', CONCAT_WS('|', ...))` over business columns and compare hashes.
- **Column compare** -- when few columns exist. Compare each column individually in the MERGE condition.

### T3 -- Transformations

**Purpose:** Business logic, conformance, and star schema construction.

**Engine:** Dataflows Gen2 (Power Query M) for transformation logic; Warehouse for storage.

**Sub-layers:**

| Sub-Layer | Purpose | Example |
|-----------|---------|---------|
| `t3.ref_*` | Reference/lookup tables | `t3.ref_region`, `t3.ref_category` |
| `t3.[entity]_01` | Base transformations | Column selection, type casting, renaming |
| `t3.[entity]_02` | Enrichment joins | Join with reference tables, add derived columns |
| `t3.agg_*` | Pre-computed aggregations | Monthly summaries, running totals |

**Write modes:**
- Transformation tables: **Append** (data already versioned in T2).
- Reference tables: **Replace** (full refresh of lookups).

### T3._FINAL -- Validated Snapshots

**Purpose:** Isolate the semantic layer from pipeline failures. If a T3 refresh fails mid-way, reports continue reading the last successful snapshot.

**Pattern:** Zero-copy clones, drop-and-recreate after successful T3 completion.

```sql
-- Procedure: t3.usp_refresh_final_clones
-- 1. Drop dependent T5 views
-- 2. Drop existing clone tables
-- 3. Recreate clones from T3 tables
-- 4. Recreate T5 views
```

### T5 -- Presentation

**Purpose:** Business-friendly interface for reporting consumers.

**Rules:**
- Views only -- no base tables.
- Business-friendly column names (e.g., `[Employee Name]`, `[Total Revenue]`).
- Row-level security (RLS) filters applied here.
- Version-controlled SQL scripts deployed via CI/CD.

```sql
CREATE VIEW t5.vw_[entity] AS
SELECT
    [surrogate_key]   AS [Entity Key],
    [business_name]   AS [Entity Name],
    [measure_column]  AS [Measure Label]
FROM t3.[entity]_FINAL;
```

---

## 4. Naming Conventions

### Fabric Items

| Item Type | Convention | Example |
|-----------|-----------|---------|
| Lakehouse | `LH_T1_RAW` | `LH_T1_RAW` |
| Warehouse | `WH_MAIN` | `WH_MAIN` |
| Pipeline | `PL_[SCOPE]_[Action]` | `PL_MASTER_Sales`, `PL_T1_Master_Ingest` |
| Dataflow | `DF_T3_[Entity]` | `DF_T3_Employee_Enrichment` |
| Semantic Model | `SM_[Project]` | `SM_SalesAnalytics` |
| Report | `RPT_[Name]` | `RPT_Monthly_Revenue` |

### Database Objects

| Object Type | Convention | Example |
|-------------|-----------|---------|
| Schema | `t0`, `t2`, `t3`, `t5` | `t2` |
| Dimension | `t2.dim_[entity]` | `t2.dim_customer` |
| Fact | `t2.fact_[event]` | `t2.fact_order` |
| Reference | `t3.ref_[entity]` | `t3.ref_region` |
| Clone | `t3.[entity]_FINAL` | `t3.dim_customer_FINAL` |
| View | `t5.vw_[entity]` | `t5.vw_customer` |
| Procedure | `t[n].usp_[action]_[entity]` | `t2.usp_merge_dim_customer` |
| Watermark | `t0.watermark` | `t0.watermark` |
| Log | `t0.pipeline_log` | `t0.pipeline_log` |

### Column Naming

- Snake_case for all database columns: `customer_id`, `effective_date`.
- Business-friendly aliases in T5 views only: `[Customer Name]`.
- Surrogate keys: `[entity]_key` (e.g., `customer_key`).
- Business keys: use source system name (e.g., `customer_id`, `order_number`).
- Timestamps: `created_at`, `updated_at`, `effective_date`, `expiry_date`.

---

## 5. Pipeline Orchestration

### Master Pipeline Pattern

```
PL_MASTER_[Domain]
  |-- PL_T1_Master_Ingest         (Data Factory copy activities)
  |-- PL_T2_Process_SCD2          (execute T-SQL stored procedures)
  |-- PL_T3_Transform             (trigger Dataflows Gen2)
  |-- PL_T5_Clone_Refresh         (execute clone refresh procedure)
```

Each step depends on the prior step succeeding. Independent entities within a step run in parallel.

### Watermark-Based Incremental Loading

1. **Lookup** watermark from `t0.watermark` (last processed timestamp for entity).
2. **Filter** source data: `WHERE updated_at > @watermark`.
3. **Copy** to T1 Lakehouse (append mode).
4. **Process** through T2 stored procedures.
5. **Update** watermark only after confirmed T2 success.
6. **Truncate** T1 staging data after watermark update.

### Full-Refresh Loading (Unreliable Sources)

When source timestamps cannot be trusted:

1. **Truncate** T1 staging table before extraction.
2. **Extract** full dataset from source (paginated if API).
3. **Hash** all business columns in a staging view.
4. **Merge** using hash comparison instead of timestamp watermark.
5. **Log** to `t0.pipeline_log`.

### Error Handling

- **Retry policy:** Exponential backoff at the activity level (3 retries, 30s/60s/120s).
- **Logging:** Every pipeline execution writes to `t0.pipeline_log` with procedure name, row counts, and execution timestamp.
- **Truncation guard:** T1 data is only truncated after confirmed T2 success, followed by watermark update.
- **Clone isolation:** T3._FINAL clones are only refreshed after all T3 Dataflows complete successfully.

---

## 6. Capacity and Environment Planning

### Capacity Sizing

| Environment | Recommended SKU | Use Case |
|-------------|----------------|----------|
| Development | F2 or F4 | Individual developer work, unit testing |
| Test/UAT | F8 (16 CU) | Integration testing, performance validation |
| Production | F64+ | Full workloads, concurrent users, scheduled refreshes |

### Environment Promotion

```
Dev Workspace  -->  Test Workspace  -->  Prod Workspace
     |                    |                    |
  Feature branch      Release branch      Main branch
     |                    |                    |
  Manual deploy      CI gate deploy      CI/CD deploy
```

**Deployment artefacts:**
- SQL scripts (schemas, tables, procedures, views) -- deployed via Azure Pipelines.
- Pipeline definitions -- exported/imported as JSON.
- Dataflow definitions -- exported/imported via Fabric APIs.
- Semantic model -- deployment rules handle environment-specific connection remapping.

---

## 7. Deployment Checklist

### Pre-Deployment

- [ ] All SQL scripts are version-controlled and peer-reviewed.
- [ ] T5 views use business-friendly names and include RLS filters.
- [ ] Watermark table has entries for all source entities.
- [ ] Pipeline error handling configured (retry, logging).
- [ ] Capacity SKU matches expected workload.

### Post-Deployment

- [ ] Run master pipeline end-to-end in the target environment.
- [ ] Verify T0 watermark updates after successful processing.
- [ ] Verify T3._FINAL clones are populated.
- [ ] Verify T5 views return expected data.
- [ ] Verify semantic model connects and Direct Lake mode is active.
- [ ] Verify Power BI reports render correctly.
- [ ] Set up Azure Monitor alerts for pipeline failures and capacity thresholds.

---

## 8. Semantic Layer Configuration

### Direct Lake Setup

1. Connect the semantic model to the Warehouse (or Lakehouse).
2. Select T3._FINAL tables as the primary source -- these use Direct Lake mode (in-memory Parquet reads).
3. T5 views automatically fall back to DirectQuery with SQL pushdown.
4. Define star schema relationships: many-to-one, single-direction cross-filter.
5. Add DAX measures in the semantic layer for calculated metrics.

### Performance Optimisation

- Partition large fact tables by date.
- Z-order on frequently filtered columns.
- Use zstd compression for Parquet files.
- Pre-compute common aggregations as T3 aggregation tables.
- Monitor with DAX Studio: cache hit rate, SE/FE query split, server timings.

---

## 9. Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Correct Approach |
|-------------|-------------|------------------|
| Business logic in Data Factory | Pipelines become untestable and opaque | Use Dataflows Gen2 (Power Query M) for all T3 logic |
| Skipping T3._FINAL clones | Pipeline failures corrupt live reports | Always clone before exposing to the semantic layer |
| Base tables in T5 | Breaks separation of concerns; ungoverned writes | T5 is views only -- never base tables |
| Hardcoded connection strings | Blocks environment promotion | Use Key Vault and workspace-level settings |
| VARIANT tables without materialised views | Downstream queries fail on VARIANT type mismatches | Always flatten via materialised view before shortcutting |
| Truncating T1 before confirming T2 | Data loss if T2 fails | Truncate only after watermark update confirms success |

---

## Cross-References

- [[Microsoft Fabric & Azure Data Services]] -- detailed Fabric patterns and code examples
- [[Data Warehouse Design Patterns]] -- star schema, SCD types, fact table design
- [[ETL & ELT Patterns]] -- incremental loading, watermark patterns, CDC
- [[Delta Lake Fundamentals]] -- Delta table format, time travel, OPTIMIZE/VACUUM
- [[Power Query M Language]] -- M language reference for Dataflows Gen2
- [[CI/CD for Data Pipelines]] -- deployment automation patterns
