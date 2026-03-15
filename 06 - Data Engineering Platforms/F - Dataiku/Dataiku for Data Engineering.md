# Dataiku for Data Engineering

## Platform Overview

Dataiku DSS (Data Science Studio) is an end-to-end data science and engineering platform that bridges the gap between data engineers, analysts, and data scientists within a single collaborative environment. Rather than requiring separate tools for ingestion, transformation, ML, and deployment, Dataiku provides a unified workspace with both visual (no-code) and programmatic (code) interfaces.

### Positioning

Dataiku sits at the intersection of data engineering and data science. It is not a pure-play transformation tool like [[Core dbt Fundamentals|dbt]] nor a notebook-first analytics platform like [[Databricks & Delta Lake|Databricks]]. Its value proposition is accessibility — enabling teams with mixed skill levels to collaborate on the same pipeline using visual recipes or code recipes interchangeably.

### Editions

| Edition | Target | Key Constraints |
|---|---|---|
| **Free** (Community) | Individuals, learning | Single user, local engine only, limited connections |
| **Team** | Small teams | Multi-user, limited governance features |
| **Enterprise** | Organizations | Full governance, elastic compute, SSO, audit, automation nodes |
| **Cloud Starters** | Cloud-native teams | Managed DSS on AWS/GCP/Azure, quick provisioning |

### When to Use Dataiku

- Teams with mixed technical skill levels (analysts alongside engineers)
- Organizations wanting a governed, end-to-end ML lifecycle without stitching tools together
- Projects requiring rapid prototyping that can be promoted to production
- Environments where visual auditability of data lineage is critical

When **not** to use Dataiku: if you need deep infrastructure control, a pure SQL transformation layer ([[Core dbt Fundamentals|dbt]] is better), or you are already standardized on a cloud-native ML platform with no governance gaps.

---

## Architecture

### Node Types

Dataiku deployments typically involve multiple node types:

- **Design Node** — the primary development environment where users build flows, train models, and explore data interactively. This is where all authoring happens.
- **Automation Node** — the production execution environment. Receives bundled projects from the design node and runs scheduled scenarios. No interactive editing.
- **API Node** — serves real-time prediction endpoints. Receives model packages and exposes them as REST APIs with load balancing and high availability.
- **Govern Node** — (Enterprise) centralized governance layer for managing model approvals, sign-offs, and regulatory compliance workflows.

### Computation Engines

DSS pushes work to external engines where possible: **local** (small datasets, Python/R), **SQL pushdown** (recipes translated to SQL and run in-database on Snowflake/PostgreSQL/Redshift/BigQuery), **Spark** (distributed processing via Hadoop/YARN, Databricks, or standalone), and **Kubernetes** (containerized execution for recipes, training, and API serving).

### Elastic Compute (CDE/CDA)

- **Cloud Data Engineering (CDE)** — ephemeral Kubernetes clusters for recipe execution, scaling to zero when idle
- **Cloud Data Analytics (CDA)** — elastic scaling for interactive notebooks and exploration

### Connection Management

Connections are centrally configured by admins and shared across projects. Each connection defines credentials, default databases/schemas, and per-user credential mapping. Supported connections include JDBC databases, cloud storage (S3, GCS, ADLS), HDFS, Kafka, Elasticsearch, and more via plugins.

---

## Flow & Datasets

### Project Flow

The core abstraction in Dataiku is the **Flow** — a visual directed acyclic graph (DAG) showing how datasets are connected through recipes. Every project has one flow. Each node is either a dataset (blue squares) or a recipe (circles colored by type). This provides immediate visual lineage for every output.

The flow is conceptually similar to the DAG in [[Core dbt Fundamentals|dbt]] or an [[Apache Airflow Core Concepts|Airflow]] DAG, but it is both the development canvas and the execution plan.

### Managed vs External Datasets

