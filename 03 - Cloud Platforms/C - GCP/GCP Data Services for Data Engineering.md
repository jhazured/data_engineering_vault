# GCP Data Services for Data Engineering

> Production patterns for Google Cloud data services, drawn from real ETL framework implementations. Covers BigQuery, Cloud Storage, Cloud SQL, Dataflow, Secret Manager, Composer, IAM, and cost management.

---

## 1. BigQuery

### Architecture
BigQuery is a **serverless, columnar** analytics warehouse. Storage and compute are decoupled -- you pay for bytes scanned (on-demand) or reserve slots (flat-rate). Data is stored in Google's Capacitor columnar format and queried via Dremel execution trees.

### Write Dispositions
The write disposition controls how a load job interacts with an existing table:

| Disposition | Behavior | Use Case |
|---|---|---|
| `WRITE_TRUNCATE` | Replaces all existing data | Full refreshes, dimension reloads |
| `WRITE_APPEND` | Adds rows to existing data | Incremental fact loading |
| `WRITE_EMPTY` | Fails if table is not empty | Safety guard for first-load jobs |

```python
from google.cloud import bigquery

client = bigquery.Client(project=project_id)

job_config = bigquery.LoadJobConfig(
    write_disposition="WRITE_APPEND",      # or WRITE_TRUNCATE
    create_disposition="CREATE_IF_NEEDED",
)

# Schema definition when table may not exist
job_config.schema = [
    bigquery.SchemaField("event_id", "STRING", mode="REQUIRED"),
    bigquery.SchemaField("event_ts", "TIMESTAMP", mode="NULLABLE"),
    bigquery.SchemaField("amount", "FLOAT64", mode="NULLABLE"),
]

job = client.load_table_from_dataframe(df, table_ref, job_config=job_config)
job.result()  # block until complete
```

### Partitioning and Clustering
Partitioning prunes data at the storage level; clustering sorts data within partitions.

```python
# Time-based partitioning (most common)
job_config.time_partitioning = bigquery.TimePartitioning(
    type_=bigquery.TimePartitioningType.DAY,   # also MONTH, YEAR, HOUR
    field="event_ts",
)

# Clustering -- up to 4 columns, order matters
job_config.clustering_fields = ["region", "product_category"]
```

**Range partitioning** works on INTEGER columns -- useful for ID-based sharding when time is not available.

### Dry-Run Queries and Cost Control
Always guard against runaway scans in production pipelines:

```python
job_config = bigquery.QueryJobConfig(
    dry_run=True,               # estimate only -- no data scanned
    use_query_cache=True,       # reuse cached results when possible
    maximum_bytes_billed=5 * (1024 ** 3),  # 5 GB hard cap
)

query_job = client.query(sql, job_config=job_config)
print(f"Estimated scan: {query_job.total_bytes_processed / 1e9:.2f} GB")
```

### Streaming Inserts vs Batch Loads
- **Batch loads** (load jobs) are free and preferred for ETL. They support all write dispositions and partitioning.
- **Streaming inserts** (`tabledata.insertAll`) have per-row costs but sub-second latency. Use only when real-time dashboards require it. Streamed data has a brief buffer period before it is available for DML.

See also: [[Apache Spark Fundamentals]] for Spark-BigQuery integration patterns.

---

## 2. Cloud Storage (GCS)

### Bucket Design
Organize buckets by environment and purpose, not by project team:

```
gs://<project>-<env>-etl-data/
    raw/            # landing zone -- immutable source files
    staging/        # intermediate transforms
    curated/        # production-ready datasets
    archive/        # cold storage after retention window
```

From Ansible config, a typical bucket setup:
```yaml
gcs_bucket_name: "my-gcp-project-dev-etl-data"
gcs_bucket_location: "us-central1"
```

### File Format Best Practices

| Format | Strengths | When to Use |
|---|---|---|
| **Parquet** | Columnar, splittable, schema-embedded, snappy compression | Default for analytics workloads |
| **Avro** | Row-based, schema evolution, compact | Streaming landing zones, [[Apache Kafka Fundamentals]] sinks |
| **CSV** | Human-readable, universal | Legacy system interchange only |
| **JSON** | Semi-structured, nested data | API responses, config payloads |

```python
# Loading Parquet from GCS into pandas
from google.cloud import storage
import pandas as pd, io

client = storage.Client(project=project_id)
bucket = client.bucket(bucket_name)
blob = bucket.blob("curated/events/2026/03/events.parquet")

content = blob.download_as_bytes()
df = pd.read_parquet(io.BytesIO(content))
```

### Lifecycle Policies
Set lifecycle rules to auto-transition or delete objects:
- Move to **Nearline** after 30 days, **Coldline** after 90, **Archive** after 365.
- Auto-delete temporary staging files after 7 days.

