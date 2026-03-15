# AWS Data Services for Data Engineering

AWS provides the most mature and broadly adopted set of cloud data services for building production data platforms.

See also: [[GCP Data Services for Data Engineering]] | [[Microsoft Fabric & Azure Data Services]] | [[Terraform for Data Infrastructure]]

---

## 1. S3 for Data Lakes

### Bucket Design — Prefix Strategy
Organize with prefixes that mirror the medallion architecture:

```
s3://company-data-lake-prod/
  raw/              # untouched source extracts
  staging/          # in-flight or intermediate
  curated/          # cleaned, conformed, business-ready
  archive/          # cold storage exports
```

Partition by date within each prefix: `raw/orders/year=2026/month=03/day=15/`

### File Format Best Practices

| Format  | Use Case | Strengths | Weaknesses |
|---------|----------|-----------|------------|
| Parquet | Analytics, data lakes | Columnar, compressed, predicate pushdown | Not human-readable |
| Avro    | Streaming, schema evolution | Row-based, compact, strong schema evolution | Less efficient for analytics |
| JSON    | APIs, semi-structured logs | Flexible, human-readable | Verbose, no schema enforcement |
| CSV     | Legacy ingestion, exports | Universal compatibility | No types, no compression |

**Rule of thumb**: land in source format (JSON/CSV), convert to Parquet or Delta as early as possible.

### Lifecycle Policies
Automate cost optimization: `Standard --30d--> Standard-IA --90d--> Glacier --365d--> Deep Archive`

From the Databricks Delta Lake project Terraform:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  rule {
    id = "data_lifecycle"; status = "Enabled"
    filter { prefix = "" }
    transition { days = 30;  storage_class = "STANDARD_IA" }
    transition { days = 90;  storage_class = "GLACIER" }
    transition { days = 365; storage_class = "DEEP_ARCHIVE" }
  }
}
```

### Versioning, Encryption, and Replication
- **Versioning** — enable to protect against accidental deletes and support Delta Lake time-travel
- **Encryption** — SSE-S3 (AES256, default), SSE-KMS (customer-managed key, audit/rotation), SSE-C (bring-your-own-key, rare). The Databricks project uses SSE-S3; the GCP equivalent uses Cloud KMS via `default_kms_key_name`
- **Cross-Region Replication (CRR)** — for DR or placing data closer to compute in another region

---

## 2. IAM for Data Pipelines

### Identity Types

| Identity | Purpose | Data Engineering Use |
|----------|---------|---------------------|
| IAM User | Long-lived credentials | CI/CD service users (prefer roles instead) |
| IAM Role | Assumed identity, temporary creds | EC2, Lambda, Glue jobs, cross-account |
| Service-linked Role | AWS-managed | Glue, Redshift, EMR auto-created roles |

**Best practice**: never embed access keys — always use IAM roles with `sts:AssumeRole`.

### Least-Privilege Policy Pattern
From the Databricks project — scoped to exactly the needed S3 actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::my-data-lake", "arn:aws:s3:::my-data-lake/*"]
  }]
}
```

Two ARN forms: bucket-level for `ListBucket`, object-level (`/*`) for Get/Put/Delete.

### Resource-Based Policies, Cross-Account, and STS
- **Bucket policies** — enforce encryption in transit (`aws:SecureTransport`), grant cross-account access
- **Cross-account** — Account B creates a role with trust policy for Account A; pipeline calls `sts:AssumeRole` for temporary creds
- **STS AssumeRole** — always prefer over long-lived keys; session duration 15 min to 12 hours; use external ID for third-party access

**GCP parallel**: service accounts with `roles/storage.objectAdmin` and workload identity federation — same least-privilege model.

---

## 3. AWS Glue

- **Crawlers** — scan S3 or JDBC sources to infer schema, populate the Data Catalog
- **Data Catalog** — Hive-compatible metastore used by Athena, Redshift Spectrum, EMR
- **ETL Jobs** — serverless PySpark with DPU-based billing
- **Glue Studio** — visual drag-and-drop editor (generates PySpark)
- **Job Bookmarks** — track processed data for incremental loads; critical for append-heavy lakes

### Typical Glue ETL Pattern

