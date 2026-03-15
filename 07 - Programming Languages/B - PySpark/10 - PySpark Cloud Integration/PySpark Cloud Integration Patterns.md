
#pyspark #gcp #cloud #bigquery #gcs #spark #connectors #authentication #docker

## Overview

Patterns for integrating PySpark with cloud storage, data warehouses, and managed databases. Examples drawn from a production GCP data migration framework; principles apply to AWS and Azure equivalents. See also [[PySpark Performance Optimization]], [[GCP Data Services for Data Engineering]], and [[Docker & Container Patterns]].

---

## GCS as Spark Filesystem

Spark treats GCS as a Hadoop-compatible filesystem via the `gcs-connector` JAR. Once configured, any Spark read/write API accepts `gs://` URIs transparently. The JAR is downloaded in the Docker build:

```dockerfile
ARG GCS_CONNECTOR_VERSION=hadoop3-2.2.21
RUN curl -sSL https://storage.googleapis.com/hadoop-lib/gcs/gcs-connector-${GCS_CONNECTOR_VERSION}.jar \
    -o ${SPARK_HOME}/jars/gcs-connector-${GCS_CONNECTOR_VERSION}.jar
```

Register the `gs://` scheme and configure auth in `spark-defaults.conf`:

```properties
spark.hadoop.fs.gs.impl                    com.google.cloud.hadoop.fs.gcs.GoogleHadoopFileSystem
spark.hadoop.fs.AbstractFileSystem.gs.impl com.google.cloud.hadoop.fs.gcs.GoogleHadoopFS
spark.hadoop.fs.gs.auth.service.account.enable true
spark.hadoop.fs.gs.project.id              your-gcp-project-id
spark.hadoop.google.cloud.auth.service.account.json.keyfile /app/service-account-key.json
```

Reading and writing works identically to local paths:

```python
df = spark.read.parquet("gs://my-bucket/data/events/")
df = spark.read.csv("gs://my-bucket/data/customers.csv", header=True, inferSchema=True)
df.write.mode("overwrite").parquet("gs://my-bucket/output/events/")
```

For smaller datasets without Spark, the framework uses the `google-cloud-storage` Python client directly (`GCSExtractor` in `extract.py`), downloading blobs as text/bytes and parsing with pandas.

---

## BigQuery Spark Connector

The `spark-bigquery-with-dependencies` uber-JAR is downloaded alongside the GCS connector:

```dockerfile
RUN curl -sSL https://repo1.maven.org/maven2/com/google/cloud/spark/spark-bigquery-with-dependencies_2.12/0.36.1/spark-bigquery-with-dependencies_2.12-0.36.1.jar \
    -o ${SPARK_HOME}/jars/spark-bigquery-with-dependencies_2.12-0.36.1.jar
```

Reading tables, views, and writing back:

```python
# Direct table read
df = spark.read.format("bigquery").option("table", "project.dataset.table_name").load()

# Query-based read -- viewMaterializationDataset stores temp results for views
df = spark.read.format("bigquery") \
    .option("query", "SELECT * FROM `project.dataset.view` WHERE date > '2025-01-01'") \
    .option("viewMaterializationDataset", "temp_dataset").load()

# Write -- requires a temporary GCS bucket for staging
df.write.format("bigquery") \
    .option("table", "project.dataset.target_table") \
    .option("temporaryGcsBucket", "my-staging-bucket") \
    .option("partitionField", "event_date") \
    .option("clusteredFields", "user_id,event_type") \
    .mode("overwrite").save()
```

The framework's `BigQueryLoader` uses the Python client (`google-cloud-bigquery`) for non-distributed loads, supporting `write_disposition`, time partitioning, and clustering via `LoadJobConfig`.

---

## Cloud SQL from Spark

The framework uses `google-cloud-sql-connector` with SQLAlchemy (`creator=getconn` gives SQLAlchemy a connection pool backed by the connector, handling SSL/IAM automatically):

```python
from google.cloud.sql.connector import Connector
connector = Connector()
def getconn():
    return connector.connect(
        "project:region:instance", "pymysql",  # or "pg8000" for PostgreSQL
        user="etl_user", password=secret, db="mydb"
    )
engine = sqlalchemy.create_engine("mysql+pymysql://", creator=getconn)
df = pd.read_sql("SELECT * FROM orders", engine)
```

For distributed reads across Spark executors, use JDBC with partition hints:

```python
df = spark.read.format("jdbc") \
    .option("url", "jdbc:postgresql://10.0.0.5:5432/mydb") \
    .option("dbtable", "public.orders") \
    .option("partitionColumn", "order_id") \
    .option("lowerBound", 1).option("upperBound", 1000000) \
    .option("numPartitions", 8).load()
```

---

## Connector JARs and Dependency Management

The project pins versions via Docker build args and places JARs directly in `$SPARK_HOME/jars/`:

```dockerfile
ARG SPARK_VERSION=3.5.1
ARG HADOOP_VERSION=3
ARG GCS_CONNECTOR_VERSION=hadoop3-2.2.21
```

Key Python dependencies from `requirements/prod.txt`:

```
pyspark==3.5.1
pyarrow==14.0.2
google-cloud-storage==2.10.0
google-cloud-bigquery==3.13.0
google-auth==2.23.4
sqlalchemy==2.0.23
```

> Always pin `pyarrow` to a version compatible with your `pyspark` version -- Arrow is required when `spark.sql.execution.arrow.pyspark.enabled` is true.