### GCS-Hadoop Connector for Spark
The `gcs-connector` JAR enables Spark to read/write `gs://` paths natively:

```properties
# spark-defaults.conf
spark.hadoop.fs.gs.impl  com.google.cloud.hadoop.fs.gcs.GoogleHadoopFileSystem
spark.hadoop.fs.AbstractFileSystem.gs.impl  com.google.cloud.hadoop.fs.gcs.GoogleHadoopFS
spark.hadoop.fs.gs.auth.service.account.enable  true
spark.hadoop.google.cloud.auth.service.account.json.keyfile  /app/service-account-key.json
```

The BigQuery connector allows Spark SQL to query BQ tables directly:
```properties
spark.sql.catalog.spark_catalog  com.google.cloud.spark.bigquery.v2.BigQueryTableProvider
```

See also: [[Docker & Container Patterns]] for how the Dockerfile installs both connectors.

---

## 3. Cloud SQL

### Connection Patterns
Cloud SQL offers managed MySQL and PostgreSQL. Two connection strategies exist:

**Cloud SQL Python Connector** (preferred for application code):
```python
from google.cloud.sql.connector import Connector
import sqlalchemy

connector = Connector()

def getconn():
    return connector.connect(
        "project:region:instance",        # instance_connection_name
        "pg8000",                         # or "pymysql" for MySQL
        user="etl_user",
        password=password,               # pull from Secret Manager
        db="warehouse",
    )

engine = sqlalchemy.create_engine("postgresql+pg8000://", creator=getconn)
```

**Cloud SQL Auth Proxy** (preferred for long-running services and local dev):
```bash
cloud-sql-proxy project:region:instance --port 5432 &
# then connect to localhost:5432 as usual
```

### Batch Loading
Use chunked inserts for large DataFrames to avoid memory pressure and lock contention:

```python
df.to_sql(
    "target_table",
    engine,
    if_exists="append",   # or "replace" for full refresh
    index=False,
    chunksize=10000,      # commit every 10k rows
)
```

### Failover
Enable high availability (regional) for production instances. Configure read replicas for analytics queries to keep OLTP performance isolated.

---

## 4. Dataflow

### Overview
Dataflow is the managed runner for [[Apache Beam]] pipelines. It handles autoscaling, work rebalancing, and exactly-once processing.

### Batch vs Streaming
- **Batch**: bounded `PCollection`, processes all data then terminates. Use for nightly ETL.
- **Streaming**: unbounded `PCollection`, runs continuously. Use for real-time enrichment from Pub/Sub.

### Job Lifecycle (from CI/CD)
```groovy
// Check job status from Jenkins
def status = sh(
    script: "gcloud dataflow jobs describe ${jobId} --region=${region} --format='value(currentState)'",
    returnStdout: true
).trim()

// List running jobs to avoid duplicates before launching
def running = sh(
    script: "gcloud dataflow jobs list --region=${region} --filter='state=JOB_STATE_RUNNING' --format='csv[no-heading](JOB_ID,JOB_NAME)'",
    returnStdout: true
).trim()
```

### Monitoring
- Use `gcloud dataflow jobs list` with filters for automated alerting.
- Clean up stale jobs older than a retention window to avoid quota exhaustion.

---

## 5. Secret Manager

### Credential Management
Store all service account keys, database passwords, and API tokens in Secret Manager -- never in environment files or source control.

```groovy
// Retrieve a secret in CI/CD
def dbPassword = sh(
    script: "gcloud secrets versions access latest --secret=etl-db-password",
    returnStdout: true
).trim()
```

```groovy
// Create or rotate a secret
def secretExists = sh(
    script: "gcloud secrets describe ${secretName} 2>/dev/null || echo 'NOT_FOUND'",
    returnStdout: true
).trim()

if (secretExists.contains('NOT_FOUND')) {
    sh "echo '${value}' | gcloud secrets create ${secretName} --data-file=-"
} else {
    sh "echo '${value}' | gcloud secrets versions add ${secretName} --data-file=-"
}
```

### Rotation Pattern
1. Add a new secret version (old versions remain accessible).
2. Deploy application update referencing `latest` (or pin a version).
3. Disable old versions after confirming rollout.
4. Destroy disabled versions after retention period.

---

## 6. Cloud Composer

### Managed Airflow
Cloud Composer runs Apache [[Airflow Orchestration Patterns|Airflow]] on GKE. DAGs are synced from a GCS bucket.

### Triggering from CI/CD
```bash
# Deploy ETL script to GCS staging
gsutil cp "etl_jobs/${ETL_SCRIPT}.py" "gs://my-etl-bucket/${ENV}/jobs/"

# Trigger a Composer DAG
gcloud composer environments run my-composer-env \
    --location us-central1 \
    trigger_dag -- "my_${ETL_SCRIPT}_dag"
```

