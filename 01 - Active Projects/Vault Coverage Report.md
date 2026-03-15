# Vault Coverage Report

> Updated 2026-03-15 — ~100 markdown files across ~94 directories

---

## At a Glance

| Area | Files | Lines (est.) | Verdict |
|------|------:|-------------:|---------|
| 01 Active Projects | 1 | ~200 | Meta (this report) |
| 02 Templates | 1 | ~487 | ETL Pipeline Templates & Patterns |
| 03 Cloud Platforms | 3 | ~955 | **Solid** — AWS, Azure/Fabric, GCP all populated |
| 04 Data Engineering | 16 | ~3,180 | **Strong** — lifecycle, ingestion (3), storage, transformation, testing & quality (2), monitoring (2), security, cataloguing |
| 05 Data Streaming | 3 | ~574 | **Solid** — Kafka, stream theory, event-driven architecture |
| 06 Data Engineering Platforms | 14 | ~5,950 | **Excellent** — dbt (5), Snowflake (4), Databricks (1), Informatica (1), Matillion (1), Dataiku (1) |
| 07 Programming Languages | 35 | ~24,300 | **Deep** — PySpark (25), Python (6 incl. pandas/Polars, Streamlit), SQL (8), Bash (1) |
| 08 DevOps & Orchestration | 10 | ~3,600 | **Strong** — Docker (2), K8s (1), Terraform (1), Ansible (1), Jenkins (1), CI/CD (3), MCP (1) |
| 09 Data Modelling | 5 | ~820 | **Good** — Kimball, Data Vault 2.0, star schema, DFD and sequence diagrams |
| 10 Protocols | 3 | ~1,070 | **Adequate** — REST and SOAP comprehensive; SFTP stub |
| 11 Learning Resources | 7 | ~16,080 | **Strong** — 4 interview guides + DP-600 study guide + best practices + cheat sheets |

---

## Coverage Heatmap

```
██████████ PySpark              Excellent — 25 files, + Delta Lake patterns + cloud integration
█████████░ Snowflake            Excellent — RBAC, cost, SnowPro guide, Cortex, SQL pipelines
█████████░ dbt                  Excellent — 5 notes: fundamentals, incremental, macros, tags, advanced/cost
████████░░ Interview/Cert Prep  Strong — 5 guides (Snowflake, SQL, PySpark, dbt, DP-600)
████████░░ Data Engineering     Strong — ingestion (3), RAG, architecture, security, monitoring (2), quality, lifecycle
████████░░ DevOps/Orchestration Strong — Docker (2), K8s, Terraform, Ansible, Jenkins, CI/CD (3), MCP
███████░░░ Cloud Platforms      Solid — AWS, Azure/Fabric, GCP all populated
███████░░░ Databricks           Solid — medallion, quality scoring, API, Unity Catalog
██████░░░░ Data Streaming       Solid — Kafka, event-driven, stream theory
██████░░░░ SQL                  Good — 8 notes including Snowflake pipeline patterns
██████░░░░ Data Modelling       Good — Kimball + Data Vault 2.0 + star schema + diagrams
██████░░░░ Python (non-Spark)   Good — 6 notes including core patterns, pandas/Polars, Streamlit
█████░░░░░ Informatica          Adequate — comprehensive reference guide
█████░░░░░ Matillion            Adequate — comprehensive reference guide
█████░░░░░ Dataiku              Adequate — comprehensive reference guide
█████░░░░░ Protocols            Adequate — REST/SOAP good, SFTP stub
████░░░░░░ Bash                 Improved — deployment + scripting fundamentals + Makefile
███░░░░░░░ Templates            Partial — ETL patterns populated; Common Patterns still empty
░░░░░░░░░░ Orchestration        Still missing (Airflow/Dagster)
```

---

## Remaining Gaps

### Empty Folders

| Folder | Expected Content |
|--------|-----------------|
| `02/A — Common Patterns` | Reusable design/code pattern templates |
| `08/E/A — Postman` | API testing, collection management |

