# Matillion for Data Engineering

> [!summary]
> Matillion is a cloud-native ELT platform that pushes transformations down to the target warehouse. It provides a visual, low-code interface for building data pipelines that execute SQL natively on Snowflake, Databricks, Redshift, and BigQuery.

---

## 1. Platform Overview

### Product Evolution

| Product | Model | Status |
|---|---|---|
| **Matillion ETL** | Single VM/EC2 instance per warehouse | Legacy (still supported) |
| **Matillion Data Productivity Cloud (DPC)** | SaaS with hub-and-agent architecture | Current platform |

Matillion ETL was deployed as a dedicated instance per target warehouse. The Data Productivity Cloud replaced this with a centralised SaaS control plane and lightweight agents.

### Cloud-Native ELT Positioning

Matillion generates SQL and submits it to the warehouse engine rather than processing data in its own memory. This "pushdown" model means performance scales with the warehouse. See [[Data Ingestion Patterns]] for broader ELT context.

### Supported Warehouses

- **Snowflake** — deepest integration, COPY INTO support, variant handling
- **Databricks** (Lakehouse) — Spark SQL pushdown
- **Amazon Redshift** — native Redshift SQL generation
- **Google BigQuery** — BigQuery SQL generation

Each warehouse target has its own component set since the generated SQL is dialect-specific.

### When to Use Matillion

| Scenario | Recommended Tool |
|---|---|
| Visual pipeline development, mixed technical team | **Matillion** |
| SQL-first transformation with testing | **dbt** ([[Core dbt Fundamentals]]) |
| Enterprise integration with on-prem/mainframe | **Informatica** |
| Microsoft-ecosystem orchestration | **Azure Data Factory** |

---

## 2. Architecture

### Matillion ETL (Legacy)

A Matillion instance runs on a VM (EC2/Azure VM/GCE) in the same cloud as the target warehouse. It hosts the web UI, job definitions, scheduler, and execution engine, connecting directly to the warehouse.

### Data Productivity Cloud (Current)

```
[Matillion Cloud Hub]  <-->  [Agent (lightweight VM)]  <-->  [Target Warehouse]
       |                            |
   UI / API / Git                Network access to
   Scheduling / Projects         sources & warehouse
```

- **Hub**: SaaS control plane for projects, pipelines, scheduling, users, and Git
- **Agent**: Lightweight process in the customer's VPC handling warehouse/source connectivity. Data never passes through the hub
- Multiple agents can serve different environments or network zones

### Pushdown Execution Model

1. Matillion compiles the visual pipeline into SQL statements
2. SQL is submitted to the target warehouse
3. The warehouse executes using its own compute
4. Matillion monitors execution and captures metadata/row counts

See [[Snowflake SQL Pipeline Patterns]] for warehouse-side optimisation.

---

## 3. Orchestration Jobs

Matillion has two job types: **orchestration** (control flow, loading, API calls — runs on agent) and **transformation** (SQL pushed to warehouse).

### Core Orchestration Components

| Component | Purpose |
|---|---|
| **Start / End** | Entry and termination points |
| **Run Transformation / Orchestration** | Execute child jobs |
| **If** | Conditional branching on variables or query results |
| **Python Script / Bash Script** | Execute code on the agent |
| **API Query** | HTTP requests to external APIs |
| **SNS Message / SQS Message** | AWS messaging integration |
| **Iterator** | Loop over grid variable rows or query results |
| **Retry** | Retry downstream components with configurable backoff |
| **And / Or** | Logical gates for parallel branch convergence |

### Variables and Grid Variables

**Scalar variables** hold single values, referenced via `${variable_name}`. They can be scoped to job, project, or environment level.

**Grid variables** hold tabular data and pair with the Iterator component for looping. Common pattern: query a metadata table for a list of schemas, store in a grid variable, iterate to run a parameterised job per row.

Environment variables override project variables, enabling dev/test/prod switching (see section 7).

---

## 4. Transformation Jobs

Transformation jobs define a DAG of components compiled into warehouse SQL. The visual canvas shows data flow from source tables through transformations to a target.

### Key Components

