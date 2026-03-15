tags: #databricks #delta-live-tables #unity-catalog #workflows #schema-management #data-engineering #2025

# Databricks Modern Patterns (2025)

Source: *The Big Book of Data Engineering*, 3rd Edition (Databricks, January 2025). This note covers platform capabilities and production patterns beyond the fundamentals documented in [[Databricks & Delta Lake]].

---

## Delta Live Tables (DLT)

DLT is a declarative ETL framework that handles task orchestration, cluster management, monitoring, data quality and error handling automatically. Engineers define *what* transformations to apply; DLT determines *how* to execute them, managing checkpoint locations, dependency resolution and the pipeline execution graph.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Streaming Table** | Created via `dlt.read_stream()` -- incrementally processes new data |
| **Materialized View** | Created via `dlt.read()` -- periodically recalculates aggregates |
| **View** | Created via `@dlt.view` -- ephemeral, not persisted to storage |
| **Pipeline DAG** | Automatically inferred from table dependencies |

DLT supports both Python and SQL APIs. The framework manages checkpointing out-of-the-box, removing the need for explicit `checkpointLocation` configuration that Structured Streaming requires.

### Expectations (Data Quality)

Expectations are declarative data quality constraints applied to DLT tables. Three enforcement levels control how violations are handled:

| Decorator | Behaviour on Violation |
|-----------|----------------------|
| `@dlt.expect` | Warn and retain the record |
| `@dlt.expect_or_drop` | Silently drop the record |
| `@dlt.expect_or_fail` | Halt the entire pipeline |

Batch variants (`expect_all`, `expect_all_or_drop`, `expect_all_or_fail`) accept dictionaries of multiple constraints:

```python
@dlt.expect_all({
    "valid_claim_amount": "total_claim_amount > 0",
    "valid_coverage": "months_since_covered > 0",
})
@dlt.expect_all_or_drop({
    "valid_claim_number": "claim_number IS NOT NULL",
    "valid_policy_number": "policy_number IS NOT NULL",
    "valid_claim_date": "claim_date < current_date()",
})
def curate_claims():
    ...
```

Expectation results are logged to the DLT event log and can be queried for reporting:

```sql
SELECT
  row_expectations.dataset AS dataset,
  row_expectations.name AS expectation,
  SUM(row_expectations.passed_records) AS passing_records,
  SUM(row_expectations.failed_records) AS failing_records
FROM (
  SELECT explode(from_json(
    details:flow_progress:data_quality:expectations,
    "array<struct<name:string, dataset:string, passed_records:int, failed_records:int>>"
  )) row_expectations
  FROM event_log_raw
  WHERE event_type = 'flow_progress'
    AND origin.update_id = '${latest_update.id}'
)
GROUP BY row_expectations.dataset, row_expectations.name;
```

### Materialized Views and Aggregates

For continuously running streams, resource-intensive aggregates (e.g. median over an unbounded stream) are impractical to recalculate on every arriving record. DLT addresses this by materialising results only periodically, with a configurable trigger interval via a table property. The developer does not need to manage this logic -- the declarative framework handles it by design.

```python
def build_gold(gname):
    @dlt.table(name=f"gold_{gname}_player_agg")
    def gold_agg():
        return (
            dlt.read(f"silver_{gname}_events")
            .groupBy(["gamer_id"])
            .agg(
                F.count("*").alias("session_count"),
                F.min(F.col("event_timestamp")).alias("min_timestamp"),
                F.max(F.col("event_timestamp")).alias("max_timestamp"),
            )
        )
```

### Metadata-Driven Table Generation

DLT pipelines can dynamically create tables from configuration. A single pipeline can produce 1 + 2N tables (one Bronze, N Silver Streaming Tables, N Gold Materialized Views):

```python
GAMES_ARRAY = spark.conf.get("games").split(",")
for game in GAMES_ARRAY:
    build_silver(game)
    build_gold(game)
```

### Table Properties for Auto-Optimisation

