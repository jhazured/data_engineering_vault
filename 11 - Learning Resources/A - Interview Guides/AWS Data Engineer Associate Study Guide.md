tags: #aws #certification #data-engineering #dea-c01 #study-guide

# AWS Data Engineer Associate Study Guide

> Comprehensive revision guide for the AWS Certified Data Engineer - Associate (DEA-C01) examination. Cross-references vault notes on [[AWS Data Services for Data Engineering]], [[Data Ingestion Patterns]], and [[Apache Kafka Fundamentals]].

---

## Table Of Contents

1. [[#Exam Overview]]
2. [[#Domain 1 Data Ingestion And Transformation (34%)]]
3. [[#Domain 2 Data Store Management (26%)]]
4. [[#Domain 3 Data Operations And Support (22%)]]
5. [[#Domain 4 Data Security And Governance (18%)]]
6. [[#Key Exam Tips And Decision Matrix]]
7. [[#Selected Practice Questions With Explanations]]

---

## Exam Overview

### Format And Logistics

| Detail           | Value                                      |
| ---------------- | ------------------------------------------ |
| Exam code        | DEA-C01                                    |
| Level            | Associate                                  |
| Duration         | 130 minutes                                |
| Questions        | 65 (multiple choice / multiple response)   |
| Passing score    | ~720-750 out of 1000 (approximately 72-75%)|
| Cost             | $150 USD                                   |

### Domain Weights And Study Time Allocation

| Domain | Weight | Recommended Study Time |
| ------ | ------ | ---------------------- |
| 1. Data Ingestion and Transformation | 34% | 40% -- highest priority |
| 2. Data Store Management            | 26% | 30% -- second priority  |
| 3. Data Operations and Support      | 22% | 20% -- medium priority  |
| 4. Data Security and Governance     | 18% | 10% -- lower priority but essential |

---

## Domain 1 Data Ingestion And Transformation (34%)

This is the heaviest-weighted domain. Mastering it is the single most impactful thing you can do for your score. See also [[Data Ingestion Patterns]] and [[ETL Pipeline Design]].

### 1.1 AWS Glue -- The ETL Foundation

#### Core Components

- **Data Catalog** -- centralised metadata repository (databases, tables, partitions, connections)
- **Crawlers** -- automatic schema discovery; scan data stores, infer schema, create/update catalog tables
- **ETL Jobs** -- transform data with Python (PySpark) or Scala
- **Triggers** -- automate job execution (scheduled, conditional, on-demand)
- **Development Endpoints** -- interactive development environment

#### DynamicFrame vs DataFrame

DynamicFrame is the Glue-native abstraction; it handles schema evolution and missing fields gracefully. DataFrame (Spark) requires a fixed schema but offers richer transformations.

```python
# DynamicFrame -- handles schema variations
dynamic_frame = glueContext.create_dynamic_frame.from_catalog(
    database="my_database",
    table_name="my_table"
)

# Convert to DataFrame when you need Spark SQL features
data_frame = dynamic_frame.toDF()
```

Key DynamicFrame transformations to know:

| Transformation   | Purpose                                |
| ---------------- | -------------------------------------- |
| `ApplyMapping`   | Rename and recast columns              |
| `Filter`         | Remove unwanted records                |
| `Join`           | Combine datasets                       |
| `Relationalize`  | Flatten nested structures              |
| `ResolveChoice`  | Handle data type ambiguity             |

#### Job Optimisation Strategies

- **Job Bookmarks** -- process only new/changed data; supported for S3, RDS, and JDBC sources (not NoSQL)
- **Worker types** -- Standard (balanced), G.1X (4 vCPU / 16 GB), G.2X (8 vCPU / 32 GB)
- **File format** -- use Parquet for analytics workloads
- **Partitioning** -- enables parallel processing and partition pruning
- **Spark UI** -- identify bottlenecks via executor metrics
- **Adaptive file grouping** -- optimises processing of mixed small/large files

#### Error Handling

```python
try:
    transformed_data = ApplyMapping.apply(
        frame=source_data,
        mappings=mapping_spec
    )
except Exception as e:
    logger.error(f"Transformation failed: {str(e)}")
    # Alert via CloudWatch or SNS
```

**Exit code 143** on a Glue job almost always means the container was killed due to memory pressure -- increase DPU allocation.

#### Glue Schema Registry

Supports schema evolution with compatibility modes:

- **BACKWARD** -- consumers using old schema can read new data
- **FORWARD** -- consumers can read data produced with a newer schema (allows adding optional fields)
- **FULL** -- both directions
- **NONE** -- no compatibility checks

#### Glue Data Quality

Rule types: Completeness (null checks), Uniqueness, Validity, Consistency.

### 1.2 Amazon Kinesis -- Real-Time Streaming

See also [[Apache Kafka Fundamentals]] for comparison with MSK.

#### Kinesis Family

| Service            | Purpose                              |
| ------------------ | ------------------------------------ |
| Data Streams       | Real-time data ingestion             |
| Data Firehose      | Managed delivery to S3, Redshift, etc.|
| Data Analytics     | Real-time SQL/Flink on streams       |
| Video Streams      | Video ingestion and processing       |

#### Data Streams -- Capacity Planning

Each shard provides:
- **Write**: 1,000 records/sec OR 1 MB/sec
- **Read**: 2 MB/sec (shared) or 2 MB/sec per consumer (enhanced fan-out)

```
Required shards = max(
    records_per_second / 1000,
    data_throughput_MB_per_second / 1
)
```

**Example**: 50,000 records/sec at 2 KB each = 100 MB/sec throughput.
`max(50, 100) = 100 shards`.

Exceeding shard limits throws `ProvisionedThroughputExceededException`.

**Producer patterns**: KPL (high-throughput batching), AWS SDK (simple), Kinesis Agent (log files).
**Consumer patterns**: KCL (auto load-balancing), Lambda (event-driven), Kinesis Analytics (SQL).
**Scaling**: resharding (split/merge), on-demand mode, enhanced fan-out.

#### Data Firehose

- Delivery destinations: S3 (most common), Redshift, OpenSearch, HTTP endpoints
- **Minimum buffer interval: 60 seconds** -- if you need sub-60-second delivery, use Data Streams + Lambda instead
- Buffer size: 1-128 MB
- Supports inline Lambda transformation
- Error records routed to a separate S3 prefix

#### Data Analytics -- Window Types

```sql
-- TUMBLING (non-overlapping, fixed intervals)
SELECT ticker, AVG(price)
FROM SOURCE_SQL_STREAM_001
GROUP BY ticker,
  ROWTIME RANGE INTERVAL '1' MINUTE PRECEDING;

-- SLIDING (overlapping, ideal for anomaly detection)
SELECT ticker, AVG(price)
FROM SOURCE_SQL_STREAM_001
GROUP BY ticker,
  ROWTIME RANGE INTERVAL '5' MINUTE PRECEDING;

-- SESSION (activity-based, gaps trigger new window)
SELECT user_id, COUNT(*)
FROM SOURCE_SQL_STREAM_001
GROUP BY user_id,
  SESSION(ROWTIME RANGE INTERVAL '10' MINUTE);
```

`ROWTIME` refers to ingestion time, not application time. Use SLIDING windows with watermarks to handle late-arriving data.

### 1.3 Amazon EMR -- Big Data Processing

#### Deployment Models

| Model          | Best For                              | Startup Time |
| -------------- | ------------------------------------- | ------------ |
| EMR on EC2     | Full control, custom configs          | Fastest (long-running clusters) |
| EMR on EKS     | Kubernetes integration                | Medium       |
| EMR Serverless | Pay-per-use, auto-scaling             | Medium       |

#### Cluster Architecture

- **Master node** -- ResourceManager, query coordination
- **Core nodes** -- DataNode + NodeManager (store data)
- **Task nodes** -- compute only (ideal for Spot instances)

#### Spark Optimisation On EMR

```properties
spark.executor.memory=4g
spark.executor.cores=2
spark.executor.instances=10
spark.driver.memory=2g
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
```

- **Broadcast joins** -- broadcast the smaller table to avoid expensive shuffles
- **Bootstrap Actions** -- run scripts on nodes during startup to install software or configure settings
- **YARN "Container killed"** errors -- typically indicate executor memory issues; increase `spark.executor.memory`

#### Cost Optimisation

- **Spot instances** for task nodes (50-90% savings)
- **Transient clusters** -- terminate after job completion, store data in S3
- **Reserved instances** -- for predictable, long-running workloads
- **Auto-scaling** -- resize cluster based on load

### 1.4 AWS Database Migration Service (DMS)

#### Migration Types

| Type             | Use Case                                     |
| ---------------- | -------------------------------------------- |
| Full load        | One-time migration of existing data          |
| CDC              | Ongoing replication of changes               |
| Full load + CDC  | Complete migration with continuous sync       |

For minimal-downtime migrations (e.g. 50 TB Oracle to RDS), use **Full load + CDC**.

#### Key Concepts

- **Replication instance** -- Multi-AZ for HA; right-size for throughput
- **AWS SCT** (Schema Conversion Tool) -- converts schema between heterogeneous engines
- **Transactional apply mode** -- maintains transaction boundaries during replication
- **Increasing target lag** -- usually means the target database has insufficient IOPS

### 1.5 Data Format Optimisation

| Format  | Type     | Best For              | Pros                      | Cons                 |
| ------- | -------- | --------------------- | ------------------------- | -------------------- |
| CSV     | Row      | Simple interchange    | Human-readable            | Large, no schema     |
| JSON    | Row      | Semi-structured data  | Flexible schema           | Verbose, slow parse  |
| Parquet | Columnar | Analytics             | Column pruning, compressed| Write overhead       |
| ORC     | Columnar | Hive/Hadoop workloads | Optimised compression     | Limited ecosystem    |
| Avro    | Row      | Schema evolution      | Schema embedded           | Row-based            |

**Compression trade-offs**:

| Algorithm | Ratio  | Speed     | Use Case                  |
| --------- | ------ | --------- | ------------------------- |
| GZIP      | High   | Slow      | Archival, space-critical  |
| SNAPPY    | Medium | Fast      | Real-time, analytics      |
| LZ4       | Low    | Very fast | High-throughput streaming  |
| ZSTD      | High   | Medium    | Balanced performance      |

**Parquet + Snappy** is the gold standard for analytical queries on Athena.

---

## Domain 2 Data Store Management (26%)

See also [[Data Modelling Fundamentals]] and [[AWS Data Services for Data Engineering]].

### 2.1 Amazon S3 -- The Data Lake Foundation

#### Storage Classes

| Class              | Access Pattern          | Retrieval  | Relative Cost |
| ------------------ | ----------------------- | ---------- | ------------- |
| Standard           | Frequent                | ms         | Highest       |
| Standard-IA        | Infrequent              | ms         | Medium        |
| One Zone-IA        | Non-critical, infrequent| ms         | Lower         |
| Glacier Instant    | Archive, instant access | ms         | Low           |
| Glacier Flexible   | Archive, occasional     | 1-5 min    | Very low      |
| Deep Archive       | Long-term archive       | 12 hours   | Lowest        |
| Intelligent-Tiering| Unknown patterns        | Variable   | Optimised     |

#### Lifecycle Policy Example

```json
{
  "Rules": [{
    "Id": "DataLakeLifecycle",
    "Status": "Enabled",
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ]
  }]
}
```

#### Partitioning Strategies

```
s3://bucket/year=2024/month=01/day=15/    -- Best for date-range queries
s3://bucket/data_type=logs/date=2024-01-15/
s3://bucket/region=us-east-1/date=2024-01-15/
```

Year/Month/Day hierarchy allows efficient partition pruning in Athena and Redshift Spectrum.

#### Performance Tips

- Multipart upload for large files
- Transfer Acceleration via CloudFront edge locations
- S3 Select for extracting subsets without downloading entire objects
- Compact many small files into larger ones to reduce metadata overhead

### 2.2 Amazon Redshift -- Data Warehousing

#### Distribution Strategies

```sql
-- KEY: co-locate rows that join together
CREATE TABLE sales (
    sale_id INT, customer_id INT, amount DECIMAL(10,2)
) DISTKEY(customer_id);

-- ALL: replicate small dimension tables to every node
CREATE TABLE products (
    product_id INT, product_name VARCHAR(100)
) DISTSTYLE ALL;

-- EVEN: round-robin for standalone tables
CREATE TABLE logs (
    log_id BIGINT, message TEXT
) DISTSTYLE EVEN;
```

**Rule of thumb**: use `DISTSTYLE ALL` for dimension tables under ~2 million rows; use `DISTKEY` on the join column of large fact tables.

#### Sort Key Strategies

- **Compound sort key** -- optimised for prefix-matching filters (e.g. date then event_type)
- **Interleaved sort key** -- gives equal weight to multiple filter columns; better when queries filter on different column combinations

#### Maintenance Operations

```sql
VACUUM table_name;   -- reclaim space, re-sort
ANALYZE table_name;  -- update statistics for query planner
```

Poor performance with high disk I/O? Run `VACUUM` and `ANALYZE` first before adding nodes.

#### Redshift Spectrum

Query data directly in S3 via external tables. Convert JSON to Parquet for dramatically better Spectrum performance.

```sql
CREATE EXTERNAL SCHEMA s3_data
FROM DATA CATALOG DATABASE 's3_database'
IAM_ROLE 'arn:aws:iam::account:role/RedshiftSpectrumRole';
```

#### Concurrency Scaling And WLM

- **Concurrency scaling** -- automatically adds transient clusters during peak query load
- **Workload Management (WLM)** -- allocate memory and concurrency slots per queue to isolate workloads
- **Materialised views** -- require manual `REFRESH MATERIALIZED VIEW` to reflect base table changes
- **Redshift Serverless** -- auto-scales, pay per usage; good for unpredictable/spiky workloads

### 2.3 Amazon DynamoDB -- NoSQL At Scale

#### Core Constraints

- Maximum item size: **400 KB** (including all attribute names and values)
- Partition key design is critical to avoid hot partitions

#### Partition Key Design

**Anti-patterns**: sequential IDs, timestamps alone, single high-volume user IDs.

**Good patterns**:
```python
# Composite partition key
partition_key = f"{user_id}#{date}"

# Random suffix for write-heavy workloads
partition_key = f"{base_key}#{random.randint(1, 10)}"
```

#### Access Patterns And GSI Design

```
Primary Key:  PK=user_id, SK=order_date
GSI1:         PK=order_status, SK=order_date   -- query by status
GSI2:         PK=product_id, SK=order_date     -- query by product
```

Single-table design with composite sort keys supports the most access patterns efficiently.

#### Capacity Modes

| Mode         | Best For                          |
| ------------ | --------------------------------- |
| Provisioned  | Predictable traffic; use auto-scaling |
| On-demand    | Unpredictable/spiky traffic       |

**Throttling with even distribution?** Enable auto-scaling (most cost-effective fix).

#### Global Tables And Streams

- **Global Tables** -- multi-region replication with **eventual consistency** across regions
- **DynamoDB Streams** -- change data capture; `NEW_AND_OLD_IMAGES` gives full before/after snapshots

### 2.4 Other Storage Services

| Service     | Primary Use Case                        |
| ----------- | --------------------------------------- |
| RDS/Aurora  | OLTP, ACID compliance, relational       |
| DocumentDB  | MongoDB-compatible document storage     |
| Neptune     | Graph databases (social, fraud)         |
| Timestream  | Time-series (IoT, telemetry); auto-tiering with up to 365-day memory store |
| FSx Lustre  | High-performance file system with S3 integration; ideal for ML training |
| ElastiCache | Sub-millisecond lookups; Redis Sorted Sets for leaderboards |

---

## Domain 3 Data Operations And Support (22%)

See also [[Data Pipeline Orchestration]] and [[Observability in Data Systems]].

### 3.1 Monitoring And Observability

#### Key CloudWatch Metrics

| Service   | Critical Metrics                    | Alarm Threshold Guidance   |
| --------- | ----------------------------------- | -------------------------- |
| Glue      | Job duration, success rate          | Duration > 2x baseline     |
| Kinesis   | IncomingRecords, IteratorAge        | IteratorAge > 30 sec       |
| EMR       | CPU utilisation, memory usage       | CPU > 80%                  |
| Redshift  | CPU utilisation, query duration     | Query time > 5 min         |
| DynamoDB  | ConsumedCapacity, ThrottledRequests | Throttles > 0              |

#### Custom Metrics For Data Pipelines

```python
import boto3
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='DataPipeline/Quality',
    MetricData=[{
        'MetricName': 'RecordsProcessed',
        'Value': record_count,
        'Unit': 'Count',
        'Dimensions': [
            {'Name': 'Pipeline', 'Value': 'daily-etl'},
            {'Name': 'Environment', 'Value': 'production'}
        ]
    }]
)
```

Use **Spark UI** (not just CloudWatch Logs) for detailed memory/executor metrics when debugging Glue job failures.

### 3.2 Automation And Orchestration

#### AWS Step Functions

Best for complex dependency management, error handling, and parallel processing in time-critical workflows.

```json
{
  "StartAt": "DataValidation",
  "States": {
    "DataValidation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:validate-data",
      "Next": "ProcessData",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "HandleError"
      }]
    },
    "ProcessData": {
      "Type": "Parallel",
      "Branches": [{
        "StartAt": "TransformData",
        "States": {
          "TransformData": {
            "Type": "Task",
            "Resource": "arn:aws:states:::glue:startJobRun.sync",
            "Parameters": { "JobName": "transform-job" },
            "End": true
          }
        }
      }],
      "Next": "DataQualityCheck"
    }
  }
}
```

#### Glue Workflows

```python
# Scheduled trigger
glue.create_trigger(
    Name='schedule-trigger',
    Type='SCHEDULED',
    Schedule='cron(0 2 * * ? *)',
    Actions=[{'JobName': 'extract-job'}]
)

# Conditional trigger -- runs only after upstream job succeeds
glue.create_trigger(
    Name='conditional-trigger',
    Type='CONDITIONAL',
    Predicate={
        'Conditions': [{
            'LogicalOperator': 'EQUALS',
            'JobName': 'extract-job',
            'State': 'SUCCEEDED'
        }]
    },
    Actions=[{'JobName': 'transform-job'}]
)
```

#### Amazon EventBridge

Event-driven architecture: S3 object creation triggers Step Functions for dependency-aware processing.

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {"name": ["my-data-bucket"]},
    "object": {"key": [{"prefix": "raw-data/"}]}
  }
}
```

### 3.3 Performance Optimisation

#### Athena Optimisation

- Use **partition projection** to avoid Glue Catalog calls at query time
- Store data as **Parquet with Snappy** compression
- **Compact small files** -- many small files cause metadata overhead and non-linear query time growth

```sql
CREATE TABLE logs (timestamp string, message string, level string)
PARTITIONED BY (year string, month string, day string)
STORED AS PARQUET
TBLPROPERTIES (
    'projection.enabled'='true',
    'projection.year.type'='integer',
    'projection.year.range'='2020,2030',
    'projection.month.type'='integer',
    'projection.month.range'='1,12',
    'projection.day.type'='integer',
    'projection.day.range'='1,31'
);
```

#### Spark Join Optimisation

```python
from pyspark.sql.functions import broadcast

large_df = spark.read.parquet("s3://bucket/large-dataset/")
small_df = spark.read.parquet("s3://bucket/small-dataset/")

# Broadcast the smaller table to avoid shuffles
result = large_df.join(broadcast(small_df), "key")
```

### 3.4 Error Handling And Recovery

- **Exponential backoff with jitter** for transient failures
- **Dead Letter Queues (DLQ)** on SQS for failed messages (`maxReceiveCount: 3`)
- **Idempotency keys in DynamoDB** for exactly-once Lambda processing
- **SQS buffer** between producers and Kinesis to prevent data loss during backpressure

---

## Domain 4 Data Security And Governance (18%)

See also [[Data Governance Frameworks]] and [[Encryption at Rest and in Transit]].

### 4.1 IAM Best Practices

#### Principle Of Least Privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::data-bucket/processed/*"
    },
    {
      "Effect": "Allow",
      "Action": "glue:StartJobRun",
      "Resource": "arn:aws:glue:region:account:job/etl-*"
    }
  ]
}
```

#### Cross-Account Access

Use **cross-account IAM roles with external IDs** -- never shared IAM users or access keys.

```json
{
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-external-id"
    }
  }
}
```

#### Multi-Account Strategy

Separate AWS accounts per business domain provides the strongest isolation for data mesh architectures. Use **AWS Organisations SCPs** to enforce guardrails.

### 4.2 Encryption And Key Management

#### KMS Key Types

| Type              | Control Level        | Use Case                          |
| ----------------- | -------------------- | --------------------------------- |
| AWS managed       | Low (AWS controls)   | Default, low-effort encryption    |
| Customer managed  | High (you control)   | Regulatory, key rotation control  |
| CloudHSM          | Highest (dedicated)  | FIPS 140-2 Level 3, custom        |

**Customer managed KMS keys** provide full control over encryption key lifecycle -- prefer these for sensitive financial data in Redshift.

#### Encryption At Rest

```python
# S3 default encryption with KMS
s3_client.put_bucket_encryption(
    Bucket='encrypted-data-bucket',
    ServerSideEncryptionConfiguration={
        'Rules': [{
            'ApplyServerSideEncryptionByDefault': {
                'SSEAlgorithm': 'aws:kms',
                'KMSMasterKeyID': 'arn:aws:kms:region:account:key/key-id'
            }
        }]
    }
)
```

#### Encryption In Transit

Force HTTPS on S3 with a bucket policy denying requests where `aws:SecureTransport` is `false`.

### 4.3 AWS Lake Formation

#### Fine-Grained Access Control

Lake Formation provides the most comprehensive data lake security: database, table, column, row, and cell-level permissions.

- **LF-TBAC** -- Lake Formation Tag-Based Access Control; assign tags to resources and grant permissions by tag
- **Row-level security** -- filter rows automatically based on user context
- **Column-level permissions** -- exclude sensitive columns (e.g. SSN, credit card) per role

```python
# Grant column-level access, excluding PII
lakeformation.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::account:role/AnalystRole'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'sales_db',
            'Name': 'customers',
            'ColumnWildcard': {
                'ExcludedColumnNames': ['ssn', 'credit_card']
            }
        }
    },
    Permissions=['SELECT']
)
```

### 4.4 Data Privacy And Compliance

| Requirement | Solution |
| ----------- | -------- |
| GDPR right to erasure | Delta Lake with ACID transactions for true record-level deletes |
| Dynamic data masking   | Lake Formation column/cell-level security, or Glue DataBrew masking |
| PCI tokenisation       | AWS Payment Cryptography |
| Comprehensive audit    | CloudTrail with data event logging for S3 and Glue |
| Compliance rules       | AWS Config (e.g. `S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED`) |
| HIPAA                  | Use only HIPAA-eligible services, sign BAA, encrypt everything, audit logs |

### 4.5 Audit And Compliance Monitoring

Configure **CloudTrail** with:
- Multi-region trail
- Log file validation enabled
- Data event logging for S3 objects and Glue tables
- Delivery to a secured, encrypted S3 bucket

---

## Key Exam Tips And Decision Matrix

### Service Selection Decision Matrix

| Scenario | Choose | Why |
| -------- | ------ | --- |
| Near real-time (<60 sec) delivery to consumers | Kinesis Data Streams + Lambda | Firehose minimum buffer is 60 sec |
| Managed delivery to S3/Redshift, >60 sec OK | Kinesis Data Firehose | Fully managed, no consumer code |
| ETL on semi-structured data with schema drift | Glue DynamicFrame | Handles missing fields, schema evolution |
| Large-scale distributed processing with custom libs | EMR with Spark | Full Spark ecosystem, custom packages |
| Sub-millisecond key-value lookups | DynamoDB (+ DAX for caching) | Single-digit ms latency by design |
| Ad-hoc SQL on S3 data lake | Athena | Serverless, pay per query |
| Complex multi-step pipeline orchestration | Step Functions | Error handling, parallel branches, wait states |
| Simple sequential Glue job chaining | Glue Workflows | Native integration, simpler setup |
| Unpredictable Redshift workloads | Redshift Serverless | Auto-scaling, pay per usage |
| Time-series IoT data with retention policies | Timestream | Purpose-built, auto-tiering |
| ML training on large S3 datasets | FSx for Lustre | High-performance caching of S3 data |
| Microsecond latency financial data | Direct EC2 + Redis/UDP | Managed services add latency |

### Common Exam Traps

1. **Firehose latency**: minimum 60-second buffer -- never choose Firehose for sub-minute requirements.
2. **DynamoDB item size**: 400 KB hard limit -- if larger, store in S3 and keep a pointer.
3. **Glue bookmarks**: only work with S3, RDS, and JDBC sources -- not DynamoDB or DocumentDB.
4. **Redshift VACUUM/ANALYZE**: always the first step for performance issues before scaling the cluster.
5. **Crawler column renames**: treated as deletion + addition; not seamlessly detected.
6. **Global Tables consistency**: eventual consistency across regions, not strong.
7. **Lake Formation vs IAM**: Lake Formation is always preferred when the question mentions "fine-grained" data lake security.
8. **Parquet vs JSON at scale**: JSON parsing overhead grows non-linearly; always convert to columnar for large-scale analytics.

### Architecture Pattern Recognition

- **Lambda architecture**: batch layer (EMR/Glue) + speed layer (Kinesis/Lambda) + serving layer (Redshift/DynamoDB)
- **Kappa architecture**: single stream-processing pipeline; simpler but less suited for complex reprocessing
- **Data mesh**: separate AWS accounts per domain with shared services account
- **Exactly-once processing**: SQS FIFO with deduplication, or Lambda with idempotency keys in DynamoDB

---

## Selected Practice Questions With Explanations

### Domain 1: Data Ingestion And Transformation

**Q1.** A data engineer needs to process JSON files from S3 using AWS Glue. The JSON files have nested structures and some records have missing fields. What is the BEST approach?

- A) Static schema definition
- B) Glue Schema Registry with BACKWARD compatibility
- **C) DynamicFrame with `relationalize()` transformation** (correct)
- D) Convert to DataFrame immediately

> **Explanation**: DynamicFrame handles schema variations and missing fields automatically. `relationalize()` flattens nested structures. DataFrames require a fixed schema and would fail on missing fields.

---

**Q2.** A Glue job fails with "Container killed on request. Exit code is 143". What is the MOST likely cause?

- **A) Insufficient memory -- increase DPU allocation** (correct)
- B) Job timeout
- C) Permission issues
- D) Network connectivity

> **Explanation**: Exit code 143 (SIGTERM) typically indicates the container was killed due to memory pressure. Increase DPU count or switch to a larger worker type (G.2X).

---

**Q3.** A real-time application ingests 50,000 records/sec, each 2 KB. Which Kinesis configuration provides optimal performance?

- A) 10 shards, batch size 100
- **B) 50 shards, batch size 500** (correct)
- C) 25 shards, batch size 1000
- D) 100 shards, batch size 50