| Component | SQL Equivalent |
|---|---|
| **Table Input** | `SELECT * FROM table` |
| **Calculator** | `SELECT col, expression AS new_col` |
| **Filter** | `WHERE` clause |
| **Join** | `JOIN` (inner, left, right, full, cross) |
| **Union** | `UNION / UNION ALL` |
| **Aggregate** | `GROUP BY` with SUM, COUNT, AVG, etc. |
| **Rank** | `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` |
| **Window Functions** | `OVER (PARTITION BY ... ORDER BY ...)` |
| **Pivot / Unpivot** | Rows-to-columns and reverse |
| **Rewrite Table** | `CREATE TABLE AS SELECT` / `INSERT OVERWRITE` |
| **SQL Script** | Raw SQL escape hatch |

### Shared Jobs

Reusable jobs that accept parameters, functioning like dbt macros or stored procedures. Define input/output parameters, call from multiple parent jobs, and centralise common logic (SCD Type 2 merge, audit columns). Compare with [[Core dbt Fundamentals]].

---

## 5. Data Loading

### Connector Categories

- **Databases**: PostgreSQL, MySQL, SQL Server, Oracle, MongoDB
- **Cloud storage**: S3, GCS, ADLS
- **SaaS/API**: Salesforce, HubSpot, Google Analytics, Jira, ServiceNow, Workday
- **File-based**: SFTP, FTP
- **Streaming**: Kafka (via API), SQS

### S3 Load (Snowflake)

Generates `COPY INTO` commands with format options (CSV, JSON, Parquet, Avro, ORC). Supports pattern matching for file selection and handles Snowflake stages transparently.

### API Extract with Pagination

For REST APIs without a dedicated connector: configure endpoint, auth (OAuth/API key/Basic), pagination (offset, cursor, or next-link), map JSON response to flat table, and load to staging. See [[Data Ingestion Patterns]].

### CDC Patterns

- **Watermark columns**: Filter by `last_modified`, stored as a job variable between runs
- **Merge/upsert**: Rewrite Table in merge mode for inserts and updates
- **Full-refresh fallback**: Truncate-and-reload for small dimension tables

Native CDC (Debezium, AWS DMS) is typically handled outside Matillion, with Matillion consuming CDC output from staging.

---

## 6. Snowflake Integration

### COPY INTO Optimisation

S3 Load and Azure Blob Load components generate `COPY INTO` with full format/compression support. Can leverage external or temporary internal stages with parallel file loading.

### Variant and Semi-Structured Data

- **Flatten** component: equivalent to `LATERAL FLATTEN()` for nested JSON/arrays
- **Extract Nested Data**: dot-notation access (`payload:customer.name`)
- Calculator supports `GET_PATH`, `PARSE_JSON`, `ARRAY_SIZE`

### Warehouse, Role, and Schema Management

Each environment specifies warehouse, role, database, and schema. For example, `TRANSFORM_WH_DEV` + `TRANSFORMER_DEV` role in dev vs `TRANSFORM_WH_PROD` + `TRANSFORMER_PROD` in prod. Aligns with Snowflake RBAC best practices. See [[Snowflake SQL Pipeline Patterns]].

---

## 7. Environment Management

### Environment Variables

Environments promote pipelines across stages without changing job logic:

```
Project
  |-- Environment: DEV   (warehouse=TRANSFORM_WH_DEV,  database=ANALYTICS_DEV,  role=TRANSFORMER_DEV)
  |-- Environment: PROD  (warehouse=TRANSFORM_WH_PROD, database=ANALYTICS_PROD, role=TRANSFORMER_PROD)
```

Components reference `${database}`, `${schema}`, etc. — values resolve per environment.

### Git Integration (DPC)

- Link projects to GitHub, GitLab, Bitbucket, or Azure DevOps
- Feature branching, commits, and pull requests for pipeline changes
- Pipeline definitions stored as JSON, enabling diff-based review

### Branching Strategy

| Branch | Purpose | Environment |
|---|---|---|
| `main` | Production-ready pipelines | PROD |
| `develop` | Integration branch | TEST/UAT |
| `feature/*` | Individual changes | DEV |