```python
@dlt.table(
    name="curated_claims",
    comment="Curated claim records",
    table_properties={
        "layer": "silver",
        "pipelines.autoOptimize.managed": "true",
        "delta.autoOptimize.optimizeWrite": "true",
        "delta.autoOptimize.autoCompact": "true",
    },
)
```

### DLT and Schema Evolution

When a source stream's schema evolves, DLT detects the change and restarts the pipeline. In Production mode, this restart is automatic. Upon restart, the latest schema definitions are retrieved (e.g. from Confluent Schema Registry) and the stream continues with the evolved schema.

### DLT with Protobuf and Schema Registry

Databricks provides native `from_protobuf` / `to_protobuf` support (Runtime 12.1+), integrating with Confluent Schema Registry. This eliminates the need for manual `protoc` compilation:

```python
from pyspark.sql.protobuf.functions import from_protobuf

@dlt.view
def bronze_events():
    return (
        spark.readStream.format("kafka")
        .options(**kafka_options)
        .load()
        .withColumn("decoded", from_protobuf(
            F.col("value"), options=schema_registry_options
        ))
        .selectExpr("decoded.*")
    )
```

---

## DLT DevOps Best Practices

### Code Structure

Separate transformation logic from DLT pipeline definitions. Transformations live in standalone Python modules that receive and return Spark DataFrames; DLT notebooks import and call these functions. This enables:

- Unit testing transformations locally with pytest (no Databricks cluster required)
- Interactive testing from notebooks on interactive clusters
- Shared transformation libraries across multiple DLT pipelines

Recommended repository layout:

```
repo_root/
  dlt_code/          # DLT pipeline notebooks
  transformations/   # Python modules (pure Spark logic)
  tests/
    unit/            # pytest-based, runnable locally
    integration/     # DLT expectations or Workflow-based
  terraform/         # Infrastructure deployment
```

### Integration Testing

Two approaches:

1. **Databricks Workflows** -- multi-task workflow: setup test data, execute pipeline, validate results. Requires additional compute and auxiliary code.
2. **DLT Expectations (recommended)** -- extend the pipeline with additional DLT tables that apply `expect_all_or_fail` to validate results. No extra compute needed; everything runs in the same pipeline and results are logged to the event log.

```python
@dlt.table(comment="Check type")
@dlt.expect_all_or_fail({
    "valid type": "type in ('link', 'redlink')",
    "type is not null": "type is not null",
})
def filtered_type_check():
    return dlt.read("clickstream_filtered").select("type")
```

### CI/CD Pipeline

A typical CI/CD pipeline (e.g. Azure DevOps):

- **onPush** (any branch except releases): update staging Databricks Repo, run unit tests (local + notebook-based via Nutter)
- **onRelease** (releases branch): unit tests + DLT integration tests, then release pipeline updates the production Repo

Promotion: DLT separates code from pipeline configuration. The same code is used across environments; pipeline settings (schemas, data locations, cluster sizes) are environment-specific. Terraform's `databricks_pipeline` resource handles deployment with dependency management.

---

## Unity Catalog Governance

Unity Catalog (UC) provides a single governance solution across all data and AI assets, covering multiple clouds and data platforms. See [[Databricks & Delta Lake]] for the three-level namespace fundamentals and [[Data Cataloguing & Discovery]] for broader cataloguing patterns.

### System Tables for Observability

UC exposes governance metadata through queryable system tables:

| System Table Category | Content |
|----------------------|---------|
| **Audit tables** | Actions performed against the metastore -- who accessed what and when |
| **Billing / pricing** | Billable usage records across the entire account |
| **Table lineage** | Read-and-write events on UC tables (job runs, notebook runs, dashboards) |
| **Column lineage** | Column-level data flow tracking |
| **Query history** | SQL commands, I/O performance, rows returned |
| **Clusters** | Full history of cluster configurations |
| **Predictive optimisation** | Compaction and vacuum operation history |
| **Node types / utilisation** | Available hardware configurations and usage metrics |

These tables can be queried with SQL or surfaced in activity dashboards for governance reporting (billing trends, user counts, ML model counts, monitoring coverage percentages).

### Lakehouse Monitoring

Powered by UC, Lakehouse Monitoring enables monitoring of the entire data pipeline -- from data and features to ML models -- without additional tooling. Capabilities include:

