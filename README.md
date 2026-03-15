# Data Engineering Vault

A personal knowledge base for data engineering — built in [Obsidian](https://obsidian.md), covering cloud platforms, data pipelines, programming, DevOps, data modelling, and interview prep.

**113 notes | 70,000+ lines | 11 topic areas**

---

## Structure

```
01 - Active Projects        Vault audit & coverage report
02 - Templates              ETL pipeline templates, common patterns, reference architectures
03 - Cloud Platforms        AWS, Azure/Microsoft Fabric, GCP
04 - Data Engineering       Ingestion, storage, transformation, testing, monitoring, security, cataloguing
05 - Data Streaming         Kafka, event-driven architecture, stream processing theory
06 - Data Engineering Platforms
    dbt (5)                 Fundamentals, incremental, macros, tags, advanced/cost optimisation
    Snowflake (4)           SnowPro study guide, Cortex AI, troubleshooting
    Databricks (3)          Delta Lake, medallion, Unity Catalog, DLT, modern patterns (2025)
    DuckDB (1)              Embedded OLAP, Python/dbt integration, CI/CD
    Informatica (1)         PowerCenter, IDMC, mapping design, ETL patterns
    Matillion (1)           Cloud-native ELT, Snowflake integration, orchestration
    Dataiku (1)             DSS, visual/code recipes, MLOps, governance
07 - Programming Languages
    Bash (1)                Deployment patterns, scripting fundamentals
    PySpark (25)            Core → streaming → testing → production → cloud integration
    Python (6)              Core patterns, pandas & Polars, pytest, data generation, Streamlit
    SQL (8)                 CTEs, window functions, optimisation, Snowflake pipeline patterns
08 - DevOps & Orchestration
    Ansible (1)             Playbooks, roles, GCP integration
    Docker (2)              Container patterns, Kubernetes for data workloads
    Jenkins (1)             Pipeline patterns, shared Groovy libraries
    Terraform (1)           Modules, GCP networking/compute/IAM/monitoring
    Airflow (2)             DAGs, scheduling, deep dive, Dagster/Prefect alternatives
    CI/CD (3)               GitHub Actions, GitLab CI, Snowflake deployment
09 - Data Modelling         Kimball, Data Vault 2.0, star schema, data flow & sequence diagrams
10 - Protocols              REST, SOAP, SFTP, gRPC & GraphQL
11 - Learning Resources     Interview guides (Snowflake, SQL, PySpark, dbt, AWS DEA, Databricks),
                            DP-600 study guide, best practices, cheat sheets
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
| GCP ETL framework | GCP services, Ansible, Jenkins, Docker, Python patterns, observability |
| GCP Terraform infra | Terraform modules (networking, compute, storage, IAM, monitoring) |
| Fabric lakehouse | Microsoft Fabric, Delta Lake, SCD2, medallion architecture |
| Databricks platform | Medallion pipeline, quality scoring, K8s manifests, Databricks API |
| Fabric DW wiki | T0-T5 architecture, DP-600 prep, star schema, ADRs |
| Logistics analytics | dbt advanced patterns, Fivetran cost optimisation, ML features |
| Flights pipeline | Snowflake JS stored procs, star schema DDL, incremental loading |

## Conventions

- **Folders**: Numbered prefixes (`01 -`, `02 -`) for top-level ordering, letter prefixes (`A -`, `B -`) for subfolders
- **File names**: Title Case, `&` for compound terms, brand casing preserved (`dbt`, `PySpark`)
- **Spelling**: British English (`Modelling`, `Optimisation`, `Cataloguing`)
- **Links**: `[[wikilinks]]` for internal cross-references

## Coverage

See [`01 - Active Projects/Vault Coverage Report.md`](01%20-%20Active%20Projects/Vault%20Coverage%20Report.md) for a detailed gap analysis with heatmap, per-folder assessment, and prioritised next steps.

### Remaining Gaps

- PySpark Security/Governance, MLOps, GraphFrames (subfolders scaffolded, empty)
- Iceberg / Hudi (brief coverage in existing note, no dedicated deep dive)

---

Built with [Obsidian](https://obsidian.md) and [Claude Code](https://claude.ai/code).