> **Explanation**: Each shard handles 1,000 records/sec. 50,000 / 1,000 = 50 shards needed. Data throughput is 100 MB/sec (50,000 x 2 KB), which would need 100 shards by throughput alone -- but the question focuses on record count. Larger batch sizes improve efficiency.

---

**Q4.** An application uses Kinesis Data Firehose to deliver data to S3. The business requires data queryable within 30 seconds. What configuration is needed?

- A) Reduce buffer interval to 30 seconds
- B) Enable dynamic partitioning
- **C) Use Kinesis Data Streams instead** (correct)
- D) Lambda-based processing

> **Explanation**: Firehose has a **minimum 60-second** buffer interval. For sub-minute requirements, use Data Streams with Lambda.

---

**Q5.** An EMR cluster processes daily batch jobs but remains idle for 16 hours. What is the MOST cost-effective strategy?

- A) Spot instances for all nodes
- **B) Transient clusters with S3 for storage** (correct)
- C) Auto-scaling
- D) Reserved instances

> **Explanation**: Transient clusters terminate after job completion. Storing data persistently in S3 means no cluster is needed during idle hours.

---

### Domain 2: Data Store Management

**Q6.** A data lake stores 5 years of daily files. Recent data (last 30 days) is accessed frequently, older data occasionally. What minimises cost?

