tags: #interview #system-design #data-engineering #architecture #trade-offs

# System Design for Data Engineers

This guide covers how to approach data engineering system design interviews, with five common problems and structured solutions. The framework and examples are tailored to data platform and pipeline design rather than general software system design.

---

## Interview Framework

Every data engineering system design question should be approached with a consistent framework. Spend the first 5-10 minutes on requirements before drawing any architecture.

### Step 1: Requirements Gathering (5-10 Minutes)

**Functional requirements** -- what does the system need to do?
- What data sources exist? (batch files, streaming events, APIs, databases)
- What are the output consumers? (dashboards, ML models, APIs, downstream teams)
- What transformations are needed? (aggregations, joins, enrichment, deduplication)
- What query patterns will consumers use? (ad hoc SQL, pre-aggregated metrics, key-value lookups)

**Non-functional requirements** -- how does it need to perform?
- **Latency** -- real-time (seconds), near-real-time (minutes), or batch (hours)?
- **Volume** -- how much data per day? Growth rate?
- **Freshness** -- how stale can data be before it is unacceptable?
- **Availability** -- what is the SLA? Can consumers tolerate downtime?
- **Consistency** -- is eventual consistency acceptable, or do consumers need strong consistency?
- **Compliance** -- PII handling, data residency, retention requirements?

**Clarifying questions to ask:**
- "How many concurrent users will query the output?"
- "What is the expected data growth rate over the next 2 years?"
- "Are there any existing systems this needs to integrate with?"
- "What is the team's technical profile -- SQL-proficient, Python engineers, or mixed?"

### Step 2: Data Model (5 Minutes)

- Sketch the key entities and their relationships
- Identify grain (what does one row represent?)
- Decide on modelling approach: star schema, wide denormalised tables, or event-based
- Note slowly changing dimensions and how to handle them

### Step 3: Pipeline Architecture (10-15 Minutes)

- Draw the end-to-end data flow: sources, ingestion, storage, transformation, serving
- Choose the medallion/layered architecture pattern (raw/staging/curated or bronze/silver/gold)
- Select technologies for each layer with brief justification
- Identify orchestration and scheduling approach
- Address data quality and validation points

### Step 4: Scale (5 Minutes)

- Identify bottlenecks at current and 10x scale
- Discuss partitioning, clustering, and indexing strategies
- Address compute scaling (auto-scaling, serverless, reserved capacity)
- Consider cost implications of scaling choices

### Step 5: Trade-Offs (5 Minutes)

- Explicitly name 2-3 trade-offs you made and why
- Discuss alternatives you considered and why you rejected them
- Mention operational concerns: monitoring, alerting, incident response
- Address what you would do differently with more time or budget

---

## Problem 1: Design a Real-Time Analytics Pipeline

### Scenario

A digital media company needs to analyse clickstream data from their website and mobile apps. Product managers need dashboards showing page views, session duration, conversion funnels, and A/B test results updated within 60 seconds of user activity.

### Requirements Gathering

- **Sources**: Clickstream events from web (JavaScript SDK) and mobile (iOS/Android SDKs), ~50,000 events/second at peak
- **Consumers**: Product managers via dashboards, data scientists via SQL for ad hoc analysis
- **Freshness**: Dashboard metrics updated within 60 seconds
- **Retention**: Raw events retained for 2 years, aggregates retained indefinitely
- **Scale**: 4 billion events/day, growing 30% annually

### Architecture

```
[Web/Mobile SDKs] --> [Event Collector API] --> [Kafka] --> [Stream Processor] --> [Serving Layer]
                                                  |               |
                                                  v               v
                                            [Object Storage]  [OLAP Engine]
                                            (raw archive)     (dashboards)
```

**Ingestion layer**: Events flow from SDKs to an event collector (API Gateway + Lambda or a dedicated service) that validates schema, enriches with server-side metadata (timestamp, geo-IP), and publishes to [[Apache Kafka Deep Dive|Kafka]] topics partitioned by `session_id`.

**Stream processing**: [[PySpark Structured Streaming|Spark Structured Streaming]] or Flink consumes from Kafka, performing:
- Sessionisation (group events into sessions using a 30-minute inactivity window)
- Deduplication (exactly-once via event IDs)
- Pre-aggregation of key metrics per minute (page views, unique users, conversions)