### PySpark Empty Subfolders

| Folder | Expected Content |
|--------|-----------------|
| `07/B/05 — GraphFrames Notebooks` | Graph processing with PySpark |
| `07/B/05 — MLlib Examples` | ML pipelines, feature engineering |
| `07/B/08 — Security & Governance` (3 subfolders) | Auditing, data masking, row-level security |
| `07/B/09 — Production Patterns` (4 subfolders) | CI/CD configs, deployment templates, monitoring dashboards, testing examples |
| `07/B/11 — MLOps` | Model training, serving, experiment tracking |

### Topics Still Missing or Limited

| Topic | Why It Matters |
|-------|---------------|
| **Airflow / Dagster** | Industry-standard orchestration — one overview note only |
| **Data contracts** | Schema enforcement between producers and consumers |
| **gRPC / GraphQL** | Modern API protocols |
| **Data mesh** | Organisational pattern for decentralised data ownership |
| **DuckDB** | Embedded analytics engine |

---

## Recommended Next Steps

### Priority 1 — High-Value Gaps

| # | Action | Effort | Rationale |
|---|--------|--------|-----------|
| 1 | **Airflow / orchestration deep dive** | Medium | Biggest functional gap — one overview note exists but needs expansion |
| 2 | **Data contracts & schema enforcement** | Small | Emerging pattern, growing in importance |
| 3 | **Data mesh** | Small | Organisational pattern for decentralised data ownership |

### Priority 2 — Extend Coverage

| # | Action | Effort |
|---|--------|--------|
| 4 | **gRPC & GraphQL** in `10` | Small |
| 5 | **DuckDB** standalone guide | Small |
| 6 | **Common Patterns** templates in `02/A` | Small |

### Priority 3 — Nice to Have

| # | Action | Effort |
|---|--------|--------|
| 7 | Populate PySpark MLOps & Security subfolders | Medium |
| 8 | Expand SQL notes (QUALIFY, LATERAL, JSON functions) | Small |

---

## Vault Health Metrics

| Metric | Before | After |
|--------|--------|-------|
| Total markdown files | 72 | ~100 |
| Notes created/expanded | — | 28+ |
| Comprehensive files (8+/10) | ~25 | ~50 |
| Empty critical folders | 9 | 0 |
| Cloud platform coverage | 0/3 | 3/3 |
| Data engineering platform coverage | 3/6 | 6/6 |
| DevOps tool coverage | 3/6 | 6/6 |
| Data modelling coverage | Kimball only | Kimball + Data Vault 2.0 |
| Biggest remaining gap | Orchestration | Orchestration (Airflow/Dagster) |

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
| No project (reference guides) | 3 |

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

### From data-engineering-books project (9 notes)

| # | Note | Folder |
|---|------|--------|
| 19 | RAG Patterns | `04/B - Query & Analysis` |
| 20 | Vector Embeddings & Semantic Search | `04/B - Query & Analysis` |
| 21 | Snowflake Cortex AI | `06/B - Snowflake` |
| 22 | Document Ingestion & Chunking | `04/A - Ingestion` |
| 23 | Model Context Protocol (MCP) | `08/E - API Management/B - API Gateway Patterns` |
| 24 | Streamlit Data Apps | `07/C - Python` |
| 25 | GitHub Actions for Python Projects | `08/F - CI-CD Patterns` |
| 26 | Python Testing with pytest | `07/C - Python` |
| 27 | Markdown to Notebook Conversion | `07/C - Python` |

### From extracted book PDFs (8 notes)