```python
from awsglue.context import GlueContext
from pyspark.context import SparkContext

glueContext = GlueContext(SparkContext())
source = glueContext.create_dynamic_frame.from_catalog(database="raw_db", table_name="orders")
cleaned = ApplyMapping.apply(frame=source, mappings=[
    ("order_id", "string", "order_id", "string"),
    ("amount", "string", "amount", "double"),
])
glueContext.write_dynamic_frame.from_options(
    frame=cleaned, connection_type="s3",
    connection_options={"path": "s3://bucket/curated/orders/"}, format="parquet"
)
```

---

## 4. Amazon Redshift

### Architecture
**Leader node** parses queries and coordinates; **compute nodes** store columnar data and execute in parallel.

### Distribution Styles

| Style | Behavior | Use When |
|-------|----------|----------|
| KEY   | Same key value on same node | Large fact tables joined on a key |
| EVEN  | Round-robin | No clear join key |
| ALL   | Full copy on every node | Small dimension tables (<~2M rows) |
| AUTO  | Redshift decides | Default starting point |

### Sort Keys
- **Compound** — optimizes range queries on leading columns
- **Interleaved** — equal weight to all key columns (ad-hoc queries)

### COPY Command and Spectrum
- **COPY** — fastest load path, reads S3 in parallel: `COPY orders FROM 's3://...' IAM_ROLE '...' FORMAT AS PARQUET;`
- **Spectrum** — query S3 data without loading; uses Glue Data Catalog; ideal for cold historical data

### Workload Management (WLM)
Isolate ETL loads from interactive queries with separate queues (ETL: high concurrency/low memory; BI: low concurrency/high memory).

---

## 5. Lambda for Data

### Event-Driven Processing
S3 PUT events trigger Lambda for lightweight transforms: `S3 PUT (raw/) -> Lambda -> validate -> S3 PUT (staging/)`

Common patterns: file format conversion, data validation/routing, metadata extraction, triggering downstream workflows.

### Step Functions for Orchestration
Compose Lambda, Glue jobs, and ECS tasks into workflows with retry logic, parallel execution, error handling, and visual monitoring.

### Lambda Layers
Package shared dependencies (pandas, pyarrow) as layers. AWS provides managed layers for common data libraries.

---

## 6. Amazon Kinesis

### Data Streams vs Data Firehose

| Feature | Data Streams | Data Firehose |
|---------|-------------|---------------|
| Latency | Real-time (~200ms) | Near real-time (60s buffer) |
| Management | Manual shard provisioning | Fully managed, auto-scaling |
| Consumers | Custom (KCL, Lambda) | Built-in delivery (S3, Redshift) |
| Cost model | Per-shard hour + PUT | Per GB ingested |

### Shard Management
Each shard: 1 MB/s in, 2 MB/s out, 1000 records/s. Monitor `WriteProvisionedThroughputExceeded` to split/merge. Consider on-demand mode.

### Consumer Patterns
- **KCL** — stateful consumer with checkpointing
- **Lambda** — serverless with event-source mapping
- **Kinesis Data Analytics (Flink)** — SQL or Flink for windowed aggregations

---

## 7. Amazon EMR

### Managed Spark/Hadoop
EMR on EC2, EMR on EKS, or EMR Serverless. Runs Spark, Hive, Presto, HBase.

### Cluster Sizing
- **Master**: coordination; modest sizing unless running Hive metastore
- **Core nodes**: HDFS + tasks; use on-demand for stability
- **Task nodes**: compute-only; use **spot instances** (up to 90% savings)

### EMRFS and Orchestration
- **EMRFS** — S3 connector with consistent reads; replaces HDFS to decouple storage from compute
- **Step Functions** — submit EMR steps, monitor, terminate transient clusters

---

## 8. Other AWS Data Services

| Service | Purpose | Key Detail |
|---------|---------|------------|
| **Athena** | Serverless SQL on S3 | Pay per TB scanned; optimize with Parquet + partitioning |
| **Lake Formation** | Data lake governance | Column/row-level security, cross-account sharing |
| **EventBridge** | Event routing | Trigger Lambda/Glue on schedules or service events |
| **SNS** | Fan-out notifications | Pipeline failure alerts, multi-consumer triggers |
| **SQS** | Message queuing | Decouple producers/consumers, dead-letter queues |
| **Secrets Manager** | Credential storage | Auto-rotation, SDK access from Lambda/Glue/EMR |

**GCP parallels**: GCS notifications use Pub/Sub (`google_storage_notification` in GCP storage module); Secret Manager with `roles/secretmanager.secretAccessor` (GCP IAM module).

---

## 9. Terraform for AWS Data Infrastructure