- **External datasets** — point to data that exists outside DSS (a table in Snowflake, files in S3). DSS reads from them but does not control their lifecycle.
- **Managed datasets** — DSS owns these. It decides where they are stored (based on the connection) and manages their creation, rebuild, and cleanup.

### Dataset Types

| Category | Examples |
|---|---|
| SQL databases | PostgreSQL, MySQL, Oracle, SQL Server, Snowflake, BigQuery, Redshift |
| Cloud storage | S3, GCS, Azure Blob/ADLS |
| Filesystem | Local, NFS, HDFS |
| Streaming | Kafka (via plugin) |
| NoSQL | MongoDB, Cassandra, Elasticsearch |
| APIs | HTTP/REST (via plugin or custom datasets) |

### Partitioning

Datasets can be partitioned by **time** (year/month/day/hour) or by **discrete** dimensions (country, region, etc.). Partitioning enables:

- Incremental builds — only rebuild changed partitions
- Partition-dependent recipes — process one partition at a time
- Storage optimization — map to physical partitions in Hive, S3 prefixes, or database columns

### Schema Management

DSS tracks dataset schemas and detects drift. When an upstream schema changes, downstream recipes can be checked for compatibility. Schema propagation can be run manually or automatically, and the flow highlights breaking changes visually.

---

## Visual Recipes

Visual recipes are the no-code transformation layer. They generate execution plans in SQL, Spark, or the local engine depending on the dataset type. Key visual recipes:

| Recipe | Purpose |
|---|---|
| **Prepare** | Data wrangling with 100+ built-in processors (rename, split, parse dates, flag/remove invalid, compute formulas, geo-processing, normalization, etc.) |
| **Join** | Left/right/inner/full/cross joins across datasets, with pre-join filters and column selection |
| **Group By** | Aggregation with multiple aggregation functions per column, pre/post-filters |
| **Pivot** | Reshape long-to-wide, with configurable aggregation |
| **Window** | SQL-style window functions (rank, lead/lag, running totals) applied visually |
| **Split** | Route rows to multiple output datasets based on conditions |
| **Stack** | Union/concatenate datasets vertically (append rows) |
| **Sort** | Order rows by one or more columns |
| **Sample/Filter** | Subset rows by condition, random sampling, top-N, stratified sampling |
| **Sync** | Copy a dataset from one connection to another (e.g., S3 to Snowflake) |

The **Prepare recipe** is particularly powerful. Each step is a processor that can be previewed immediately. Steps are recorded as a script that can be reordered, disabled, or grouped. The entire prepare script is pushed down to SQL or Spark when the input and output datasets support it.

---

## Code Recipes

For transformations that exceed what visual recipes offer, code recipes embed scripts directly in the flow:

### Supported Languages

- **Python** — general-purpose, Pandas-based I/O by default
- **R** — statistical computing with managed R environments
- **SQL** — write raw SQL that executes in the target database engine
- **PySpark** — distributed processing via the [[Databricks & Delta Lake|Spark]] engine
- **Spark SQL** — SQL dialect on Spark DataFrames
- **Hive** — HiveQL execution on Hadoop clusters
- **Shell** — system-level scripting

### Managed I/O

Code recipes declare their input and output datasets in the flow. DSS handles reading inputs and writing outputs via its API:

```python
import dataiku
import pandas as pd

# Read input
input_df = dataiku.Dataset("raw_events").get_dataframe()

# Transform
result_df = input_df.groupby("user_id").agg({"event_count": "sum"})

# Write output
output_ds = dataiku.Dataset("user_summary")
output_ds.write_with_schema(result_df)
```

This pattern keeps code recipes as first-class nodes in the flow DAG with full lineage tracking.

### Code Environments

DSS manages isolated Python/R environments (virtualenvs or conda) per project or per recipe. This avoids dependency conflicts and ensures reproducibility. Custom packages are declared in environment specifications and installed automatically.

---

## Machine Learning

### AutoML (Visual ML)

Dataiku provides a visual ML workflow:

