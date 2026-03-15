# AWS Snowflake dbt Reference Architecture

**Tags:** #aws #snowflake #dbt #terraform #lambda #kinesis #step-functions #reference-architecture

## Overview

A production reference architecture for a data engineering platform using AWS cloud services, Snowflake as the data warehouse, and dbt for transformation. This pattern covers infrastructure-as-code, serverless ingestion, real-time streaming, orchestration, and CI/CD.

See also: [[AWS Data Services for Data Engineering]] | [[Core dbt Fundamentals]] | [[Terraform for Data Infrastructure]]

---

## Architecture Components

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Cloud Platform | AWS | Hosting, compute, storage, networking |
| Data Warehouse | Snowflake | Analytics, transformation target |
| Transformation | dbt | SQL-based ELT modelling |
| Orchestration | AWS Step Functions + Lambda | Pipeline scheduling and coordination |
| Storage | Amazon S3 | Data lake (raw/processed/reference) |
| Streaming | Amazon Kinesis Data Streams | Real-time event ingestion |
| Security | AWS Secrets Manager + IAM | Credential management, access control |
| CI/CD | AWS CodePipeline + CodeBuild | Automated deployment |
| Monitoring | CloudWatch + SNS | Alerts, dashboards, notifications |

---

## Project Structure

```
project/
├── infrastructure/
│   └── terraform/
│       ├── main.tf           # Provider, backend, variables
│       ├── s3.tf             # Data lake buckets, lifecycle, events
│       ├── lambda.tf         # Ingestion, processing, dbt runner
│       ├── kinesis.tf        # Streaming infrastructure
│       ├── step-functions.tf # Pipeline orchestration
│       └── iam.tf            # Roles, policies, permissions
├── lambda-functions/
│   ├── data-ingestion/       # S3 event processor, Snowflake loader
│   └── data-generators/      # Test data generation
├── step-functions/
│   ├── daily-pipeline.json   # Batch pipeline definition
│   └── real-time-processing.json
├── dbt/
│   ├── models/
│   │   ├── staging/          # 1:1 source mirrors
│   │   ├── intermediate/     # Business logic joins
│   │   └── marts/            # Final consumption models
│   ├── macros/               # Reusable SQL functions
│   ├── seeds/                # Reference data (CSV)
│   └── tests/                # Data quality assertions
├── snowflake/
│   ├── setup/                # Database, schema, warehouse DDL
│   └── external-stages/      # S3 stage + Snowpipe config
└── monitoring/
    ├── cloudwatch-dashboards/
    └── alerts/
```

---

## Terraform Patterns