### DAG Deployment Best Practices
- Store DAGs in version control; CI pushes to the Composer bucket on merge.
- Use environment variables in Composer for project ID, dataset names, and bucket paths -- not hardcoded values.
- Pin Python package versions in Composer's PyPI config to avoid drift from local dev.

---

## 7. IAM for Data

### Service Accounts
Create narrow, purpose-specific service accounts:

```yaml
# From Ansible config
service_account_name: "etl-vm-sa"
service_account_display_name: "ETL VM Service Account"
gcp_credentials_secret_name: "etl-service-account-key"
```

### Key Roles for Data Pipelines

| Role | Scope | Purpose |
|---|---|---|
| `roles/bigquery.dataEditor` | Dataset | Read/write tables, run load jobs |
| `roles/bigquery.jobUser` | Project | Execute queries |
| `roles/storage.objectViewer` | Bucket | Read GCS objects |
| `roles/storage.objectCreator` | Bucket | Write GCS objects (no delete) |
| `roles/cloudsql.client` | Project | Connect to Cloud SQL instances |
| `roles/secretmanager.secretAccessor` | Secret | Read secret values |
| `roles/dataflow.worker` | Project | Execute Dataflow jobs |
| `roles/composer.user` | Environment | Trigger DAGs |

### Least-Privilege Patterns
- Grant roles at the **resource level** (dataset, bucket), not project level, when possible.
- Use Workload Identity Federation for GKE workloads instead of key files.
- Validate permissions in CI before deployment:

```groovy
def validatePermissions(services) {
    services.each { svc ->
        switch(svc) {
            case 'bigquery':  sh "bq ls --max_results=1 > /dev/null"; break
            case 'storage':   sh "gsutil ls > /dev/null"; break
            case 'dataflow':  sh "gcloud dataflow jobs list --limit=1 > /dev/null"; break
        }
    }
}
```

See also: [[Terraform for Data Infrastructure]] for provisioning IAM bindings as code.

---

## 8. Cost Management

### BigQuery Pricing Models

| Model | How it Works | Best For |
|---|---|---|
| **On-demand** | $6.25/TB scanned | Exploratory, variable workloads |
| **Capacity (editions)** | Slot-hours purchased | Predictable, high-volume ETL |

**Cost controls in code:**
- Set `maximum_bytes_billed` on every production query.
- Use `dry_run=True` in CI to catch expensive queries before they run.
- Partition and cluster tables to minimize scan size.
- Prefer `WRITE_TRUNCATE` on partitioned tables with `partitionOverwriteMode=dynamic` to avoid full-table rewrites.

### GCS Storage Tiers

| Tier | Min Duration | Cost (approx) | Use Case |
|---|---|---|---|
| Standard | None | $0.020/GB/mo | Active pipeline data |
| Nearline | 30 days | $0.010/GB/mo | Monthly reporting archives |
| Coldline | 90 days | $0.004/GB/mo | Quarterly compliance data |
| Archive | 365 days | $0.0012/GB/mo | Long-term audit logs |

### Compute Right-Sizing
From the project's Ansible config, dev environments use free-tier eligible instances:
```yaml
machine_type: "e2-micro"   # Always Free tier
disk_size: "30"             # GB - within free tier limits
```

Production should scale to `n2-standard-4` or higher based on workload profiling. Use preemptible/spot VMs for Dataflow batch jobs to cut compute costs by ~60-80%.

### Quota Monitoring
Proactively check quotas to prevent pipeline failures from hitting limits:

```groovy
sh """
    gcloud compute project-info describe \
        --format='table(quotas[].metric,quotas[].usage,quotas[].limit)' | \
    awk 'NR>1 {
        usage=\$2; limit=\$3;
        if (usage && limit && usage > limit * 0.8) {
            print "WARNING: " \$1 " at " usage "/" limit
        }
    }'
"""
```

---

## Quick Reference: Docker Environment for GCP ETL

The production [[Docker & Container Patterns|Docker image]] bundles the GCP SDK, Spark, and both GCS/BigQuery connectors in a multi-stage build:

```dockerfile
# Builder: install GCP SDK + Spark + connectors
RUN apt-get install -y google-cloud-sdk
RUN curl -sSL .../gcs-connector-hadoop3-2.2.21.jar -o ${SPARK_HOME}/jars/gcs-connector.jar
RUN curl -sSL .../spark-bigquery-with-dependencies_2.12-0.36.1.jar -o ${SPARK_HOME}/jars/spark-bq.jar

# Runtime: set credentials path
ENV GOOGLE_APPLICATION_CREDENTIALS=/app/service-account-key.json
```