- A) All S3 Standard
- **B) Lifecycle policy: Standard to IA (30d) to Glacier (90d)** (correct)
- C) Intelligent-Tiering everywhere
- D) Standard to IA (30d) to Deep Archive (365d)

> **Explanation**: Matches access patterns with storage classes. "Occasionally" accessed older data suits Glacier (retrievable in minutes), not Deep Archive (12-hour retrieval).

---

**Q7.** A Redshift fact table (500M rows) joins frequently with a 1M row dimension table. What distribution optimises joins?

- A) DISTKEY on fact table join column
- **B) DISTSTYLE ALL for dimension table** (correct)
- C) DISTSTYLE EVEN for both
- D) DISTSTYLE AUTO for both

> **Explanation**: Distributing the small dimension table to ALL nodes eliminates network shuffling during joins.

---

**Q8.** A gaming application queries player scores by game and time period. Millions of records. What partition key avoids hot partitions?

- A) `game_id`
- B) `player_id`
- C) `timestamp`
- **D) `game_id + date` as composite** (correct)

> **Explanation**: Composite keys distribute load across partitions while still enabling efficient game+time queries.

---

**Q9.** A financial institution needs to archive 500 TB of transaction data for 10 years. Must be retrievable within 12 hours for auditors. What minimises cost?

