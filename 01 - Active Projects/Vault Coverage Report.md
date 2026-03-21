# Vault Coverage Report

> Updated 2026-03-21 — 121 markdown files | ~79,400 lines

---

## At a Glance

| Area | Files | Lines | Verdict |
|------|------:|------:|---------|
| 01 Active Projects | 1 | ~372 | Meta (this report) |
| 02 Templates | 3 | ~1,480 | **Strong** — ETL templates + REST API pagination, common patterns (12), reference architecture |
| 03 Cloud Platforms | 4 | ~2,190 | **Strong** — AWS, Azure/Fabric (incl. ADF + hash merge + orchestration + audit), GCP (incl. BigQuery ETL framework) |
| 04 Data Engineering | 23 | ~8,180 | **Excellent** — lifecycle, ingestion (3), storage (incl. Hadoop), transformation (data contracts, data mesh, SCD2 incl. T-SQL MERGE), testing (2), monitoring (3 incl. Fabric T0-T4 views), security (4 incl. IAM + compliance), cataloguing |
| 05 Data Streaming | 5 | ~1,755 | **Strong** — Kafka (incl. Schema Registry + exactly-once), stream theory, event-driven, Snowflake Streams, Flink & Kinesis |
| 06 Data Engineering Platforms | 16 | ~6,132 | **Excellent** — dbt (5), Snowflake (4), Databricks (3), Informatica, Matillion, Dataiku, DuckDB, Fivetran |
| 07 Programming Languages | 36 | ~31,335 | **Deep** — PySpark (26), Python (6), SQL (8 incl. LATERAL FLATTEN + QUALIFY), Bash (1, expanded) |
| 08 DevOps & Orchestration | 12 | ~5,330 | **Excellent** — Docker (2), K8s, Terraform, Ansible, Jenkins, CI/CD (3), MCP, Airflow (2), Dagster & Prefect |
| 09 Data Modelling | 6 | ~2,490 | **Strong** — Kimball, advanced dimensional patterns, Data Vault 2.0, star schema + Fabric load patterns + OLS/RLS, DFD, sequence diagrams |
| 10 Protocols | 4 | ~2,031 | **Strong** — REST, SOAP, SFTP (comprehensive), gRPC & GraphQL |
| 11 Learning Resources | 9 | ~17,962 | **Excellent** — 6 interview/cert guides + DP-600 + best practices + cheat sheets |

---

## Coverage Heatmap

```
██████████ PySpark              Excellent — 26 files incl. Spark architecture, Delta Lake, cloud integration
██████████ SCD2 / Transformation Complete — dbt snapshots + T-SQL MERGE (Fabric), hash detection, soft deletes
█████████░ Snowflake            Excellent — RBAC, cost monitoring, SnowPro guide, Cortex, SQL pipelines
█████████░ dbt                  Excellent — 5 notes: fundamentals, incremental, macros (expanded), tags, advanced/cost
█████████░ Interview/Cert Prep  Excellent — 6 guides (Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks) + DP-600
█████████░ Data Engineering     Excellent — 23 notes: ingestion, storage, transformation, testing, monitoring, security, compliance, cataloguing
█████████░ DevOps/Orchestration Excellent — Docker, K8s, Terraform, Ansible, Jenkins, CI/CD (3), MCP, Airflow (2), Dagster & Prefect
█████████░ Monitoring           Excellent — observability, cost, reliability, SLOs, Fabric T0-T4 monitoring views
████████░░ Databricks           Strong — 3 notes: platform, modern patterns (2025), exam guide
████████░░ Data Streaming       Strong — Kafka (+ Schema Registry + exactly-once), Snowflake Streams, Flink & Kinesis
████████░░ Data Modelling       Strong — Kimball, advanced dimensional, Data Vault 2.0, star schema + Fabric load patterns
████████░░ SQL                  Strong — 8 notes incl. LATERAL FLATTEN, QUALIFY, JSON, execution plans
████████░░ ETL Templates        Strong — YAML config, pipeline archetypes, REST API pagination (3 load strategies)
███████░░░ Cloud Platforms      Solid — AWS, Azure/Fabric + orchestration + audit logging, GCP + BigQuery ETL
███████░░░ Protocols            Solid — REST, SOAP, SFTP (comprehensive), gRPC & GraphQL
███████░░░ Bash                 Solid — deployment patterns + scripting fundamentals (1,487 lines)
██████░░░░ Python (non-Spark)   Good — 6 notes including core patterns, pandas/Polars, Streamlit
██████░░░░ Cheat Sheets         Good — per-tool references for dbt, Snowflake, Docker, Terraform, Git
█████░░░░░ Informatica          Adequate — comprehensive reference guide
█████░░░░░ Matillion            Adequate — comprehensive reference guide
█████░░░░░ Dataiku              Adequate — comprehensive reference guide
█████░░░░░ DuckDB              Adequate — embedded OLAP, Python/dbt integration, CI/CD
```