Key Spark tuning for GCP ETL from `spark-defaults.conf`:
```properties
spark.sql.sources.partitionOverwriteMode  dynamic
spark.sql.parquet.compression.codec       snappy
spark.sql.parquet.filterPushdown          true
spark.sql.adaptive.enabled                true
```

---

## 9. ETL Framework Patterns

The following sections document production patterns from a config-driven GCP ETL framework built on the factory method pattern. Each pipeline stage -- extraction, transformation, loading -- is selected at runtime from a registry of typed handlers.

### Framework Architecture

The framework uses a **factory pattern** to dispatch to the correct extractor, transformer, or loader based on a YAML configuration. A single `DataExtractor`, `DataTransformer`, or `DataLoader` class delegates to a specialised implementation:

```python
# Factory dispatch -- source type selects the handler
extractors = {
    'bigquery':   BigQueryExtractor,
    'cloudsql':   CloudSQLExtractor,
    'gcs':        GCSExtractor,
    'local_file': LocalFileExtractor,
    'api':        APIExtractor,
}

extractor = extractors[source_config['type']](source_config)
df = extractor.extract()
```

All handlers inherit from an abstract base class that enforces `extract()` / `load()` contracts and provides config validation:

```python
class BaseExtractor(ABC):
    def __init__(self, config: Dict[str, Any]):
        self.config = config

    def _validate_config(self, required_fields: List[str]) -> None:
        missing = [f for f in required_fields if f not in self.config]
        if missing:
            raise ValueError(f"Missing required fields: {missing}")

    @abstractmethod
    def extract(self) -> pd.DataFrame:
        pass
```

### Configuration Management

Pipeline configs are YAML files rendered through [[Jinja2]] templating, allowing environment-specific variable injection:

```python
from jinja2 import Environment, FileSystemLoader

class ConfigManager:
    def __init__(self, environment: str = 'dev'):
        self.environment = environment

    def load_config(self, config_path, variables=None):
        env_vars = {
            'ENVIRONMENT': self.environment,
            'PROJECT_ID': os.getenv('PROJECT_ID'),
            'REGION': os.getenv('REGION', 'us-central1'),
            'BQ_LOCATION': os.getenv('BQ_LOCATION', 'US'),
            'GCS_TEMP_BUCKET': os.getenv('GCS_TEMP_BUCKET'),
        }
        template_vars = {**env_vars, **(variables or {})}

        # Render Jinja2 template then parse as YAML
        env = Environment(loader=FileSystemLoader(config_path.parent))
        template = env.get_template(config_path.name)
        config = yaml.safe_load(template.render(**template_vars))
        return config
```

Required config sections: `job_name`, `source` (with `type`), and `destination` (with `type`). An optional `transformations` list defines the pipeline's transformation chain.

---

## 10. Extraction Patterns

### BigQuery Extraction with Query Job Config

The `BigQueryExtractor` wraps `QueryJobConfig` to control cost and caching at extraction time. Queries can be inline strings or `.sql` file references:

```python
from google.cloud import bigquery
from pathlib import Path

client = bigquery.Client(project=project_id)

# Load query from file or inline
query = config['query']
if query.endswith('.sql'):
    with open(Path(query), 'r') as f:
        query = f.read()

# Configure extraction job
job_config = bigquery.QueryJobConfig()
job_config.dry_run = config.get('dry_run', False)
job_config.use_query_cache = config.get('use_query_cache', True)
job_config.maximum_bytes_billed = config.get('maximum_bytes_billed')  # e.g. 5 * 1024**3

df = client.query(query, job_config=job_config).to_dataframe()
```

**Key parameters:**

| Parameter | Purpose | Production Default |
|---|---|---|
| `dry_run` | Estimate bytes scanned without executing | `False` (set `True` in CI) |
| `use_query_cache` | Reuse previous results for identical queries | `True` |
| `maximum_bytes_billed` | Hard cap on scan cost; query fails if exceeded | Set per-job based on expected size |

### Cloud SQL Extraction with Connector Switching

The `CloudSQLExtractor` uses the Cloud SQL Python Connector and dynamically selects the driver based on database type -- `pymysql` for MySQL, `pg8000` for PostgreSQL:

```python
from google.cloud.sql.connector import Connector
import sqlalchemy, pandas as pd

connector = Connector()

def getconn():
    driver = "pymysql" if db_type == "mysql" else "pg8000"
    return connector.connect(
        "project:region:instance",
        driver,
        user=user,
        password=password,
        db=database,
    )

# SQLAlchemy engine URL must match the driver
url = "mysql+pymysql://" if db_type == "mysql" else "postgresql+pg8000://"
engine = sqlalchemy.create_engine(url, creator=getconn)

try:
    df = pd.read_sql(query, engine)
finally:
    connector.close()  # always close to release resources
```

