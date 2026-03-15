# DP-600 Microsoft Fabric Analytics Engineer Study Guide

## Exam Overview

**Certification:** Microsoft Certified: Fabric Analytics Engineer Associate
**Exam code:** DP-600 — Implementing Analytics Solutions Using Microsoft Fabric
**Target audience:** Analytics engineers who design, build, and deploy enterprise-scale analytics solutions on [[Microsoft Fabric & Azure Data Services]]. Candidates should be proficient in Power BI, T-SQL, Spark, and data modeling.

**Format and scoring:**
- 40-60 questions (multiple-choice, multi-select, drag-and-drop, case studies)
- 120 minutes
- Passing score: 700 / 1000
- Proctored online or at a test center

**Skills measured domains (January 2026):**

| Domain | Weight |
|--------|--------|
| Plan, implement, and maintain a data analytics solution | 25-30% |
| Prepare and transform data | 45-50% |
| Implement and manage semantic models | 25-30% |

> The "Prepare and transform data" domain carries nearly half the exam weight. Prioritize it.

---

## Domain 1: Plan and Implement Analytics (25-30%)

### OneLake
- Fabric's unified storage layer built on top of ADLS Gen2
- Single namespace for all Fabric workloads (lakehouse, warehouse, eventhouse)
- Supports Delta/Parquet open formats — no vendor lock-in
- Use the **OneLake catalog** (data hub) to discover and browse assets across workspaces
- **Shortcuts** provide zero-copy references to external or cross-workspace data

### Lakehouse vs Warehouse Decision

| Factor | Lakehouse | Warehouse |
|--------|-----------|-----------|
| Query language | Spark (PySpark/SQL), T-SQL via endpoint | T-SQL natively |
| Schema flexibility | Schema-on-read, supports [[VARIANT Data Type]] | Schema-on-write, relational |
| Best for | Raw/semi-structured ingestion, data science | Star schema, BI-optimized queries, stored procedures |
| Storage format | Delta tables in OneLake | Managed Delta tables |
| SCD2 / MERGE | Limited | Full T-SQL MERGE support |

### Direct Lake vs DirectQuery

| Aspect | Direct Lake | DirectQuery |
|--------|-------------|-------------|
| Data access | Reads Parquet/Delta from OneLake directly into memory | Pushes SQL to source at query time |
| Performance | Near-Import speed, no scheduled refresh needed | Depends on source query performance |
| Fallback | Automatically falls back to DirectQuery if query exceeds memory or hits unsupported patterns | N/A |
| Best for | Large fact tables with columnar scans | Real-time views, complex SQL logic |

**Composite models** combine two or more storage modes (e.g., Direct Lake + DirectQuery) in one semantic model.

### Capacity Planning
- Fabric capacities are measured in Capacity Units (CUs) — F2, F4, ... F64, F128+
- F64 or higher recommended for production workloads
- Monitor utilization via the **Microsoft Fabric Capacity Metrics** app
- Capacity admins manage capacity assignment; workspace admins manage workspace membership

### Security and Governance
- **Workspace roles:** Admin, Member, Contributor, Viewer
- **Item-level permissions** for sharing individual assets
- **Sensitivity labels** for regulatory classification
- **Endorsement:** Promoted vs Certified to signal data quality trust
- **Deployment pipelines** with autobinding for Dev/Test/Prod lifecycle
- **.pbip** format enables Git-based source control for semantic models

---

## Domain 2: Prepare and Transform Data (45-50%)

### Data Factory Pipelines
- Orchestrate data movement and processing across T0-T5 layers
- **Copy Activity** for bulk ingestion into lakehouse or warehouse
- **Stored Procedure Activity** to call T-SQL logic (e.g., SCD2 MERGE)
- **Dataflow Activity** to invoke Dataflows Gen2 transformations
- Schedule pipelines with per-pipeline time zone settings
- Connect **Office 365 Outlook Activity** on fail for alerting

### Dataflows Gen2
- Code-free, Power Query (M language) transformation engine
- Ideal for joins, enrichments, aggregations, and star schema assembly
- **Query folding** pushes operations to the source for performance
- Append mode for pre-versioned data; Replace mode for reference tables
- Runs in Fabric compute — no external infrastructure needed

### T-SQL Transformations (Warehouse)
- **Views** expose reusable query logic in the presentation layer
- **Table-valued functions** return result sets for composition
- **Stored procedures** encapsulate MERGE logic for [[Dimensional Modelling (Kimball)]] SCD2 patterns
- Surrogate keys generated for dimension tracking
- **Zero-copy clones** provide stable snapshots without data duplication

### Spark Notebooks
- PySpark / Spark SQL for complex transformations and data science
- Lakehouse is the native integration point
- Use for heavy data wrangling that exceeds Power Query capabilities
- Delta Lake ACID transactions, time travel, and schema evolution