---

## Project Tree

```
data_engineering_vault/
├── 01 - Active Projects/
│   └── Vault Coverage Report.md
├── 02 - Templates/
│   ├── A - Common Patterns/
│   │   └── Common Data Engineering Patterns.md
│   ├── B - ETL vs ELT Workflows/
│   │   └── ETL Pipeline Templates & Patterns.md
│   └── C - Reference Architectures/
│       └── AWS Snowflake dbt Reference Architecture.md
├── 03 - Cloud Platforms/
│   ├── A - AWS/
│   │   └── AWS Data Services for Data Engineering.md
│   ├── B - Azure/
│   │   ├── Azure Data Factory Project Patterns.md
│   │   └── Microsoft Fabric & Azure Data Services.md
│   └── C - GCP/
│       └── GCP Data Services for Data Engineering.md
├── 04 - Data Engineering/
│   ├── A - Ingestion/
│   │   ├── Data Ingestion Patterns.md
│   │   ├── Document Ingestion & Chunking.md
│   │   └── Incremental Loading Strategies.md
│   ├── B - Query & Analysis/
│   │   ├── RAG Patterns.md
│   │   └── Vector Embeddings & Semantic Search.md
│   ├── C - Storage and Warehousing/
│   │   ├── Distributed Systems Fundamentals.md
│   │   ├── Hadoop & MapReduce Fundamentals.md
│   │   ├── Multi-Tier Data Architecture.md
│   │   └── Open Table Formats - Iceberg Hudi & Delta.md
│   ├── D - Transformation/
│   │   ├── Data Contracts & Schema Enforcement.md
│   │   ├── Data Mesh & Domain-Driven Data.md
│   │   └── SCD Type 2 Patterns.md
│   ├── E - Testing & Quality Assurance/
│   │   ├── Data Validation & Quality Frameworks.md
│   │   └── dbt Testing & Data Quality.md
│   ├── F - Monitoring & Observability/
│   │   ├── Pipeline Observability & Monitoring.md
│   │   └── Snowflake Cost Monitoring.md
│   ├── G - Security & Governance/
│   │   ├── Snowflake RBAC & Data Security.md
│   │   └── Trust Stores & Certificate Management.md
│   ├── H - Data Cataloguing/
│   │   └── Data Cataloguing & Discovery.md
│   └── I - Lifecycle/
│       └── Data Engineering Lifecycle.md
├── 05 - Data Streaming/
│   ├── A - Publish-Subscribe/
│   │   └── Stream Processing Theory.md
│   ├── B - Apache Kafka/
│   │   └── Apache Kafka Fundamentals.md
│   └── C - Event-Driven Architecture/
│       └── Event-Driven Architecture.md
├── 06 - Data Engineering Platforms/
│   ├── A - dbt/
│   │   ├── Core dbt Fundamentals.md
│   │   ├── dbt Advanced Patterns & Cost Optimisation.md
│   │   ├── dbt Incremental Loading Patterns.md
│   │   ├── dbt Macro Patterns.md
│   │   └── dbt Tag & Execution Strategy.md
│   ├── B - Snowflake/
│   │   ├── SnowPro Advanced Data Engineer (DEA-C02) Complete Study Guide.md
│   │   ├── Snowflake & dbt Troubleshooting.md
│   │   ├── Snowflake Cortex AI.md
│   │   └── Snowflake Native dbt Workspace.md
│   ├── C - Databricks/
│   │   ├── Databricks & Delta Lake.md
│   │   └── Databricks Modern Patterns (2025).md
│   ├── D - Informatica/
│   │   └── Informatica for Data Engineering.md
│   ├── E - Matillion/
│   │   └── Matillion for Data Engineering.md
│   ├── F - Dataiku/
│   │   └── Dataiku for Data Engineering.md
│   └── G - DuckDB/
│       └── DuckDB for Data Engineering.md
├── 07 - Programming Languages/
│   ├── A - Bash/
│   │   └── Bash Deployment Patterns.md
│   ├── B - PySpark/
│   │   ├── 01 - Core Concepts/ (2 files: Spark Architecture, PySpark Core)
│   │   ├── 02 - Data Operations/ (1 file)
│   │   ├── 03 - Window Functions/ (1 file)
│   │   ├── 04 - Performance Optimization/ (1 file)
│   │   ├── 05 - Advanced Features/ (2 files: Advanced Features, Delta Lake)
│   │   ├── 06 - Streaming/ (1 file)
│   │   ├── 07 - Testing & Quality/ (6 files)
│   │   ├── 09 - Production Patterns/ (1 file)
│   │   ├── 10 - Cloud Integration/ (1 file)
│   │   └── 12 - Reference & Utilities/ (5 files: MOC, Quick Ref, Error Codes, Troubleshooting, Functions)
│   ├── C - Python/
│   │   ├── pandas & Polars for Data Engineering.md
│   │   ├── Python Core Patterns for Data Engineering.md
│   │   ├── Python Data Generation Patterns.md
│   │   ├── Python Testing with pytest.md
│   │   ├── Streamlit Data Apps.md
│   │   └── Markdown to Notebook Conversion.md
│   └── D - SQL/
│       ├── A - Data Aggregation & Filtering/ (1 file)
│       ├── B - CTEs/ (2 files)
│       ├── C - Query Optimization/ (1 file)
│       ├── D - Window Functions/ (1 file)
│       ├── E - Performance Tuning/ (1 file)
│       ├── F - Stored Procedures/ (1 file)
│       └── Snowflake SQL Pipeline Patterns.md
├── 08 - DevOps & Orchestration/
│   ├── A - Ansible/ (1 file)
│   ├── B - Docker/ (2 files: Container Patterns, Kubernetes)
│   ├── C - Jenkins/ (1 file)
│   ├── D - Terraform/ (1 file)
│   ├── E - API Management/ (1 file: MCP)
│   ├── F - CI-CD Patterns/ (3 files: GitHub Actions, GitLab CI, Snowflake Deploy)
│   └── G - Orchestration/ (2 files: Airflow Overview, Airflow Deep Dive)
├── 09 - Data Modelling/
│   ├── A - Entity Relationship Diagrams/ (3 files: Kimball, Data Vault 2.0, Star Schema)
│   ├── B - Data Flow Diagrams/ (1 file)
│   └── C - Sequence Diagrams/ (1 file)
├── 10 - Protocols/
│   ├── A - REST/ (1 file)
│   ├── B - SOAP/ (1 file)
│   ├── C - SFTP/ (1 file: GoAnywhere MFT)
│   └── D - gRPC & GraphQL/ (1 file)
└── 11 - Learning Resources/
    ├── A - Interview Guides/ (7 files: Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks, DP-600)
    ├── B - Best Practices/ (1 file)
    └── C - Cheat Sheets/ (1 file)
```

