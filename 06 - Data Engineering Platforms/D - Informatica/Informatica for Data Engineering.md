# Informatica for Data Engineering

## Platform Overview

Informatica is an enterprise data integration platform with two primary product lines: **PowerCenter** (on-premises ETL engine, established 1993) and **IDMC** (Intelligent Data Management Cloud, the SaaS successor). Most large organisations run both during migration.

**Product family:**
- **PowerCenter** (on-prem) -- legacy ETL, regulated environments needing full infrastructure control
- **IDMC / CDI** (SaaS + Secure Agent) -- cloud-native integration, hybrid connectivity
- **Data Quality / CDQ** -- profiling, standardisation, matching, address validation
- **MDM** -- master data management, golden record creation
- **CDGC** (cloud) -- catalog, lineage, policy management

**When to use Informatica vs alternatives:**
- **Informatica over [[Core dbt Fundamentals|dbt]]** -- when you need extract + load (not just transform), visual mapping design for non-SQL teams, enterprise governance, or pre-built connectors to legacy systems (SAP, Mainframe, Oracle EBS)
- **Informatica over Spark** -- when you want a managed GUI-based platform without custom code, or need out-of-box CDC and SCD handling
- **dbt/Spark over Informatica** -- when your team prefers code-first workflows, you operate exclusively in a cloud warehouse, or you need open-source flexibility and lower licensing costs

### Secure Agent Architecture

The Secure Agent bridges the IDMC cloud control plane and on-premises/private-cloud data sources. It runs behind the firewall, initiating **outbound-only HTTPS** connections to Informatica's cloud -- no inbound firewall ports required.

- Agent groups provide horizontal scaling and high availability
- Each agent bundles its own DTM (Data Transformation Manager) engine
- Connectors are auto-updated from the cloud

---

## PowerCenter Architecture

PowerCenter separates metadata management, execution, and monitoring into distinct services:

- **Domain** -- top-level administrative unit; contains nodes, services, and folders
- **Repository Service** -- manages the repository database (mappings, workflows, metadata)
- **Integration Service** -- runtime engine that executes workflows and sessions
- **Web Services Hub** -- exposes mappings as SOAP/REST services (optional)

### Hierarchy: Mappings → Sessions → Workflows

```
Workflow (WF_DAILY_LOAD)
 ├── Start Task
 ├── Session (S_STG_CUSTOMERS)      ← runs mapping M_STG_CUSTOMERS
 ├── Decision Task                  ← conditional branching
 ├── Session (S_DIM_CUSTOMERS)      ← runs mapping M_DIM_CUSTOMERS
 ├── Command Task                   ← shell commands / SQL
 └── Email Task                     ← success/failure notification
```

- **Mapping** -- the data flow definition: sources, transformations, targets (pure logic, no runtime config)
- **Session** -- a runnable instance of a mapping with connection assignments, performance settings, error handling
- **Workflow** -- orchestrates one or more sessions, commands, decisions, and timers into an execution sequence
- **Worklet** -- a reusable sub-workflow embedded in a parent workflow

### Key Tools

- **Designer** -- build mappings, mapplets, source/target definitions
- **Workflow Manager** -- create sessions, workflows, connections, schedules
- **Workflow Monitor** -- monitor runs, view logs, restart failed sessions
- **Repository Manager** -- manage folders, permissions, deploy objects

---

## Mapping Design

A mapping is a directed graph of sources, transformations, and targets. Data flows through transformation ports (input, output, input/output, variable, local variable).

### Core Transformations

| Transformation | Type | Purpose |
|---------------|------|---------|
| **Source Qualifier** | Active | Generates the SELECT query; allows source filter, join override, SQL override |
| **Expression** | Passive | Row-level calculations, type conversions, string manipulation |
| **Filter** | Active | Removes rows that fail a condition (like a WHERE clause) |
| **Joiner** | Active | Joins two heterogeneous pipelines (Normal, Master Outer, Detail Outer, Full Outer) |
| **Lookup** | Active/Passive | Retrieves values from a table or flat file; connected or unconnected |
| **Aggregator** | Active | GROUP BY operations: SUM, AVG, COUNT, MIN, MAX |
| **Router** | Active | Splits data into multiple groups based on conditions (like a CASE statement with multiple outputs) |
| **Union** | Active | Merges multiple pipelines (UNION ALL) |
| **Normalizer** | Active | Pivots repeating columns into rows (wide to long) |
| **Sequence Generator** | Passive | Generates unique numeric IDs (NEXTVAL/CURRVAL) |
| **Update Strategy** | Active | Flags rows as INSERT (0), UPDATE (1), DELETE (2), or REJECT (3) |
| **Sorter** | Active | Sorts data in-memory by specified ports |
| **Rank** | Active | Returns top/bottom N rows per group |
| **Stored Procedure** | Passive | Invokes a database stored procedure |