- **Data integrity and drift detection** over time
- **ML model performance tracking** (R2, RMSE, MAPE) with accuracy trends
- **Proactive alerts** for errors, data drift, model failures, quality issues and potential PII breaches
- **Root cause analysis** via lineage from table-level to column-level

### Lineage

End-to-end lineage from raw data sources through transformations to models:

- What are the raw data sources?
- Who created the data and when?
- How was data merged and transformed?
- What is the traceability from models back to training datasets?

Lineage is visible both table-level and column-level, and works across data platforms (e.g. Snowflake sources are tracked).

### Metadata Tagging

Column and table descriptions can be entered manually or auto-generated by Databricks Assistant using GenAI. Tags provide contextual insights including frequent users, associated notebooks, query patterns, join relationships and billing trends.

---

## Lakehouse Federation

Federation allows querying external data sources (Snowflake, Azure SQL, Synapse and others) without moving or copying data, governed through Unity Catalog.

### Setup

Three-step process:

1. **Create a connection** -- specify connection type (e.g. Azure SQL) and credentials
2. **Create a foreign catalog** -- register the external database as a UC catalog (type "Foreign")
3. **Query** -- access the federated data as any other UC catalog, including joins with local Delta tables

### When to Use (and When Not To)

**Good fit:** ad hoc exploration, creating a holistic view of the data estate, agile direct-source access.

**Poor fit:** real-time processing (queries are slower than local data), complex transformations at scale (better to ingest and process via medallion architecture).

Federation is an augmentation of analytics capability, not a replacement for ETL pipelines.

---

## Databricks Workflows

Workflows is a fully managed orchestration service native to the Data Intelligence Platform. See [[Databricks & Delta Lake]] for Terraform-based job definitions.

### Task Types

| Task Type | Description |
|-----------|-------------|
| **SQL Query** | Execute queries written in the SQL Editor |
| **SQL Alert** | Trigger notifications based on query conditions |
| **Dashboard** | Refresh dashboards and notify subscribers |
| **File** | Execute `.sql` or `.py` files from a Git repository (latest branch version) |
| **DLT Pipeline** | Trigger a Delta Live Tables pipeline |
| **Notebook** | Run a Databricks notebook |

Tasks can be chained with dependencies or executed in parallel. Task values can pass data between tasks.

### Monitoring and Scheduling

- **Individual run monitoring** -- task outcomes, execution times, bottleneck identification
- **Scheduling** -- cron-based intervals or file-arrival triggers
- **Notifications** -- alerts on success, failure or long-running jobs
- **Automatic retries** -- configurable retry policies per task
- **Serverless compute** -- smart scaling and efficient task execution

### Example Analyst Workflow

A typical Gold-layer refresh workflow:

1. `Create_State_Speed_Records` (SQL Query) -- insert data into Gold table and optimise
2. `Data_Available_Alert` (Alert) -- notify consumers of new records (parallel with step 3)
3. `Update_Dashboard_Dataset` (SQL Query) -- refresh the dataset view feeding the dashboard
4. `Dashboard_Refresh` (Dashboard) -- update visualisations and notify subscribers

---

## Schema Management and Data Drift

Auto Loader (AL) extends Spark Structured Streaming with schema tracking, drift detection and evolution support. See [[Databricks & Delta Lake]] for Auto Loader fundamentals.

### Schema Evolution Modes

| Mode | `schemaEvolutionMode` Value | Behaviour |
|------|----------------------------|-----------|
| **Rescue** | `"rescue"` | Store drifted columns/types in `_rescued_data` JSON column; stream continues |
| **Add New Columns** | `"addNewColumns"` | Merge new columns into schema; fail and restart stream on detection |
| **Enforcement** | `"none"` | Reject non-matching data; stream continues without failure |

### Schema Hints

Override dynamic inference for known columns using SQL DDL syntax:

```python
.option("cloudFiles.schemaHints",
    "coordinates STRUCT<latitude:DOUBLE, longitude:DOUBLE>, "
    "humidity LONG, temp DOUBLE")
```