The `connector.close()` call in `finally` is essential -- the connector maintains IAM-authenticated tunnels that leak if not cleaned up.

### GCS Multi-File Extraction with Format Handling

The `GCSExtractor` lists blobs by prefix, reads each file according to its format, and concatenates the results:

```python
from google.cloud import storage
import pandas as pd

client = storage.Client(project=project_id)
bucket = client.bucket(bucket_name)
blobs = list(bucket.list_blobs(prefix=file_pattern))

dataframes = []
for blob in blobs:
    if file_format == 'csv':
        content = blob.download_as_text()
        df = pd.read_csv(
            pd.io.common.StringIO(content),
            **csv_options  # e.g. delimiter, encoding, dtype
        )
    elif file_format == 'json':
        content = blob.download_as_text()
        df = pd.read_json(
            pd.io.common.StringIO(content),
            **json_options  # e.g. orient, lines
        )
    elif file_format == 'parquet':
        content_bytes = blob.download_as_bytes()
        df = pd.read_parquet(
            pd.io.common.BytesIO(content_bytes),
            **parquet_options  # e.g. columns, filters
        )
    dataframes.append(df)

result = pd.concat(dataframes, ignore_index=True)
```

Note that Parquet requires `download_as_bytes()` (binary), whilst CSV and JSON use `download_as_text()` (string). Each format accepts pass-through options for fine-grained control (delimiters, encodings, column selection).

### API Extraction with Nested JSON Support

The `APIExtractor` handles REST endpoints and uses `pd.json_normalize` for flattening nested JSON responses. A `data_path` parameter navigates into the response structure:

```python
import requests
import pandas as pd

response = requests.get(url, headers=headers, params=params)
response.raise_for_status()

json_data = response.json()

# Navigate nested response: e.g. data_path = "results.items"
if data_path:
    for key in data_path.split('.'):
        json_data = json_data[key]

# Flatten nested objects into columns
df = pd.json_normalize(json_data)
```

`json_normalize` recursively flattens nested dictionaries into dot-separated column names (e.g. `address.city`, `address.postcode`), which is preferable to manual parsing for complex API responses.

---

## 11. Transformation Patterns

The `DataTransformer` applies a sequential chain of named transformations from configuration. Each transformation is a dictionary with a `type` key that maps to a handler function.

### Transformation Registry

Available transformations and their config keys:

| Transformation | Purpose | Key Config Fields |
|---|---|---|
| `filter` | Row-level filtering | `conditions` (column, operator, value) |
| `select_columns` | Column projection | `columns` |
| `rename_columns` | Column renaming | `mapping` |
| `add_column` | Derived columns | `columns` (name, type, value) |
| `drop_columns` | Remove columns | `columns` |
| `convert_types` | Type casting | `conversions` (column: target_type) |
| `fill_nulls` | Null imputation | `fill_values` or `method` |
| `drop_nulls` | Remove null rows | `columns`, `how` |
| `drop_duplicates` | Deduplication | `columns`, `keep` |
| `aggregate` | Group-by aggregation | `group_by`, `aggregations` |
| `pivot` | Rows to columns | `index`, `columns`, `values`, `aggfunc` |
| `unpivot` | Columns to rows | `id_vars`, `value_vars` |
| `window_functions` | Windowed calculations | `windows` (column, function, partition_by) |
| `standardize_text` | Text normalisation | `columns`, `operations` |
| `extract_date_parts` | Date decomposition | `date_columns`, `parts` |
| `validate_data_quality` | Quality checks | `rules` |

### Type Conversions

The framework uses pandas nullable types and coercion to handle dirty data gracefully:

```python
conversions = {
    'revenue':     'float',     # pd.to_numeric with errors='coerce'
    'quantity':    'int',       # nullable Int64 to preserve NaN
    'event_date':  'datetime',  # pd.to_datetime with errors='coerce'
    'customer_id': 'string',   # .astype(str)
    'is_active':   'boolean',  # .astype(bool)
}

for column, target_type in conversions.items():
    if target_type == 'int':
        df[column] = pd.to_numeric(df[column], errors='coerce').astype('Int64')
    elif target_type == 'float':
        df[column] = pd.to_numeric(df[column], errors='coerce')
    elif target_type == 'datetime':
        df[column] = pd.to_datetime(df[column], errors='coerce')
```

Using `errors='coerce'` converts unparseable values to `NaN`/`NaT` rather than raising exceptions -- essential for production pipelines where source data quality varies.

### Null Handling Strategies

Four strategies are available, selectable by configuration:

