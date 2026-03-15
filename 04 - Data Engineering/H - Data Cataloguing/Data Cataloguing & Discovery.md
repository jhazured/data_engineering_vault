# Data Cataloguing & Discovery

## Why Data Cataloguing

A data catalogue is the central inventory of an organisation's data assets. Without one, analysts spend up to 30% of their time just finding and understanding data.

| Driver | What It Solves |
|--------|---------------|
| **Discoverability** | Find tables, columns, and metrics without asking around |
| **Governance** | Enforce ownership, access policies, and classification consistently |
| **Data Lineage** | Trace data from source to dashboard — debug issues in minutes, not days |
| **Compliance** | GDPR, CCPA, HIPAA — know where PII lives and who accessed it |
| **Self-Service Analytics** | Analysts query confidently without engineering hand-holding |
| **Data Democratisation** | Every team gets equal, governed access to trusted data |

A catalogue is the foundation that makes [[Snowflake RBAC & Data Security]] enforceable and [[Pipeline Observability & Monitoring]] meaningful.

---

## Core Capabilities

- **Technical metadata** — column names, data types, partitioning, row counts, storage format
- **Business metadata** — descriptions, owners, domain, SLA, refresh cadence
- **Operational metadata** — last run time, query frequency, cost per table
- **Column-level lineage** — trace how a field moves from source through staging, integration, to presentation
- **Data profiling** — automated stats (nulls, cardinality, min/max, distribution) feeding [[Data Validation & Quality Frameworks]]
- **Search & discovery** — full-text search across names, descriptions, tags; ranked by usage and freshness
- **Access control** — RBAC integration with audit trails
- **Tagging & classification** — PII, financial, operational labels driving masking and retention policies

---

## Unity Catalog (Databricks)

[[Databricks & Delta Lake]]'s built-in governance layer with a three-level namespace:

```
catalog.schema.table → analytics.logistics.dim_customers
```

| Level | Purpose | Example |
|-------|---------|---------|
| **Catalog** | Top-level container (environment or domain) | `prod`, `dev` |
| **Schema** | Logical grouping | `logistics`, `finance` |
| **Table** | The data asset | `dim_customers` |

**Managed vs external tables** — managed tables are deleted when dropped; external tables (`LOCATION 's3://...'`) leave files intact.

**Lineage** — captured automatically for Spark SQL, DataFrame ops, and dbt models. Visible at table and column level in the UI.

**Fine-grained permissions** — row filters and column masks as SQL functions:

```sql
-- Row filter: restrict by region
ALTER TABLE fact_shipments SET ROW FILTER region_filter ON (region);

-- Column mask: hash PII for non-privileged roles
ALTER TABLE dim_customers ALTER COLUMN email SET MASK mask_email;
```

**Delta Sharing** — open protocol for cross-organisation data sharing without copying. Recipients read with Spark, pandas, or any Delta Sharing client.

---

## DataHub (LinkedIn OSS)

Open-source metadata platform supporting tables, dashboards, pipelines, and ML models.

### Architecture

```
Frontend (React) → Metadata Service (GMS — GraphQL + REST)
  → Metadata Store (MySQL/Postgres) + Search (Elasticsearch)
  → Kafka (change log) → Ingestion Framework (Python)
```

**Entities and aspects** — metadata modelled as entities (datasets, dashboards, users) with versioned aspects (ownership, schema, lineage).

### Ingestion Example (Snowflake)

```yaml
source:
  type: snowflake
  config:
    account_id: "org-account"
    username: "${SNOWFLAKE_USER}"
    password: "${SNOWFLAKE_PASSWORD}"
    database_pattern: { allow: ["PROD_.*"] }
    profiling: { enabled: true }
sink:
  type: datahub-rest
  config: { server: "http://datahub-gms:8080" }
```

| Source | Captures |
|--------|----------|
| **Snowflake** | Tables, views, column stats, usage, lineage |
| **dbt** | Models, tests, descriptions, lineage from manifest |
| **Airflow** | DAGs, tasks, run history, dependencies |
| **Kafka** | Topics, schemas, consumer groups |
| **BigQuery** | Datasets, tables, audit log lineage |

**GraphQL API** — query any entity's metadata, ownership, schema, and upstream/downstream lineage programmatically.

---

## OpenMetadata

Open-source catalogue with built-in data quality, lineage, and collaboration.

**Architecture** — Java API server + Elasticsearch + MySQL/Postgres + Airflow (ingestion orchestration).

**Data quality integration** — define tests (uniqueness, not-null, row count bounds) directly in the catalogue; results appear alongside metadata.

**Glossary** — business terms linked to columns across tables (e.g., "Customer Lifetime Value" maps to `dim_customers.clv` and `rpt_customer_value.lifetime_value`).

**Custom properties** — extend metadata with domain fields (`data_classification`, `refresh_sla`, `cost_centre`).

**Connectors** — Snowflake, BigQuery, Redshift, dbt, [[Apache Kafka Fundamentals|Kafka]], Airflow, Tableau, Looker, and 50+ sources.

---

## Atlan

Commercial active metadata platform designed for collaboration-first cataloguing.

- **Persona-based discovery** — engineers, analysts, and business users each see relevant metadata
- **Active metadata** — automated actions on metadata events (notify on SLA breach, auto-classify PII)
- **Cross-platform lineage** — Snowflake, dbt, Airflow, Tableau, Looker
- **Governance policies** — purpose-based access control, auto-masking, data contracts
- **Integrations** — Slack/Teams, Jira, embedded in BI tools

Best for organisations wanting managed, low-ops cataloguing with strong collaboration. Trade-off is vendor lock-in and per-seat licensing.

---