---

## Section Ratings

| Section | Rating | Strengths | Gaps / Notes |
|---------|:------:|-----------|--------------|
| **1. Templates & Patterns** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Common Patterns | 8/10 | 12 reusable DE patterns with code examples, pseudo-code, and decision matrix | Missing: multi-tenancy, observability-as-code patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;B - ETL vs ELT Workflows | 8/10 | Pipeline templates, YAML config patterns, decision framework, REST API pagination (OData $top/$skip/$filter, 3 load strategies) | Comprehensive after Fabric API ingestion patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Reference Architectures | 7/10 | AWS/Snowflake/dbt end-to-end with Terraform, Lambda, Step Functions | AWS-only; missing Azure/GCP reference architectures |
| **2. Cloud Platforms** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - AWS | 7/10 | Comprehensive service overview, exam-aligned | No hands-on project pattern; missing FinOps, VPC/security |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Azure | 9/10 | Fabric T0-T5, ADF project patterns, hash merge SCD2, pagination, metadata-driven orchestration (control table, parent/child pipelines, audit logging, SP routing) | Most complete cloud section; deepened with production pipeline patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;C - GCP | 8/10 | Service overview, BigQuery ETL framework, Cloud SQL, GCS extraction, transformations | Now includes practical project patterns |
| **3. Data Engineering Core** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Ingestion | 8/10 | 3 notes covering batch, incremental, document ingestion | CDC could be expanded; missing Fivetran/Airbyte specifics |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Query & Analysis | 6/10 | RAG patterns, vector embeddings | Niche (Snowflake Cortex-specific); not core DE |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Storage | 8/10 | Distributed systems, Hadoop/MapReduce, multi-tier, open table formats | Iceberg/Hudi brief; missing lakehouse comparison framework |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Transformation | 10/10 | Data contracts (Protobuf/Avro/JSON Schema), data mesh (4 principles), SCD2 (dbt snapshots + T-SQL MERGE for Fabric with hash detection, soft deletes, health checks) | Complete — both dbt and T-SQL SCD2 patterns covered |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Testing | 8/10 | Quality frameworks + dbt testing | Missing Great Expectations hands-on, data profiling |
| &nbsp;&nbsp;&nbsp;&nbsp;F - Monitoring | 9/10 | Pipeline observability, cost monitoring (expanded), reliability patterns, SLOs, data freshness, Fabric monitoring views (cross-layer reconciliation, SCD2 health, referential integrity, dim coverage) | Now includes Fabric-specific T-SQL monitoring patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;G - Security | 8/10 | Snowflake RBAC (deep), cross-platform IAM, compliance (GDPR/SOC2/HIPAA/PCI-DSS), trust stores | Now 4 notes; major improvement |
| &nbsp;&nbsp;&nbsp;&nbsp;H - Cataloguing | 8/10 | DataHub, Unity Catalog, OpenMetadata, decision matrix | Well-rounded |
| &nbsp;&nbsp;&nbsp;&nbsp;I - Lifecycle | 8/10 | End-to-end framework (Reis & Housley), technology selection matrix | Good foundational overview |
| **4. Data Streaming** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Publish-Subscribe | 7/10 | Stream processing theory, windowing, watermarks | Conceptual; could add more hands-on |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Apache Kafka | 9/10 | Kafka fundamentals, Schema Registry, exactly-once, producer/consumer reliability, backpressure | Comprehensive after enhancement |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Event-Driven Architecture | 7/10 | Event sourcing, CQRS concepts | Solid conceptual coverage |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Snowflake Streams | 8/10 | CDC, real-time alert tasks, task scheduling, stream consumption patterns | Practical production patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Flink & Kinesis | 8/10 | Flink architecture/checkpointing/CDC, Kinesis streams/firehose/analytics, 4-way comparison | Broad streaming coverage |
| **5. Data Engineering Platforms** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - dbt | 9/10 | 5 notes: fundamentals through advanced/cost optimisation | Missing dbt Cloud features (scheduling, metadata API) |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Snowflake | 9/10 | SnowPro guide, Cortex AI, troubleshooting, native dbt, cost monitoring (expanded) | Comprehensive after cost/monitoring enhancements |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Databricks | 9/10 | Platform + modern 2025 patterns (DLT, Unity Catalog) | Missing MLflow, Feature Store, AutoML |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Informatica | 6/10 | Comprehensive single-note reference with positioning | Adequate; lacks hands-on examples |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Matillion | 6/10 | Comprehensive single-note reference with positioning | Adequate; lacks hands-on examples |
| &nbsp;&nbsp;&nbsp;&nbsp;F - Dataiku | 6/10 | Comprehensive single-note reference with positioning | Adequate; lacks hands-on examples |
| &nbsp;&nbsp;&nbsp;&nbsp;G - DuckDB | 7/10 | Architecture, Python/dbt integration, CI/CD, limitations | Single note; could expand with recipes |
| &nbsp;&nbsp;&nbsp;&nbsp;H - Fivetran | 8/10 | Setup, RBAC, REST API, key-pair auth, cost optimisation, MAR reduction | Production-ready operational guide |
| **6. Programming Languages** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Bash | 8/10 | Deployment patterns, scripting fundamentals, error handling, parameter expansion, log parsing, process management | Comprehensive after expansion (1,487 lines) |
| &nbsp;&nbsp;&nbsp;&nbsp;B - PySpark | 9/10 | 26 files: architecture through production, MOC, testing (6), troubleshooting | Vault's strongest section; missing MLlib/GraphFrames |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Python | 7/10 | Core patterns, pandas/Polars, pytest, Streamlit | Missing async patterns, packaging (poetry/uv), type hints |
| &nbsp;&nbsp;&nbsp;&nbsp;D - SQL | 9/10 | 8 files: CTEs, window functions, optimisation, Snowflake pipelines, LATERAL FLATTEN, QUALIFY, JSON, execution plans | Comprehensive after advanced SQL expansion |
| **7. DevOps & Orchestration** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Ansible | 7/10 | Playbooks, roles, GCP integration | Single note; adequate for DE context |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Docker | 8/10 | Container patterns + Kubernetes for data workloads | Solid; missing Spark-on-K8s detail |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Jenkins | 7/10 | Pipeline patterns, shared Groovy libraries | Single note; adequate |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Terraform | 8/10 | Modules, GCP networking/compute/IAM/monitoring | Well-expanded; could add Snowflake/Databricks provider examples |
| &nbsp;&nbsp;&nbsp;&nbsp;E - API Management | 6/10 | Model Context Protocol (MCP) only | Niche; missing API gateway patterns (Kong, AWS API Gateway) |
| &nbsp;&nbsp;&nbsp;&nbsp;F - CI/CD Patterns | 8/10 | GitHub Actions, GitLab CI, Snowflake deployment | 3 notes; missing artifact management, blue/green patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;G - Orchestration | 9/10 | Airflow (2 notes), Dagster & Prefect (861 lines), 18-dimension comparison, migration patterns | All three major orchestrators now covered in depth |
| **8. Data Modelling** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - ERDs | 9/10 | Kimball, advanced dimensional (junk/factless/bridge/mini), Data Vault 2.0, star schema, SCD Types 0-7, Fabric TRUNCATE+INSERT load pattern, OLS/RLS security | Now 4 notes; deepened with Fabric implementation |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Data Flow Diagrams | 7/10 | DFD notation, levels, Mermaid examples (batch ETL, streaming, medallion), common mistakes | Practical after enhancement |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Sequence Diagrams | 7/10 | Notation, Mermaid examples (REST pagination, dbt orchestration, Kafka, CI/CD) | Practical after enhancement |
| **9. Protocols** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - REST | 8/10 | CRUD, OAuth 2.0, Azure AD, pagination, response codes | Missing webhook patterns, API versioning strategies |
| &nbsp;&nbsp;&nbsp;&nbsp;B - SOAP | 8/10 | WSDL, namespaces, Postman setup, complex examples | Solid for legacy integration reference |
| &nbsp;&nbsp;&nbsp;&nbsp;C - SFTP | 7/10 | Protocol comparison, paramiko patterns, key management, security best practices, GoAnywhere MFT | Full rewrite from stub (683 lines) |
| &nbsp;&nbsp;&nbsp;&nbsp;D - gRPC & GraphQL | 7/10 | Both protocols, Protobuf, code examples, decision matrix | Missing federation, schema stitching, streaming depth |
| **10. Learning Resources** | | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Interview Guides | 8/10 | 7 guides: Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks, DP-600 | Some content duplicated from core notes; cert-heavy, light on system design |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Best Practices | 7/10 | Cross-cutting DE best practices | Single note; could expand per domain |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Cheat Sheets | 8/10 | Per-tool references for dbt, Snowflake, Docker, Terraform, Git, Python packaging | Expanded to 613 lines |