Hints can be combined with dynamic inference -- enforce known types while letting AL infer unknown ones.

### Dynamic Schema Inference

AL samples the dataset to determine schema without a full scan. Configurable via:

- `spark.databricks.cloudFiles.schemaInference.sampleSize.numBytes` (default 50 GB)
- `spark.databricks.cloudFiles.schemaInference.sampleSize.numFiles` (default 1,000 files)

### Schema Repository

AL stores schema versions, metadata and change history at the `schemaLocation` path. Each schema change creates a new version file, providing:

- Full history of schema evolution over time
- DDL retrieval on the fly for enforcement
- Integration with Delta Lake's `DESCRIBE HISTORY` and Time Travel

### Data Type Changes

Adding new columns is straightforward (existing data gets NULLs for new columns). Data type changes are harder -- the safest approach is a complete overwrite of the target Delta table. If data types change frequently, it indicates a weak data governance strategy that should be addressed upstream.

> "Constantly changing schemas can be a sign of a weak data governance strategy and lack of communication with the data business owners." -- Garrett Peternel

---

## Production Case Studies

### Insurance Claims Processing (Financial Services)

**Problem:** FSI with policy and claims data scattered across on-premises EDWs, operational databases and third-party sources.

**Architecture:**
- **Ingestion:** Fivetran with CDC support, configurable sync frequency (5 min to 24 hours), storing as Delta tables
- **Transformation:** DLT pipeline with multi-notebook medallion architecture (3 source tables to 13 output tables)
- **Quality:** Extensive DLT expectations on claims data (valid licence dates, claim amounts, coverage periods, incident timing)
- **Serving:** Databricks SQL dashboards with parameterised date ranges and Delta time travel for version comparison
- **SCD support:** DLT CDC for SCD Type 1 and Type 2 updates

Key insight: Batch processing remains vital in financial services for back-office functions requiring systematic review of aggregate data. It provides the most cost-effective method for processing large volumes and can be done offline.

### Gaming Telemetry (IoT Streaming)

**Problem:** Video gaming company streaming events from multiple games through Kafka, needing per-game tables and pre-aggregated analytics.

**Architecture:**
- **Bronze:** DLT view consuming Kafka with Protobuf deserialisation via Confluent Schema Registry
- **Silver:** Dynamically generated Streaming Tables per game (demultiplexing a single stream)
- **Gold:** Materialized Views with player-level aggregates (session counts, min/max timestamps)
- **Schema evolution:** Handled automatically -- Production mode restarts pipeline on schema change

The pipeline uses metadata-driven generation: a single configuration parameter (comma-separated game list) drives creation of all Silver and Gold tables.

### Federated Lakehouse (Cross-Platform Analytics)

**Problem:** Organisation needing to query Azure SQL Database alongside lakehouse data without data movement.

**Solution:** Lakehouse Federation via Unity Catalog foreign catalog, enabling:
- Direct SQL queries against external databases from Databricks SQL Warehouse
- Joins between federated external data and local Delta tables
- Unified governance and permissions through UC

---

## LakeFlow

Databricks' unified data engineering solution spanning ingestion, transformation and orchestration:

- **LakeFlow Connect** -- native connectors for SaaS applications, databases and file sources with incremental ingestion, simple UI/API setup and UC governance
- Compatible with existing tooling (Auto Loader, Structured Streaming, DLT)

---

## Databricks SDK for Python

The SDK covers the entire Databricks API surface with benefits over raw REST calls:

- Unified client authentication (U2M OAuth with short-lived tokens)
- Two clients: `WorkspaceClient` (workspace-level) and `AccountClient` (account-level)
- Built-in debug logging with sensitive data redacted
- Automatic retries on transient errors
- Standard iterators for paginated APIs
- Long-running operation support (wait for job completion, cluster start)

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()  # auto-discovers auth from environment
for job in w.jobs.list():
    ...