- A) S3 Glacier with expedited retrieval
- **B) S3 Deep Archive with standard retrieval** (correct)
- C) EBS snapshots
- D) Tape Gateway

> **Explanation**: Deep Archive is the cheapest option and standard retrieval completes within 12 hours, meeting the requirement.

---

### Domain 3: Data Operations And Support

**Q10.** A data pipeline processes files from S3 every 15 minutes. How should you monitor for missing or delayed files?

- **A) CloudWatch custom metric with Lambda** (correct)
- B) S3 event notifications to SQS
- C) CloudTrail API logging
- D) AWS Config rules

> **Explanation**: Custom metrics can track file arrival patterns and trigger CloudWatch Alarms for delays. S3 events fire on arrival but do not detect *missing* files.

---

**Q11.** A daily batch job processes 1 TB of CSV files. Processing time has grown from 2 to 6 hours over 6 months. What is the FIRST optimisation step?

- **A) Convert CSV to Parquet format** (correct)
- B) Increase Glue job DPUs
- C) Implement partitioning
- D) Enable job bookmarks

> **Explanation**: Columnar format (Parquet) provides the single biggest performance improvement for large analytical datasets. Address format before throwing more compute at the problem.

---

**Q12.** A data pipeline needs to process files in a specific order based on dependencies. What orchestration is MOST suitable?