**Vault Average: 8.0/10**

---

## Roadmap to 10/10

What each section needs to reach a perfect score. Items marked with effort estimates.

### 1. Templates & Patterns (7 → 10)

| # | Action | Effort |
|---|--------|--------|
| 1 | Add Azure/Fabric reference architecture (T0-T5 as template) | Small |
| 2 | Add GCP reference architecture (BigQuery + Dataflow + Composer) | Small |
| 3 | Add multi-tenancy pipeline template | Small |
| 4 | Add observability-as-code template (Terraform for dashboards + alerts) | Small |

### 2. Cloud Platforms (7-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 5 | AWS: add hands-on project pattern (Step Functions + Glue + Redshift) | Medium |
| 6 | AWS: add FinOps patterns (RI strategies, Savings Plans, cost allocation tags) | Small |
| 7 | GCP: add Cloud Composer (managed Airflow) patterns | Small |
| 8 | GCP: add Dataflow (Apache Beam) patterns | Medium |
| 9 | Multi-cloud comparison framework (when to choose AWS vs Azure vs GCP) | Small |

### 3. Data Engineering Core (6-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 10 | Query & Analysis: add general analytical SQL patterns beyond RAG/vector | Small |
| 11 | Storage: expand Iceberg/Hudi to dedicated deep-dive note | Medium |
| 12 | Testing: add Great Expectations hands-on with code examples | Small |
| 13 | Testing: add data profiling patterns (whylogs, pandas-profiling) | Small |
| 14 | Monitoring: add Datadog/Grafana dashboard patterns | Small |
| 15 | Security: add platform-agnostic masking policy patterns | Small |

