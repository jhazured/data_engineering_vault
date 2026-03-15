# Vault Coverage Report

> Updated 2026-03-15 — 113 markdown files | 70,241 lines

---

## At a Glance

| Area | Files | Lines | Verdict |
|------|------:|------:|---------|
| 01 Active Projects | 1 | ~238 | Meta (this report) |
| 02 Templates | 3 | ~1,351 | **Good** — ETL templates, common patterns (12), reference architecture |
| 03 Cloud Platforms | 4 | ~1,469 | **Solid** — AWS, Azure/Fabric (incl. ADF patterns), GCP |
| 04 Data Engineering | 20 | ~5,436 | **Excellent** — lifecycle, ingestion (3), storage (incl. Hadoop), transformation (incl. data contracts, data mesh), testing (2), monitoring (2), security (incl. trust stores), cataloguing |
| 05 Data Streaming | 3 | ~574 | **Solid** — Kafka, stream theory, event-driven architecture |
| 06 Data Engineering Platforms | 15 | ~5,393 | **Excellent** — dbt (5), Snowflake (4), Databricks (3), Informatica, Matillion, Dataiku, DuckDB |
| 07 Programming Languages | 36 | ~30,369 | **Deep** — PySpark (26 incl. Spark architecture), Python (6), SQL (8), Bash (1) |
| 08 DevOps & Orchestration | 11 | ~4,469 | **Strong** — Docker (2), K8s, Terraform, Ansible, Jenkins, CI/CD (3), MCP, Airflow (2) |
| 09 Data Modelling | 5 | ~1,320 | **Good** — Kimball, Data Vault 2.0, star schema, DFD, sequence diagrams |
| 10 Protocols | 4 | ~1,500 | **Good** — REST, SOAP, SFTP, gRPC & GraphQL |
| 11 Learning Resources | 9 | ~17,962 | **Excellent** — 6 interview/cert guides + DP-600 + best practices + cheat sheets |

---

## Coverage Heatmap

```
██████████ PySpark              Excellent — 26 files incl. Spark architecture, Delta Lake, cloud integration
█████████░ Snowflake            Excellent — RBAC, cost, SnowPro guide, Cortex, SQL pipelines
█████████░ dbt                  Excellent — 5 notes: fundamentals, incremental, macros, tags, advanced/cost
█████████░ Interview/Cert Prep  Excellent — 6 guides (Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks) + DP-600
████████░░ Data Engineering     Strong — ingestion, storage, Hadoop, data contracts, data mesh, lifecycle, security
████████░░ DevOps/Orchestration Strong — Docker (2), K8s, Terraform, Ansible, Jenkins, CI/CD (3), MCP, Airflow (2)
████████░░ Databricks           Strong — 3 notes: platform, modern patterns (2025), exam guide
███████░░░ Cloud Platforms      Solid — AWS, Azure/Fabric + ADF patterns, GCP
███████░░░ Templates            Solid — ETL templates, 12 common patterns, reference architecture
██████░░░░ Data Streaming       Solid — Kafka, event-driven, stream theory
██████░░░░ SQL                  Good — 8 notes including Snowflake pipeline patterns
██████░░░░ Data Modelling       Good — Kimball + Data Vault 2.0 + star schema + diagrams
██████░░░░ Python (non-Spark)   Good — 6 notes including core patterns, pandas/Polars, Streamlit
██████░░░░ Protocols            Good — REST, SOAP, gRPC & GraphQL; SFTP stub
█████░░░░░ Informatica          Adequate — comprehensive reference guide
█████░░░░░ Matillion            Adequate — comprehensive reference guide
█████░░░░░ Dataiku              Adequate — comprehensive reference guide
█████░░░░░ DuckDB              Adequate — embedded OLAP, Python/dbt integration, CI/CD
████░░░░░░ Bash                 Improved — deployment + scripting fundamentals + Makefile
```

---

## Remaining Gaps

### PySpark Empty Subfolders

| Folder | Expected Content |
|--------|-----------------|
| `07/B/05 — GraphFrames Notebooks` | Graph processing with PySpark |
| `07/B/05 — MLlib Examples` | ML pipelines, feature engineering |
| `07/B/08 — Security & Governance` (3 subfolders) | Auditing, data masking, row-level security |
| `07/B/09 — Production Patterns` (4 subfolders) | CI/CD configs, deployment templates, monitoring dashboards |
| `07/B/11 — MLOps` | Model training, serving, experiment tracking |

### Topics with Limited Coverage

| Topic | Status |
|-------|--------|
| **Iceberg / Hudi** | Brief coverage in existing open table formats note |
| **SFTP / GoAnywhere MFT** | Stub only |
| **Postman / API testing** | Empty folder at `08/E/A` |

---

## Recommended Next Steps

### Priority 1 — Specialist Topics

| # | Action | Effort |
|---|--------|--------|
| 1 | Populate PySpark MLOps subfolder | Medium |
| 2 | Populate PySpark Security & Governance subfolder | Medium |
| 3 | Expand Iceberg / Hudi to dedicated note | Small |

### Priority 2 — Polish

| # | Action | Effort |
|---|--------|--------|
| 4 | Expand SQL notes (QUALIFY, LATERAL, JSON functions) | Small |
| 5 | Standardise British English across all notes | Small |

---

## Vault Health Metrics

| Metric | Start | Current |
|--------|-------|---------|
| Total markdown files | 31 | 113 |
| Total lines | ~8,000 | 70,241 |
| Comprehensive files (8+/10) | ~10 | ~70 |
| Empty critical folders | 9 | 0 |
| Cloud platform coverage | 0/3 | 3/3 |
| Data engineering platform coverage | 3/6 | 7/7 (incl. DuckDB) |
| DevOps tool coverage | 3/6 | 7/7 (incl. Airflow) |
| Protocol coverage | 2/4 | 4/4 (incl. gRPC/GraphQL) |
| Interview/cert guides | 4 | 6 + DP-600 |
| Biggest remaining gap | — | PySpark specialist subfolders |

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