If JARs are not in `$SPARK_HOME/jars/`, specify them explicitly:

```properties
spark.jars                    /opt/connectors/gcs-connector.jar,/opt/connectors/spark-bigquery.jar
spark.driver.extraClassPath   /opt/connectors/*
spark.executor.extraClassPath /opt/connectors/*
```

---

## Authentication Patterns

**Service account key file** -- set an env var and mirror it in Spark config. Both Python client libraries and the Hadoop GCS connector honour `GOOGLE_APPLICATION_CREDENTIALS`:

```dockerfile
ENV GOOGLE_APPLICATION_CREDENTIALS=/app/service-account-key.json
```

```properties
spark.hadoop.google.cloud.auth.service.account.json.keyfile /app/service-account-key.json
```

**Workload identity (GKE)** -- no key file needed. Annotate the K8s service account with `iam.gke.io/gcp-service-account`, then set Spark to use ADC:

```properties
spark.hadoop.fs.gs.auth.service.account.enable  false
spark.hadoop.fs.gs.auth.type                    APPLICATION_DEFAULT
```

**Application default credentials** -- for local development, run `gcloud auth application-default login`. Client libraries pick up credentials automatically.

---

## Spark Configuration for Cloud

From the project's `spark-defaults.conf`. See [[PySpark Performance Optimization]] for deeper coverage.

### Adaptive Query Execution

```properties
spark.sql.adaptive.enabled                        true
spark.sql.adaptive.coalescePartitions.enabled      true
spark.sql.adaptive.skewJoin.enabled                true
spark.sql.adaptive.advisoryPartitionSizeInBytes    128MB
```

AQE is essential for cloud workloads where data distribution is unpredictable.

### Arrow, Serialisation, and Parquet

```properties
spark.sql.execution.arrow.pyspark.enabled    true
spark.sql.execution.arrow.maxRecordsPerBatch 10000
spark.serializer                             org.apache.spark.serializer.KryoSerializer
spark.sql.sources.partitionOverwriteMode     dynamic
spark.sql.parquet.compression.codec          snappy
spark.sql.parquet.filterPushdown             true
```

Arrow accelerates `toPandas()`/`createDataFrame()` conversions. Kryo reduces shuffle payload sizes. Dynamic partition overwrite only replaces partitions present in the output DataFrame.

### Network and Broadcast Timeouts

Cloud storage has higher latency than local disks -- increase defaults:

```properties
spark.network.timeout       800s
spark.sql.broadcastTimeout  36000
spark.rpc.askTimeout        600s
```

---

## Checkpoint and State Management

For production streaming, checkpoints and event logs should live on durable cloud storage. The project defaults to local paths for development:

```properties
# Checkpoints -- local for dev, GCS for production
spark.sql.streaming.checkpointLocation  gs://my-bucket/spark/checkpoints

# Event logs for the History Server
spark.eventLog.enabled        true
spark.eventLog.dir            gs://my-bucket/spark-events
spark.history.fs.logDirectory gs://my-bucket/spark-events
```

Structured Streaming state stores work transparently with `gs://` paths via the GCS connector:

```python
query = df.writeStream.format("parquet") \
    .option("checkpointLocation", "gs://my-bucket/checkpoints/orders") \
    .option("path", "gs://my-bucket/output/orders") \
    .trigger(processingTime="5 minutes").start()
```

See [[PySpark Streaming]] for detailed streaming patterns.

---

## AWS and Azure Equivalents

| Concern | GCP | AWS | Azure |
|---|---|---|---|
| **Object storage** | GCS (`gs://`) | S3 (`s3a://`) | ADLS Gen2 (`abfss://`) |
| **Hadoop connector** | `gcs-connector-hadoop3-X.jar` | `hadoop-aws-X.jar` + `aws-java-sdk-bundle` | `hadoop-azure-X.jar` + `azure-storage` |
| **FS impl class** | `GoogleHadoopFileSystem` | `S3AFileSystem` | `AzureBlobFileSystem` |
| **Auth** | SA key / workload identity | Access key / IAM role | Account key / managed identity |
| **DW connector** | `spark-bigquery` JAR | `spark-redshift` / JDBC | `spark-synapse` / JDBC |
| **Staging bucket** | `temporaryGcsBucket` | `tempdir` (S3) | `tempDir` (ADLS) |

### AWS S3

```properties
spark.hadoop.fs.s3a.impl                          org.apache.hadoop.fs.s3a.S3AFileSystem
spark.hadoop.fs.s3a.aws.credentials.provider      com.amazonaws.auth.InstanceProfileCredentialsProvider
```

### Azure ADLS Gen2

```properties
spark.hadoop.fs.azure.account.auth.type.myaccount.dfs.core.windows.net             OAuth
spark.hadoop.fs.azure.account.oauth.provider.type.myaccount.dfs.core.windows.net   org.apache.hadoop.fs.azurebfs.oauth2.MsiTokenProvider
```

---

## Related Notes

- [[PySpark Performance Optimization]] -- AQE, shuffle tuning, broadcast joins
- [[PySpark Streaming]] -- structured streaming, watermarks, state management
- [[PySpark Production Engineering]] -- error handling, logging, deployment
- [[GCP Data Services for Data Engineering]] -- BigQuery, GCS, Cloud SQL overview
- [[Docker & Container Patterns]] -- multi-stage builds, dependency management