```python
# Strategy 1: Column-specific fill values
df = df.fillna({'status': 'unknown', 'amount': 0.0, 'region': 'UNSPECIFIED'})

# Strategy 2: Forward fill (carry last known value)
df = df.fillna(method='ffill')

# Strategy 3: Backward fill
df = df.fillna(method='bfill')

# Strategy 4: Statistical imputation (numeric columns only)
numeric_cols = df.select_dtypes(include=[np.number]).columns
df[numeric_cols] = df[numeric_cols].fillna(df[numeric_cols].mean())   # or .median()
```

### Deduplication

```python
# Keep first occurrence, deduplicate on specific columns
df = df.drop_duplicates(
    subset=['customer_id', 'order_date'],  # None = all columns
    keep='first',                           # 'first', 'last', or False (drop all dupes)
)
```

### Aggregation with Multi-Level Flattening

Grouped aggregations produce multi-level column names which the framework flattens automatically:

```python
aggregations = {
    'revenue': ['sum', 'mean'],
    'quantity': 'sum',
    'order_id': 'count',
}

result = df.groupby(['region', 'product_category']).agg(aggregations).reset_index()

# Flatten multi-level columns: ('revenue', 'sum') -> 'revenue_sum'
if isinstance(result.columns, pd.MultiIndex):
    result.columns = [
        '_'.join(col).strip() if col[1] else col[0]
        for col in result.columns.values
    ]
```

### Pivot and Unpivot

**Pivot** (rows to columns):
```python
pivoted = df.pivot_table(
    index='region',
    columns='quarter',
    values='revenue',
    aggfunc='sum',
    fill_value=0,
).reset_index()
```

**Unpivot / Melt** (columns to rows):
```python
melted = df.melt(
    id_vars=['customer_id', 'region'],
    value_vars=['q1_revenue', 'q2_revenue', 'q3_revenue', 'q4_revenue'],
    var_name='quarter',
    value_name='revenue',
)
```

### Window Functions

The framework supports row numbering, ranking, and running aggregations partitioned by group:

```python
windows = [
    {
        'column': 'revenue',
        'function': 'row_number',
        'partition_by': ['region'],
        'order_by': ['event_date'],
        'new_column': 'revenue_row_number',
    },
]

# Partition then apply
if partition_by:
    grouped = df.groupby(partition_by)
else:
    grouped = df.groupby(lambda x: True)

if function == 'row_number':
    df = df.sort_values(order_by)
    df[new_column] = grouped.cumcount() + 1
```

### Text Standardisation

Multiple operations can be chained on text columns:

```python
operations = ['strip', 'lower', 'remove_extra_spaces']

for column in text_columns:
    if 'strip' in operations:
        df[column] = df[column].str.strip()
    if 'lower' in operations:
        df[column] = df[column].str.lower()
    if 'remove_extra_spaces' in operations:
        df[column] = df[column].str.replace(r'\s+', ' ', regex=True)
```

### Date Part Extraction

Decompose datetime columns into numeric components for downstream analytics:

```python
parts = ['year', 'month', 'day', 'weekday', 'quarter']

df['event_date'] = pd.to_datetime(df['event_date'], errors='coerce')

df['event_date_year']    = df['event_date'].dt.year
df['event_date_month']   = df['event_date'].dt.month
df['event_date_quarter'] = df['event_date'].dt.quarter
df['event_date_weekday'] = df['event_date'].dt.dayofweek  # 0 = Monday
```

---

## 12. Loading Patterns

### BigQuery Loading with Partitioning and Clustering

The `BigQueryLoader` builds a `LoadJobConfig` dynamically from configuration, supporting schema definition, time partitioning (DAY or MONTH), and clustering:

```python
from google.cloud import bigquery

client = bigquery.Client(project=project_id)
table_ref = f"{project_id}.{dataset_id}.{table_id}"

job_config = bigquery.LoadJobConfig(
    write_disposition='WRITE_APPEND',       # or WRITE_TRUNCATE
    create_disposition='CREATE_IF_NEEDED',
)

# Dynamic schema from config
job_config.schema = [
    bigquery.SchemaField(f['name'], f['type'], mode=f.get('mode', 'NULLABLE'))
    for f in schema_config
]

# Time partitioning
job_config.time_partitioning = bigquery.TimePartitioning(
    type_=bigquery.TimePartitioningType.DAY,  # or MONTH
    field='event_date',
)

# Clustering
job_config.clustering_fields = ['region', 'product_category']

job = client.load_table_from_dataframe(df, table_ref, job_config=job_config)
job.result()

# Verify load
table = client.get_table(table_ref)
print(f"Loaded {job.output_rows} rows; table now has {table.num_rows} total")
```

### GCS Loading with Multi-Format Support