1. Select a training dataset and target variable
2. DSS analyzes the data and suggests feature handling (imputation, encoding, rescaling)
3. Configure algorithms to train — options include scikit-learn models, XGBoost, LightGBM, and deep learning (Keras/TensorFlow)
4. Train multiple models in parallel with cross-validation
5. Compare results on a leaderboard with metrics, confusion matrices, and ROC curves

### Supported Algorithms

- **Classification/Regression** — logistic/linear regression, random forest, gradient boosting (XGBoost, LightGBM), SVM, neural networks, ridge/lasso
- **Clustering** — K-means, hierarchical, DBSCAN, Gaussian mixture
- **Time series** — Prophet, ARIMA (via plugins), custom models
- **Deep learning** — Keras with visual architecture builder, custom TensorFlow/PyTorch code

### Model Evaluation & Interpretability

- Standard metrics (accuracy, F1, AUC, RMSE, MAE) with per-class breakdowns
- Feature importance (built-in and SHAP-based), partial dependence plots
- Individual prediction explanations and subpopulation analysis
- Automatic feature generation (polynomial, interactions) and feature stores for reuse

---

## Deployment & MLOps

### Model Deployment Options

| Method | Use Case |
|---|---|
| **Batch scoring** | Apply a trained model to a dataset via a Scoring recipe in the flow |
| **REST API endpoint** | Deploy to an API node for real-time, low-latency predictions |
| **Edge deployment** | Export models as standalone packages (PMML, Python pickles) for external systems |

### Bundles & Promotion

The design-to-production workflow uses **bundles**:

1. Develop and validate on the **Design Node**
2. Package the project (or specific items) into a **bundle** — a versioned, immutable snapshot
3. Push the bundle to the **Automation Node**
4. Activate the bundle on automation — it becomes the live production version

This mirrors CI/CD patterns and supports rollback by activating a previous bundle.

### Scenarios

Scenarios are Dataiku's orchestration mechanism (analogous to [[Apache Airflow Core Concepts|Airflow]] DAGs):

- **Triggers** — time-based (cron), dataset-change, API-triggered, or manual
- **Steps** — build datasets, train models, run code, send notifications, check conditions
- **Reporters** — send results via email, Slack, webhook, or custom channels
- Scenarios can include **checks** (data quality assertions) and **gates** (conditional logic)

### Model Monitoring

- **Data drift detection** — compare incoming data distributions against training data
- **Performance tracking** — monitor prediction accuracy over time when ground truth becomes available
- **Model comparison** — automatically retrain and compare against the deployed model
- **Alerting** — trigger notifications or retraining scenarios when drift exceeds thresholds

---

## Governance & Collaboration

### Project Permissions

DSS uses role-based access at the project level:

| Role | Capabilities |
|---|---|
| **Reader** | View flow, dashboards, and wiki; cannot modify |
| **Contributor** | Build recipes, train models, run scenarios |
| **Admin** | Manage project settings, connections, permissions |

Global permissions, group-based policies, and per-connection security provide additional layers.

### Collaboration Features

- **Project wiki** — markdown-based documentation within each project
- **Dashboards** — visual reporting with charts, metrics, and embedded datasets
- **Discussions** — threaded comments on any object (dataset, recipe, model)
- **Tagging** — organize objects with custom tags for discoverability
- **Data catalogue** — searchable inventory of datasets across all projects with lineage
- **Usage tracking** — monitor who accesses which data and when
- **Audit trail** — (Enterprise) full log of all actions for compliance

---

## Integration Patterns

### Snowflake Pushdown

When datasets live in Snowflake, DSS pushes visual recipe logic down as SQL executed inside Snowflake. Prepare, Join, Group By, and Window recipes run as Snowflake queries without moving data to the DSS server. Ensure managed datasets target the same Snowflake connection for optimal pushdown.

### Databricks / Spark Integration