```

---

## MLflow on Databricks

[[Databricks & Delta Lake|Databricks]] provides a fully managed MLflow deployment integrated with [[#Unity Catalog Governance|Unity Catalog]]. This eliminates the need to self-host MLflow Tracking Server infrastructure.

### Tracking Server and Experiment UI

Every Databricks workspace includes a hosted MLflow Tracking Server. Experiments are first-class objects in the workspace, accessible from the sidebar. The Experiment UI surfaces:

- **Run comparison** -- side-by-side metrics, parameters, and artefacts across runs
- **Charts** -- automatic visualisation of metric trends across runs (loss curves, accuracy over epochs)
- **Artefact browser** -- inspect logged models, plots, and custom files
- **Search and filter** -- query runs by parameters, metrics, tags, or status

Logging is automatic when using Databricks AutoML or manual via the MLflow API:

```python
import mlflow

with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("rmse", 0.87)
    mlflow.sklearn.log_model(model, "model")
```

### Model Registry and Stages

Unity Catalog Model Registry replaces the legacy workspace-level registry. Models are registered as three-level namespace objects (`catalog.schema.model_name`) with full governance:

| Stage | Purpose |
|-------|---------|
| **None** | Initial registration, experimental |
| **Champion** | Production-serving model (replaces legacy "Production" stage) |
| **Challenger** | Candidate model under evaluation against the champion |
| **Archived** | Retired model versions retained for audit |

Transitions between stages can be gated by approval workflows. Each model version records lineage back to the training run, dataset, and code.

### Model Serving Endpoints

Databricks Model Serving deploys registered models as auto-scaling REST endpoints:

- **Serverless compute** -- scales to zero when idle, scales out on demand
- **GPU support** -- for deep learning and LLM inference
- **Environment management** -- dependencies resolved from the logged model's `conda.yaml` or `requirements.txt`
- **Authentication** -- integrated with workspace tokens and service principals

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

endpoint = w.serving_endpoints.create(
    name="fraud-detection-v2",
    config={
        "served_models": [{
            "model_name": "ml_catalog.fraud.detector",
            "model_version": "3",
            "workload_size": "Small",
            "scale_to_zero_enabled": True,
        }]
    }
)
```

### A/B Testing with Traffic Splitting

Model Serving supports traffic splitting across multiple model versions on a single endpoint:

```python
config = {
    "served_models": [
        {"model_name": "ml.prod.recommender", "model_version": "5",
         "workload_size": "Small", "scale_to_zero_enabled": False},
        {"model_name": "ml.prod.recommender", "model_version": "6",
         "workload_size": "Small", "scale_to_zero_enabled": False},
    ],
    "traffic_config": {
        "routes": [
            {"served_model_name": "recommender-5", "traffic_percentage": 90},
            {"served_model_name": "recommender-6", "traffic_percentage": 10},
        ]
    }
}
```

Inference logs capture request/response pairs per model version, enabling offline analysis of A/B test results. Gradually shift traffic from champion to challenger as confidence grows.

---

## Databricks Feature Store

The Feature Store provides a centralised repository for feature engineering, ensuring consistency between training and serving.

### Feature Tables

Feature tables are Delta tables registered in Unity Catalog with a designated primary key. They support:

- **Time-series features** -- optional timestamp column for temporal joins
- **Composite keys** -- multi-column primary keys (e.g. `customer_id` + `product_id`)
- **Auto-documentation** -- descriptions, tags, and lineage tracked in UC

```python
from databricks.feature_engineering import FeatureEngineeringClient

fe = FeatureEngineeringClient()

fe.create_table(
    name="ml_catalog.features.customer_features",
    primary_keys=["customer_id"],
    timestamp_keys=["feature_timestamp"],
    df=customer_features_df,
    description="Customer behavioural features computed daily",
)
```

### Online and Offline Serving

| Mode | Storage | Latency | Use Case |
|------|---------|---------|----------|
| **Offline** | Delta Lake | Seconds | Batch training, batch scoring |
| **Online** | DynamoDB / Cosmos DB / Feature Serving | Milliseconds | Real-time model inference |

Online stores are synchronised from the offline Delta tables via scheduled or triggered publish jobs. Feature Serving endpoints (serverless) can serve features directly without external online stores.

### Point-in-Time Lookups

