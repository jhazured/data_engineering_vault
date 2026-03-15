# Vault Audit — 2026-03-15

## Summary (Updated)

- **Total markdown files:** 72 (was 31 — 18 from review_bundle, 9 from data-engineering-books code, 8 from extracted book PDFs, 6 final gap closers)
- **Total directories:** 88
- **Empty directories:** 50 (was 61 — 11 previously-empty folders now have content)
- **PySpark files:** 20 (46% of vault, was 65%)
- **Interview guides:** 4 (3 enriched with real-world supplement sections)
- **SQL files:** 3 (was 2)

### Notes Created from review_bundle.txt (2026-03-15)

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

### Notes Created from data-engineering-books project (2026-03-15)

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

### Notes Created from extracted book PDFs (2026-03-15)

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

### Final gap closers (2026-03-15)

| # | Note | Folder |
|---|------|--------|
| 36 | Terraform for Data Infrastructure | `08/D - Terraform` |
| 37 | SQL Query Optimization | `07/D/C - Query Optimization` |
| 38 | SQL Stored Procedures | `07/D/F - Stored Procedures` |
| 39 | SQL Performance Tuning | `07/D/E - Performance Tuning` |
| 40 | Data Flow Diagrams | `09/B - Data Flow Diagrams` |
| 41 | Sequence Diagrams | `09/C - Sequence Diagrams` |

### Files Enriched from data-engineering-books (2026-03-15)

- Data Ingestion Patterns — staging/final atomic loading, incremental vs full reload, batch resilience
- Python Data Generation Patterns — requirements splitting, dependency verification, .env.example
- Snowflake RBAC & Data Security — SQL injection prevention, parameterized queries, model allowlisting, Cortex grants
- Bash Deployment Patterns — Makefile patterns, .PHONY targets, review bundle generation

### Files Enriched (real-world supplement sections appended)

- Snowflake DEA-C02 Study Guide — resource monitors, warehouse sizing, LATERAL FLATTEN, data classification
- dbt Interview Guide — incremental watermark, snapshot strategies, tag execution, pipeline logging
- SQL Interview Guide — CASE classification, NULLIF, MD5 keys, LISTAGG, DATE_TRUNC patterns
- **Protocol files:** 3

PySpark is the dominant focus. The vault is well-scaffolded but most folders are still empty.

---

## Priority 1 — Critical gaps (folders exist, zero content)

Core DE topics that directly complement existing PySpark, dbt, Snowflake, and SQL notes.

| Topic | Folder | Why |
|---|---|---|
| **Docker** | `08/B - Docker` | Referenced in PySpark Production Patterns but no Docker notes |
| **Kafka** | `05/B - Apache Kafka` | PySpark Streaming covers Kafka integration heavily — standalone Kafka architecture/ops notes missing |
| **Python** | `07/C - Python` | 20 PySpark files but nothing on core Python (decorators, generators, packaging, virtual envs, typing) |
| **SQL Window Functions** | `07/D/D - Window Functions` | Exhaustive PySpark window functions note exists but SQL equivalent is empty |
| **SQL Query Optimization** | `07/D/C - Query Optimization` | SQL interview guide references execution plans and index design — no standalone reference |
| **Data Ingestion** | `04/A - Ingestion` | Fundamental DE topic; dbt/Snowflake notes assume data is already landed |
| **Data Transformation** | `04/D - Transformation` | Platform-agnostic transformation patterns (SCD, dedup, normalisation) |
| **Terraform** | `08/D - Terraform` | IaC is a standard lead DE expectation; empty folder |

## Priority 2 — High-value gaps (strengthen existing content)

| Topic | Folder | Why |
|---|---|---|
| **Databricks** | `06/C - Databricks` | Folder exists, empty. Snowflake has a study guide — Databricks equivalent (Unity Catalog, Delta Lake, Workflows) would round out platforms |
| **Stored Procedures** | `07/D/F - Stored Procedures` | Referenced in SQL interview guide but no dedicated notes |
| **SQL Performance Tuning** | `07/D/E - Performance Tuning` | Indexing strategies, query plans, statistics — distinct from query optimization |
| **CI/CD Patterns** | `08/F - CI-CD Patterns` | dbt interview guide discusses CI/CD, PySpark production notes mention it — no standalone reference |
| **Data Storage & Warehousing** | `04/C - Storage and Warehousing` | Partitioning strategies, file formats (Parquet vs Avro vs ORC), lakehouse vs warehouse — platform-agnostic |
| **dbt Fundamentals (flesh out)** | `06/A - DBT` | Currently a skeleton with just headers. Interview guide is thorough but reference note needs content |

## Priority 3 — Important for completeness

| Topic | Folder | Why |
|---|---|---|
| **Event-Driven Architecture** | `05/C - Event-Driven Architecture` | Complements Kafka; patterns like CQRS, event sourcing, saga |
| **Ansible** | `08/A - Ansible` | Empty; relevant if used in projects |
| **Jenkins** | `08/C - Jenkins` | Empty; same rationale |
| **Data Modelling (all 3)** | `09/A,B,C` | ERDs, DFDs, sequence diagrams — all empty. Useful for system design interviews |
| **Monitoring & Observability** | `04/F - Monitoring & Observability` | Platform-agnostic alerting, logging, SLA tracking patterns |
| **Data Quality & Testing** | `04/E - Testing & Quality Assurance` | PySpark-specific testing exists (6 files) but no general data quality frameworks (Great Expectations, Soda, dbt tests) |
| **Bash** | `07/A - Bash` | Empty; useful for scripting patterns DEs use daily |

## Priority 4 — Nice to have

| Topic | Folder | Why |
|---|---|---|
| **Cloud Platforms (AWS/Azure/GCP)** | `03/A,B,C` | All empty. Key services: S3/ADLS/GCS, Glue/ADF/Dataflow, Redshift/Synapse/BigQuery |
| **Security & Governance** | `04/G - Security & Governance` | RBAC, data masking, PII handling — platform-agnostic |
| **Data Cataloguing** | `04/H - Data Cataloguing` | Emerging area; DataHub, OpenMetadata, Unity Catalog |
| **Templates (Common Patterns, ETL vs ELT)** | `02/A,B` | Reusable note templates for the vault |
| **Best Practices / Cheat Sheets** | `11/B,C` | Empty learning resource folders |
| **Pub/Sub patterns** | `05/A - Publish-Subscribe` | General messaging patterns beyond Kafka |

---

## Existing Content Assessment

### Well-developed (no action needed)
- PySpark Core Concepts, Data Operations, Window Functions, Performance Optimization, Advanced Features, Streaming, Production Engineering
- PySpark Reference & Utilities (MOC, Quick Reference, Error Codes, Troubleshooting, Functions)
- Snowflake DEA-C02 Study Guide
- All 4 interview guides (dbt, PySpark, Snowflake, SQL)

### Partially complete (could be expanded)
- PySpark Testing & Quality — files 03-06 appear truncated or incomplete
- SQL — only Aggregation/Filtering and CTEs have content
- dbt Fundamentals — skeleton file with headers only
- REST APIs, SOAP, GoAnywhere MFT — moderate coverage

### Suggested starting point
Docker, Kafka, and Python would give the most immediate value — they're referenced throughout existing PySpark and production notes but have no standalone content. After that, filling in the SQL folders (window functions, query optimization) would be fast since patterns can be adapted from PySpark equivalents.
