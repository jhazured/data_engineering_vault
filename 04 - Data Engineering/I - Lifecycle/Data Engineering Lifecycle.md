# Data Engineering Lifecycle

A framework from "Fundamentals of Data Engineering" (Reis & Housley) for understanding the end-to-end data engineering process.

## The Lifecycle Stages

```
Generation → Storage → Ingestion → Transformation → Serving
                ↕           ↕            ↕              ↕
         ┌──────────────────────────────────────────────────┐
         │              Undercurrents                        │
         │  Security · Data Management · DataOps            │
         │  Orchestration · Software Engineering            │
         └──────────────────────────────────────────────────┘
```

### 1. Generation

Where data originates — source systems you don't control:

- **Application databases** — OLTP systems (PostgreSQL, MySQL, SQL Server)
- **APIs** — REST endpoints, webhooks, third-party SaaS
- **Files** — CSV, JSON, Parquet drops (SFTP, S3, GCS)
- **Events/streams** — Kafka topics, message queues, IoT sensors
- **Logs** — Application logs, change data capture (CDC)

### 2. Storage

Where data lives at rest — choosing the right abstraction:

| Storage Type | Best For | Examples |
|-------------|----------|---------|
| **Object storage** | Raw files, data lake | S3, GCS, ADLS |
| **Data warehouse** | Structured analytics | Snowflake, BigQuery, Redshift |
| **Data lakehouse** | Unified raw + analytics | Databricks/Delta Lake, Iceberg |
| **OLTP database** | Transactional serving | PostgreSQL, DynamoDB |
| **Streaming store** | Event retention | Kafka, Kinesis |

### 3. Ingestion

Moving data from source to storage:

| Pattern | Latency | Complexity | Use Case |
|---------|---------|------------|----------|
| **Batch** | Minutes to hours | Lower | Nightly loads, full snapshots |
| **Micro-batch** | Seconds to minutes | Medium | Snowpipe, Spark Structured Streaming |
| **Real-time** | Milliseconds | Higher | Kafka, Kinesis, event-driven |
| **CDC** | Near-real-time | Medium | Debezium, Fivetran, database logs |

### 4. Transformation

Converting raw data into analysis-ready models:

- **ELT** (modern) — load raw, transform in warehouse (dbt, Spark SQL)
- **ETL** (traditional) — transform before loading (Informatica, SSIS)
- **Streaming transforms** — process in-flight (Kafka Streams, Flink)

### 5. Serving

Making data available to consumers:

| Consumer | Pattern | Examples |
|----------|---------|---------|
| **Business analysts** | BI dashboards, ad-hoc SQL | Tableau, Looker, Power BI |
| **Data scientists** | Feature stores, notebooks | MLflow, SageMaker, Databricks |
| **Applications** | Reverse ETL, APIs | Census, Hightouch, custom APIs |
| **Operational teams** | Alerts, reports | Email alerts, Slack notifications |

## Undercurrents

Cross-cutting concerns that span all lifecycle stages:

### Security

- Encryption at rest and in transit
- RBAC and least-privilege access
- Data classification (PII, financial, operational)
- Audit logging and access history

### Data Management

- Data quality (testing, validation, monitoring)
- Data governance (cataloguing, lineage, compliance)
- Master data management
- Data privacy (GDPR, CCPA)

### DataOps

- CI/CD for data pipelines
- Infrastructure as code
- Monitoring and alerting
- Incident response

### Orchestration

- DAG-based scheduling (Airflow, Dagster, dbt)
- Dependency management
- Retry and failure handling
- SLA monitoring

### Software Engineering

- Version control
- Code review
- Testing (unit, integration, end-to-end)
- Documentation

## Architecture Patterns

### Data Warehouse

Structured, schema-on-write approach:

```
Sources → ETL/ELT → Data Warehouse → BI Tools
                     (star schemas)
```

**When to use:** Structured data, SQL-centric teams, well-defined business metrics.

### Data Lake

Store everything raw, schema-on-read:

```
Sources → Ingest (raw) → Data Lake (S3/GCS) → Process on demand
                          (any format)
```

**When to use:** Diverse data types, ML workloads, exploration. **Risk:** Can become a "data swamp" without governance.

### Data Lakehouse

Combines lake flexibility with warehouse reliability:

```
Sources → Ingest → Delta Lake / Iceberg → SQL Analytics + ML
                   (ACID, schema enforcement, time travel)
```

**When to use:** Unified analytics and ML, open formats, cost-effective storage. See [[Databricks & Delta Lake]].

### Data Mesh

Decentralized, domain-owned data products:

```
Domain A owns: Sources → Transform → Serve (data products)
Domain B owns: Sources → Transform → Serve (data products)
                                ↕
                    Federated Governance
```

**Principles:**
1. Domain-oriented ownership
2. Data as a product
3. Self-serve data platform
4. Federated computational governance

**When to use:** Large organizations with multiple data-producing domains.

### Lambda Architecture

Parallel batch + streaming paths:

```
Sources → Batch Layer (complete, accurate) ─────→ Serving Layer ← Queries
       → Speed Layer (fast, approximate) ────────↗
```

**Trade-off:** Complete + accurate (batch) vs fast + approximate (speed).
**Problem:** Maintaining two codebases (batch + streaming) for the same logic.

### Kappa Architecture

Streaming-only — eliminates the batch layer:

```
Sources → Stream Processing → Serving Layer ← Queries
          (replay from log when needed)
```

**Advantage:** Single codebase for all processing.
**Requirement:** Kafka or similar log that supports full replay.

## Technology Evaluation Framework

When choosing tools, evaluate along these axes:

| Axis | Questions |
|------|-----------|
| **Team** | What skills does the team have? What's the learning curve? |
| **Speed to market** | How fast can we deliver value? |
| **Interoperability** | Does it work with our existing stack? |
| **Cost** | TCO including license, compute, people, maintenance |
| **Managed vs self-hosted** | Operational burden vs control |
| **Community** | Documentation, support, hiring pool |
| **Lock-in** | Open formats/standards vs proprietary |
| **Scale** | Current needs and 2-year growth projection |

**Principle:** Choose boring technology when possible. Only adopt new tools when they solve a problem your current stack cannot.

---

**Related:** [[Data Mesh & Domain-Driven Data]] | [[Data Contracts & Schema Enforcement]] | [[Common Data Engineering Patterns]] | [[Data Ingestion Patterns]] | [[Apache Airflow & Orchestration Patterns]]