### Incremental Refresh
- Load only new/changed records using **watermarks** (timestamps or IDs)
- Configure in semantic model settings or via pipeline parameters
- Deployment pipeline rules can filter data volume per stage (Dev vs Prod)
- Critical for large datasets to minimize refresh time and capacity cost

---

## Domain 3: Model and Serve Data (25-30%)

### Star Schema in Fabric
- Central **fact tables** (measures, foreign keys) surrounded by **dimension tables** (attributes)
- Optimized for analytical queries and Direct Lake performance
- Build in the warehouse T3 layer, expose via T5 views
- See [[Dimensional Modelling (Kimball)]] for design principles

### Relationships
- Define in the semantic model: one-to-many, many-to-one
- Use `USERELATIONSHIP()` in DAX to activate inactive relationships
- Bridge tables handle many-to-many scenarios
- Cross-filter direction impacts query behavior — prefer single-direction unless required

### DAX Measures
- **Aggregation functions:** SUM, AVERAGE, COUNT, DISTINCTCOUNT
- **Iterator (X) functions:** SUMX, AVERAGEX — evaluate row-by-row
- **Filter functions:** CALCULATE, FILTER, ALL, ALLEXCEPT
- **Time intelligence:** TOTALYTD, SAMEPERIODLASTYEAR, DATEADD
- **Variables:** `VAR ... RETURN` pattern for readability and performance
- **RANKX**, **WINDOW** functions for windowing calculations
- **Calculation groups** reduce measure explosion
- **Field parameters** enable dynamic axis switching in reports
- **Dynamic format strings** apply conditional formatting to measures

### Semantic Models
- The metadata layer connecting data to Power BI reports
- Storage modes: Import, DirectQuery, Direct Lake, Composite
- **Large semantic model format** enables models beyond the default size limit
- Connect via **XMLA endpoint** for external tooling (ALM Toolkit, Tabular Editor)
- Use **Impact Analysis** to assess downstream dependencies before changes

### Row-Level Security (RLS)
- DAX filter expressions restrict which rows a user sees
- Defined on the semantic model, enforced at query time
- Test with "View as role" in Power BI Service
- Combine with **Object-Level Security** (OLS) to hide tables/measures and **Column-Level Security** (CLS) to hide columns
- **Dynamic data masking** protects sensitive values without changing the underlying data

### Composite Models
- Mix Direct Lake tables (fast columnar) with DirectQuery views (real-time, complex logic)
- Useful when some tables need Import-speed performance and others need live data
- Aggregation tables can accelerate high-cardinality DirectQuery sources

---

## Domain 4: Monitor and Optimize

### Query Performance
- **V-Order** optimization in Delta tables for columnar read performance
- **Partitioning** by date for large fact tables — enables partition pruning
- **Z-Ordering** clusters data by frequently filtered columns
- Aggregation tables pre-compute common summaries
- DAX performance: prefer simple measures over iterators; use variables; avoid FILTER on large tables

### Capacity Metrics and Monitoring
- **Microsoft Fabric Capacity Metrics** app tracks CU utilization, throttling, and overages
- **Query insights** in the warehouse for slow query identification
- **Refresh history** in semantic model settings for failure diagnosis
- **Pipeline monitoring** via Data Factory run history and T0 logging tables

### Cost Management
- Right-size capacity: start with F64, scale up based on metrics
- Use incremental refresh to reduce compute per cycle
- Zero-copy clones avoid storage duplication
- Pause capacity during off-hours for non-production workloads
- Shortcuts avoid data duplication across workspaces

### Workspace Administration
- Deployment pipelines: Dev -> Test -> Prod with autobinding and deployment rules
- Git integration for version control (.pbip, TMDL files)
- Impact analysis before semantic model changes
- ALM Toolkit for metadata comparison between environments

---

## Key Concepts Quick Reference

| Term | Definition |
|------|-----------|
| **OneLake** | Unified data lake storage for all Fabric workloads; Delta/Parquet format |
| **Direct Lake** | Semantic model mode reading Parquet from OneLake into memory; no refresh needed |
| **DirectQuery** | SQL pushdown mode; automatic fallback from Direct Lake |
| **VARIANT** | Data type for semi-structured data (JSON/XML) in lakehouse ingestion |
| **Shortcut** | Zero-copy reference to data in another location |
| **Zero-copy clone** | Copy-on-write table clone; no storage duplication until modified |
| **Dataflows Gen2** | Cloud-based Power Query (M) transformation service in Fabric |
| **SCD2** | Slowly Changing Dimension Type 2; tracks full history via effective/expiry dates |
| **Star schema** | Fact + dimension tables optimized for analytics |
| **Watermark** | Timestamp/ID tracking last processed record for incremental loads |
| **Query folding** | Pushing transformations to the source system for performance |
| **RLS** | Row-Level Security; DAX filters restricting data by user role |
| **XMLA endpoint** | External connectivity for semantic model management tooling |
| **Composite model** | Semantic model combining multiple storage modes |
| **.pbip** | Power BI project format enabling Git source control |