| # | Note | Source Book(s) | Folder |
|---|------|---------------|--------|
| 28 | Apache Kafka Fundamentals | Kafka Definitive Guide + Designing Event Driven Systems | `05/B - Apache Kafka` |
| 29 | Event-Driven Architecture | Designing Event Driven Systems | `05/C - Event-Driven Architecture` |
| 30 | Databricks & Delta Lake | Data Engineering with Databricks | `06/C - Databricks` |
| 31 | Dimensional Modelling (Kimball) | The Data Warehouse Toolkit | `09/A - Entity Relationship Diagrams` |
| 32 | Distributed Systems Fundamentals | Designing Data Intensive Applications | `04/C - Storage and Warehousing` |
| 33 | Data Engineering Lifecycle | Fundamentals of Data Engineering | `04 - Data Engineering` |
| 34 | Stream Processing Theory | Streaming Systems + Big Data Principles | `05/A - Publish-Subscribe` |
| 35 | Docker & Container Patterns | Designing Distributed Systems | `08/B - Docker` |

### Gap closers (6 notes)

| # | Note | Folder |
|---|------|--------|
| 36 | Terraform for Data Infrastructure | `08/D - Terraform` |
| 37 | SQL Query Optimization | `07/D/C - Query Optimization` |
| 38 | SQL Stored Procedures | `07/D/F - Stored Procedures` |
| 39 | SQL Performance Tuning | `07/D/E - Performance Tuning` |
| 40 | Data Flow Diagrams | `09/B - Data Flow Diagrams` |
| 41 | Sequence Diagrams | `09/C - Sequence Diagrams` |

### From workspace project extraction (24 notes)

| # | Note | Action | Lines |
|---|------|--------|------:|
| 1 | GCP Data Services for Data Engineering | Created | ~290 |
| 2 | Ansible for Data Infrastructure | Created | ~290 |
| 3 | Jenkins Pipeline Patterns for Data Engineering | Created | ~300 |
| 4 | Docker & Container Patterns | Expanded | +170 |
| 5 | Data Validation & Quality Frameworks | Created | ~270 |
| 6 | Python Core Patterns for Data Engineering | Created | ~310 |
| 7 | ETL Pipeline Templates & Patterns | Created | ~487 |
| 8 | PySpark Cloud Integration Patterns | Created | ~260 |
| 9 | Bash Deployment Patterns | Expanded | +170 |
| 10 | Pipeline Observability & Monitoring | Created | ~280 |
| 11 | Microsoft Fabric & Azure Data Services | Created | ~352 |
| 12 | Star Schema Implementation Patterns | Created | ~297 |
| 13 | DP-600 Microsoft Fabric Study Guide | Created | ~282 |
| 14 | Databricks & Delta Lake | Rewritten | 330 |
| 15 | Kubernetes for Data Workloads | Created | ~604 |
| 16 | Delta Lake Operations & Patterns | Created | ~270 |
| 17 | Terraform for Data Infrastructure | Expanded | +210 |
| 18 | dbt Advanced Patterns & Cost Optimisation | Created | ~326 |
| 19 | Snowflake SQL Pipeline Patterns | Created | ~317 |
| 20 | Incremental Loading Strategies | Created | ~300 |
| 21 | AWS Data Services for Data Engineering | Created | ~313 |
| 22 | Informatica for Data Engineering | Created | ~324 |
| 23 | Matillion for Data Engineering | Created | ~301 |
| 24 | Dataiku for Data Engineering | Created | ~327 |

### Notes Enriched (supplements appended)

- Snowflake DEA-C02 Study Guide — resource monitors, warehouse sizing, LATERAL FLATTEN, data classification
- dbt Interview Guide — incremental watermark, snapshot strategies, tag execution, pipeline logging
- SQL Interview Guide — CASE classification, NULLIF, MD5 keys, LISTAGG, DATE_TRUNC patterns
- Data Ingestion Patterns — staging/final atomic loading, incremental vs full reload, batch resilience
- Python Data Generation Patterns — requirements splitting, dependency verification, .env.example
- Snowflake RBAC & Data Security — SQL injection prevention, parameterized queries, model allowlisting
- Bash Deployment Patterns — Makefile patterns, .PHONY targets, review bundle generation
