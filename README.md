# Data Engineering Vault

A personal knowledge base for data engineering — built in [Obsidian](https://obsidian.md), covering cloud platforms, data pipelines, programming, DevOps, data modelling, and interview prep.

**137 notes | 90,000+ lines | 11 topic areas**

---

## Structure

### 01 - Active Projects
Vault coverage report with heatmap, section ratings, project tree, and roadmap.

### 02 - Templates
| Area | Notes | Topics |
|------|------:|--------|
| Common Patterns | 1 | 12 reusable DE patterns with code examples |
| ETL vs ELT Workflows | 1 | Pipeline templates, YAML config patterns |
| Reference Architectures | 2 | AWS/Snowflake/dbt, Azure Fabric T0-T5 |

### 03 - Cloud Platforms
| Area | Notes | Topics |
|------|------:|--------|
| AWS | 2 | Data services, project patterns (Step Functions + Glue + Redshift) |
| Azure | 2 | Microsoft Fabric (T0-T5, hash merge, pagination), ADF project patterns |
| GCP | 2 | BigQuery ETL framework, Cloud Composer, Dataflow & Apache Beam |
| Cross-cloud | 1 | Multi-cloud comparison framework |

### 04 - Data Engineering
| Area | Notes | Topics |
|------|------:|--------|
| Ingestion | 3 | Batch, incremental loading, document ingestion & chunking |
| Query & Analysis | 3 | RAG patterns, vector embeddings, analytical SQL patterns |
| Storage | 5 | Distributed systems, Hadoop/MapReduce, multi-tier, open table formats, Iceberg & Hudi deep dive |
| Transformation | 3 | Data contracts & schema enforcement, data mesh, SCD Type 2 |
| Testing | 2 | Quality frameworks (+ profiling), dbt testing |
| Monitoring | 3 | Pipeline observability, Snowflake cost monitoring, reliability patterns & SLOs |
| Security | 4 | Snowflake RBAC, cross-platform IAM & governance, compliance (GDPR/SOC2/HIPAA), trust stores |
| Cataloguing | 1 | DataHub, Unity Catalog, OpenMetadata |
| Lifecycle | 1 | End-to-end DE framework (Reis & Housley) |

### 05 - Data Streaming
| Area | Notes | Topics |
|------|------:|--------|
| Publish-Subscribe | 1 | Stream processing theory, windowing, watermarks |
| Apache Kafka | 1 | Fundamentals, Schema Registry, Connect, exactly-once, backpressure |
| Event-Driven Architecture | 1 | Event sourcing, CQRS, materialized views |
| Snowflake Streams | 1 | CDC, real-time alert tasks, task scheduling |
| Flink & Kinesis | 1 | Flink architecture/checkpointing, Kinesis streams/firehose |
| GCP Pub/Sub | 1 | Topics, subscriptions, Dataflow integration |

### 06 - Data Engineering Platforms
| Platform | Notes | Topics |
|----------|------:|--------|
| dbt | 5 | Fundamentals (+ Cloud), incremental, macros, tags, advanced/cost optimisation |
| Snowflake | 4 | SnowPro guide, Cortex AI, troubleshooting, cost monitoring |
| Databricks | 3 | Platform, modern patterns 2025 (DLT, MLflow, AutoML), exam guide |
| Fivetran | 1 | Setup, RBAC, REST API, cost optimisation |
| DuckDB | 1 | Embedded OLAP, Python/dbt integration, recipe cookbook |
| Informatica | 1 | PowerCenter, IDMC, migration patterns |
| Matillion | 1 | Cloud-native ELT, migration patterns |
| Dataiku | 1 | DSS, visual/code recipes, migration patterns |

### 07 - Programming Languages
| Language | Notes | Topics |
|----------|------:|--------|
| Bash | 1 | Deployment patterns, scripting fundamentals, error handling, log parsing |
| PySpark | 30 | Architecture, core, data ops, window functions, performance, streaming, testing (6), production, cloud, Delta Lake, MLlib, GraphFrames, Security & Governance, MLOps |
| Python | 6 | Core patterns, pandas & Polars, pytest, async, packaging, Streamlit |
| SQL | 8 | CTEs, window functions, aggregation, optimisation, performance tuning, stored procedures, LATERAL FLATTEN, QUALIFY |

### 08 - DevOps & Orchestration
| Area | Notes | Topics |
|------|------:|--------|
| Ansible | 1 | Playbooks, roles, GCP integration |
| Docker | 2 | Container patterns, Kubernetes (+ Spark-on-K8s) |
| Jenkins | 1 | Pipeline patterns, shared Groovy libraries |
| Terraform | 1 | Modules, GCP/Snowflake/Databricks providers |
| API Management | 3 | API Gateway patterns, Postman & API testing, MCP |
| CI/CD | 3 | GitHub Actions (+ advanced), GitLab CI, Snowflake deployment |
| Orchestration | 3 | Airflow (overview + deep dive), Dagster & Prefect |

### 09 - Data Modelling
| Area | Notes | Topics |
|------|------:|--------|
| ERDs | 4 | Kimball, advanced dimensional (junk/factless/bridge/mini + Inmon), Data Vault 2.0, star schema |
| Data Flow Diagrams | 1 | Notation, levels, Mermaid examples, domain-specific (healthcare, finance, e-commerce) |
| Sequence Diagrams | 1 | Notation, Mermaid examples (REST, dbt, Kafka, CI/CD, error handling) |

### 10 - Protocols
| Protocol | Notes | Topics |
|----------|------:|--------|
| REST | 1 | CRUD, OAuth 2.0, pagination, webhooks, API versioning |
| SOAP | 1 | WSDL, namespaces, complex examples |
| SFTP | 1 | Protocol comparison, paramiko, key management, GoAnywhere MFT |
| gRPC & GraphQL | 1 | Protobuf, streaming, decision matrix vs REST |
| WebSocket & SSE | 1 | Real-time protocols, comparison with polling |

### 11 - Learning Resources
| Area | Notes | Topics |
|------|------:|--------|
| Interview Guides | 8 | Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks, DP-600, system design |
| Best Practices | 1 | Cross-cutting + per-domain (SQL, Python, dbt, Snowflake) |
| Cheat Sheets | 1 | Per-tool references (dbt, Snowflake, Docker, Terraform, Git, PySpark, Airflow) |
| Troubleshooting | 1 | Runbooks for Snowflake, PySpark, dbt, Airflow, Kafka |

## Usage

### Open in Obsidian

1. Clone this repo
2. Open the folder as a vault in Obsidian (`Open folder as vault`)
3. Trust the plugins when prompted (obsidian-git, smart-connections, terminal)

Notes use `[[wikilinks]]` for cross-referencing — Obsidian will render these as clickable links and build a knowledge graph.

### Browse on GitHub

All notes are standard Markdown and readable directly on GitHub. Wikilinks won't resolve, but the content is fully accessible.

## Content Sources

Many notes were extracted from real project codebases — code examples, configurations, and patterns are drawn from production implementations rather than documentation alone.

| Source Project | Topics Extracted |
|----------------|-----------------|
| GCP ETL framework | GCP services, BigQuery ETL, Ansible, Jenkins, Docker, Python, observability |
| GCP Terraform infra | Terraform modules, GCP IAM, Cloud Monitoring dashboards |
| Fabric lakehouse | Microsoft Fabric, Delta Lake, SCD2, medallion, RLS patterns |
| Databricks platform | Medallion pipeline, quality scoring, K8s manifests, Databricks API |
| Fabric DW wiki | T0-T5 architecture, DP-600 prep, star schema, security patterns |
| Logistics analytics | dbt patterns, Fivetran, Snowflake cost/streaming/RBAC, data lineage |
| Flights pipeline | Snowflake JS stored procs, star schema DDL, incremental loading |
| Data engineering books | Spark, Hadoop, Kafka, DDIA, Kimball, Airflow, Databricks (2025) |

## Conventions

- **Folders**: Numbered prefixes (`01 -`, `02 -`) for top-level ordering, letter prefixes (`A -`, `B -`) for subfolders
- **File names**: Title Case, `&` for compound terms, brand casing preserved (`dbt`, `PySpark`)
- **Spelling**: British English (`Modelling`, `Optimisation`, `Cataloguing`)
- **Links**: `[[wikilinks]]` for internal cross-references

## Coverage

See [`01 - Active Projects/Vault Coverage Report.md`](01%20-%20Active%20Projects/Vault%20Coverage%20Report.md) for the full coverage heatmap, section ratings, project tree, and 10/10 roadmap.

**Vault Average: 7.9/10** — comprehensive coverage across all data engineering domains with production-grounded code examples.

---

Built with [Obsidian](https://obsidian.md) and [Claude Code](https://claude.ai/code).