---

## Implementation Checklist

Condensed from the 10-phase T0-T5 project setup checklist:

- [ ] **Environment** — Create workspace, configure permissions, provision Git repo
- [ ] **Control layer (T0)** — Warehouse with watermark, pipeline log, and error log tables
- [ ] **Ingestion (T1)** — Lakehouse with VARIANT tables, materialized views, Data Factory pipelines
- [ ] **Historical record (T2)** — Warehouse shortcuts to lakehouse, SCD2 MERGE stored procedures, orchestration pipeline
- [ ] **Transformations (T3)** — Dataflows Gen2 for joins, enrichments, star schema assembly
- [ ] **Presentation (T5)** — Zero-copy clones (T3._FINAL), SQL views, clone refresh pipeline
- [ ] **Semantic layer** — Semantic model (Direct Lake + DirectQuery), relationships, DAX measures, RLS
- [ ] **Master orchestration** — End-to-end pipeline with dependencies, error handling, scheduling
- [ ] **Monitoring and security** — Dashboards, alerts, RLS, workspace and data source permissions
- [ ] **Deployment** — Deployment pipeline (Dev/Test/Prod), CI/CD, datasource rules, autobinding
- [ ] **Testing** — Data quality checks, performance baselines, end-to-end validation, recovery procedures

---

## Study Tips

**Recommended approach:**
1. Read the official [DP-600 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-600) and map each bullet to your knowledge gaps
2. Complete the Microsoft Learn paths in order — they align with the exam domains
3. Build a hands-on Fabric project (even a small lakehouse-to-report pipeline exercises all domains)
4. Take practice exams under timed conditions, then review every wrong answer

**Practice exam resources:**
- 246 practice questions available in the data-engineering-wiki project at `/home/jhark/workspace/data-engineering-wiki/fabric/practice-exams/`:
  - `azure-fabric-practice-exam.md` — 150 questions covering 100% of DP-600 skills
  - `microsoft-learn-dp600-assessment.md` — 96 unique questions + 10 gap-coverage questions
- Official [Microsoft Learn Practice Assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/dp-600/practice/assessment)

**Microsoft Learn paths:**
- [Get started with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/get-started-fabric/)
- [Implement a data warehouse with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/work-with-data-warehouses-using-microsoft-fabric/)
- [Implement a lakehouse with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/implement-lakehouse-microsoft-fabric/)
- [Implement Real-Time Intelligence with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/explore-real-time-analytics-microsoft-fabric/)

**Hands-on labs:**
- Microsoft Fabric trial (60 days free) — build a lakehouse, warehouse, and semantic model
- Practice creating deployment pipelines with Git integration
- Experiment with Direct Lake vs DirectQuery mode switching

---

## Common Exam Traps

### Data Factory vs Dataflows Gen2
- **Data Factory pipelines** orchestrate and move data (Copy Activity, Stored Procedure Activity)
- **Dataflows Gen2** transform data using Power Query M (code-free)
- They are complementary: a pipeline can invoke a dataflow as an activity
- Trap: choosing Data Factory when the question asks for "code-free transformations" — the answer is Dataflows Gen2

### Direct Lake Modes
- Direct Lake reads Delta/Parquet from OneLake; it does NOT require scheduled refresh
- If a query exceeds memory or hits unsupported patterns, it **automatically falls back to DirectQuery**
- Direct Lake on OneLake vs Direct Lake on SQL endpoints — know when each applies
- Trap: assuming Direct Lake always stays in Direct Lake mode — fallback behavior is testable

### Lakehouse vs Warehouse Choice
- Lakehouse: Spark-first, semi-structured data, schema-on-read, VARIANT support
- Warehouse: T-SQL-first, relational schema, stored procedures, MERGE operations
- Eventhouse: streaming/event data with KQL — do not confuse with the other two
- Trap: choosing warehouse for semi-structured ingestion or lakehouse for SCD2 MERGE operations

### VARIANT Usage
- Used in lakehouse T1 layer for schema-agnostic ingestion of JSON/XML
- Stores semi-structured data without pre-defined schema
- Flattened via materialized views for downstream consumption
- Trap: assuming VARIANT is available in the warehouse — it is a lakehouse data type

---

*See also: [[Microsoft Fabric & Azure Data Services]] | [[Dimensional Modelling (Kimball)]] | [[Data Warehouse Design Patterns]]*