### Mapping Variables & Parameters

```
$$PARAM_NAME     -- mapping parameter: value set externally (parameter file, session config)
$PARAM_NAME      -- session parameter: resolved at session level
$$VAR_NAME       -- mapping variable: retains value between runs (via SETVARIABLE)
```

**Parameter file format** (`param_file.txt`):
```
[Global]
$$DB_CONNECTION=PROD_ORACLE
$$BATCH_SIZE=10000

[S_STG_CUSTOMERS.WF_DAILY_LOAD]
$$LAST_EXTRACT_DATE=2026-03-14 00:00:00
```

---

## IDMC / Cloud Data Integration (CDI)

IDMC replaces PowerCenter's desktop tooling with a browser-based platform:

- **Mapping** -- same concept as PowerCenter but designed in a web canvas; supports both ETL and [[Data Ingestion Patterns|ELT pushdown]] modes
- **Mapping Task** -- the runtime wrapper (like a PowerCenter session): assigns connections, sets pushdown mode, configures error handling
- **Taskflow** -- the orchestration layer (like a PowerCenter workflow): chains mapping tasks, human tasks, timers, and fault handlers
- **Synchronisation Task** -- codeless point-and-click replication for simple source-to-target data movement
- **Data Transfer Task** -- bulk file transfer between cloud storage endpoints

### Connectors

Pre-built connectors span the modern data stack:
- **Cloud DW** -- [[SnowPro Advanced Data Engineer (DEA-C02) Complete Study Guide|Snowflake]], BigQuery, Redshift, Azure Synapse, [[Databricks & Delta Lake|Databricks]]
- **Cloud Storage** -- S3, ADLS Gen2, GCS
- **Databases** -- Oracle, SQL Server, PostgreSQL, MySQL, Db2, SAP HANA
- **SaaS** -- Salesforce, Workday, SAP, ServiceNow, NetSuite, HubSpot
- **Files** -- CSV, JSON, XML, Parquet, Avro, ORC
- **Messaging** -- Kafka, Azure Event Hubs, Amazon Kinesis

### Pushdown Optimisation (ELT Mode)

Pushdown optimisation generates SQL that executes directly in the target database engine, avoiding data movement through the Secure Agent.

**Pushdown modes:**
- **None** -- all processing in the Secure Agent DTM (classic ETL)
- **Source** -- push filter/join logic into source SQL
- **Target** -- push transformation logic into target database
- **Full** -- push entire mapping to the target engine (maximum performance)

Transformations that support full pushdown: Expression, Filter, Joiner, Lookup, Aggregator, Union, Sorter. Transformations like Router, Normalizer, and custom Java typically break pushdown.

---

## Data Quality (CDQ)

Informatica's data quality tools run within both PowerCenter (via Data Quality transformations) and IDMC (via Cloud Data Quality).

### Capabilities

- **Profiling** -- column statistics, pattern analysis, value frequency, null rates
- **Standardisation** -- normalise dates, phones, addresses; apply reference tables
- **Matching & Deduplication** -- fuzzy matching (Jaro-Winkler, Soundex, edit distance), identity resolution, survivorship rules
- **Address Validation** -- postal validation (USPS, Royal Mail, Australia Post), geocoding
- **Scorecards** -- rule-based validation tracking completeness, conformity, consistency over time

Typical pipeline: `Source -> Profile -> Standardise -> Parse -> Match -> Merge/Survive -> Scorecard -> Target`

---

## Common ETL Patterns

### SCD Type 1 (Overwrite)

