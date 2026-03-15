# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is an Obsidian knowledge vault — a personal reference library for data engineering topics. It is **not** a software project. There is no build system, test suite, or application code. The content is ~113 markdown files organized into numbered topic folders.

## Vault Structure

Notes are organized by numbered prefix into domain areas:

- **01 - Active Projects** — workspace for current initiatives (vault audit, coverage report)
- **02 - Templates** — ETL pipeline templates, common DE patterns (12 reusable patterns), reference architectures (AWS/Snowflake/dbt)
- **03 - Cloud Platforms** — AWS, Azure/Microsoft Fabric (incl. ADF patterns), GCP
- **04 - Data Engineering** — core DE topics: ingestion (3), storage (incl. Hadoop/MapReduce), transformation (incl. data contracts, data mesh), testing & quality (2), monitoring (2), security (incl. trust stores), cataloguing
- **05 - Data Streaming** — pub/sub, Kafka, event-driven architecture
- **06 - Data Engineering Platforms** — dbt (5), Snowflake (4), Databricks (3 incl. modern 2025 patterns), Informatica, Matillion, Dataiku, DuckDB
- **07 - Programming Languages** — Bash (1), PySpark (26 incl. Spark architecture + Delta Lake + cloud integration), Python (6 incl. pandas/Polars, Streamlit), SQL (8 incl. Snowflake pipeline patterns)
- **08 - DevOps & Orchestration** — Ansible, Docker (2), Kubernetes, Jenkins, Terraform, API management/MCP, CI/CD (3), Airflow (2 incl. deep dive)
- **09 - Data Modelling** — Kimball, Data Vault 2.0, star schema implementation, data flow diagrams, sequence diagrams
- **10 - Protocols** — REST, SOAP, SFTP, gRPC & GraphQL
- **11 - Learning Resources** — interview guides (6 incl. AWS DEA, Databricks), DP-600 study guide, best practices, cheat sheets

## Obsidian Plugins

- **obsidian-git** — version control for the vault
- **smart-connections** — AI-powered note linking
- **terminal** — embedded terminal

## Conventions

- Folders use numbered prefixes (`01 -`, `02 -`, etc.) for ordering; subfolders use letter prefixes (`A -`, `B -`, etc.)
- Each leaf folder typically contains one or a few markdown files on that topic
- PySpark is the most developed section with 12 subsections covering core concepts through production patterns
- The vault setting `alwaysUpdateLinks` is enabled — Obsidian auto-updates internal links when files are moved/renamed

## Current Priorities

See `01 - Active Projects/Vault Coverage Report.md` for the full prioritised gap analysis, per-folder assessment, and recommended next steps.

### Remaining Gaps (as of 2026-03-15)

- **PySpark Security/Governance, MLOps, GraphFrames** — subfolders scaffolded but empty
- **Iceberg / Hudi** — open table formats beyond Delta Lake (brief coverage in existing note)

### Workspace Projects (source material)

These projects at `/home/jhark/workspace/` have been reviewed and extracted into vault notes:

| Project | Content Extracted |
|---------|-------------------|
| `gcp_datamigration` | GCP, Ansible, Jenkins, Docker, Python, Bash, ETL templates, observability, data quality |
| `gcp_infra_terraform` | Terraform modules (networking, compute, storage, IAM, monitoring) |
| `fabric-aged-care-lakehouse` | Microsoft Fabric, Delta Lake, SCD2, medallion architecture |
| `databricks-delta-lake-project` | Databricks platform, quality scoring, K8s manifests, Terraform |
| `data-engineering-wiki` | Microsoft Fabric T0-T5, DP-600 study guide, star schema, ADRs |
| `logistics-analytics-platform` | dbt advanced patterns, Fivetran cost optimisation, ML features |
| `US-flights-data-pipeline` | Snowflake JS stored procs, star schema DDL, incremental loading |
| `data-engineering-books` | RAG architecture reference (already covered in vault) |

## Working With This Vault

- When creating new notes, follow the existing naming conventions (numbered/lettered prefix folders, descriptive filenames)
- Use Obsidian-compatible markdown: `[[wikilinks]]` for internal links, standard markdown for everything else
- Place new notes in the appropriate topic folder; create new lettered subfolders if a new subtopic is needed
- The `01 - Active Projects` folder is reserved for current work-in-progress notes
