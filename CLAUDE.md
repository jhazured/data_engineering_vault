# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is an Obsidian knowledge vault — a personal reference library for data engineering topics. It is **not** a software project. There is no build system, test suite, or application code. The content is ~137 markdown files (~90,000 lines) organised into numbered topic folders.

## Vault Structure

Notes are organised by numbered prefix into domain areas:

- **01 - Active Projects** — vault coverage report with heatmap, section ratings, project tree, and 10/10 roadmap
- **02 - Templates** — common DE patterns (12 reusable patterns), ETL pipeline templates, reference architectures (AWS, Azure Fabric)
- **03 - Cloud Platforms** — AWS (services + project patterns), Azure/Microsoft Fabric (T0-T5 + ADF + hash merge), GCP (BigQuery ETL + Composer + Dataflow/Beam), multi-cloud comparison
- **04 - Data Engineering** — ingestion (3), storage (Hadoop, distributed systems, Iceberg/Hudi deep dive), transformation (data contracts, data mesh, SCD2), testing (quality frameworks + profiling), monitoring (observability + cost + reliability + SLOs), security (RBAC, cross-platform IAM, compliance/GDPR/SOC2/HIPAA, masking, trust stores), cataloguing, lifecycle, analytical SQL patterns
- **05 - Data Streaming** — Kafka (+ Schema Registry + Connect + exactly-once), stream processing theory, event-driven architecture, Snowflake Streams, Flink & Kinesis, GCP Pub/Sub
- **06 - Data Engineering Platforms** — dbt (5 + dbt Cloud), Snowflake (4 + cost monitoring), Databricks (3 + MLflow/AutoML), Fivetran, DuckDB (+ recipes), Informatica, Matillion, Dataiku (all with migration patterns)
- **07 - Programming Languages** — Bash (deployment + scripting fundamentals, 1,487 lines), PySpark (30 files incl. Spark architecture, MLlib, GraphFrames, Security & Governance, MLOps), Python (6 incl. async, packaging), SQL (8 incl. LATERAL FLATTEN, QUALIFY, JSON, execution plans)
- **08 - DevOps & Orchestration** — Ansible, Docker (2 + Spark-on-K8s), Jenkins, Terraform (+ Snowflake/Databricks providers), API management (gateway patterns, Postman, MCP), CI/CD (3 + advanced blue/green), Orchestration (Airflow deep dive, Dagster & Prefect)
- **09 - Data Modelling** — Kimball, advanced dimensional (junk/factless/bridge/mini + Inmon + lakehouse-era), Data Vault 2.0, star schema, DFD (+ domain examples), sequence diagrams (+ error handling flows)
- **10 - Protocols** — REST (+ webhooks + API versioning), SOAP, SFTP (comprehensive), gRPC & GraphQL, WebSocket & SSE
- **11 - Learning Resources** — interview guides (7 incl. system design), DP-600 study guide, best practices (per-domain), cheat sheets (per-tool), troubleshooting runbooks

## Obsidian Plugins

- **obsidian-git** — version control for the vault
- **smart-connections** — AI-powered note linking
- **terminal** — embedded terminal

## Conventions

- Folders use numbered prefixes (`01 -`, `02 -`, etc.) for ordering; subfolders use letter prefixes (`A -`, `B -`, etc.)
- Each leaf folder typically contains one or a few markdown files on that topic
- PySpark is the most developed section with 30 files across 12 subsections
- The vault setting `alwaysUpdateLinks` is enabled — Obsidian auto-updates internal links when files are moved/renamed
- British English throughout (`Modelling`, `Optimisation`, `Cataloguing`, `Serialisation`)

## Current State

See `01 - Active Projects/Vault Coverage Report.md` for the full coverage heatmap, section ratings (7.9/10 average), project tree, and roadmap.

### Vault Metrics

- **137 notes | 90,656 lines | 11 topic areas**
- **Vault Average: 7.9/10** across all sections
- No empty critical folders
- All major DE domains covered

### Workspace Projects (source material)

These projects at `/home/jhark/workspace/` have been reviewed and extracted into vault notes:

| Project | Content Extracted |
|---------|-------------------|
| `gcp_etl_framework` | BigQuery ETL, Cloud SQL, GCS extraction, transformation patterns |
| `gcp_infra_terraform` | Terraform modules, GCP IAM, Cloud Monitoring dashboards |
| `fabric-aged-care-lakehouse` | Microsoft Fabric, Delta Lake, SCD2, medallion, RLS patterns |
| `databricks-delta-lake-project` | Databricks platform, quality scoring, K8s manifests, Terraform |
| `data-engineering-wiki` | Microsoft Fabric T0-T5, DP-600 study guide, star schema, security patterns |
| `logistics-analytics-platform` | dbt patterns, Fivetran, Snowflake cost/streaming/RBAC, data lineage, macros |
| `US-flights-data-pipeline` | Snowflake JS stored procs, star schema DDL, incremental loading |
| `data-engineering-books` | Spark, Hadoop, Kafka, DDIA, Kimball, Airflow, Databricks (2025), AWS DEA |
| `fabric-logistics-warehouse` | Fabric T-SQL SCD2 MERGE, metadata-driven pipeline orchestration, monitoring views (T0-T4), API pagination patterns, star schema load SPs, OLS/RLS security |

## Working With This Vault

- When creating new notes, follow the existing naming conventions (numbered/lettered prefix folders, descriptive filenames)
- Use Obsidian-compatible markdown: `[[wikilinks]]` for internal links, standard markdown for everything else
- Place new notes in the appropriate topic folder; create new lettered subfolders if a new subtopic is needed
- The `01 - Active Projects` folder is reserved for current work-in-progress notes
- Run `make bundle` to generate a `review_bundle.txt` for sharing/review (if Makefile exists)