- A) CloudWatch Events with Lambda
- **B) Step Functions with sequential states** (correct)
- C) Glue workflows with triggers
- D) EventBridge with SQS ordering

> **Explanation**: Step Functions handle complex dependencies with built-in error handling, parallel branches, and sequential state transitions.

---

### Domain 4: Data Security And Governance

**Q13.** A data lake contains PII that must be encrypted at rest and in transit. Multiple teams need different access levels. What provides comprehensive security?

- A) S3 default encryption with IAM
- B) S3 KMS encryption with bucket policies
- C) Client-side encryption
- **D) AWS Lake Formation with fine-grained permissions** (correct)

> **Explanation**: Lake Formation provides comprehensive data lake security including column-level and row-level access control, integrated with the Glue Data Catalog.

---

**Q14.** Cross-account access is needed for a shared S3 data lake. What is the MOST secure approach?

- A) S3 bucket policies with account principals
- **B) Cross-account IAM roles with external IDs** (correct)
- C) Shared IAM users with access keys
- D) S3 Access Points with VPC restrictions

> **Explanation**: Cross-account roles with external IDs provide secure, auditable, credential-free access. Shared users/keys are an anti-pattern.

---

**Q15.** GDPR requires the ability to delete individual customer records across the data lake. What architecture supports this?

