# Data Engineering Vault

A personal knowledge base for data engineering — built in [Obsidian](https://obsidian.md), covering cloud platforms, data pipelines, programming, DevOps, data modelling, and interview prep.

**137 notes | 90,000+ lines | 11 topic areas**

---

## Structure

| Section | Notes | Topics |
|---------|------:|--------|
| **1. Templates & Patterns** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Common Patterns | 1 | 12 reusable DE patterns with code examples |
| &nbsp;&nbsp;&nbsp;&nbsp;B - ETL vs ELT Workflows | 1 | Pipeline templates, YAML config patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Reference Architectures | 2 | AWS/Snowflake/dbt, Azure Fabric T0-T5 |
| **2. Cloud Platforms** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - AWS | 2 | Data services, project patterns (Step Functions + Glue + Redshift) |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Azure | 2 | Microsoft Fabric (T0-T5, hash merge, pagination), ADF project patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;C - GCP | 2 | BigQuery ETL framework, Cloud Composer, Dataflow & Apache Beam |
| &nbsp;&nbsp;&nbsp;&nbsp;Multi-cloud | 1 | AWS vs Azure vs GCP comparison framework |
| **3. Data Engineering Core** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Ingestion | 3 | Batch, incremental loading, document ingestion & chunking |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Query & Analysis | 3 | RAG patterns, vector embeddings, analytical SQL patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Storage | 5 | Distributed systems, Hadoop/MapReduce, multi-tier, open table formats, Iceberg & Hudi |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Transformation | 3 | Data contracts & schema enforcement, data mesh, SCD Type 2 |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Testing | 2 | Quality frameworks (+ profiling), dbt testing |
| &nbsp;&nbsp;&nbsp;&nbsp;F - Monitoring | 3 | Pipeline observability, Snowflake cost monitoring, reliability patterns & SLOs |
| &nbsp;&nbsp;&nbsp;&nbsp;G - Security | 4 | Snowflake RBAC, cross-platform IAM, compliance (GDPR/SOC2/HIPAA/PCI-DSS), trust stores |
| &nbsp;&nbsp;&nbsp;&nbsp;H - Cataloguing | 1 | DataHub, Unity Catalog, OpenMetadata |
| &nbsp;&nbsp;&nbsp;&nbsp;I - Lifecycle | 1 | End-to-end DE framework (Reis & Housley) |
| **4. Data Streaming** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Publish-Subscribe | 1 | Stream processing theory, windowing, watermarks |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Apache Kafka | 1 | Fundamentals, Schema Registry, Connect, exactly-once, backpressure |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Event-Driven Architecture | 1 | Event sourcing, CQRS, materialised views |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Snowflake Streams | 1 | CDC, real-time alert tasks, task scheduling |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Flink & Kinesis | 1 | Flink architecture/checkpointing, Kinesis streams/firehose |
| &nbsp;&nbsp;&nbsp;&nbsp;F - GCP Pub/Sub | 1 | Topics, subscriptions, Dataflow integration |
| **5. Data Engineering Platforms** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - dbt | 5 | Fundamentals (+ Cloud), incremental, macros, tags, advanced/cost optimisation |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Snowflake | 4 | SnowPro guide, Cortex AI, troubleshooting, cost monitoring |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Databricks | 3 | Platform, modern patterns 2025 (DLT, MLflow, AutoML), exam guide |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Informatica | 1 | PowerCenter, IDMC, migration patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;E - Matillion | 1 | Cloud-native ELT, migration patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;F - Dataiku | 1 | DSS, visual/code recipes, migration patterns |
| &nbsp;&nbsp;&nbsp;&nbsp;G - DuckDB | 1 | Embedded OLAP, Python/dbt integration, recipe cookbook |
| &nbsp;&nbsp;&nbsp;&nbsp;H - Fivetran | 1 | Setup, RBAC, REST API, cost optimisation |
| **6. Programming Languages** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Bash | 1 | Deployment patterns, scripting fundamentals, error handling, log parsing |
| &nbsp;&nbsp;&nbsp;&nbsp;B - PySpark | 30 | Architecture, core, streaming, testing (6), production, cloud, Delta Lake, MLlib, GraphFrames, Security, MLOps |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Python | 6 | Core patterns, pandas & Polars, pytest, async, packaging, Streamlit |
| &nbsp;&nbsp;&nbsp;&nbsp;D - SQL | 8 | CTEs, window functions, optimisation, LATERAL FLATTEN, QUALIFY, JSON, execution plans |
| **7. DevOps & Orchestration** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Ansible | 1 | Playbooks, roles, GCP integration |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Docker | 2 | Container patterns, Kubernetes (+ Spark-on-K8s) |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Jenkins | 1 | Pipeline patterns, shared Groovy libraries |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Terraform | 1 | Modules, GCP/Snowflake/Databricks providers |
| &nbsp;&nbsp;&nbsp;&nbsp;E - API Management | 3 | API Gateway patterns, Postman & API testing, MCP |
| &nbsp;&nbsp;&nbsp;&nbsp;F - CI/CD | 3 | GitHub Actions (+ advanced), GitLab CI, Snowflake deployment |
| &nbsp;&nbsp;&nbsp;&nbsp;G - Orchestration | 3 | Airflow (overview + deep dive), Dagster & Prefect |
| **8. Data Modelling** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - ERDs | 4 | Kimball, advanced dimensional (+ Inmon), Data Vault 2.0, star schema |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Data Flow Diagrams | 1 | Notation, Mermaid examples, domain-specific (healthcare, finance, e-commerce) |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Sequence Diagrams | 1 | Notation, Mermaid examples (REST, dbt, Kafka, CI/CD, error handling) |
| **9. Protocols** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - REST | 1 | CRUD, OAuth 2.0, pagination, webhooks, API versioning |
| &nbsp;&nbsp;&nbsp;&nbsp;B - SOAP | 1 | WSDL, namespaces, complex examples |
| &nbsp;&nbsp;&nbsp;&nbsp;C - SFTP | 1 | Protocol comparison, paramiko, key management, GoAnywhere MFT |
| &nbsp;&nbsp;&nbsp;&nbsp;D - gRPC & GraphQL | 1 | Protobuf, streaming, decision matrix vs REST |
| &nbsp;&nbsp;&nbsp;&nbsp;E - WebSocket & SSE | 1 | Real-time protocols, comparison with polling |
| **10. Learning Resources** | | |
| &nbsp;&nbsp;&nbsp;&nbsp;A - Interview Guides | 8 | Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks, DP-600, system design |
| &nbsp;&nbsp;&nbsp;&nbsp;B - Best Practices | 1 | Cross-cutting + per-domain (SQL, Python, dbt, Snowflake) |
| &nbsp;&nbsp;&nbsp;&nbsp;C - Cheat Sheets | 1 | Per-tool references (dbt, Snowflake, Docker, Terraform, Git, PySpark, Airflow) |
| &nbsp;&nbsp;&nbsp;&nbsp;D - Troubleshooting | 1 | Runbooks for Snowflake, PySpark, dbt, Airflow, Kafka |

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
