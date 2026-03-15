

#pyspark #interview #data-engineering #spark #big-data #apache-spark

## Table of Contents

- [[#PySpark Fundamentals]]
- [[#Core Concepts]]
- [[#Data Operations]]
- [[#Performance & Optimization]]
- [[#Advanced Topics]]
- [[#Stream Processing & Real-time]]
- [[#Production & Engineering]]
- [[#Data Governance & Compliance]]
- [[#Security & Access Control]]
- [[#Architecture & System Design]]

---

## PySpark Fundamentals

### What is PySpark and its Key Advantages?

**PySpark** is the Python API for Apache Spark, an open-source distributed computing system designed for large-scale data processing and analytics.

#### Key Advantages over Traditional Python:

- **Scalability**: Handle datasets from gigabytes to petabytes by distributing computation across clusters
- **High Performance**: 10-100x faster processing through in-memory computing and parallel execution
- **Fault Tolerance**: Automatic recovery from node failures using RDD lineage information
- **Integration**: Seamless integration with Hadoop ecosystem (HDFS, Hive, HBase) and cloud storage (S3, ADLS, GCS)
- **Unified Platform**: Single framework for batch processing, streaming, ML, and graph processing
- **Lazy Evaluation**: Optimizes entire execution pipeline before running any computation

#### Example Comparison:

```python
# Traditional Python (single-threaded)
import pandas as pd
df = pd.read_csv("large_file.csv")  # Loads entire file into memory
result = df.groupby("category").sum()  # Limited by single machine memory

# PySpark (distributed)
df = spark.read.csv("large_file.csv", header=True)  # Distributed across cluster
result = df.groupBy("category").sum()  # Processed in parallel across nodes
```

### SparkSession Creation and Responsibilities

**SparkSession** is the entry point for all Spark functionality in PySpark 2.0+. It encapsulates SparkContext, SQLContext, and HiveContext into a single unified API.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MyDataApplication") \
    .master("local[*]") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()
```

#### Main Responsibilities:

- **DataFrame Operations**: Creating and manipulating DataFrames
- **SQL Interface**: Executing SQL queries on distributed data
- **Configuration Management**: Setting Spark properties and cluster configurations
- **Resource Management**: Managing SparkContext lifecycle and cluster resources
- **Catalog Access**: Interacting with metadata catalogs (Hive, Delta Lake)
- **UDF Registration**: Registering user-defined functions for SQL use

---

## Core Concepts

### Spark DAG (Directed Acyclic Graph)

A **DAG** in Spark represents the logical execution plan of transformations applied to RDDs or DataFrames.

#### Key Characteristics:

- **Directed**: Each node represents a transformation with dependencies flowing in one direction
- **Acyclic**: No loops or cycles exist, ensuring finite execution paths
- **Lazy**: Built during transformation definitions, executed only when actions are called

#### DAG Optimization:

- **Pipeline Optimization**: Combines narrow transformations into single stages
- **Predicate Pushdown**: Moves filter operations closer to data sources
- **Column Pruning**: Eliminates unnecessary columns early in the pipeline
- **Constant Folding**: Pre-computes constant expressions
- **Join Reordering**: Optimizes join order based on statistics

```python
# Example: DAG optimization in action
df1 = spark.read.parquet("sales.parquet")
df2 = spark.read.parquet("customers.parquet")

# These transformations build the DAG but don't execute
filtered_sales = df1.filter(df1.amount > 100)  # Will be pushed down
selected_customers = df2.select("customer_id", "name")  # Column pruning
joined = filtered_sales.join(selected_customers, "customer_id")  # Join optimization

# Action triggers DAG optimization and execution
result = joined.count()  # Catalyst optimizer creates optimal physical plan
```

### RDDs vs DataFrames vs Datasets

|Feature|RDD|DataFrame|Dataset|
|---|---|---|---|
|API Level|Low-level|High-level|High-level|
|Schema|No schema|Schema-aware|Schema + compile-time type safety|
|Optimization|Manual|Catalyst optimizer|Catalyst optimizer|
|Type Safety|Runtime|Runtime|Compile-time (Scala/Java only)|
|Performance|Slower|Fast|Fastest|
|Memory Usage|Higher|Lower (columnar)|Lower (columnar)|
|Language Support|All|All|Scala/Java only|

#### When to Use Each:

**RDDs**: Unstructured data or complex transformations

```python
text_rdd = spark.sparkContext.textFile("logs.txt")
parsed_rdd = text_rdd.map(lambda line: parse_complex_format(line))
filtered_rdd = parsed_rdd.filter(lambda record: custom_business_logic(record))
```

**DataFrames**: Structured/semi-structured data with SQL-like operations (recommended for most cases)

```python
df = spark.read.json("user_events.json")
result = df.select("user_id", "timestamp", "event_type") \
           .filter(df.timestamp > "2024-01-01") \
           .groupBy("event_type").count()
```

### Transformations vs Actions

#### Transformations (Lazy Operations):

- Create new RDDs/DataFrames from existing ones
- Build computation graph (DAG) without executing
- Enable optimization across entire pipeline

#### Actions (Eager Operations):

- Trigger execution of entire transformation pipeline
- Return results to driver or write to external storage
- Force DAG optimization and execution

```python
# All these are transformations - no computation happens yet
df_customers = spark.read.parquet("customers.parquet")
df_filtered = df_customers.filter(df_customers.age > 25)
df_selected = df_filtered.select("customer_id", "name", "age")
df_grouped = df_selected.groupBy("age").count()

# These actions trigger execution of all previous transformations
df_grouped.show()           # Display results
count = df_grouped.count()  # Return count to driver
df_grouped.write.parquet("age_distribution.parquet")  # Write to storage
```

#### Lazy Evaluation Benefits:

- **Optimization**: Catalyst optimizer sees entire pipeline and applies optimizations
- **Efficiency**: Eliminates intermediate results that aren't needed
- **Resource Management**: Delays resource allocation until necessary
- **Pipeline Fusion**: Combines operations to minimize data movement

---

## Data Operations

### Reading Data into PySpark

#### File-based Sources:

```python
# CSV files - good for data exchange, human-readable
df_csv = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .option("multiline", "true") \
    .csv("data.csv")

# Parquet files - optimized for analytics, columnar storage
df_parquet = spark.read.parquet("data.parquet")

# JSON files - semi-structured data
df_json = spark.read.json("data.json")

# Delta Lake - ACID transactions, time travel
df_delta = spark.read.format("delta").load("delta-table")
```

#### Database Sources:

```python
# JDBC connections
df_postgres = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/db") \
    .option("dbtable", "sales") \
    .option("user", "username") \
    .option("password", "password") \
    .option("numPartitions", "10") \
    .load()
```

#### Performance Considerations:

- **Parquet**: Best for analytical workloads, 10x faster than CSV
- **Delta Lake**: Best for data lakes requiring ACID properties
- **Partitioning**: Use partitioned datasets for better query performance
- **Schema Inference**: Disable for large CSV files to improve performance

### Handling Missing Data

#### 1. Dropping Missing Values:

```python
# Drop rows with any null values
df_clean = df.dropna(how="any")

# Drop rows only if all values are null
df_clean = df.dropna(how="all")

# Drop rows with nulls in specific columns
df_clean = df.dropna(subset=["customer_id", "amount"])

# Drop rows with nulls in at least 2 columns
df_clean = df.dropna(thresh=2)
```

#### 2. Filling Missing Values:

```python
# Fill with constant values
df_filled = df.fillna({"age": 0, "name": "Unknown", "salary": -1})

# Forward fill (use previous non-null value)
from pyspark.sql.window import Window
window = Window.partitionBy("customer_id").orderBy("timestamp")
df_ffill = df.withColumn("amount_filled", 
    last(col("amount"), ignorenulls=True).over(window))
```

#### 3. Statistical Imputation:

```python
from pyspark.ml.feature import Imputer

# Impute with mean, median, or mode
imputer = Imputer(
    strategy="median",  # or "mean", "mode"
    inputCols=["age", "income"],
    outputCols=["age_imputed", "income_imputed"]
)

model = imputer.fit(df)
df_imputed = model.transform(df)
```

### Join Types and Performance

#### Join Types:

```python
# Inner join - only matching records
inner_result = df1.join(df2, df1.id == df2.id, "inner")

# Left outer join - all records from left, matching from right
left_result = df1.join(df2, df1.id == df2.id, "left")

# Right outer join - all records from right, matching from left
right_result = df1.join(df2, df1.id == df2.id, "right")

# Full outer join - all records from both sides
full_result = df1.join(df2, df1.id == df2.id, "outer")

# Left semi join - left records that have matches in right (no right columns)
semi_result = df1.join(df2, df1.id == df2.id, "left_semi")

# Left anti join - left records that don't have matches in right
anti_result = df1.join(df2, df1.id == df2.id, "left_anti")
```

#### Performance Optimization:

**Broadcast Joins** (for small tables < 10MB):

```python
from pyspark.sql.functions import broadcast

# Automatically broadcast small table
result = large_df.join(broadcast(small_df), "key")

# Configure broadcast threshold
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")
```

**Bucketed Joins** (for repeated joins):

```python
# Pre-bucket tables for efficient joins
df1.write \
   .bucketBy(10, "join_key") \
   .saveAsTable("bucketed_table1")

# Join bucketed tables (no shuffle required)
result = spark.table("bucketed_table1").join(
    spark.table("bucketed_table2"), "join_key"
)
```

---

## Performance & Optimization

### Caching and Storage Levels

Caching stores computed DataFrames/RDDs in memory or disk to avoid recomputation.

```python
# Simple memory caching (default: MEMORY_AND_DISK)
df.cache()

# Explicit persistence with storage levels
from pyspark import StorageLevel

df.persist(StorageLevel.MEMORY_ONLY)
df.persist(StorageLevel.MEMORY_AND_DISK)
df.persist(StorageLevel.DISK_ONLY)
df.persist(StorageLevel.MEMORY_ONLY_SER)  # Serialized
```

#### Storage Level Comparison:

|Storage Level|Memory|Disk|Serialized|Recompute on Loss|
|---|---|---|---|---|
|MEMORY_ONLY|✓|✗|✗|✓|
|MEMORY_AND_DISK|✓|✓|✗|✗|
|MEMORY_ONLY_SER|✓|✗|✓|✓|
|DISK_ONLY|✗|✓|✓|✗|

#### Cache Management:

```python
# Check what's cached
print(spark.catalog.cacheTable("table_name"))

# Unpersist when no longer needed
df.unpersist()
```

### Data Partitioning

Data partitioning divides data into smaller chunks that can be processed in parallel.

#### Types of Partitioning:

**Hash Partitioning**:

```python
# Repartition by column (hash-based)
df_partitioned = df.repartition("customer_id")

# Specify number of partitions
df_partitioned = df.repartition(200, "customer_id")

# Multiple columns
df_partitioned = df.repartition("year", "month")
```

**Range Partitioning**:

```python
# Sort and partition by ranges
df_range_partitioned = df.repartitionByRange("timestamp")
```

**Coalesce**:

```python
# Reduce partitions without shuffle (when possible)
df_coalesced = df.coalesce(50)
```

#### Optimal Partition Size:

```python
# Rule of thumb: 128MB - 1GB per partition
target_partition_size_mb = 128
file_size_mb = 10000  # 10GB file
optimal_partitions = file_size_mb // target_partition_size_mb

df = spark.read.option("maxPartitionBytes", "128MB").parquet("large_file.parquet")
```

### Broadcast Variables and Accumulators

#### Broadcast Variables (Read-only shared data):

Use cases: Large lookup tables, configuration data, ML models

```python
# Create broadcast variable
large_lookup = {"key1": "value1", "key2": "value2", ...}
broadcast_lookup = spark.sparkContext.broadcast(large_lookup)

# Use in transformations
def enrich_data(row):
    lookup_value = broadcast_lookup.value.get(row.key, "default")
    return (row.id, row.data, lookup_value)

enriched_rdd = rdd.map(enrich_data)

# Cleanup when done
broadcast_lookup.destroy()
```

#### Accumulators (Write-only shared counters):

Use cases: Counting events/errors, debugging, data quality metrics

```python
# Create accumulators
error_count = spark.sparkContext.accumulator(0)
valid_records = spark.sparkContext.accumulator(0)

def process_record(record):
    try:
        if is_valid(record):
            valid_records.add(1)
            return process_valid_record(record)
        else:
            error_count.add(1)
            return None
    except Exception as e:
        error_count.add(1)
        return None

# Process data and trigger action to update accumulators
processed_rdd = rdd.map(process_record).filter(lambda x: x is not None)
result = processed_rdd.collect()

# Access accumulator values
print(f"Valid records: {valid_records.value}")
print(f"Errors: {error_count.value}")
```

### Data Skew Handling

Data skew occurs when data is unevenly distributed across partitions.

#### Detection Techniques:

```python
# Check partition sizes
partition_sizes = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
print(f"Partition sizes: {partition_sizes}")
print(f"Max/Min ratio: {max(partition_sizes) / min(partition_sizes)}")

# Check key distribution
key_distribution = df.groupBy("skewed_key").count().orderBy(col("count").desc())
key_distribution.show(20)
```

#### Salting Technique:

```python
from pyspark.sql.functions import rand, concat, lit, regexp_replace

def apply_salting(df, skewed_column, salt_range=100):
    """Add random salt to skewed keys for better distribution"""
    
    # Add salt to the skewed column
    salted_df = df.withColumn(
        "salted_key",
        concat(col(skewed_column), lit("_"), (rand() * salt_range).cast("int"))
    )
    
    # Perform operations on salted data
    intermediate_result = salted_df.groupBy("salted_key").agg(
        sum("amount").alias("total_amount"),
        count("*").alias("record_count")
    )
    
    # Remove salt and aggregate final results
    final_result = intermediate_result.withColumn(
        "original_key",
        regexp_replace(col("salted_key"), "_\\d+$", "")
    ).groupBy("original_key").agg(
        sum("total_amount").alias("final_amount"),
        sum("record_count").alias("final_count")
    )
    
    return final_result

# Apply salting to skewed dataset
result = apply_salting(skewed_df, "customer_id", salt_range=50)
```

---

## Advanced Topics

### Window Functions

Window functions perform calculations across a set of rows related to the current row.

#### Window Function Components:

- **Partition By**: Divides data into groups
- **Order By**: Defines row ordering within partitions
- **Frame Specification**: Defines which rows to include in calculation

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import *

# Basic window specification
window_spec = Window.partitionBy("department").orderBy("salary")
```

#### Ranking Functions:

```python
# Ranking functions
df_ranked = df.withColumn("row_number", row_number().over(window_spec)) \
             .withColumn("rank", rank().over(window_spec)) \
             .withColumn("dense_rank", dense_rank().over(window_spec)) \
             .withColumn("percent_rank", percent_rank().over(window_spec))

# Find top N employees per department
top_earners = df.withColumn("rank", rank().over(window_spec)) \
                .filter(col("rank") <= 3)
```

#### Analytical Functions:

```python
# Lead and lag functions
window_time = Window.partitionBy("customer_id").orderBy("timestamp")

df_analytics = df.withColumn("previous_amount", lag("amount", 1).over(window_time)) \
                 .withColumn("next_amount", lead("amount", 1).over(window_time)) \
                 .withColumn("first_amount", first("amount").over(window_time)) \
                 .withColumn("last_amount", last("amount").over(window_time))

# Calculate differences and trends
df_trends = df_analytics.withColumn("amount_change", 
    col("amount") - col("previous_amount"))
```

#### Aggregate Functions with Windows:

```python
# Running totals and moving averages
window_unbounded = Window.partitionBy("account_id") \
                         .orderBy("timestamp") \
                         .rowsBetween(Window.unboundedPreceding, Window.currentRow)

window_moving = Window.partitionBy("account_id") \
                      .orderBy("timestamp") \
                      .rowsBetween(-6, Window.currentRow)  # 7-day window

df_aggregates = df.withColumn("running_total", sum("amount").over(window_unbounded)) \
                  .withColumn("moving_avg", avg("amount").over(window_moving)) \
                  .withColumn("max_in_window", max("amount").over(window_moving))
```

### Custom Transformations and UDFs

#### Custom Transformations using DataFrame.transform():

```python
def standardize_column_names(df):
    """Convert all column names to lowercase and replace spaces with underscores"""
    new_columns = [col(c).alias(c.lower().replace(" ", "_")) for c in df.columns]
    return df.select(*new_columns)

def add_derived_features(df):
    """Add commonly used derived features"""
    return df.withColumn("full_name", concat(col("first_name"), lit(" "), col("last_name"))) \
             .withColumn("age_group", 
                 when(col("age") < 18, "Minor")
                 .when(col("age") < 65, "Adult")
                 .otherwise("Senior"))

# Chain transformations
result_df = source_df.transform(standardize_column_names) \
                     .transform(add_derived_features)
```

#### User Defined Functions (UDFs):

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import *

# Simple UDF
@udf(returnType=StringType())
def extract_domain(email):
    if email and "@" in email:
        return email.split("@")[1]
    return None

# Complex UDF with multiple outputs
@udf(returnType=StructType([
    StructField("is_valid", BooleanType(), True),
    StructField("formatted", StringType(), True),
    StructField("country_code", StringType(), True)
]))
def parse_phone_number(phone):
    if not phone:
        return {"is_valid": False, "formatted": None, "country_code": None}
    
    # Processing logic here...
    return {"is_valid": True, "formatted": formatted_phone, "country_code": "US"}

# Use UDFs
df_enriched = df.withColumn("email_domain", extract_domain(col("email"))) \
                .withColumn("phone_info", parse_phone_number(col("phone")))
```

#### Pandas UDFs (Vectorized UDFs):

```python
from pyspark.sql.functions import pandas_udf, PandasUDFType
import pandas as pd

@pandas_udf(returnType=DoubleType(), functionType=PandasUDFType.SCALAR)
def calculate_zscore(values: pd.Series) -> pd.Series:
    """Calculate z-scores efficiently using pandas operations"""
    return (values - values.mean()) / values.std()

# Use pandas UDFs
df_analyzed = df.withColumn("zscore", calculate_zscore(col("value")))
```

### Narrow vs Wide Transformations

#### Narrow Transformations:

- Each input partition contributes to at most one output partition
- No data shuffling across network
- Can be pipelined together for efficiency

```python
# All these operations don't require shuffling
df_narrow = df.filter(col("amount") > 100) \           # Filter
              .select("customer_id", "amount", "date") \  # Select/Project
              .withColumn("tax", col("amount") * 0.1) \   # Map/Transform
              .union(other_df) \                          # Union
              .drop("temp_column")                        # Drop
```

#### Wide Transformations:

- Input partitions contribute to multiple output partitions
- Require shuffling data across network
- Create stage boundaries in execution plan

```python
# These operations require shuffling
df_wide = df.groupBy("customer_id").agg(sum("amount")) \  # GroupBy/Aggregation
            .join(other_df, "customer_id") \               # Join
            .orderBy("customer_id") \                      # Sort
            .distinct()                                    # Distinct
```

---

## Stream Processing & Real-time

### Structured Streaming Fundamentals

Spark Structured Streaming treats streaming data as an unbounded table that is continuously appended.

#### Basic Streaming Setup:

```python
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Define schema for streaming data
schema = StructType([
    StructField("timestamp", TimestampType(), True),
    StructField("user_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("value", DoubleType(), True)
])

# Read from Kafka stream
streaming_df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "user_events") \
    .option("startingOffsets", "latest") \
    .load()

# Parse JSON messages
parsed_df = streaming_df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")
```

### Watermarks and Late Data Handling

Watermarks define how long to wait for late-arriving data before considering a time window complete.

```python
# Add watermark for handling late data
watermarked_df = parsed_df \
    .withWatermark("timestamp", "10 minutes")  # Wait 10 minutes for late data

# Windowed aggregation with watermark
windowed_counts = watermarked_df \
    .groupBy(
        window(col("timestamp"), "5 minutes", "1 minute"),  # 5-min windows, 1-min slide
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        avg("value").alias("avg_value")
    )
```

### Trigger Types and Output Modes

#### Trigger Types:

```python
# Continuous Processing (experimental, ultra-low latency)
continuous_query = streaming_df.writeStream \
    .trigger(continuous="1 second") \
    .format("console") \
    .start()

# Micro-batch Processing (default)
microbatch_query = streaming_df.writeStream \
    .trigger(processingTime="30 seconds") \
    .outputMode("append") \
    .format("delta") \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .start("/path/to/delta-table")
```

#### Output Modes:

- **Append Mode**: Only new rows (default for most operations)
- **Update Mode**: Only updated rows (for aggregations)
- **Complete Mode**: Entire result table (memory intensive)

```python
# Append Mode
append_query = windowed_counts.writeStream \
    .outputMode("append") \
    .format("console") \
    .start()

# Update Mode
update_query = windowed_counts.writeStream \
    .outputMode("update") \
    .format("delta") \
    .start("/path/to/delta-updates")
```

---

## Production & Engineering

### Error Handling and Debugging

#### Structured Logging:

```python
import logging
from datetime import datetime

class SparkLogger:
    def __init__(self, app_name, log_level="INFO"):
        self.logger = logging.getLogger(app_name)
        self.logger.setLevel(getattr(logging, log_level))
        
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        self.logger.addHandler(console_handler)
    
    def log_dataframe_info(self, df, operation_name):
        """Log DataFrame operation details"""
        try:
            count = df.count()
            schema_info = str(df.schema)
            self.logger.info(f"Operation: {operation_name}, Records: {count}")
        except Exception as e:
            self.logger.error(f"Failed to log DataFrame info: {str(e)}")

# Decorator for automatic error handling
def with_error_handling(operation_name):
    def decorator(func):
        def wrapper(*args, **kwargs):
            try:
                spark_logger.logger.info(f"Starting operation: {operation_name}")
                result = func(*args, **kwargs)
                spark_logger.logger.info(f"Completed operation: {operation_name}")
                return result
            except Exception as e:
                spark_logger.logger.error(f"Operation failed: {operation_name}, Error: {str(e)}")
                raise
        return wrapper
    return decorator
```

#### Circuit Breaker Pattern:

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing fast
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN - failing fast")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
```

### Testing Strategies

#### Unit Testing Framework:

```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import *
import pandas as pd

class SparkTestBase:
    """Base class for Spark tests with common utilities"""
    
    @pytest.fixture(scope="class")
    def spark(self):
        """Create Spark session for testing"""
        return SparkSession.builder \
            .appName("test_session") \
            .master("local[2]") \
            .config("spark.sql.shuffle.partitions", "2") \
            .getOrCreate()
    
    @pytest.fixture
    def sample_data(self, spark):
        """Create sample data for testing"""
        schema = StructType([
            StructField("id", StringType(), False),
            StructField("name", StringType(), True),
            StructField("value", DoubleType(), True)
        ])
        
        data = [
            ("1", "John", 100.0),
            ("2", "Jane", 200.0),
            ("3", "Bob", 150.0)
        ]
        
        return spark.createDataFrame(data, schema)
    
    def assert_dataframes_equal(self, actual_df, expected_df):
        """Compare two DataFrames for equality"""
        assert actual_df.schema == expected_df.schema
        actual_count = actual_df.count()
        expected_count = expected_df.count()
        assert actual_count == expected_count
        
        actual_pd = actual_df.toPandas().sort_values(by=actual_df.columns[0])
        expected_pd = expected_df.toPandas().sort_values(by=expected_df.columns[0])
        pd.testing.assert_frame_equal(actual_pd, expected_pd)
```

---

## Data Governance & Compliance

### Data Lineage Tracking

Data lineage tracking is crucial for understanding data dependencies and managing changes safely.

```python
import json
import uuid
from datetime import datetime
from dataclasses import dataclass
import networkx as nx

@dataclass
class DataAsset:
    """Represents a data asset in the lineage graph"""
    asset_id: str
    asset_type: str  # table, view, file, api
    location: str
    owner: str
    created_at: datetime

@dataclass
class LineageRecord:
    """Represents a lineage relationship"""
    record_id: str
    source_assets: list
    target_asset: str
    transformation_type: str
    transformation_code: str
    execution_id: str
    user: str
    timestamp: datetime

class DataLineageTracker:
    def __init__(self, spark_session, lineage_store_path="/data/lineage"):
        self.spark = spark_session
        self.lineage_store_path = lineage_store_path
        self.lineage_graph = nx.DiGraph()
        self.session_assets = {}
    
    def register_dataset(self, dataset_id: str, dataset_type: str, 
                        location: str, owner: str) -> DataAsset:
        """Register a new dataset in the lineage system"""
        asset = DataAsset(
            asset_id=dataset_id,
            asset_type=dataset_type,
            location=location,
            owner=owner,
            created_at=datetime.now()
        )
        
        self.session_assets[dataset_id] = asset
        self.lineage_graph.add_node(dataset_id, **asset.__dict__)
        return asset
    
    def track_transformation(self, source_datasets: list, target_dataset: str,
                           transformation_type: str, transformation_code: str = None):
        """Track a data transformation"""
        lineage_record = LineageRecord(
            record_id=str(uuid.uuid4()),
            source_assets=source_datasets,
            target_asset=target_dataset,
            transformation_type=transformation_type,
            transformation_code=transformation_code or "",
            execution_id=self.spark.sparkContext.applicationId,
            user=self.spark.sparkContext.sparkUser(),
            timestamp=datetime.now()
        )
        
        # Add edges to graph
        for source in source_datasets:
            self.lineage_graph.add_edge(source, target_dataset, record=lineage_record)
        
        return lineage_record
    
    def impact_analysis(self, dataset_id: str) -> dict:
        """Perform impact analysis for a dataset change"""
        downstream = list(nx.descendants(self.lineage_graph, dataset_id))
        
        return {
            "source_dataset": dataset_id,
            "affected_datasets": len(downstream),
            "downstream_dependencies": downstream,
            "risk_assessment": self._assess_change_risk(dataset_id, downstream)
        }
```

### GDPR Compliance Implementation

#### Right to be Forgotten:

```python
from pyspark.sql.functions import *

class GDPRCompliance:
    def __init__(self, spark_session):
        self.spark = spark_session
        self.deletion_log = []
    
    def right_to_be_forgotten(self, user_id: str, tables: list) -> dict:
        """Implement GDPR right to be forgotten"""
        deletion_results = {}
        
        for table_name in tables:
            try:
                # Read table
                df = self.spark.read.table(table_name)
                
                # Identify user data
                user_columns = self._identify_user_columns(df, table_name)
                
                # Apply deletion/pseudonymization
                if user_columns:
                    cleaned_df = self._remove_user_data(df, user_id, user_columns)
                    
                    # Write back cleaned data
                    cleaned_df.write.mode("overwrite").saveAsTable(table_name)
                    
                    deletion_results[table_name] = {
                        "status": "success",
                        "records_affected": df.filter(col(user_columns[0]) == user_id).count(),
                        "columns_affected": user_columns
                    }
                else:
                    deletion_results[table_name] = {
                        "status": "no_user_data",
                        "records_affected": 0
                    }
                    
            except Exception as e:
                deletion_results[table_name] = {
                    "status": "error",
                    "error": str(e)
                }
        
        # Log deletion request
        self.deletion_log.append({
            "user_id": user_id,
            "tables": tables,
            "results": deletion_results,
            "timestamp": datetime.now()
        })
        
        return deletion_results
    
    def _identify_user_columns(self, df, table_name: str) -> list:
        """Identify columns containing user data"""
        user_columns = []
        
        # Check for common user identifier columns
        common_user_cols = ["user_id", "customer_id", "email", "phone"]
        
        for col_name in df.columns:
            if any(identifier in col_name.lower() for identifier in common_user_cols):
                user_columns.append(col_name)
        
        return user_columns
    
    def _remove_user_data(self, df, user_id: str, user_columns: list):
        """Remove or pseudonymize user data"""
        # Option 1: Delete rows
        cleaned_df = df.filter(~col(user_columns[0]).isin([user_id]))
        
        # Option 2: Pseudonymize (alternative approach)
        # for col_name in user_columns:
        #     cleaned_df = cleaned_df.withColumn(col_name,
        #         when(col(col_name) == user_id, "PSEUDONYMIZED")
        #         .otherwise(col(col_name)))
        
        return cleaned_df
```

---

## Security & Access Control

### Column-Level Security Framework

```python
from typing import Dict, List, Set
from enum import Enum
from dataclasses import dataclass

class SecurityLevel(Enum):
    PUBLIC = "public"
    INTERNAL = "internal" 
    CONFIDENTIAL = "confidential"
    RESTRICTED = "restricted"

class MaskingType(Enum):
    FULL_MASK = "full_mask"
    PARTIAL_MASK = "partial_mask"
    HASH_MASK = "hash_mask"
    FORMAT_PRESERVING = "format_preserving"
    NULL_MASK = "null_mask"
    NO_MASK = "no_mask"

@dataclass
class ColumnSecurityPolicy:
    column_name: str
    security_level: SecurityLevel
    allowed_roles: Set[str]
    masking_type: MaskingType
    masking_config: Dict[str, any]
    audit_access: bool = True

class DataSecurityManager:
    def __init__(self, spark_session):
        self.spark = spark_session
        self.security_policies: Dict[str, ColumnSecurityPolicy] = {}
        self.user_roles: Dict[str, Set[str]] = {}
        self.audit_log = []
    
    def apply_column_security(self, df, table_name: str, user_id: str) -> 'DataFrame':
        """Apply column-level security policies to a DataFrame"""
        user_roles = self.user_roles.get(user_id, set())
        secured_df = df
        access_log = []
        
        for column in df.columns:
            policy_key = f"{table_name}.{column}"
            policy = self.security_policies.get(policy_key)
            
            if policy:
                has_access = bool(policy.allowed_roles & user_roles)
                
                if has_access:
                    access_decision = "granted"
                    masking_applied = MaskingType.NO_MASK
                else:
                    access_decision = "denied"
                    masking_applied = policy.masking_type
                    secured_df = self._apply_masking(secured_df, column, policy)
                
                if policy.audit_access:
                    access_log.append({
                        "user_id": user_id,
                        "table": table_name,
                        "column": column,
                        "access_decision": access_decision,
                        "masking_applied": masking_applied.value,
                        "timestamp": datetime.now()
                    })
        
        self.audit_log.extend(access_log)
        return secured_df
    
    def _apply_masking(self, df, column_name: str, policy: ColumnSecurityPolicy):
        """Apply the specified masking type to a column"""
        masking_type = policy.masking_type
        
        if masking_type == MaskingType.FULL_MASK:
            return df.withColumn(column_name, lit("***MASKED***"))
        elif masking_type == MaskingType.NULL_MASK:
            return df.withColumn(column_name, lit(None))
        elif masking_type == MaskingType.PARTIAL_MASK:
            return self._apply_partial_masking(df, column_name, policy.masking_config)
        elif masking_type == MaskingType.HASH_MASK:
            return self._apply_hash_masking(df, column_name)
        else:
            return df
    
    def _apply_partial_masking(self, df, column_name: str, config: dict):
        """Apply partial masking based on data type"""
        
        # Email masking UDF
        def mask_email(email: str) -> str:
            if not email or "@" not in email:
                return "***@***.***"
            username, domain = email.split("@", 1)
            visible_chars = config.get("visible_chars", 3)
            if len(username) <= visible_chars:
                masked_username = "*" * len(username)
            else:
                masked_username = username[:visible_chars] + "*" * (len(username) - visible_chars)
            return f"{masked_username}@{domain}"
        
        # Phone masking UDF
        def mask_phone(phone: str) -> str:
            if not phone:
                return "***-***-****"
            digits = re.sub(r'\D', '', phone)
            if len(digits) >= 4:
                return "*" * (len(digits) - 4) + digits[-4:]
            return "*" * len(digits)
        
        # Apply appropriate masking based on column type
        if "email" in column_name.lower():
            email_mask_udf = udf(mask_email, StringType())
            return df.withColumn(column_name, email_mask_udf(col(column_name)))
        elif "phone" in column_name.lower():
            phone_mask_udf = udf(mask_phone, StringType())
            return df.withColumn(column_name, phone_mask_udf(col(column_name)))
        else:
            # General string masking
            def mask_string(text: str) -> str:
                if not text:
                    return "***"
                visible_chars = config.get("visible_chars", 3)
                if len(text) <= visible_chars:
                    return "*" * len(text)
                return text[:visible_chars] + "*" * (len(text) - visible_chars)
            
            string_mask_udf = udf(mask_string, StringType())
            return df.withColumn(column_name, string_mask_udf(col(column_name)))

# Usage Example
security_manager = DataSecurityManager(spark)

# Register user roles
security_manager.user_roles["analyst_user"] = {"analyst", "internal"}
security_manager.user_roles["external_user"] = {"external", "basic"}

# Register column security policies
security_manager.security_policies["customers.email"] = ColumnSecurityPolicy(
    column_name="email",
    security_level=SecurityLevel.CONFIDENTIAL,
    allowed_roles={"analyst", "admin"},
    masking_type=MaskingType.PARTIAL_MASK,
    masking_config={"data_type": "email", "visible_chars": 3}
)

# Apply security
customers_df = spark.read.table("customers")
secured_df = security_manager.apply_column_security(customers_df, "customers", "analyst_user")
```

---

## Architecture & System Design

### Multi-Cloud Data Platform Design

Designing a multi-cloud platform requires abstraction layers to avoid vendor lock-in while maintaining performance.

#### Key Design Principles:

- **Abstraction Layers**: Cloud-agnostic interfaces for storage, compute, networking
- **Containerization**: Kubernetes for portable workload deployment
- **Open Standards**: Open-source technologies (Spark, Delta Lake, Kafka)
- **Cost Optimization**: Intelligent workload placement based on pricing
- **Data Governance**: Centralized metadata and lineage tracking

```python
from abc import ABC, abstractmethod
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class CloudProvider:
    name: str
    region: str
    cost_tier: str
    performance_tier: str

class CloudStorageAbstraction(ABC):
    """Abstract interface for cloud storage operations"""
    
    @abstractmethod
    def read_data(self, path: str, format: str = "parquet") -> 'DataFrame':
        pass
    
    @abstractmethod
    def write_data(self, df: 'DataFrame', path: str, format: str = "parquet") -> bool:
        pass
    
    @abstractmethod
    def get_cost_estimate(self, operation: str, size_gb: float) -> float:
        pass

class MultiCloudDataManager:
    def __init__(self, spark_session):
        self.spark = spark_session
        self.providers: Dict[str, CloudStorageAbstraction] = {}
        self.data_catalog: Dict[str, Dict] = {}
        self.replication_policies: Dict[str, Dict] = {}
        
    def write_dataset(self, df: 'DataFrame', dataset_name: str, 
                     provider_strategy: str = "cost_optimized") -> Dict[str, bool]:
        """Write dataset with multi-cloud strategy"""
        target_providers = self._select_providers(dataset_name, provider_strategy)
        
        results = {}
        for provider_name in target_providers:
            adapter = self.providers[provider_name]
            path = self._generate_path(dataset_name, provider_name)
            success = adapter.write_data(df, path, format="delta")
            results[provider_name] = success
        
        return results
    
    def _select_providers(self, dataset_name: str, strategy: str) -> List[str]:
        """Select optimal providers based on strategy"""
        if strategy == "cost_optimized":
            cost_rankings = self._get_cost_rankings()
            return [cost_rankings[0], cost_rankings[1]]  # Primary + backup
        elif strategy == "performance_optimized":
            performance_rankings = self._get_performance_rankings()
            return [performance_rankings[0]]
        elif strategy == "high_availability":
            return list(self.providers.keys())  # All providers
        else:
            return ["default_provider"]
    
    def optimize_placement(self) -> Dict[str, any]:
        """Recommend optimal data placement across providers"""
        cost_analysis = self.get_cost_analysis()
        recommendations = {}
        
        for dataset_name, provider_costs in cost_analysis.items():
            cheapest_provider = min(provider_costs.items(), key=lambda x: x[1]["total"])
            current_primary = self.data_catalog[dataset_name]["primary_provider"]
            
            if cheapest_provider[0] != current_primary:
                potential_savings = (provider_costs[current_primary]["total"] - 
                                   cheapest_provider[1]["total"])
                recommendations[dataset_name] = {
                    "current_primary": current_primary,
                    "recommended_primary": cheapest_provider[0],
                    "monthly_savings": potential_savings,
                    "action": "migrate" if potential_savings > 100 else "monitor"
                }
        
        return recommendations

# AWS Implementation
class AWSStorageAdapter(CloudStorageAbstraction):
    def __init__(self, spark_session):
        self.spark = spark_session
        
    def read_data(self, path: str, format: str = "parquet") -> 'DataFrame':
        s3_path = f"s3a://{path}"
        return self.spark.read.format(format).load(s3_path)
    
    def write_data(self, df: 'DataFrame', path: str, format: str = "parquet") -> bool:
        try:
            s3_path = f"s3a://{path}"
            df.write.format(format).mode("overwrite").save(s3_path)
            return True
        except Exception as e:
            print(f"AWS write failed: {e}")
            return False
    
    def get_cost_estimate(self, operation: str, size_gb: float) -> float:
        storage_cost = size_gb * 0.023  # $0.023 per GB per month
        if operation == "read":
            return size_gb * 0.0004
        elif operation == "write":
            return size_gb * 0.005
        return storage_cost

# Azure Implementation  
class AzureStorageAdapter(CloudStorageAbstraction):
    def __init__(self, spark_session):
        self.spark = spark_session
        
    def read_data(self, path: str, format: str = "parquet") -> 'DataFrame':
        adls_path = f"abfss://{path}"
        return self.spark.read.format(format).load(adls_path)
    
    def write_data(self, df: 'DataFrame', path: str, format: str = "parquet") -> bool:
        try:
            adls_path = f"abfss://{path}"
            df.write.format(format).mode("overwrite").save(adls_path)
            return True
        except Exception as e:
            print(f"Azure write failed: {e}")
            return False
    
    def get_cost_estimate(self, operation: str, size_gb: float) -> float:
        storage_cost = size_gb * 0.0208  # $0.0208 per GB per month
        if operation == "read":
            return size_gb * 0.00036
        elif operation == "write":
            return size_gb * 0.0045
        return storage_cost

# GCP Implementation
class GCPStorageAdapter(CloudStorageAbstraction):
    def __init__(self, spark_session):
        self.spark = spark_session
        
    def read_data(self, path: str, format: str = "parquet") -> 'DataFrame':
        gcs_path = f"gs://{path}"
        return self.spark.read.format(format).load(gcs_path)
    
    def write_data(self, df: 'DataFrame', path: str, format: str = "parquet") -> bool:
        try:
            gcs_path = f"gs://{path}"
            df.write.format(format).mode("overwrite").save(gcs_path)
            return True
        except Exception as e:
            print(f"GCP write failed: {e}")
            return False
    
    def get_cost_estimate(self, operation: str, size_gb: float) -> float:
        storage_cost = size_gb * 0.020  # $0.020 per GB per month
        if operation == "read":
            return size_gb * 0.0005
        elif operation == "write":
            return size_gb * 0.0050
        return storage_cost
```

### Cost Optimization Framework

```python
class CostOptimizer:
    def __init__(self, multi_cloud_manager):
        self.mcm = multi_cloud_manager
        
    def analyze_workload_costs(self) -> Dict[str, any]:
        """Analyze costs across different workload types"""
        return {
            "batch_processing": self._calculate_batch_costs(),
            "streaming": self._calculate_streaming_costs(),
            "analytics": self._calculate_analytics_costs()
        }
    
    def recommend_cost_optimizations(self) -> List[Dict[str, any]]:
        """Generate cost optimization recommendations"""
        recommendations = []
        recommendations.extend(self._analyze_storage_tiers())
        recommendations.extend(self._analyze_compute_optimization())
        recommendations.extend(self._analyze_lifecycle_policies())
        return recommendations
    
    def _analyze_storage_tiers(self) -> List[Dict[str, any]]:
        """Analyze optimal storage tier usage"""
        recommendations = []
        
        for dataset_name, metadata in self.mcm.data_catalog.items():
            access_pattern = metadata.get("access_pattern", "unknown")
            current_tier = metadata.get("storage_tier", "standard")
            
            if access_pattern == "infrequent" and current_tier == "standard":
                recommendations.append({
                    "type": "storage_tier_optimization",
                    "dataset": dataset_name,
                    "current_tier": current_tier,
                    "recommended_tier": "infrequent_access",
                    "estimated_savings": self._calculate_tier_savings(dataset_name, "infrequent_access"),
                    "priority": "medium"
                })
        
        return recommendations

# Usage Example
multi_cloud_manager = MultiCloudDataManager(spark)

# Register cloud providers
multi_cloud_manager.providers["aws"] = AWSStorageAdapter(spark)
multi_cloud_manager.providers["azure"] = AzureStorageAdapter(spark)
multi_cloud_manager.providers["gcp"] = GCPStorageAdapter(spark)

# Write data with multi-cloud strategy
customer_df = spark.read.table("customers")
write_results = multi_cloud_manager.write_dataset(
    customer_df, "customer_analytics", "high_availability"
)

# Get optimization recommendations
cost_optimizer = CostOptimizer(multi_cloud_manager)
recommendations = cost_optimizer.recommend_cost_optimizations()
```

---

## Related Topics

- [[Apache Spark Architecture]]
- [[Data Engineering]]
- [[Big Data Processing Patterns]]
- [[Cloud Data Platforms]]
- [[Data Quality and Governance]]
- [[Stream Processing Frameworks]]
- [[Performance Tuning Techniques]]
- [[Data Security and Privacy]]

## Tags

#pyspark #spark #data-engineering #big-data #distributed-computing #apache-spark #performance-optimization #streaming #data-governance #security #multi-cloud #interview-prep #python #sql

---

_Last Updated: [[2024-08-18]]_ _Source: PySpark Lead Data Engineer Interview Questions_