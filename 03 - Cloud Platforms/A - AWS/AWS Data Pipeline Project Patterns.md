#aws #data-engineering #step-functions #glue #redshift #terraform #project-patterns

# AWS Data Pipeline Project Patterns

Production-grade patterns for building end-to-end data pipelines on AWS using Step Functions for orchestration, Glue for transformation, and Redshift for warehousing. Generalised from real-world implementations with Terraform IaC throughout.

See also: [[AWS Data Services for Data Engineering]], [[Terraform Fundamentals]], [[Data Engineering Lifecycle]], [[ETL Pipeline Template]]

---

## 1 — Project Structure

A well-organised project separates infrastructure, application logic, and configuration:

```
aws-data-pipeline/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── backend.tf
│   ├── modules/
│   │   ├── glue/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── iam.tf
│   │   ├── step_functions/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── state_machine.asl.json
│   │   ├── redshift/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── iam.tf
│   │   ├── s3/
│   │   │   ├── main.tf
│   │   │   └── variables.tf
│   │   ├── eventbridge/
│   │   │   ├── main.tf
│   │   │   └── variables.tf
│   │   └── monitoring/
│   │       ├── main.tf
│   │       └── variables.tf
│   └── environments/
│       ├── dev.tfvars
│       ├── staging.tfvars
│       └── prod.tfvars
├── glue_jobs/
│   ├── raw_to_staging.py
│   ├── staging_to_curated.py
│   ├── curated_to_redshift.py
│   └── shared/
│       ├── utils.py
│       └── quality_checks.py
├── redshift/
│   ├── ddl/
│   │   ├── schemas.sql
│   │   ├── tables.sql
│   │   └── views.sql
│   └── migrations/
│       └── V001__initial_schema.sql
├── tests/
│   ├── unit/
│   │   └── test_glue_transforms.py
│   └── integration/
│       └── test_pipeline_e2e.py
├── scripts/
│   ├── deploy.sh
│   └── seed_test_data.sh
└── Makefile
```

**Key principles:**
- Terraform modules per service — each independently testable
- Glue jobs in a flat directory with a shared utilities module
- Redshift DDL versioned with migration scripts
- Environment-specific variables via `.tfvars` files

---

## 2 — Terraform IaC Patterns

### S3 Bucket Layout

```hcl
# terraform/modules/s3/main.tf

resource "aws_s3_bucket" "data_lake" {
  for_each = toset(["raw", "staging", "curated", "artifacts"])
  bucket   = "${var.project_name}-${each.key}-${var.environment}"
}

resource "aws_s3_bucket_lifecycle_configuration" "raw_lifecycle" {
  bucket = aws_s3_bucket.data_lake["raw"].id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 365
      storage_class = "GLACIER"
    }
  }
}

resource "aws_s3_bucket_versioning" "raw_versioning" {
  bucket = aws_s3_bucket.data_lake["raw"].id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### Glue Resources

```hcl
# terraform/modules/glue/main.tf

resource "aws_glue_catalog_database" "pipeline_db" {
  name = "${var.project_name}_${var.environment}"
}

resource "aws_glue_job" "etl_job" {
  for_each = var.glue_jobs

  name         = "${var.project_name}-${each.key}-${var.environment}"
  role_arn     = aws_iam_role.glue_role.arn
  glue_version = "4.0"
  worker_type  = each.value.worker_type
  number_of_workers = each.value.num_workers
  timeout      = each.value.timeout_minutes

  command {
    script_location = "s3://${var.artifacts_bucket}/glue_jobs/${each.key}.py"
    python_version  = "3"
  }

  default_arguments = merge(
    {
      "--job-language"             = "python"
      "--enable-glue-datacatalog"  = ""
      "--enable-continuous-cloudwatch-log" = "true"
      "--enable-metrics"           = "true"
      "--additional-python-modules" = "great_expectations==0.18.0"
      "--TempDir"                  = "s3://${var.artifacts_bucket}/glue-temp/"
      "--source_bucket"            = each.value.source_bucket
      "--target_bucket"            = each.value.target_bucket
      "--database"                 = aws_glue_catalog_database.pipeline_db.name
    },
    each.value.extra_args
  )
}