## Snowflake Native Cataloguing

Built-in cataloguing through system views and governance features. See [[Snowflake RBAC & Data Security]].

### ACCOUNT_USAGE Views

```sql
-- Table metadata
SELECT table_catalog, table_schema, table_name, row_count, bytes, last_altered
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL;

-- Column-level access audit
SELECT user_name, query_start_time,
       obj.value:objectName::STRING AS table_name,
       col.value:columnName::STRING AS column_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
     LATERAL FLATTEN(direct_objects_accessed) obj,
     LATERAL FLATTEN(obj.value:columns) col
WHERE query_start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP());
```

### TAG-Based Classification

```sql
CREATE TAG IF NOT EXISTS pii_classification
  ALLOWED_VALUES 'PII_HIGH', 'PII_LOW', 'FINANCIAL', 'OPERATIONAL';

ALTER TABLE dim_customers ALTER COLUMN customer_name
  SET TAG pii_classification = 'PII_HIGH';
```

### Horizon Governance

Bundles cataloguing features: Universal Search, automatic PII detection, object-level lineage (Enterprise), DMFs for quality monitoring, and secure data sharing via listings.

---

## dbt as a Catalogue

dbt generates documentation that functions as a lightweight catalogue. See [[Core dbt Fundamentals]].

```bash
dbt docs generate   # Build from manifest + catalog
dbt docs serve      # Serve searchable site with DAG visualisation
```

### Model Descriptions (schema.yml)

```yaml
models:
  - name: dim_customers
    description: "Customer dimension. Grain: one row per customer. Refreshed daily 06:00 AEST."
    columns:
      - name: customer_id
        description: "Surrogate key (SHA-256 hash)"
        tests: [unique, not_null]
      - name: customer_name
        description: "Full legal name from ERP. PII_HIGH."
```

### Exposures

Extend lineage beyond dbt to downstream consumers:

```yaml
exposures:
  - name: logistics_dashboard
    type: dashboard
    url: https://tableau.company.com/views/logistics
    depends_on: [ref('fact_shipments'), ref('dim_customers')]
    owner: { name: Analytics Team, email: analytics@company.com }
```

**dbt Cloud Explorer** — hosted catalogue with full-text search, column-level lineage, cross-project references, and per-model performance metrics.

---

## Implementation Patterns

### Metadata Ingestion Pipelines

Treat metadata ingestion like data ingestion — schedule, monitor, alert on failures. A typical Airflow DAG runs parallel tasks for Snowflake metadata, dbt manifest parsing, and Airflow DAG extraction, then stitches cross-platform lineage.

### Automated Lineage Capture

- **dbt** — parse `manifest.json` for model lineage and `catalog.json` for column metadata
- **Snowflake** — `ACCESS_HISTORY` provides column-level read/write lineage
- **Airflow** — inlets/outlets on tasks declare data dependencies
- **SQL parsing** — tools like sqlglot extract lineage from raw SQL when no native lineage exists

### Business Glossary Design

| Term | Definition | Owner | Linked Assets |
|------|-----------|-------|---------------|
| Customer Lifetime Value | Total revenue over customer relationship | Finance | `dim_customers.clv` |
| On-Time Delivery Rate | % shipments delivered by promised date | Logistics | `fact_shipments.is_on_time` |
| Active Customer | At least one order in last 12 months | Sales | `dim_customers.is_active` |

### Data Stewardship Workflows

1. **New table registered** — auto-assign owner based on schema/database mapping
2. **Owner reviews** — add descriptions, classify columns, set SLA
3. **Quality tests defined** — link tests from [[dbt Testing & Data Quality]] to catalogue entries
4. **Ongoing maintenance** — quarterly review of stale assets, orphaned tables, unused columns

### Catalogue-First Development

Before writing any new model: check if data exists, identify upstream owners, verify tests/SLAs, and confirm governance constraints. Prevents duplicate models and respects existing contracts.

---

## Choosing a Catalogue

### Comparison Table

| Feature | Unity Catalog | DataHub | OpenMetadata | Atlan | Snowflake Native | dbt Docs |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| **Licence** | Databricks | OSS | OSS | Commercial | Snowflake | Free / Cloud |
| **Multi-Platform Lineage** | Databricks only | Yes | Yes | Yes | Snowflake only | dbt only |
| **Column-Level Lineage** | Yes | Yes | Yes | Yes | Enterprise | Cloud |
| **Data Quality** | Partner tools | Integrations | Built-in | Integrations | DMFs | Via tests |
| **Business Glossary** | Basic | Yes | Yes | Yes | Tags only | Descriptions |
| **Search UX** | Good | Good | Good | Excellent | Basic | Basic |
| **Setup Complexity** | None | Medium-High | Medium | None (SaaS) | None | Low |

### Decision Factors

**Already on Databricks?** Start with Unity Catalog. Add DataHub or Atlan for cross-platform coverage.

**Snowflake-centric?** Use native features (tags, ACCESS_HISTORY, Horizon) for governance. Pair with dbt docs for business context. Graduate to OpenMetadata or DataHub when you outgrow native capabilities.

**Multi-cloud / multi-warehouse?** DataHub (stronger community) or OpenMetadata (better built-in quality) for vendor-neutral cataloguing.

**Enterprise with budget?** Atlan provides the best out-of-the-box experience. Weigh per-seat licensing against self-hosting OSS.

**dbt-heavy team?** Start with dbt docs and [[dbt Tag & Execution Strategy|dbt tags]]. Exposures extend lineage to dashboards. Upgrade when you need access governance or cross-tool lineage.

**Budget-constrained?** Combine dbt docs + Snowflake native + lightweight DataHub — covers 80% of needs at minimal cost.