### 4. Data Streaming (7-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 16 | Publish-Subscribe: add hands-on code examples (Python Kafka producer/consumer) | Small |
| 17 | Kafka: add Connect framework deep dive (source/sink connectors, SMTs) | Medium |
| 18 | Add GCP Pub/Sub patterns note | Small |

### 5. Data Engineering Platforms (6-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 19 | dbt: add dbt Cloud features (scheduling, metadata API, multi-tenancy) | Small |
| 20 | Databricks: add MLflow, Feature Store, AutoML patterns | Medium |
| 21 | Informatica/Matillion/Dataiku: add hands-on migration examples | Medium |
| 22 | DuckDB: add recipe cookbook (common analytical queries, benchmarks) | Small |

### 6. Programming Languages (7-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 23 | PySpark: populate MLlib subfolder (ML pipelines, feature engineering) | Medium |
| 24 | PySpark: populate Security & Governance subfolder (auditing, masking) | Medium |
| 25 | PySpark: populate GraphFrames subfolder (graph processing) | Small |
| 26 | PySpark: populate MLOps subfolder (training, serving, experiment tracking) | Medium |
| 27 | Python: add async/await patterns for data engineering | Small |
| 28 | Python: add packaging guide (poetry, uv, pyproject.toml) | Small |

### 7. DevOps & Orchestration (7-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 29 | API Management: add API gateway patterns (Kong, AWS API Gateway) | Small |
| 30 | Docker: add Spark-on-K8s detail (spark-submit, operator) | Small |
| 31 | Terraform: add Snowflake/Databricks provider examples | Small |
| 32 | CI/CD: add artifact management and blue/green deployment patterns | Small |