### S3 Bucket with Encryption and Versioning
From the [[Terraform for Data Infrastructure]] patterns in the Databricks Delta Lake project:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "${var.project_name}-${var.environment}-data-lake-${random_string.suffix.result}"
  tags   = merge(var.tags, { Name = "Delta Lake Data Lake", Type = "data-lake" })
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  rule { apply_server_side_encryption_by_default { sse_algorithm = "AES256" } }
}
```

### IAM Role with AssumeRole for Pipelines

```hcl
resource "aws_iam_role" "pipeline_role" {
  name = "${var.project_name}-${var.environment}-databricks-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Action = "sts:AssumeRole", Effect = "Allow",
                    Principal = { Service = "ec2.amazonaws.com" } }]
  })
}
```

### GCP Comparison — Parallel Terraform Patterns

| AWS Terraform | GCP Terraform | Purpose |
|---------------|---------------|---------|
| `aws_s3_bucket_versioning` | `versioning { enabled = true }` | Object versioning |
| `aws_s3_bucket_lifecycle_configuration` | `lifecycle_rule { condition { age } }` | Storage transitions |
| `aws_s3_bucket_server_side_encryption_configuration` | `encryption { default_kms_key_name }` | Encryption at rest |
| `aws_iam_role` + `assume_role_policy` | `google_service_account` + workload identity | Identity for compute |
| `aws_iam_policy` (resource ARNs) | `google_project_iam_member` (roles) | Least-privilege access |

---

## 10. AWS vs GCP vs Azure — Service Mapping

| Category | AWS | GCP | Azure |
|----------|-----|-----|-------|
| **Object Storage** | S3 | GCS | ADLS Gen2 / Blob Storage |
| **Data Warehouse** | Redshift | BigQuery | Synapse Analytics |
| **Serverless SQL** | Athena | BigQuery | Synapse Serverless |
| **ETL / ELT** | Glue | Dataflow / Dataproc | Data Factory |
| **Data Catalog** | Glue Data Catalog | Data Catalog | Purview |
| **Stream Ingestion** | Kinesis Streams | Pub/Sub | Event Hubs |
| **Stream Delivery** | Kinesis Firehose | Dataflow (streaming) | Stream Analytics |
| **Managed Spark** | EMR | Dataproc | HDInsight / Synapse Spark |
| **Serverless Compute** | Lambda | Cloud Functions | Azure Functions |
| **Orchestration** | Step Functions / MWAA | Cloud Composer | Data Factory / Logic Apps |
| **IAM Model** | Users / Roles / Policies | Service Accounts / Roles | Service Principals / RBAC |
| **Secrets** | Secrets Manager | Secret Manager | Key Vault |
| **Governance** | Lake Formation | Dataplex | Purview |
| **ML Platform** | SageMaker | Vertex AI | Azure ML |

### Key Differences for Data Engineers
- **Pricing**: Redshift = per-node-hour; BigQuery = per-TB-scanned; Synapse = both models
- **Storage/compute separation**: BigQuery and Synapse Serverless fully separate; Redshift couples them (Spectrum for external)
- **IAM philosophy**: AWS has fine-grained policy documents; GCP uses predefined roles on service accounts; Azure uses RBAC with built-in roles

---

## Quick Reference — When to Use What

| Scenario | Primary Service | Alternative |
|----------|----------------|-------------|
| Land raw files | S3 | Kinesis Firehose (streaming) |
| Schema discovery | Glue Crawlers | Manual catalog registration |
| Batch ETL (large) | Glue / EMR | Databricks on AWS |
| Batch ETL (small) | Lambda + Step Functions | Glue Python Shell |
| Real-time ingestion | Kinesis Data Streams | MSK (managed Kafka) |
| Ad-hoc SQL | Athena | Redshift Spectrum |
| Production warehouse | Redshift | Athena + Glue (serverless) |
| Orchestration | Step Functions / MWAA | EventBridge Scheduler |
| Governance | Lake Formation | Manual IAM + Glue Catalog |

---

**Related notes**:
- [[GCP Data Services for Data Engineering]] — GCS, BigQuery, Dataflow, Pub/Sub
- [[Microsoft Fabric & Azure Data Services]] — ADLS, Synapse, Data Factory
- [[Terraform for Data Infrastructure]] — IaC patterns across clouds
- [[Apache Spark for Data Engineering]] — Spark fundamentals used by EMR, Glue, Databricks
- [[Data Lake Architecture Patterns]] — medallion architecture, file formats, partitioning