### S3 Data Lake with Lifecycle

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "${var.project_name}-data-lake-${var.environment}-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_lifecycle_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "tiered_storage"
    status = "Enabled"

    transition { days = 30;  storage_class = "STANDARD_IA" }
    transition { days = 90;  storage_class = "GLACIER" }
    transition { days = 365; storage_class = "DEEP_ARCHIVE" }
  }
}
```

### S3 Event-Driven Lambda Trigger

```hcl
resource "aws_s3_bucket_notification" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.data_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "raw-data/"
    filter_suffix       = ".json"
  }

  depends_on = [aws_lambda_permission.allow_s3]
}
```

### Lambda Function Pattern

```hcl
resource "aws_lambda_function" "data_processor" {
  filename         = "data-processor.zip"
  function_name    = "${var.project_name}-data-processor-${var.environment}"
  role             = aws_iam_role.lambda_execution_role.arn
  handler          = "lambda_function.lambda_handler"
  runtime          = "python3.11"
  timeout          = 900
  memory_size      = 1024

  environment {
    variables = {
      SNOWFLAKE_ACCOUNT   = var.snowflake_account
      S3_BUCKET           = aws_s3_bucket.data_lake.bucket
      SECRETS_MANAGER_ARN = aws_secretsmanager_secret.creds.arn
    }
  }
}
```

---

## Kinesis Streaming Pattern

```hcl
resource "aws_kinesis_stream" "transaction_stream" {
  name             = "${var.project_name}-stream-${var.environment}"
  shard_count      = 2
  retention_period = 24

  encryption_type = "KMS"
  kms_key_id      = aws_kms_key.kinesis_key.arn

  shard_level_metrics = ["IncomingRecords", "OutgoingRecords"]
}
```

Key design decisions:
- **Shard count** — 1 shard = 1,000 records/sec or 1 MB/sec. Scale shards based on throughput.
- **Retention** — 24 hours default; increase for replay capability.
- **Encryption** — Always use KMS for data at rest.

---

## Step Functions Orchestration

Daily pipeline pattern using the Map state for parallel account processing:

```json
{
  "StartAt": "GetAccountList",
  "States": {
    "GetAccountList": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Next": "ProcessAccounts"
    },
    "ProcessAccounts": {
      "Type": "Map",
      "ItemsPath": "$.Payload.accounts",
      "MaxConcurrency": 5,
      "Iterator": {
        "StartAt": "ExtractData",
        "States": {
          "ExtractData": { "Type": "Task", "Next": "SaveToS3" },
          "SaveToS3": { "Type": "Task", "End": true }
        }
      },
      "Next": "RunDBTModels"
    },
    "RunDBTModels": {
      "Type": "Task",
      "Next": "RunDBTTests"
    },
    "RunDBTTests": {
      "Type": "Task",
      "Next": "SendNotification"
    },
    "SendNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "End": true
    }
  }
}
```

**Key patterns:**
- **Map state** with `MaxConcurrency` for parallel processing
- **dbt run → dbt test** as sequential steps (test only after models build)
- **SNS notification** on completion (extend with error-catching for failures)

---

## Snowflake Integration

### External Stage + Snowpipe (Auto-ingest from S3)

```sql
CREATE STAGE aws_s3_stage
  URL = 's3://data-lake-bucket/raw-data/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'JSON' STRIP_OUTER_ARRAY = TRUE);

CREATE PIPE auto_ingest_pipe
  AUTO_INGEST = TRUE
AS
  COPY INTO RAW_SCHEMA.RAW_TABLE
  FROM @aws_s3_stage
  FILE_FORMAT = (TYPE = 'JSON');
```

Snowpipe uses SQS notifications from S3 events to trigger automatic loading.

### dbt Profile for Snowflake

```yaml
project_name:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: TRANSFORMER
      database: ANALYTICS
      warehouse: COMPUTE
```

---

## dbt Model Organisation

Following the [[Core dbt Fundamentals|staging → intermediate → marts]] pattern:

| Layer | Purpose | Naming |
|-------|---------|--------|
| **staging/** | 1:1 source mirrors, light cleansing | `stg_<source>_<entity>` |
| **intermediate/** | Business logic, joins, enrichment | `int_<entity>_<action>` |
| **marts/** | Final consumption models per domain | `dim_<entity>`, `fct_<entity>` |

See [[dbt Advanced Patterns & Cost Optimisation]] for incremental strategies.

---

## Lambda Data Loader Pattern

Core pattern for S3-triggered Snowflake loading:

```python
def lambda_handler(event, context):
    # 1. Get credentials from Secrets Manager
    creds = get_secret(os.environ['SECRETS_MANAGER_ARN'])

    # 2. Process each S3 event record
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']

        # 3. Download and parse
        data = s3.get_object(Bucket=bucket, Key=key)

        # 4. Load to Snowflake
        load_to_snowflake(data, creds)
```

**Security:** Never hard-code credentials. Use Secrets Manager for Snowflake passwords, IAM roles for AWS service access.

---

## Cost Optimisation

| Strategy | Service | Savings |
|----------|---------|---------|
| S3 lifecycle policies | S3 | 60-80% on cold data |
| Lambda right-sizing | Lambda | Match memory to workload |
| Snowflake warehouse auto-suspend | Snowflake | Only pay when running |
| Step Functions Express Workflows | Step Functions | 90% cheaper for short pipelines |
| Reserved capacity | Kinesis | Up to 50% on steady-state streams |

---

**Related:** [[AWS Data Services for Data Engineering]] | [[Core dbt Fundamentals]] | [[dbt Advanced Patterns & Cost Optimisation]] | [[Terraform for Data Infrastructure]] | [[Incremental Loading Strategies]] | [[Data Ingestion Patterns]]