### 8. Data Modelling (7-9 → 10)

| # | Action | Effort |
|---|--------|--------|
| 33 | ERDs: add Inmon 3NF approach (compare with Kimball) | Small |
| 34 | ERDs: add modern lakehouse dimensional modelling (how Delta/Iceberg change the game) | Small |
| 35 | DFD: add more domain-specific examples (healthcare, finance, logistics) | Small |
| 36 | Sequence Diagrams: add error handling and retry flow examples | Small |

### 9. Protocols (7-8 → 10)

| # | Action | Effort |
|---|--------|--------|
| 37 | REST: add webhook patterns and API versioning strategies | Small |
| 38 | gRPC: add federation and schema stitching depth | Small |
| 39 | Add WebSocket / Server-Sent Events note for real-time data flows | Small |

### 10. Learning Resources (7-8 → 10)

| # | Action | Effort |
|---|--------|--------|
| 40 | Interview Guides: add system design interview patterns (design a data pipeline, design a data lake) | Medium |
| 41 | Interview Guides: reduce duplication with core notes (cross-link instead) | Small |
| 42 | Best Practices: expand with per-domain guides (SQL, Python, dbt, Snowflake) | Medium |
| 43 | Cheat Sheets: add PySpark and Airflow command references | Small |
| 44 | Add troubleshooting runbooks (Snowflake perf, PySpark OOM, dbt compilation errors) | Medium |

