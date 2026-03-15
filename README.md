# Data Engineering Vault

A personal knowledge base for data engineering — built in [Obsidian](https://obsidian.md), covering cloud platforms, data pipelines, programming, DevOps, data modelling, and interview prep.

**130+ notes | 85,000+ lines | 11 topic areas**

---

## Structure

```
01 - Active Projects        Vault coverage report & roadmap
02 - Templates              Common patterns (12), ETL templates, reference architectures (AWS, Azure)
03 - Cloud Platforms        AWS (+ project patterns), Azure/Fabric (+ ADF + hash merge),
                            GCP (+ BigQuery ETL + Composer + Dataflow), multi-cloud comparison
04 - Data Engineering       Ingestion (3), storage (Hadoop, open table formats, Iceberg/Hudi),
                            transformation (data contracts, data mesh, SCD2), testing (+ profiling),
                            monitoring (observability, cost, reliability, SLOs),
                            security (RBAC, cross-platform IAM, compliance, masking, trust stores),
                            cataloguing, lifecycle, analytical SQL patterns
05 - Data Streaming         Kafka (+ Schema Registry + Connect + exactly-once), stream theory,
                            event-driven, Snowflake Streams, Flink & Kinesis, GCP Pub/Sub
06 - Data Engineering Platforms
    dbt (5)                 Fundamentals, incremental, macros (expanded), tags, advanced/cost, dbt Cloud
    Snowflake (4)           SnowPro guide, Cortex AI, troubleshooting, cost monitoring (expanded)
    Databricks (3)          Platform, modern patterns (2025, DLT, MLflow), exam guide
    Fivetran (1)            Setup, RBAC, REST API, cost optimisation
    DuckDB (1)              Embedded OLAP, Python/dbt integration, recipe cookbook
    Informatica (1)         PowerCenter, IDMC, migration patterns
    Matillion (1)           Cloud-native ELT, migration patterns
    Dataiku (1)             DSS, visual/code recipes, migration patterns
07 - Programming Languages
    Bash (1)                Deployment patterns + scripting fundamentals (1,487 lines)
    PySpark (30)            Core → architecture → streaming → testing → production → cloud
                            + MLlib + GraphFrames + Security & Governance + MLOps
    Python (6)              Core patterns, pandas/Polars, pytest, async, packaging, Streamlit
    SQL (8)                 CTEs, window functions, LATERAL FLATTEN, QUALIFY, JSON, execution plans
08 - DevOps & Orchestration
    Ansible (1)             Playbooks, roles, GCP integration
    Docker (2)              Container patterns, Kubernetes (+ Spark-on-K8s)
    Jenkins (1)             Pipeline patterns, shared Groovy libraries
    Terraform (1)           Modules, GCP/Snowflake/Databricks providers
    API Management (3)      API Gateway patterns, Postman & API testing, MCP
    CI/CD (3)               GitHub Actions (+ advanced), GitLab CI, Snowflake deployment
    Orchestration (3)       Airflow (overview + deep dive), Dagster & Prefect
09 - Data Modelling         Kimball, advanced dimensional (junk/factless/bridge/mini), Inmon,
                            Data Vault 2.0, star schema, DFD (+ domain examples), sequence diagrams
10 - Protocols              REST (+ webhooks + versioning), SOAP, SFTP (comprehensive),
                            gRPC & GraphQL, WebSocket & SSE
11 - Learning Resources     Interview guides (Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks),
                            system design interviews, DP-600, best practices (per-domain),
                            cheat sheets (per-tool), troubleshooting runbooks
```

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