For training data, point-in-time correctness prevents data leakage. The Feature Store performs temporal joins using the timestamp key, ensuring each training example only sees features that were available at that point in time:

```python
training_set = fe.create_training_set(
    df=labels_df,
    feature_lookups=[
        FeatureLookup(
            table_name="ml_catalog.features.customer_features",
            lookup_key=["customer_id"],
            timestamp_lookup_key=["event_timestamp"],
        ),
    ],
    label="is_fraud",
)
training_df = training_set.load_df()
```

### Feature Engineering with Delta

Feature pipelines are typically implemented as DLT pipelines or Databricks Workflows that write to feature tables. The medallion architecture maps naturally:

- **Bronze** -- raw event data
- **Silver** -- cleansed, deduplicated entities
- **Gold / Feature** -- aggregated features (e.g. `avg_transaction_amount_30d`, `login_count_7d`)

Feature tables benefit from Delta's ACID transactions, time travel, and schema evolution.

---

## AutoML

Databricks AutoML automates the model development workflow for tabular data, producing transparent, editable code.

### Supported Problem Types

| Problem Type | Algorithms Explored | Output |
|-------------|-------------------|--------|
| **Classification** | Logistic regression, decision trees, random forest, XGBoost, LightGBM | Best model + notebook per trial |
| **Regression** | Linear regression, decision trees, random forest, XGBoost, LightGBM | Best model + notebook per trial |
| **Forecasting** | Prophet, ARIMA | Best model + notebook per trial |

### Glass-Box Models

AutoML generates complete, editable Python notebooks for every trial run. Engineers can:

- Inspect the exact preprocessing, feature engineering, and hyperparameter choices
- Modify the generated notebook and re-run with adjustments
- Promote the notebook to a production pipeline

This "glass-box" approach avoids the opacity of black-box AutoML systems. Every decision is visible and auditable.

### Custom Metrics

Beyond default metrics (F1, AUC, RMSE), AutoML supports custom evaluation metrics:

```python
from databricks import automl

summary = automl.classify(
    dataset=train_df,
    target_col="churn",
    primary_metric="f1",
    timeout_minutes=30,
    max_trials=50,
)

# Access best model
best_model = summary.best_trial.model
```

Results are logged to an MLflow experiment with full tracking of parameters, metrics, and artefacts for every trial.

---

## Mosaic AI

Mosaic AI extends the Databricks platform with generative AI capabilities.

### Foundation Model APIs

Databricks hosts curated open-source foundation models (DBRX, Llama, Mixtral) as pay-per-token endpoints:

- **Provisioned throughput** -- dedicated capacity for production workloads with guaranteed latency
- **Pay-per-token** -- shared capacity for development and experimentation
- **External models** -- gateway to OpenAI, Anthropic, and other providers via a unified API, enabling governance and rate limiting through Databricks

All requests are routed through the AI Gateway, which provides logging, access control, and cost tracking via Unity Catalog.

### RAG with Vector Search

Databricks Vector Search provides a managed vector database for retrieval-augmented generation (RAG) workflows:

1. **Embedding** -- compute embeddings using Foundation Model APIs or custom models
2. **Indexing** -- create a vector search index on a Delta table column (auto-sync keeps the index current)
3. **Retrieval** -- query the index for semantically similar documents
4. **Generation** -- pass retrieved context to an LLM for grounded responses

```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()
index = vsc.get_index(
    endpoint_name="vector-search-endpoint",
    index_name="ml_catalog.docs.knowledge_base_index",
)

results = index.similarity_search(
    query_text="How do I configure auto-scaling?",
    columns=["content", "doc_url"],
    num_results=5,
)
```

Vector Search integrates with the MLflow evaluation framework for measuring RAG quality (relevance, faithfulness, groundedness). The entire pipeline -- from document ingestion through chunking, embedding, indexing, and serving -- can be orchestrated as a Databricks Workflow.

---

**Related:** [[Databricks & Delta Lake]] | [[Delta Lake Operations & Patterns]] | [[Data Cataloguing & Discovery]] | [[Data Validation & Quality Frameworks]] | [[Terraform for Data Infrastructure]]