DSS connects to [[Databricks & Delta Lake|Databricks]] clusters or standalone Spark for distributed processing. PySpark and Spark SQL recipes execute on these clusters. DSS can also use Databricks as a computation backend for visual recipes on large datasets. Delta Lake tables are supported as dataset sources.

### Kubernetes for Elastic Compute

DSS offloads recipe execution, model training, and model serving to Kubernetes. Each recipe runs in an isolated pod with its own code environment, with auto-scaling and scale-to-zero support. Works with EKS, GKE, AKS, or self-managed clusters.

### Git Integration

Each DSS project can be linked to a Git remote repository. Version control tracks changes to recipes, notebooks, project settings, and model configurations. Supports branching workflows for parallel development. Git integration complements (but differs from) bundle-based promotion.

### Plugin System

Dataiku's plugin ecosystem extends the platform with custom dataset types (connectors), custom recipes, custom ML algorithms, and custom web apps. Community plugins are available from the Dataiku Plugin Store.

---

## Comparison with Other Platforms

### Dataiku vs [[Databricks & Delta Lake|Databricks]]

| Dimension | Dataiku | Databricks |
|---|---|---|
| Primary interface | Visual flow + code | Notebooks + SQL |
| Target user | Mixed teams (analysts + engineers) | Engineers + data scientists |
| Compute | Delegates to external engines | Built-in Spark clusters |
| ML workflow | Visual AutoML + code | MLflow-based, code-first |
| Governance | Built-in project-level governance | Unity Catalog (data), MLflow (models) |
| Best for | Organizations needing accessibility | Teams standardized on Spark |

**Complementary pattern**: Use Databricks as the compute engine and Dataiku as the orchestration/governance layer on top.

### Dataiku vs [[Core dbt Fundamentals|dbt]]

| Dimension | Dataiku | dbt |
|---|---|---|
| Scope | Full platform (ingest to deploy) | SQL transformation layer only |
| Interface | Visual + code | Code-only (SQL + Jinja) |
| Testing | Scenarios with checks | Built-in test framework |
| Orchestration | Built-in scenarios | Requires external orchestrator |
| Best for | End-to-end workflows with ML | Pure SQL transformation pipelines |

**Complementary pattern**: Use dbt for SQL transformations and Dataiku for the ML/deployment layer on top of dbt-produced tables.

### Dataiku vs SageMaker / Vertex AI

| Dimension | Dataiku | SageMaker / Vertex AI |
|---|---|---|
| Cloud | Cloud-agnostic | AWS-only / GCP-only |
| Data engineering | Visual + code recipes | Limited (relies on external pipelines) |
| ML focus | AutoML + custom | Deep cloud-native ML tooling |
| Lock-in | Portable across clouds | Tied to single cloud provider |
| Best for | Multi-cloud or hybrid orgs | Teams deeply invested in one cloud |

**Complementary pattern**: Use Dataiku for development and governance, deploy models to SageMaker/Vertex endpoints for cloud-native serving.

---

## Key Takeaways

- Dataiku DSS is most valuable when the goal is to **democratize** data work across skill levels while maintaining governance
- The flow-based visual DAG provides immediate lineage and auditability that code-only tools require more effort to achieve
- SQL pushdown and Spark delegation mean Dataiku is an orchestration and governance layer more than a compute engine
- The design-to-automation promotion model via bundles provides a structured path from development to production
- Dataiku complements rather than replaces specialized tools like [[Core dbt Fundamentals|dbt]], [[Databricks & Delta Lake|Databricks]], and [[Apache Airflow Core Concepts|Airflow]] — it can sit on top of any of them

---

**Related Notes**: [[Databricks & Delta Lake]] | [[Core dbt Fundamentals]] | [[Apache Airflow Core Concepts]] | [[Python Core Patterns for Data Engineering]] | [[Snowflake Architecture & Key Concepts]]

**Tags**: #dataiku #data-engineering #mlops #platform #no-code