The `GCSLoader` serialises DataFrames to CSV, JSON, or Parquet and uploads to a blob:

```python
from google.cloud import storage
import io

client = storage.Client(project=project_id)
bucket = client.bucket(bucket_name)
blob = bucket.blob(file_path)

if file_format == 'csv':
    buffer = io.StringIO()
    df.to_csv(buffer, index=False, **csv_options)
    blob.upload_from_string(buffer.getvalue(), content_type='text/csv')

elif file_format == 'json':
    buffer = io.StringIO()
    df.to_json(buffer, orient='records', **json_options)
    blob.upload_from_string(buffer.getvalue(), content_type='application/json')

elif file_format == 'parquet':
    buffer = io.BytesIO()
    df.to_parquet(buffer, index=False, **parquet_options)
    blob.upload_from_string(buffer.getvalue(), content_type='application/octet-stream')
```

### Cloud SQL Loading

Uses `pd.to_sql` with chunked inserts and `if_exists` control:

```python
df.to_sql(
    table_name,
    engine,           # SQLAlchemy engine from Cloud SQL Connector
    if_exists='append',  # 'append', 'replace', or 'fail'
    index=False,
    chunksize=10000,
)
```

The `if_exists` parameter maps to common ETL strategies:
- `'append'` -- incremental loading (equivalent to BigQuery `WRITE_APPEND`)
- `'replace'` -- full refresh (drops and recreates the table)
- `'fail'` -- safety guard for first-load scenarios

### Multi-Destination Loading

The `MultiDestinationLoader` fans out a single DataFrame to multiple targets, tracking success/failure per destination:

```python
config = {
    'type': 'multi',
    'destinations': [
        {'type': 'bigquery', 'dataset': 'analytics', 'table': 'events'},
        {'type': 'gcs', 'bucket': 'archive', 'file_path': 'events.parquet', 'format': 'parquet'},
    ]
}

# Each destination is loaded independently
results = []
for dest_config in config['destinations']:
    loader = DataLoader(dest_config)
    result = loader.load(df)
    results.append(result)

# Check overall success
all_succeeded = all(r['success'] for r in results)
```

This pattern is useful when the same transformed data must land in both an analytics warehouse (BigQuery) and a cold archive (GCS) simultaneously.

---

## Cloud Composer (Managed Airflow)

Cloud Composer is Google Cloud's fully managed [[Airflow Orchestration Patterns|Apache Airflow]] service, built on GKE. It handles Airflow infrastructure -- web server, scheduler, workers, metadata database, and Redis queue -- so teams focus on DAG development rather than cluster operations.

### Composer 2 Architecture

Composer 2 runs on an **autopilot GKE cluster** with autoscaling workers. Key architectural components:

| Component | Implementation | Notes |
|-----------|---------------|-------|
| Web Server | Cloud Run-backed | Always available, independent of workers |
| Scheduler | GKE Pod | Scales based on DAG parse load |
| Workers | GKE Pods (CeleryExecutor) | Autoscale between min and max worker counts |
| Metadata DB | Cloud SQL (PostgreSQL) | Managed, automatic backups |
| DAG Storage | GCS Bucket | Synced to workers automatically |
| Logs | Cloud Logging | Integrated, searchable, exportable |

The Airflow metadata database is fully managed -- no need to provision or tune Cloud SQL separately.

### Environment Sizing

| Size | Workers (min-max) | Scheduler CPU/Memory | Use Case |
|------|-------------------|---------------------|----------|
| Small | 1-3 | 0.5 vCPU / 2 GB | Development, fewer than 50 DAGs |
| Medium | 2-6 | 2 vCPU / 7.5 GB | Production, 50-200 DAGs |
| Large | 3-12 | 4 vCPU / 15 GB | High-throughput, 200+ DAGs, short task intervals |

**Right-sizing tips:**
- Start with a Small environment and scale up based on scheduler parse time and task queue depth.
- Monitor `scheduler_heartbeat` and `dag_processing.total_parse_time` metrics.
- Set `min_workers` to at least 1 in production to avoid cold-start delays.

### DAG Deployment via GCS Sync

Composer syncs DAGs from a GCS bucket. The recommended CI/CD pattern:

```bash
# 1. CI pipeline uploads DAGs to the Composer bucket on merge to main
COMPOSER_BUCKET=$(gcloud composer environments describe my-composer-env \
    --location us-central1 \
    --format='value(config.dagGcsPrefix)')

gsutil -m rsync -r -d dags/ "${COMPOSER_BUCKET}/"
```

**Deployment rules:**
- Store DAGs in version control; CI pushes to the Composer bucket on merge to the main branch.
- Use `gsutil rsync -d` to mirror the repository state (deletes removed DAGs from the bucket).
- Avoid uploading DAGs directly to the bucket outside CI -- this bypasses code review and creates drift.
- DAG parse time affects scheduler performance; keep imports lightweight and avoid top-level database calls.

