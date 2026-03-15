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

**Related notes:** [[Apache Spark Fundamentals]] | [[Airflow Orchestration Patterns]] | [[Terraform for Data Infrastructure]] | [[Docker & Container Patterns]] | [[Apache Kafka Fundamentals]] | [[CI/CD for Data Pipelines]]