Direct update of dimension attributes without history:
- Lookup dimension by business key
- **Match + changed** -- Update Strategy (`DD_UPDATE`) to target
- **No match** -- Update Strategy (`DD_INSERT`) to target

### SCD Type 2 (Historical Tracking)

Preserves full history with effective dating. See [[SCD Type 2 Patterns]] for warehouse-agnostic approaches.

- Lookup the dimension by business key (`WHERE is_current = 'Y'`)
- **Match + changed** -- expire old row (`DD_UPDATE`: set `end_date`, `is_current='N'`), insert new row (`DD_INSERT`: set `start_date`, `is_current='Y'`)
- **Match + unchanged** -- filter out (no action)
- **No match** -- insert new dimension member (`DD_INSERT`)
- Use Sequence Generator for surrogate keys and `SESSSTARTTIME` for effective dates

### Incremental Extraction with $$LAST_EXTRACT_DATE

```sql
-- Source Qualifier SQL override:
SELECT * FROM orders
WHERE modified_date > TO_DATE('$$LAST_EXTRACT_DATE', 'YYYY-MM-DD HH24:MI:SS')
```

Set `$$LAST_EXTRACT_DATE` as a mapping variable and update it via `SETVARIABLE($$LAST_EXTRACT_DATE, SESSSTARTTIME)` in an Expression transformation. PowerCenter persists the value in the repository between runs.

### Lookup Caching Strategies

- **Static (default)** -- cache built once at session start; use when lookup table is stable
- **Dynamic** -- cache updates as rows are inserted/updated; essential for SCD patterns
- **Persistent** -- cache saved to disk and reused across runs; for large, rarely changing tables
- **Uncached** -- SQL per row; slow but low memory for very large lookups with few hits
- **Shared** -- multiple lookups share one cache to reduce memory

### Error Handling

- **Error tables** -- configure on the session to capture rejected rows with error codes and timestamps
- **Row error logging** -- built-in feature that writes bad rows to `PMERR_*` tables in a designated error log database
- **ERROR() function** -- programmatically reject rows: `IIF(amount < 0, ERROR('Negative amount'), amount)`
- **Fault handlers (IDMC)** -- taskflow-level try/catch that routes failures to notification or retry logic

---

## Performance Tuning

### Session-Level Partitioning

Partitioning parallelises data flow by splitting the pipeline into multiple threads:
- **Round-robin** -- even distribution (default)
- **Key-range** -- partitions based on value ranges of a key column
- **Hash (auto-keys)** -- hashes a key for even distribution; best for Aggregator/Joiner
- **Pass-through** -- no redistribution; each partition processes its own slice
- **Database** -- leverages native database partitioning for source reads

Match partition count to available CPU cores. Over-partitioning adds thread management overhead.

### DTM Buffer & Memory

- **DTM Buffer Size** -- memory for data movement between transformations (default 12 MB; typically 64-256 MB for wide rows or many partitions)
- **Index/data cache** -- per-transformation memory for Aggregator, Joiner, Lookup, Rank, Sorter; size for input cardinality

### Mapping-Level & Pushdown Optimisation

- Minimise connected ports -- remove unused ports to reduce memory and I/O
- Push filters early -- filter at Source Qualifier to reduce rows entering the pipeline
- Use sorted input for Aggregator/Joiner when source is pre-sorted
- Use SQL override in Source Qualifier to push complex joins/filters to the database
- For IDMC pushdown, verify generated SQL via the validation report; common blockers: unsupported functions, Router/Java transformations, cross-database joins

---

## Connectivity

### Connection Types

- **Native** -- built-in high-performance drivers (Oracle, SQL Server, DB2); best throughput
- **ODBC** -- OS-level driver; broadest compatibility but slower; requires DSN config
- **JDBC** -- Java-based; used for some cloud connectors; simpler deployment than ODBC

### Snowflake Integration

- **Snowflake V2 connector (IDMC)** -- bulk load via stage + `COPY INTO`, pushdown optimisation, [[Snowflake & dbt Troubleshooting|Snowflake-specific]] SQL generation
- **Key-pair authentication** supported for service accounts; parameterise warehouse/role for dev/prod switching

### Cloud Storage & API Connectors