- A) Partition by customer ID
- **B) Delta Lake format with ACID transactions** (correct)
- C) Soft deletes with markers
- D) Store customer data separately

> **Explanation**: Delta Lake provides true ACID transactions enabling record-level deletes in a data lake -- essential for GDPR "right to be forgotten" compliance.

---

**Q16.** In AWS Lake Formation, what does "LF-TBAC" refer to?

- A) Table-Based Access Control
- B) Time-Based Access Control
- **C) Tag-Based Access Control** (correct)
- D) Token-Based Access Control

> **Explanation**: LF-TBAC allows you to assign LF-Tags to databases, tables, and columns, then grant permissions based on those tags rather than individually naming each resource.

---

**Q17.** A multi-tenant SaaS application stores customer data in a shared data lake. Queries must automatically filter to show only tenant-specific data. What ensures data isolation?

- A) Separate S3 prefixes with IAM policies
- **B) Lake Formation with row-level security using session context** (correct)
- C) Application-level filtering
- D) Separate databases per tenant

> **Explanation**: Lake Formation row-level security automatically filters data based on the calling user's context, ensuring tenant isolation without relying on application code.

---

### Integration And Architecture

**Q18.** A streaming IoT application ingests 100,000 events/sec. Data must be available for real-time dashboards AND stored for ML. What architecture is optimal?