resource "aws_glue_crawler" "source_crawler" {
  for_each      = var.crawler_targets
  database_name = aws_glue_catalog_database.pipeline_db.name
  name          = "${var.project_name}-${each.key}-crawler-${var.environment}"
  role          = aws_iam_role.glue_role.arn

  s3_target {
    path = each.value.s3_path
  }

  schema_change_policy {
    update_behavior = "UPDATE_IN_DATABASE"
    delete_behavior = "LOG"
  }
}
```

### IAM Roles (Least Privilege)

```hcl
# terraform/modules/glue/iam.tf

resource "aws_iam_role" "glue_role" {
  name = "${var.project_name}-glue-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "glue.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "glue_s3_access" {
  name = "glue-s3-access"
  role = aws_iam_role.glue_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          "arn:aws:s3:::${var.project_name}-*-${var.environment}",
          "arn:aws:s3:::${var.project_name}-*-${var.environment}/*"
        ]
      },
      {
        Effect   = "Allow"
        Action   = ["glue:*Database*", "glue:*Table*", "glue:*Partition*"]
        Resource = "*"
      }
    ]
  })
}
```

---

## 3 — Glue Job Patterns (PySpark on Glue)

### Standard Job Template

```python
# glue_jobs/raw_to_staging.py

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql import functions as F

args = getResolvedOptions(sys.argv, [
    "JOB_NAME", "source_bucket", "target_bucket", "database"
])

sc = SparkContext()
glue_context = GlueContext(sc)
spark = glue_context.spark_session
job = Job(glue_context)
job.init(args["JOB_NAME"], args)

# --- Read from Glue Catalog ---
source_dyf = glue_context.create_dynamic_frame.from_catalog(
    database=args["database"],
    table_name="raw_events"
)

# --- Convert to DataFrame for complex transforms ---
df = source_dyf.toDF()

# --- Apply transforms ---
df_clean = (
    df
    .filter(F.col("event_id").isNotNull())
    .withColumn("event_date", F.to_date("event_timestamp"))
    .withColumn("processed_at", F.current_timestamp())
    .withColumn("_source_file", F.input_file_name())
    .dropDuplicates(["event_id"])
)

# --- Write partitioned output ---
df_clean.write.mode("overwrite").partitionBy("event_date").parquet(
    f"s3://{args['target_bucket']}/staging/events/"
)

# --- Update Glue Catalog ---
sink = glue_context.getSink(
    connection_type="s3",
    path=f"s3://{args['target_bucket']}/staging/events/",
    enableUpdateCatalog=True,
    updateBehavior="UPDATE_IN_DATABASE",
    partitionKeys=["event_date"]
)
sink.setCatalogInfo(
    catalogDatabase=args["database"],
    catalogTableName="staging_events"
)
sink.setFormat("glueparquet")
sink.writeFrame(source_dyf)

job.commit()
```

### Shared Utilities Module

```python
# glue_jobs/shared/quality_checks.py

from pyspark.sql import DataFrame
import logging

logger = logging.getLogger(__name__)


def check_not_null(df: DataFrame, columns: list[str]) -> dict:
    """Return null counts for specified columns."""
    results = {}
    for col in columns:
        null_count = df.filter(df[col].isNull()).count()
        results[col] = null_count
        if null_count > 0:
            logger.warning(f"Column '{col}' has {null_count} null values")
    return results


def check_uniqueness(df: DataFrame, key_columns: list[str]) -> bool:
    """Verify no duplicate records on key columns."""
    total = df.count()
    distinct = df.dropDuplicates(key_columns).count()
    is_unique = total == distinct
    if not is_unique:
        logger.error(f"Uniqueness check failed: {total - distinct} duplicates")
    return is_unique


def check_row_count_threshold(df: DataFrame, min_rows: int) -> bool:
    """Ensure minimum row count (catches empty/truncated loads)."""
    count = df.count()
    passed = count >= min_rows
    if not passed:
        logger.error(f"Row count {count} below threshold {min_rows}")
    return passed
```

### Bookmark Pattern for Incremental Loads

```python
# Glue job bookmarks track already-processed data
# Enable via Terraform: --job-bookmark-option = "job-bookmark-enable"

# For custom incremental logic:
from awsglue.context import GlueContext