- **S3** -- CSV, JSON, Parquet, Avro; wildcard paths; IAM role or access key auth
- **ADLS Gen2** -- service principal or managed identity auth; hierarchical namespace
- **GCS** -- service account key auth; interoperable with BigQuery external tables
- **REST V2** -- OAuth 2.0/API key/basic auth, pagination, JSON/XML response parsing
- **SOAP** -- WSDL-driven; auto-generates ports from operation schemas

---

## Deployment & Promotion

### PowerCenter: Export/Import

Promotion between environments (DEV → QA → PROD) uses XML export/import:

```
pmrep connect -r REPO_DEV -d DOMAIN_DEV -n admin -x password
pmrep objectexport -o mapping -n M_STG_CUSTOMERS -f DEV_FOLDER -u export.xml
pmrep connect -r REPO_PROD -d DOMAIN_PROD -n admin -x password
pmrep objectimport -i export.xml -c import_control.xml
```

**Import control files** resolve naming conflicts (replace, reuse, rename) and remap connections.

### IDMC: Asset Migration

- **Export/import via UI** -- select assets and dependencies, export as `.zip`, import into target org
- **InfaAssetCLI** -- command-line tool for scripted migration; supports selective export by asset type, path, or tag
- **IICS REST API** -- programmatic asset CRUD; integrate with CI/CD pipelines

### CI/CD Patterns

1. Developer builds/tests mappings in the **dev** IDMC org
2. Export assets via InfaAssetCLI, commit JSON/ZIP artefacts to Git
3. CI pipeline (Jenkins, GitHub Actions, Azure DevOps) runs InfaAssetCLI to import into **QA**
4. After QA sign-off, same pipeline promotes to **production** org

### Parameterised Connections

Use connection parameters (`$$SF_ACCOUNT`, `$$SF_WAREHOUSE`, `$$SF_DATABASE`, `$$SF_ROLE`) to avoid hard-coded environment references. Override values per environment in the parameter file or IDMC runtime environment configuration.

---

## Comparison with Modern Tools

| Dimension | Informatica (IDMC) | [[Core dbt Fundamentals|dbt]] | Matillion | ADF |
|-----------|-------------------|-----|-----------|-----|
| **Paradigm** | GUI-first ETL/ELT | Code-first SQL ELT | GUI-first ELT | GUI orchestration + data flow |
| **E & L** | Built-in connectors | Not included (Fivetran/Airbyte) | Built-in | Copy Activity |
| **Transform** | Visual mappings | SQL + Jinja | SQL components | Spark data flows |
| **Data Quality** | Full DQ suite | dbt tests | Basic | Limited |
| **Pricing** | IPU-based license | Free Core / paid Cloud | Seat-based | Consumption-based |
| **Best For** | Complex hybrid estates | Cloud DW analytics eng. | Mid-market ELT | Azure-native pipelines |

### Migration Paths to Cloud-Native

**PowerCenter to IDMC** -- use the built-in migration wizard; custom Java, complex Stored Procedures, and heavily parameterised workflows need manual rework; plan for connection remapping and Secure Agent deployment.

**Informatica to dbt** -- replace E+L with Fivetran/Airbyte, rewrite transformations as SQL models, replace SCD with [[dbt Incremental Loading Patterns|dbt snapshots]], orchestrate with dbt Cloud/Airflow/Dagster.

**Informatica to ADF/Databricks** -- ADF can orchestrate IDMC via API for phased migration; rewrite mappings as Spark notebooks or data flows; Delta Lake MERGE replaces Update Strategy for SCD.

---

## Quick Reference: Expression Language

```
-- String
LTRIM(RTRIM(field))                       IIF(ISNULL(field), 'N/A', field)
SUBSTR(field, start, length)              REG_REPLACE(field, pattern, replace)

-- Date
SESSSTARTTIME                             ADD_TO_DATE(date_field, 'DD', -7)
TO_DATE(string_field, 'YYYY-MM-DD')       DATE_DIFF(date1, date2, 'DD')

-- Control
DECODE(status, 'A','Active', 'I','Inactive', 'Unknown')
ABORT('Critical error: missing key')      -- stop session
ERROR('Row rejected: invalid code')       -- reject row to error table
```

---

## Migration Patterns

Migrating from Informatica to cloud-native alternatives is a common modernisation initiative. This section covers assessment, execution, and coexistence strategies.