### Summary

**44 items to reach 10/10 across all sections.** Effort breakdown:

| Effort | Count | Estimated Time |
|--------|------:|----------------|
| Small | 30 | 1-2 hours each |
| Medium | 14 | 3-5 hours each |
| **Total** | **44** | **~100 hours** |

---

## Vault Health Metrics

| Metric | Start | Current |
|--------|-------|---------|
| Total markdown files | 31 | 121 |
| Total lines | ~8,000 | 78,460 |
| Comprehensive files (8+/10) | ~10 | ~85 |
| Empty critical folders | 9 | 0 |
| Cloud platform coverage | 0/3 | 3/3 (all with project patterns) |
| Data engineering platform coverage | 3/6 | 8/8 (incl. DuckDB, Fivetran) |
| DevOps tool coverage | 3/6 | 8/8 (incl. Airflow, Dagster, Prefect) |
| Streaming platform coverage | 1/4 | 4/4 (Kafka, Snowflake Streams, Flink, Kinesis) |
| Protocol coverage | 2/4 | 4/4 (incl. gRPC/GraphQL, SFTP expanded) |
| Security & compliance | 1 note | 4 notes (RBAC, IAM, compliance, trust stores) |
| Interview/cert guides | 4 | 6 + DP-600 |
| Biggest remaining gap | — | PySpark specialist subfolders (MLOps, Security, GraphFrames) |

### Source Projects

| Project | Notes Extracted |
|---------|:-:|
| gcp_datamigration | 10 |
| fabric-aged-care-lakehouse | 3 |
| data-engineering-wiki | 3 |
| databricks-delta-lake-project | 3 |
| gcp_infra_terraform | 1 |
| logistics-analytics-platform | 1 |
| US-flights-data-pipeline | 2 |
| data-engineering-books (source_docs) | 14 |
| No project (reference guides) | 8 |

---

## Creation History

### From review_bundle.txt (18 notes)

| # | Note | Folder |
|---|------|--------|
| 1 | Core dbt Fundamentals (rewritten from skeleton) | `06/A - DBT` |
| 2 | dbt Incremental Loading Patterns | `06/A - DBT` |
| 3 | dbt Macro Patterns | `06/A - DBT` |
| 4 | dbt Tag & Execution Strategy | `06/A - DBT` |
| 5 | SCD Type 2 Patterns | `04/D - Transformation` |
| 6 | Snowflake RBAC & Data Security | `04/G - Security & Governance` |
| 7 | Snowflake Cost Monitoring | `04/F - Monitoring & Observability` |
| 8 | GitLab CI/CD for dbt | `08/F - CI-CD Patterns` |
| 9 | Multi-Tier Data Architecture | `04/C - Storage and Warehousing` |
| 10 | SQL Window Functions | `07/D/D - Window Functions` |
| 11 | Data Ingestion Patterns | `04/A - Ingestion` |
| 12 | Bash Deployment Patterns | `07/A - Bash` |
| 13 | Snowflake Infrastructure Deployment | `08/F - CI-CD Patterns` |
| 14 | dbt Testing & Data Quality | `04/E - Testing & Quality Assurance` |
| 15 | Snowflake & dbt Troubleshooting | `06/B - Snowflake` |
| 16 | Snowflake Native dbt Workspace | `06/B - Snowflake` |
| 17 | Python Data Generation Patterns | `07/C - Python` |
| 18 | Dashboard SQL Patterns | `07/D/B - CTEs` |

### From data-engineering-books source_docs (14 notes)

| # | Note | Source | Folder |
|---|------|--------|--------|
| 19-27 | 9 notes from project code | data-engineering-books repo | Various |
| 28-35 | 8 notes from book PDFs | Kafka, DDIA, Streaming Systems, etc. | Various |
| 36 | AWS DEA Study Guide | .docx extracts | `11/A` |
| 37 | Databricks DEA Study Guide | .docx extracts | `11/A` |
| 38 | AWS/Snowflake/dbt Reference Architecture | .docx extracts | `02/C` |
| 39 | Azure Data Factory Project Patterns | .docx extracts | `03/B` |
| 40 | Trust Stores & Certificate Management | .docx extracts | `04/G` |
| 41 | Apache Airflow Deep Dive | Data Engineering with Python | `08/G` |
| 42 | Apache Spark Architecture & Fundamentals | Spark Definitive Guide | `07/B/01` |
| 43 | Databricks Modern Patterns (2025) | Big Book of DE 3rd ed | `06/C` |
| 44 | Hadoop & MapReduce Fundamentals | Hadoop Definitive Guide + OSDI04 | `04/C` |