source_dyf = glue_context.create_dynamic_frame.from_catalog(
    database="pipeline_db",
    table_name="raw_events",
    transformation_ctx="source_bookmark"  # enables bookmarking
)
```

---

## 4 — Step Functions State Machine

### Parallel Branch Pipeline

```json
{
  "Comment": "Daily data pipeline: ingest, transform, load to Redshift",
  "StartAt": "RunCrawlers",
  "States": {
    "RunCrawlers": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "CrawlOrders",
          "States": {
            "CrawlOrders": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startCrawler.sync",
              "Parameters": { "Name": "${orders_crawler_name}" },
              "End": true
            }
          }
        },
        {
          "StartAt": "CrawlCustomers",
          "States": {
            "CrawlCustomers": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startCrawler.sync",
              "Parameters": { "Name": "${customers_crawler_name}" },
              "End": true
            }
          }
        }
      ],
      "Next": "TransformRawToStaging",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "NotifyCrawlerFailure"
      }]
    },

    "TransformRawToStaging": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "${raw_to_staging_job}",
        "Arguments": {
          "--execution_date.$": "$.execution_date"
        }
      },
      "Retry": [{
        "ErrorEquals": ["Glue.AWSGlueException"],
        "IntervalSeconds": 60,
        "MaxAttempts": 2,
        "BackoffRate": 2.0
      }],
      "Next": "TransformStagingToCurated",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "NotifyTransformFailure"
      }]
    },

    "TransformStagingToCurated": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "${staging_to_curated_job}",
        "Arguments": {
          "--execution_date.$": "$.execution_date"
        }
      },
      "Retry": [{
        "ErrorEquals": ["Glue.AWSGlueException"],
        "IntervalSeconds": 60,
        "MaxAttempts": 2,
        "BackoffRate": 2.0
      }],
      "Next": "QualityGate",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "NotifyTransformFailure"
      }]
    },

    "QualityGate": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${quality_check_lambda}",
        "Payload.$": "$"
      },
      "ResultPath": "$.quality_result",
      "Next": "QualityDecision"
    },

    "QualityDecision": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.quality_result.Payload.passed",
        "BooleanEquals": true,
        "Next": "LoadToRedshift"
      }],
      "Default": "NotifyQualityFailure"
    },

    "LoadToRedshift": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "${curated_to_redshift_job}"
      },
      "Retry": [{
        "ErrorEquals": ["Glue.AWSGlueException"],
        "IntervalSeconds": 120,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }],
      "Next": "PipelineSuccess",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "ResultPath": "$.error",
        "Next": "NotifyLoadFailure"
      }]
    },

    "PipelineSuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${sns_topic_arn}",
        "Message": "Pipeline completed successfully",
        "Subject": "Pipeline Success"
      },
      "End": true
    },

    "NotifyCrawlerFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${sns_topic_arn}",
        "Message": "Crawler step failed",
        "Subject": "Pipeline Failure - Crawlers"
      },
      "End": true
    },

    "NotifyTransformFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${sns_topic_arn}",
        "Message.$": "States.Format('Transform failed: {}', $.error.Cause)",
        "Subject": "Pipeline Failure - Transform"
      },
      "End": true
    },

    "NotifyQualityFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${sns_topic_arn}",
        "Message": "Data quality checks failed — load blocked",
        "Subject": "Pipeline Failure - Quality Gate"
      },
      "End": true
    },

    "NotifyLoadFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "${sns_topic_arn}",
        "Message.$": "States.Format('Redshift load failed: {}', $.error.Cause)",
        "Subject": "Pipeline Failure - Redshift Load"
      },
      "End": true
    }
  }
}
```

### Error Handling & Retry Strategy

| Pattern | Implementation | When to Use |
|---------|---------------|-------------|
| **Retry with backoff** | `Retry` field on Task states | Transient failures (Glue timeouts, throttling) |
| **Catch and notify** | `Catch` to SNS/Lambda | Permanent failures requiring human intervention |
| **Quality gate** | Choice state after Lambda check | Block bad data from reaching the warehouse |
| **Parallel with shared catch** | `Catch` on Parallel state | Independent tasks that can fail independently |
| **Map state** | `Map` with `MaxConcurrency` | Process multiple partitions/files in parallel |

### Step Functions Terraform

```hcl
# terraform/modules/step_functions/main.tf