- A) Kinesis Streams to Lambda to DynamoDB + S3
- **B) Kinesis Streams to Kinesis Analytics to OpenSearch + S3** (correct)
- C) Firehose to S3 to Athena + QuickSight
- D) MSK to Lambda to Timestream + S3

> **Explanation**: Kinesis Analytics provides real-time stream processing, OpenSearch powers real-time dashboards, and S3 stores raw data for ML training.

---

**Q19.** A data pipeline must handle duplicate events. Processing the same event twice would cause incorrect results. What ensures exactly-once processing?

- A) DynamoDB conditional writes
- B) SQS FIFO with deduplication
- **C) Lambda with idempotency keys in DynamoDB** (correct)
- D) Kinesis sequence numbers

> **Explanation**: Storing idempotency keys (event IDs) in DynamoDB and checking before processing ensures exactly-once semantics even across Lambda retries.

---

**Q20.** A company's data volumes grew 10x unexpectedly. The current Redshift cluster cannot handle the load. What is the BEST immediate action?

- A) Upgrade node types
- B) Implement Spectrum for historical data
- **C) Migrate to Redshift Serverless** (correct)
- D) Switch to Athena

> **Explanation**: Redshift Serverless automatically scales compute based on demand and charges only for actual usage -- ideal for unpredictable growth.

---

## Related Notes

- [[AWS Data Services for Data Engineering]]
- [[Data Ingestion Patterns]]
- [[Apache Kafka Fundamentals]]
- [[Data Modelling Fundamentals]]
- [[ETL Pipeline Design]]
- [[Data Pipeline Orchestration]]
- [[Observability in Data Systems]]
- [[Data Governance Frameworks]]
- [[Encryption at Rest and in Transit]]