### Gap closers (5 notes)

| # | Note | Folder |
|---|------|--------|
| 45 | Data Contracts & Schema Enforcement | `04/D` |
| 46 | Data Mesh & Domain-Driven Data | `04/D` |
| 47 | gRPC & GraphQL | `10/D` |
| 48 | DuckDB for Data Engineering | `06/G` |
| 49 | Common Data Engineering Patterns | `02/A` |

---

## Pending Extraction: fabric-logistics-warehouse

> Source: `/home/jhark/workspace/fabric-logistics-warehouse/`
> Status: **Extracted** — completed 2026-03-21 (5 of 6 items; #4 skipped — client-specific)
> Project: Microsoft Fabric warehouse — 5-tier architecture, 96 SQL files, 15 pipelines

### What the project contains

- **5-tier medallion variant** (T0 control → T1 transient → T2 persistent/SCD2 → T3 integration → T4 star schema)
- **Metadata-driven pipelines** — single `pipeline_control` table drives all 7 source tables across 3 load patterns (snapshot, windowed, incremental)
- **T-SQL SCD2 MERGE** with hash-based change detection, soft deletes (snapshot only), and `etl_is_current`/`etl_is_deleted`/`etl_hash` tracking columns
- **Parent/child pipeline orchestration** — master pipeline calls T1→T2→T3→T4 sequentially; each tier uses parent/child pattern with ForEach loops
- **12 monitoring views** across all layers (freshness, reconciliation, SCD2 health, referential integrity, dim coverage)
- **Star schema** — 5 dimensions + 3 facts with TRUNCATE+INSERT load via security-schema SPs, OLS/RLS placeholders
- **API pagination patterns** — offset-based with `$top`/`$skip`/`$filter`/`$orderby` for OData-style REST API

### Extraction plan (7 updates)

| # | Target Note | What Added | Status |
|---|-------------|------------|--------|
| 1 | `04/D - Transformation/SCD Type 2 Patterns.md` | T-SQL MERGE SCD2: incremental + snapshot SPs, hash change detection, ETL tracking columns | Done |
| 2 | `03/B - Azure/Microsoft Fabric & Azure Data Services.md` | Metadata-driven pipeline orchestration: control table, parent/child hierarchy, audit logging, SP routing | Done |
| 3 | `04/F - Monitoring/Pipeline Observability & Monitoring.md` | Fabric monitoring views: cross-layer reconciliation, SCD2 health, data freshness, referential integrity | Done |
| 4 | `02/C - Reference Architectures/Azure Fabric Reference Architecture.md` | Client-specific case study | Skipped |
| 5 | `02/B - ETL vs ELT Workflows/ETL Pipeline Templates & Patterns.md` | REST API pagination: OData `$top`/`$skip`/`$filter`, snapshot/windowed/incremental strategies | Done |
| 6 | `09/A - ERDs/Star Schema Implementation Patterns.md` | Fabric dim/fact load: TRUNCATE+INSERT, surrogate key lookups, date dim CTE, OLS/RLS security | Done |
| 7 | `CLAUDE.md` workspace table | Added `fabric-logistics-warehouse` as extracted project | Done |

### Vault rating impact

| Section | Before | After | Notes |
|---------|:------:|:-----:|-------|
| Templates — ETL Workflows | 7/10 | 8/10 | API pagination + load pattern variety |
| Azure | 9/10 | 9/10 | Deepens existing strength (pipeline orchestration) |
| Transformation — SCD2 | 9/10 | 10/10 | Adds T-SQL MERGE to complement dbt snapshots |
| Monitoring | 8/10 | 9/10 | Fabric-specific monitoring views |
| Star Schema | 9/10 | 9/10 | Deepens with Fabric load SP patterns |