resource "aws_sfn_state_machine" "pipeline" {
  name     = "${var.project_name}-pipeline-${var.environment}"
  role_arn = aws_iam_role.step_functions_role.arn

  definition = templatefile("${path.module}/state_machine.asl.json", {
    orders_crawler_name     = var.orders_crawler_name
    customers_crawler_name  = var.customers_crawler_name
    raw_to_staging_job      = var.raw_to_staging_job
    staging_to_curated_job  = var.staging_to_curated_job
    curated_to_redshift_job = var.curated_to_redshift_job
    quality_check_lambda    = var.quality_check_lambda_arn
    sns_topic_arn           = var.sns_topic_arn
  })

  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.sfn.arn}:*"
    include_execution_data = true
    level                  = "ALL"
  }
}
```

---

## 5 — Redshift Loading Patterns

### COPY from S3

The `COPY` command is the most efficient way to load data into Redshift. It parallelises reads across slices.

```sql
-- Standard COPY from Parquet
COPY analytics.fact_orders
FROM 's3://project-curated-prod/orders/'
IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-copy-role'
FORMAT AS PARQUET;

-- CSV with options
COPY analytics.dim_customers
FROM 's3://project-curated-prod/customers/customers.csv.gz'
IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-copy-role'
FORMAT AS CSV
GZIP
IGNOREHEADER 1
DATEFORMAT 'auto'
TIMEFORMAT 'auto'
MAXERROR 100
COMPUPDATE ON;

-- Manifest file for explicit file control
COPY analytics.fact_orders
FROM 's3://project-curated-prod/manifests/orders_20260315.manifest'
IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-copy-role'
FORMAT AS PARQUET
MANIFEST;
```

### Redshift Spectrum (Query S3 Directly)

```sql
-- Create external schema pointing to Glue Catalog
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'pipeline_db'
IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-spectrum-role'
REGION 'eu-west-2';

-- Query S3 data directly (cold/historical data)
SELECT event_date, COUNT(*) AS event_count
FROM spectrum_schema.curated_events
WHERE event_date BETWEEN '2026-01-01' AND '2026-03-01'
GROUP BY event_date;

-- Join Spectrum (S3) with local Redshift tables
SELECT c.customer_name, COUNT(e.event_id) AS events
FROM spectrum_schema.curated_events e
JOIN analytics.dim_customers c ON e.customer_id = c.customer_id
GROUP BY c.customer_name;
```

### Distribution & Sort Keys

```sql
-- Fact table: distribute on the most common join key
CREATE TABLE analytics.fact_orders (
    order_id        BIGINT       NOT NULL,
    customer_id     BIGINT       NOT NULL,
    product_id      BIGINT       NOT NULL,
    order_date      DATE         NOT NULL SORTKEY,
    quantity        INTEGER      NOT NULL,
    total_amount    DECIMAL(12,2) NOT NULL,
    region          VARCHAR(50)
)
DISTSTYLE KEY
DISTKEY (customer_id);

-- Dimension table: ALL distribution for small tables (< 3M rows)
CREATE TABLE analytics.dim_products (
    product_id      BIGINT       NOT NULL SORTKEY,
    product_name    VARCHAR(256) NOT NULL,
    category        VARCHAR(128),
    subcategory     VARCHAR(128)
)
DISTSTYLE ALL;

-- Large dimension: KEY distribution matching fact table join
CREATE TABLE analytics.dim_customers (
    customer_id     BIGINT       NOT NULL SORTKEY,
    customer_name   VARCHAR(256) NOT NULL,
    segment         VARCHAR(64),
    region          VARCHAR(50)
)
DISTSTYLE KEY
DISTKEY (customer_id);
```

| Distribution Style | When to Use |
|-------------------|-------------|
| **KEY** | Large tables frequently joined — co-locate rows with same key on same slice |
| **ALL** | Small dimension tables (< 3M rows) — replicated to every slice, avoids redistribution |
| **EVEN** | Tables not joined or no clear key — round-robin across slices |
| **AUTO** | Let Redshift choose (starts as ALL, switches to EVEN at scale) |

| Sort Key Type | When to Use |
|--------------|-------------|
| **Compound** | Queries consistently filter on a prefix of the sort columns (e.g., date then region) |
| **Interleaved** | Queries filter on different columns unpredictably — equal weight to each key column |

### Incremental Load Pattern (Glue to Redshift)

```python
# glue_jobs/curated_to_redshift.py