### Python Package Management

Composer environments support custom PyPI packages. Manage them via the console, CLI, or Terraform:

```bash
# Install packages via CLI
gcloud composer environments update my-composer-env \
    --location us-central1 \
    --update-pypi-packages-from-file requirements.txt
```

```
# requirements.txt -- pin all versions to avoid drift
apache-airflow-providers-google==10.12.0
pandas==2.1.4
requests==2.31.0
sqlalchemy==2.0.25
```

**Best practices:**
- Pin every package version. Composer upgrades can break unpinned dependencies.
- Test package compatibility in a development environment before promoting to production.
- Use `constraints.txt` if needed to lock transitive dependencies.
- Avoid packages that require system-level libraries (C extensions) unless the base Composer image supports them.

### Connections and Secrets via Secret Manager

Use [[GCP Data Services for Data Engineering#5. Secret Manager|Secret Manager]] as the secrets backend instead of storing credentials in the Airflow connections UI:

```python
# airflow.cfg override (set via Composer environment variable)
# AIRFLOW__SECRETS__BACKEND=airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend
# AIRFLOW__SECRETS__BACKEND_KWARGS={"connections_prefix": "airflow-connections", "variables_prefix": "airflow-variables", "project_id": "my-project"}
```

With this configuration:
- Airflow looks up connections from Secret Manager secrets named `airflow-connections-<conn_id>`.
- Airflow variables resolve from secrets named `airflow-variables-<var_name>`.
- The Composer service account needs `roles/secretmanager.secretAccessor` on the relevant secrets.
- Secrets are never stored in the Airflow metadata database, reducing the blast radius of a database compromise.

### Monitoring and Alerting

Composer exposes metrics to Cloud Monitoring automatically:

| Metric | Alert Threshold | Indicates |
|--------|----------------|-----------|
| `environment/healthy` | != 1 | Environment health degradation |
| `scheduler/heartbeat` | Missing for > 60s | Scheduler crash or hang |
| `dag_processing/total_parse_time` | > 30s | Slow DAG parsing; too many or heavy DAGs |
| `worker/task_queue_length` | > 50 sustained | Workers cannot keep up; scale up |
| `worker/pod_eviction_count` | > 0 | Workers running out of memory |

**Operational monitoring:**
- Set up Cloud Monitoring alerting policies for the metrics above.
- Use `gcloud composer environments describe` to check environment health programmatically.
- Review task instance logs in Cloud Logging for failed task root-cause analysis.
- Enable Airflow's built-in SLA miss callbacks for business-critical DAGs.

### Cost Optimisation

Composer 2 charges for compute (GKE), database (Cloud SQL), web server (Cloud Run), and GCS storage.

**Cost reduction strategies:**
- **Right-size the environment.** A Small environment costs roughly 60-70% less than a Large one. Upgrade only when parse times or queue depths demand it.
- **Minimise worker idle time.** Set `min_workers=1` (not 0, to avoid cold starts, but not higher than needed during off-peak).
- **Schedule DAGs efficiently.** Stagger DAG start times to avoid thundering-herd spikes that force unnecessary autoscaling.
- **Use triggerer for deferrable operators.** Deferrable operators (e.g., `BigQueryInsertJobOperator` with `deferrable=True`) release the worker slot while waiting, reducing worker count requirements.
- **Avoid over-provisioning scheduler resources.** Monitor parse times before increasing scheduler CPU/memory.
- **Use a single Composer environment per team or domain**, not per DAG or per pipeline. The fixed infrastructure cost is amortised across more workloads.
- **Delete unused development environments** promptly -- even idle Composer environments incur Cloud SQL and GKE control-plane costs.

### Example: Minimal Composer DAG

```python
from airflow import DAG
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.utils.dates import days_ago

with DAG(
    dag_id="bq_daily_transform",
    schedule_interval="@daily",
    start_date=days_ago(1),
    catchup=False,
    tags=["bigquery", "transform"],
) as dag:

    run_transform = BigQueryInsertJobOperator(
        task_id="run_transform",
        configuration={
            "query": {
                "query": "CALL `project.dataset.usp_daily_transform`()",
                "useLegacySql": False,
            }
        },
        location="US",
        deferrable=True,  # releases worker slot while BQ job runs
    )
```

---

**Related notes:** [[Apache Spark Fundamentals]] | [[Airflow Orchestration Patterns]] | [[Terraform for Data Infrastructure]] | [[Docker & Container Patterns]] | [[Apache Kafka Fundamentals]] | [[CI/CD for Data Pipelines]] | [[Cloud Platform Comparison]]