### Migration Target: Informatica to dbt + Fivetran/Airbyte

The most common cloud-native replacement separates Informatica's monolithic E+L+T into discrete components:

| Informatica Capability | Cloud-Native Replacement |
|----------------------|-------------------------|
| Extract + Load (connectors) | [[Data Ingestion Patterns\|Fivetran]], Airbyte, or [[Databricks Modern Patterns (2025)#LakeFlow\|LakeFlow Connect]] |
| Transformation (mappings) | [[Core dbt Fundamentals\|dbt]] models (SQL + Jinja) |
| SCD Type 1/2 | [[dbt Incremental Loading Patterns\|dbt snapshots]] and incremental models |
| Data quality (CDQ) | dbt tests, [[Data Validation & Quality Frameworks\|Great Expectations]], Soda |
| Orchestration (workflows) | [[Apache Airflow Core Concepts\|Airflow]], Dagster, or dbt Cloud |
| Catalogue / lineage (CDGC) | [[Data Cataloguing & Discovery\|Unity Catalog]], Atlan, DataHub |

### Migration Target: Informatica to AWS Glue

For AWS-centric organisations, AWS Glue replaces PowerCenter/IDMC:

- **Glue Crawlers** replace Informatica source discovery and schema inference
- **Glue ETL (PySpark)** replaces mapping logic with code-first transformations
- **Glue Data Catalog** replaces CDGC for metadata management
- **Step Functions** replaces workflow orchestration
- CDC via **AWS DMS** replaces Informatica CDC connectors

### Assessment Framework: What to Migrate First

Prioritise migration candidates using a scoring matrix:

| Criterion | Low Priority (1) | High Priority (5) |
|-----------|-----------------|-------------------|
| **Complexity** | Complex Java/stored proc logic | Simple SQL-translatable mappings |
| **Business criticality** | Core regulatory pipeline | Non-critical reporting |
| **Maintenance burden** | Stable, rarely changed | Frequently modified, fragile |
| **Cloud readiness** | On-prem sources only | Sources already in cloud/SaaS |
| **Team capability** | Team knows only Informatica | Team knows SQL/Python/dbt |

**Recommended migration order:**
1. **Quick wins** -- simple source-to-target loads (replace with Fivetran/Airbyte connectors)
2. **SQL-heavy transformations** -- mappings that are predominantly Expression/Filter/Aggregator (rewrite as dbt models)
3. **SCD pipelines** -- replace Update Strategy patterns with dbt snapshots
4. **Complex orchestration** -- multi-workflow chains (migrate to Airflow DAGs)
5. **Data quality** -- CDQ scorecards to dbt tests and Great Expectations suites
6. **Legacy integrations** -- mainframe/SAP connectors (keep in Informatica or use specialist tools)

### Coexistence Patterns

Running old and new systems in parallel during migration:

- **Shadow mode** -- run both Informatica and dbt pipelines, compare outputs in a reconciliation table. Differences trigger alerts before cutover
- **Strangler fig** -- migrate one mapping at a time. Informatica workflow calls dbt via CLI/API for migrated steps; remaining steps stay in Informatica
- **Dual-write** -- Fivetran loads to a new raw schema alongside Informatica's existing loads. dbt reads from the new schema; Informatica continues serving legacy consumers
- **API bridge** -- use IDMC REST API to trigger remaining Informatica jobs from Airflow, maintaining a single orchestration layer during transition

### Common Pitfalls

- **Underestimating mapping variables** -- Informatica's `$$VAR` / `SETVARIABLE` state persistence has no direct dbt equivalent. Implement via dbt run-operations writing to a state table
- **Ignoring pushdown differences** -- Informatica's Expression language is not SQL. Functions like `DECODE`, `IIF`, `REG_REPLACE` need translation to target SQL dialect
- **Skipping reconciliation** -- always run parallel comparison for at least two full data cycles before decommissioning
- **Forgetting parameter files** -- Informatica parameter files encode environment-specific configuration. Map these to dbt profiles, environment variables, or Airflow variables
- **Connector coverage gaps** -- verify Fivetran/Airbyte support for every source before committing. Mainframe, SAP, and legacy ODBC sources may require specialist connectors or middleware