---

## 8. Scheduling and Triggers

### Built-In Scheduler

Cron-style scheduling per orchestration job. Schedules are environment-aware (schedule in PROD but not DEV). In the DPC, schedules are managed centrally.

### API-Triggered Execution

REST endpoint: `POST /rest/v1/group/name/{group}/project/name/{project}/version/name/{version}/job/name/{job}/run`. Supports variable overrides in the request body. Useful for integration with [[Airflow Orchestration Patterns]] or CI/CD.

### Event-Driven Execution

- **SQS Listener**: Poll queue and trigger on message arrival
- **SNS**: Fan-out notifications to trigger jobs
- **S3 Events + SQS/SNS**: Trigger loading when files land
- **Webhooks** (DPC): HTTP-based triggers

### Job Chaining

The **Run Orchestration** component with wait-for-completion ensures sequential execution across nested orchestration and transformation jobs.

---

## 9. Error Handling and Monitoring

### Component-Level Error Handling

Every orchestration component has **success** (green) and **failure** (red) output routes, enabling granular branching: route failures to SNS notifications, log to audit tables, or trigger retry logic.

### Retry Logic

The Retry component supports max attempts, wait interval, and exponential backoff for transient failures (API rate limits, warehouse timeouts).

### Task History

Records start/end timestamps, duration, row counts, success/failure status, error messages, runtime variable values, and generated SQL statements. Centralised and searchable in the DPC.

### Monitoring and Alerting

| Method | Use Case |
|---|---|
| **SNS on failure route** | AWS-native alerting (email, Lambda, PagerDuty) |
| **Slack webhook** | Team notifications via API Query |
| **Audit table** | Persistent logging for dashboards |
| **Matillion REST API** | Programmatic status queries, external monitoring (Datadog, PagerDuty) |

---

## 10. Platform Comparison

### Matillion vs dbt

| Dimension | Matillion | dbt |
|---|---|---|
| **Interface** | Visual drag-and-drop | SQL + Jinja code |
| **Scope** | Extract + Load + Transform | Transform only |
| **Testing** | Limited | First-class `dbt test` |
| **Team profile** | Mixed technical | SQL-proficient engineers |

Many teams use both: Matillion for loading, dbt for transformation ([[Core dbt Fundamentals]]).

### Matillion vs Informatica

| Dimension | Matillion | Informatica |
|---|---|---|
| **Deployment** | Cloud-native only | Cloud, on-prem, hybrid |
| **Connectors** | ~100+ | 1000+ (mainframe, SAP) |
| **Governance** | Basic | Advanced (catalogue, lineage, MDM) |
| **Complexity** | Simpler, faster to deploy | Enterprise-grade, steeper learning curve |

### Matillion vs Azure Data Factory

| Dimension | Matillion | ADF |
|---|---|---|
| **Ecosystem** | Multi-cloud | Microsoft-centric |
| **Transform** | Pushdown SQL | Data Flows (Spark) or SQL |
| **Cost** | Licence + warehouse | Pay-per-pipeline-run |

ADF suits the Azure/Synapse ecosystem. Matillion is preferred with Snowflake or for multi-cloud portability.

### Migration Considerations

- **Informatica to Matillion**: Map mappings to transformation jobs. Redesign mapplets as shared jobs. Plan for connector gaps (mainframe, SAP may need middleware)
- **Matillion to dbt**: Extract transformation logic into dbt models. Keep Matillion or replace with Fivetran/Airbyte for loading
- **Stored procedures to Matillion**: Decompose monolithic SQL into components. Use SQL Script as a stepping stone

---

## Related Notes

- [[Core dbt Fundamentals]] — SQL-first transformation, often complementary to Matillion
- [[Data Ingestion Patterns]] — Batch and streaming loading patterns
- [[Snowflake SQL Pipeline Patterns]] — Warehouse-side optimisation for pushdown queries
- [[Airflow Orchestration Patterns]] — External orchestration triggering Matillion via API
- [[ELT vs ETL Architecture]] — Architectural context for the pushdown model

---

*Last updated: 2026-03-15*