**Storage**:
- **Raw archive**: All events land in object storage (S3/ADLS) partitioned by `date/hour` in Parquet format for long-term retention and reprocessing
- **Aggregated metrics**: Pre-computed minute-level aggregates stored in an OLAP engine ([[Snowflake Architecture & Key Concepts|Snowflake]], ClickHouse, or Druid) for dashboard queries
- **Session-level data**: Completed sessions written to Delta Lake for ad hoc SQL analysis

**Serving**: Dashboards query the OLAP engine for pre-aggregated metrics. Ad hoc queries run against Delta Lake via a SQL warehouse.

### Technology Choices

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Message broker | Kafka | High throughput, exactly-once semantics, replay capability |
| Stream processor | Spark Structured Streaming | Team familiarity, integration with Delta Lake |
| Raw storage | S3 + Parquet | Cost-effective, schema-on-read, partitioned for efficient scans |
| OLAP | ClickHouse or Snowflake | Sub-second aggregation queries at scale |
| Orchestration | [[Databricks Modern Patterns (2025)#Databricks Workflows\|Databricks Workflows]] | Manages streaming jobs and batch backfills |

### Scaling Considerations

- Kafka partitions scale horizontally; increase partitions as event volume grows
- Spark Structured Streaming auto-scales executors based on processing lag
- OLAP engine pre-aggregation bounds query cost regardless of raw data volume
- Cold storage (S3) costs are negligible compared to compute

### Follow-Up Questions

- "How would you handle late-arriving events?"
- "How would you backfill the aggregates if the stream processor fails for 2 hours?"
- "How would you support A/B test analysis with statistical significance calculations?"
- "What monitoring would you add to detect data quality issues in the stream?"

---

## Problem 2: Design a Data Lake

### Scenario

A retail company is migrating from an on-premises data warehouse to a cloud data lake. They have 50+ source systems (ERP, CRM, POS, e-commerce, logistics) and need to support analytics, reporting, and machine learning workloads.

### Requirements Gathering

- **Sources**: 50+ systems, mix of databases (CDC), APIs, and file drops
- **Consumers**: BI analysts (SQL), data scientists (Python/Spark), operational dashboards
- **Volume**: 2 TB/day incremental, 500 TB historical
- **Freshness**: Operational dashboards need hourly refresh; analytics can be daily
- **Governance**: PII must be masked for non-privileged users; full audit trail required

### Architecture

```
[Source Systems] --> [Ingestion Layer] --> [Bronze] --> [Silver] --> [Gold]
                                            |            |           |
                                         Raw files    Cleansed    Business
                                         as-is        conformed   aggregates
                                            |            |           |
                                         [Unity Catalog / Governance Layer]
```

**Ingestion layer**:
- **Database CDC**: Debezium or AWS DMS captures change events from transactional databases into Kafka, then lands as Delta tables in Bronze
- **API sources**: Fivetran or Airbyte for SaaS connectors (Salesforce, Shopify, etc.)
- **File drops**: Auto Loader monitors S3 landing zones for CSV/JSON files from legacy systems

**Medallion architecture**:
- **Bronze**: Raw data preserved exactly as received. Schema-on-read, append-only. Partitioned by `ingestion_date`. No transformations except adding metadata (`_source`, `_loaded_at`, `_file_name`).
- **Silver**: Cleansed, deduplicated, conformed. Standardised column names, data types, and date formats. Business key deduplication applied. SCD Type 2 for slowly changing dimensions. Schema enforced.
- **Gold**: Business-specific aggregates and denormalised tables optimised for query patterns. `fct_sales`, `dim_customer`, `dim_product`. Materialised as Delta tables with clustering.

**Governance**:
- [[Databricks Modern Patterns (2025)#Unity Catalog Governance|Unity Catalog]] provides the single governance layer across all data assets
- Column-level tagging for PII with dynamic masking policies
- Row-level security for multi-tenant data (regional access restrictions)
- System tables for audit logging and lineage tracking

### Technology Choices

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Storage | Delta Lake on S3/ADLS | ACID transactions, schema evolution, time travel |
| Ingestion (CDC) | Debezium + Kafka | Real-time CDC with full change history |
| Ingestion (SaaS) | Fivetran | Managed connectors, minimal maintenance |
| Transformation | dbt + Databricks SQL | SQL-first, testable, version-controlled |
| Governance | Unity Catalog | Unified catalogue, lineage, access control |
| Orchestration | Airflow | Cross-platform orchestration with dependency management |

### Scaling Considerations

- Delta Lake auto-compaction and optimised writes handle small file problems at scale
- Partition pruning and Z-ordering reduce scan volume for large tables
- Separate compute clusters for ingestion, transformation, and serving prevent resource contention
- Storage costs grow linearly; compute scales independently

### Follow-Up Questions

- "How would you handle schema evolution when a source system adds columns?"
- "How would you implement data quality gates between Silver and Gold?"
- "How would you manage cost allocation across business units?"
- "How would you support self-service analytics for non-technical users?"

---

## Problem 3: Design a CDC Pipeline

### Scenario

A financial services company needs to replicate transactional data from their PostgreSQL OLTP database to a Snowflake data warehouse with minimal latency. The source database handles 10,000 transactions/second and contains sensitive customer data.

### Requirements Gathering

- **Source**: PostgreSQL 15, 200+ tables, 10,000 TPS
- **Target**: [[Snowflake Architecture & Key Concepts|Snowflake]] for analytics
- **Latency**: Data available in Snowflake within 5 minutes of source commit
- **Consistency**: Transactional consistency within each table; cross-table consistency is eventual
- **Compliance**: PII must be encrypted in transit and masked in Snowflake for non-privileged roles
- **History**: Full change history retained for audit (who changed what, when)

### Architecture

```
[PostgreSQL] --> [Debezium] --> [Kafka] --> [Stream Processor] --> [Snowflake]
   (WAL)        (CDC agent)    (events)    (transform/route)      (warehouse)
                                  |
                              [Schema Registry]
```

**Change capture**: Debezium reads PostgreSQL's write-ahead log (WAL) via logical replication slots. Each row change (insert, update, delete) is emitted as a structured event with before/after images.

**Message broker**: Kafka receives CDC events, one topic per source table. Schema Registry (Avro) enforces schema compatibility. Events are keyed by primary key for ordering guarantees within a partition.

**Stream processing**: A lightweight stream processor (Kafka Connect with Snowflake Sink, or Spark Structured Streaming) applies:
- PII encryption/tokenisation before landing in Snowflake
- Schema mapping (source column names to target conventions)
- Dead-letter queue routing for malformed events

**Target loading**: Snowflake Snowpipe Streaming or micro-batch COPY INTO loads data into a raw/staging schema. A scheduled [[Core dbt Fundamentals|dbt]] pipeline then:
- Applies MERGE for current-state tables (SCD Type 1)
- Maintains history tables with full change audit trail (SCD Type 2 via dbt snapshots)
- Runs data quality tests before promoting to production schemas

### Technology Choices

| Layer | Technology | Justification |
|-------|-----------|---------------|
| CDC | Debezium | Open-source, WAL-based (no source impact), full change events |
| Broker | Kafka + Schema Registry | Ordering guarantees, schema evolution, replay |
| Sink | Snowflake Kafka Connector or Snowpipe Streaming | Native Snowflake integration, micro-batch loading |
| Transformation | dbt | SQL-based MERGE/snapshot logic, testable, version-controlled |
| PII handling | Tokenisation service + Snowflake masking policies | Encrypt in transit, mask at query time |

### Scaling Considerations

- Kafka partitions per table scale with transaction volume; high-throughput tables get more partitions
- Snowpipe Streaming handles continuous micro-batch loading without warehouse compute
- dbt incremental models process only new/changed records, bounding transformation cost
- WAL-based CDC adds minimal load to the source database (no trigger-based overhead)

### Follow-Up Questions

- "How would you handle schema changes in the source database?"
- "What happens if Kafka is unavailable for 30 minutes -- how do you recover?"
- "How would you validate that source and target are in sync?"
- "How would you handle deletes -- soft deletes, hard deletes, or both?"

---

## Problem 4: Design a Recommendation Engine Data Platform

### Scenario

An e-commerce company wants to build a recommendation system. Data engineers need to design the platform that ingests user behaviour, computes features, serves features to the ML model, and delivers recommendations to users in real-time.

### Requirements Gathering

- **Sources**: User clickstream (real-time), purchase history (batch), product catalogue (batch), user profiles (batch)
- **Consumers**: Recommendation model (training and real-time inference), A/B testing framework
- **Latency**: Feature retrieval for inference must be < 50ms; model training can be batch (daily)
- **Scale**: 10 million active users, 500,000 products, 100 million events/day
- **Freshness**: User behaviour features updated within 5 minutes; product features updated daily

### Architecture

```
[Clickstream] --> [Kafka] --> [Feature Pipeline] --> [Feature Store] --> [Model Serving]
                                    |                   |      |              |
[Batch Sources] --> [Batch ETL] ----+            [Offline]  [Online]    [Recommendations]
                                    |                |                       |
                                    v                v                       v
                              [Training Data]  [Model Training]         [User API]
```

**Feature engineering**:

| Feature Category | Examples | Update Frequency | Computation |
|-----------------|----------|-----------------|-------------|
| User real-time | Items viewed last 30 min, cart contents | Streaming (5 min) | Spark Structured Streaming |
| User historical | Purchase count, avg order value, category affinity | Daily batch | dbt or Spark batch |
| Product | Price, category, avg rating, popularity score | Daily batch | dbt |
| Interaction | User-item co-occurrence, collaborative filtering signals | Daily batch | Spark/MLlib |

**Feature Store**: [[Databricks Modern Patterns (2025)#Databricks Feature Store|Databricks Feature Store]] or Feast manages both offline (Delta Lake) and online (DynamoDB/Redis) feature tables. Point-in-time lookups for training prevent data leakage.

**Model training**: Daily batch job reads training data from the offline feature store with point-in-time correctness. Model trained in Databricks/SageMaker, registered in [[Databricks Modern Patterns (2025)#MLflow on Databricks|MLflow Model Registry]].

**Model serving**: Deployed as a REST endpoint via Databricks Model Serving or SageMaker. On inference request:
1. Receive user ID from the application
2. Fetch real-time + historical features from the online feature store (< 10ms)
3. Run inference (< 30ms)
4. Return ranked recommendations

**A/B testing**: Traffic splitting between model versions. Inference logs capture user ID, model version, recommendations returned, and subsequent user actions for offline evaluation.

### Technology Choices

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Real-time ingestion | Kafka + Spark Structured Streaming | Low-latency feature updates |
| Feature store | Databricks Feature Store | UC integration, point-in-time lookups, online/offline sync |
| Online store | DynamoDB | Single-digit millisecond reads at scale |
| Model registry | MLflow | Versioning, lineage, stage management |
| Model serving | Databricks Model Serving | Auto-scaling, traffic splitting, inference logging |

### Scaling Considerations

- Online feature store must handle 10,000+ reads/second at < 10ms (DynamoDB scales horizontally)
- Feature computation is the bottleneck -- pre-compute and cache rather than compute on-demand
- Model serving auto-scales based on request volume; consider GPU instances for complex models
- Cold-start problem for new users: fall back to popularity-based recommendations

### Follow-Up Questions

- "How would you handle the cold-start problem for new users with no history?"
- "How would you evaluate whether the recommendation model is actually improving business metrics?"
- "How would you retrain the model when user behaviour shifts significantly?"
- "How would you ensure feature consistency between training and serving?"

---

## Problem 5: Design a Multi-Tenant Data Platform

### Scenario

A B2B SaaS company needs a data platform that serves analytics to 500+ enterprise customers (tenants). Each tenant should see only their own data, costs should be attributable per tenant, and the platform must comply with data residency requirements for European tenants.

### Requirements Gathering

- **Tenants**: 500+ customers, ranging from 1 GB to 10 TB of data each
- **Isolation**: Strict data isolation -- no tenant can ever see another tenant's data
- **Compliance**: EU tenants' data must reside in EU regions (GDPR)
- **Cost allocation**: Platform costs attributable to individual tenants for pricing/billing
- **Scale**: Total 2 PB, growing 50% annually
- **Self-service**: Tenants can build custom dashboards and run ad hoc queries

### Architecture

```
[Tenant Applications] --> [Ingestion API] --> [Tenant Router] --> [Storage]
                                                                      |
                                                   [Shared Compute Layer]
                                                          |
                                                   [Tenant Dashboards]
```

**Isolation model** -- three approaches with trade-offs:

| Model | Isolation | Cost Efficiency | Operational Complexity |
|-------|-----------|----------------|----------------------|
| **Separate databases per tenant** | Strongest | Low (duplicate infrastructure) | High (500+ databases) |
| **Shared database, separate schemas** | Strong | Medium | Medium |
| **Shared tables with tenant_id column** | Moderate | Highest | Lowest |

**Recommended hybrid approach**: Shared tables with `tenant_id` column for most data, with row-level security policies enforcing isolation. Large enterprise tenants get dedicated schemas if they require it. EU tenants' data in an EU-region deployment.

**Data isolation implementation (Snowflake)**:
- Row access policies filter by `tenant_id` based on the querying user's role
- Each tenant's service account is mapped to a role that includes the tenant ID
- Views with `SECURE` modifier prevent policy bypass via query optimiser tricks

```sql
CREATE OR REPLACE ROW ACCESS POLICY tenant_isolation AS (tenant_id VARCHAR)
RETURNS BOOLEAN ->
    tenant_id = CURRENT_ROLE()::VARCHAR
    OR IS_ROLE_IN_SESSION('PLATFORM_ADMIN');
```

**Cost allocation**:
- Tag all queries with `tenant_id` via session variables: `ALTER SESSION SET QUERY_TAG = 'tenant_id=acme_corp'`
- Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` to attribute compute credits per tenant
- Storage attribution via `TABLE_STORAGE_METRICS` filtered by tenant partition
- Separate warehouses for large tenants; shared multi-cluster warehouse for small tenants

**Data residency**:
- EU deployment in `eu-west-1` / `eu-central-1` with separate Snowflake account
- Cross-region replication for shared reference data (product catalogue, configuration)
- Tenant onboarding process assigns region based on customer contract

**Governance**:
- Unity Catalog or Snowflake RBAC enforces access control
- Audit logging captures all data access per tenant for compliance reporting
- Data retention policies configurable per tenant (contractual requirements vary)

### Technology Choices

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Storage | Snowflake (multi-account for regions) | Row access policies, cost attribution, scaling |
| Ingestion | Fivetran or custom API | Per-tenant connector configuration |
| Transformation | dbt with tenant-aware macros | Parameterised models generating per-tenant views |
| Dashboards | Embedded Looker or Metabase | Row-level security integration, white-labelling |
| Orchestration | Airflow with tenant-parameterised DAGs | Isolated execution per tenant |

### Scaling Considerations

- Row access policies add overhead to every query; benchmark at 500+ tenants to ensure acceptable latency
- Shared warehouses risk noisy-neighbour problems; monitor per-tenant query times and isolate heavy users
- Tenant onboarding must be automated (Terraform/API) to support 500+ tenants without manual steps
- Data growth varies wildly by tenant; storage cost allocation must handle this heterogeneity

### Follow-Up Questions

- "How would you handle a tenant requesting deletion of all their data (right to be forgotten)?"
- "How would you prevent a single large tenant's queries from affecting other tenants' performance?"
- "How would you support tenant-specific custom fields or schemas?"
- "What monitoring would you implement to detect data leakage between tenants?"

---

## General Tips for Data Engineering System Design Interviews

- **Always start with requirements** -- do not jump to technology choices. Interviewers want to see structured thinking.
- **Name trade-offs explicitly** -- "I chose X over Y because of Z, accepting the downside of W."
- **Draw before you talk** -- sketch the architecture, then walk through it. A clear diagram communicates more than verbal description.
- **Discuss failure modes** -- what happens when each component fails? How do you detect and recover?
- **Mention observability** -- monitoring, alerting, and logging are not afterthoughts. Include them in the architecture.
- **Consider the team** -- technology choices should match team capabilities. The best architecture is one the team can operate.
- **Quantify where possible** -- "50,000 events/second requires N Kafka partitions" is stronger than "Kafka can handle it."
- **Know your depth** -- be prepared to go deep on any technology you mention. Do not name-drop tools you cannot discuss in detail.

---

**Related:** [[Snowflake Interview Guide - Lead Data Engineer]] | [[PySpark Lead Data Engineer Interview Questions & Answers]] | [[SQL for Lead Data Engineer - Interview Guide]] | [[dbt Data Engineer Interview Guide]] | [[Databricks Data Engineer Associate Study Guide]]