# Use staging table pattern for idempotent loads
redshift_tmp_dir = f"s3://{args['artifacts_bucket']}/redshift-temp/"

# Write to staging table
glue_context.write_dynamic_frame.from_jdbc_conf(
    frame=curated_dyf,
    catalog_connection="redshift-connection",
    connection_options={
        "dbtable": "staging.stg_orders",
        "database": "analytics_db",
        "preactions": "TRUNCATE staging.stg_orders;",
        "postactions": """
            DELETE FROM analytics.fact_orders
            USING staging.stg_orders
            WHERE analytics.fact_orders.order_id = staging.stg_orders.order_id;

            INSERT INTO analytics.fact_orders
            SELECT * FROM staging.stg_orders;

            DROP TABLE IF EXISTS staging.stg_orders;
        """
    },
    redshift_tmp_dir=redshift_tmp_dir,
    transformation_ctx="redshift_sink"
)
```

---

## 6 — EventBridge Scheduling

```hcl
# terraform/modules/eventbridge/main.tf

resource "aws_scheduler_schedule" "daily_pipeline" {
  name                = "${var.project_name}-daily-${var.environment}"
  schedule_expression = "cron(0 6 * * ? *)"   # 06:00 UTC daily
  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = var.state_machine_arn
    role_arn = aws_iam_role.scheduler_role.arn

    input = jsonencode({
      execution_date = "placeholder"  # Overridden by Step Functions input transform
      pipeline_run   = "scheduled"
    })

    retry_policy {
      maximum_event_age_in_seconds = 3600
      maximum_retry_attempts       = 2
    }

    dead_letter_config {
      arn = var.dlq_arn
    }
  }
}

# Event-driven trigger (e.g., new file lands in S3)
resource "aws_cloudwatch_event_rule" "s3_landing" {
  name        = "${var.project_name}-s3-trigger-${var.environment}"
  description = "Trigger pipeline when new data lands in raw bucket"

  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = { name = [var.raw_bucket_name] }
      object = { key  = [{ prefix = "incoming/" }] }
    }
  })
}

resource "aws_cloudwatch_event_target" "trigger_pipeline" {
  rule     = aws_cloudwatch_event_rule.s3_landing.name
  arn      = var.state_machine_arn
  role_arn = aws_iam_role.scheduler_role.arn
}
```

---

## 7 — CloudWatch Monitoring

### Glue Job Metrics

```hcl
# terraform/modules/monitoring/main.tf

resource "aws_cloudwatch_metric_alarm" "glue_job_failure" {
  for_each = var.glue_job_names

  alarm_name          = "${each.value}-failure-${var.environment}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "glue.driver.aggregate.numFailedTasks"
  namespace           = "Glue"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Glue job ${each.value} has failed tasks"
  alarm_actions       = [var.sns_topic_arn]

  dimensions = {
    JobName = each.value
    JobRunId = "ALL"
    Type     = "gauge"
  }
}

resource "aws_cloudwatch_metric_alarm" "glue_job_duration" {
  for_each = var.glue_job_names

  alarm_name          = "${each.value}-duration-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "glue.driver.aggregate.elapsedTime"
  namespace           = "Glue"
  period              = 300
  statistic           = "Maximum"
  threshold           = var.duration_threshold_ms
  alarm_description   = "Glue job ${each.value} exceeding expected duration"
  alarm_actions       = [var.sns_topic_arn]

  dimensions = {
    JobName = each.value
    JobRunId = "ALL"
    Type     = "gauge"
  }
}
```

### Step Functions Monitoring

```hcl
resource "aws_cloudwatch_metric_alarm" "sfn_execution_failure" {
  alarm_name          = "${var.project_name}-sfn-failure-${var.environment}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "ExecutionsFailed"
  namespace           = "AWS/States"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [var.sns_topic_arn]

  dimensions = {
    StateMachineArn = var.state_machine_arn
  }
}

resource "aws_cloudwatch_metric_alarm" "sfn_execution_throttled" {
  alarm_name          = "${var.project_name}-sfn-throttled-${var.environment}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "ExecutionThrottled"
  namespace           = "AWS/States"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [var.sns_topic_arn]

  dimensions = {
    StateMachineArn = var.state_machine_arn
  }
}
```

### CloudWatch Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "pipeline" {
  dashboard_name = "${var.project_name}-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Step Functions Executions"
          metrics = [
            ["AWS/States", "ExecutionsStarted", "StateMachineArn", var.state_machine_arn],
            ["AWS/States", "ExecutionsSucceeded", "StateMachineArn", var.state_machine_arn],
            ["AWS/States", "ExecutionsFailed", "StateMachineArn", var.state_machine_arn]
          ]
          period = 86400
          stat   = "Sum"
          region = var.region
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Redshift Query Performance"
          metrics = [
            ["AWS/Redshift", "QueryDuration", "ClusterIdentifier", var.redshift_cluster_id],
            ["AWS/Redshift", "CPUUtilization", "ClusterIdentifier", var.redshift_cluster_id]
          ]
          period = 300
          stat   = "Average"
          region = var.region
        }
      }
    ]
  })
}
```

---

## 8 — Operational Patterns

### Deployment Pipeline

```makefile
# Makefile

ENV ?= dev

deploy-infra:
	cd terraform && terraform init && \
	terraform plan -var-file=environments/$(ENV).tfvars -out=plan.out && \
	terraform apply plan.out

upload-glue-jobs:
	aws s3 sync glue_jobs/ s3://$(PROJECT)-artifacts-$(ENV)/glue_jobs/ \
		--exclude "__pycache__/*" --exclude "*.pyc"

deploy-all: deploy-infra upload-glue-jobs

test-unit:
	cd tests && python -m pytest unit/ -v

test-integration:
	cd tests && python -m pytest integration/ -v --env=$(ENV)
```

### Idempotency Checklist

- Glue jobs use staging table pattern — re-runs produce the same result
- Step Functions executions use unique name (date-based) to prevent duplicates
- Redshift loads use DELETE + INSERT (not just INSERT) via staging table
- S3 writes use overwrite mode per partition, not append
- EventBridge DLQ captures failed invocations for replay

### Cost Optimisation

| Lever | Approach |
|-------|----------|
| **Glue DPU** | Start with G.1X (4 vCPU, 16 GB); scale to G.2X only if OOM. Monitor DPU utilisation via CloudWatch |
| **Glue Flex** | Use Flex execution for non-urgent batch jobs (up to 35% cheaper) |
| **S3 lifecycle** | Transition raw data to IA/Glacier after retention period |
| **Redshift RA3** | Use RA3 nodes — separate compute/storage billing. Pause clusters outside business hours |
| **Redshift Serverless** | For variable workloads — pay per RPU-second |
| **Spectrum** | Keep cold/historical data in S3 and query via Spectrum instead of loading into Redshift |
| **Reserved capacity** | Reserve Redshift nodes for predictable base load |

---

## Key Takeaways

1. **Modular Terraform** — one module per AWS service, environment-specific variables, and `templatefile` for injecting resource names into state machines.
2. **Step Functions as orchestrator** — native Glue integration (`.sync` pattern), parallel branches for independent work, retry with exponential backoff, and SNS notifications for every failure path.
3. **Glue job bookmarks** — enable incremental processing without custom watermark logic.
4. **Staging table pattern** — the only safe way to do idempotent loads into Redshift. DELETE + INSERT via `preactions`/`postactions`.
5. **Distribution keys matter** — co-locate fact and dimension tables on the join key. Use ALL distribution for small dimensions.
6. **EventBridge for scheduling** — preferred over CloudWatch Events. Supports cron, rate, and event-driven triggers with DLQ and retry policies.
7. **Monitor everything** — Glue metrics, Step Functions execution status, and Redshift query performance on a single CloudWatch dashboard.

---

*Related:* [[AWS Data Services for Data Engineering]] | [[Terraform Fundamentals]] | [[Data Engineering Lifecycle]] | [[ETL Pipeline Template]] | [[PySpark on AWS Glue]] | [[Monitoring & Alerting]]
