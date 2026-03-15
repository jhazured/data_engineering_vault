
#pyspark #reference #cheat-sheet #quick-reference #daily-use #commands

---

## 🎯 Purpose

Quick reference cards for daily PySpark development. Each card focuses on a specific area with the most commonly used operations, optimized for speed and practicality.

---

## 📚 Reference Card Index

- [[#🚀 DataFrame Essentials]]
- [[#🔧 Data I/O Operations]]
- [[#🔄 Transformations & Actions]]
- [[#📊 Aggregations & GroupBy]]
- [[#🪟 Window Functions]]
- [[#🔗 Joins & Unions]]
- [[#⚡ Performance Optimization]]
- [[#🎛️ Spark Configuration]]
- [[#🐛 Debugging & Troubleshooting]]
- [[#📈 Monitoring & Metrics]]
- [[#🧪 Testing Patterns]]
- [[#🌊 Streaming Essentials]]

---

## 🚀 DataFrame Essentials

### **Creating DataFrames**

```python
# From list of tuples
df = spark.createDataFrame([(1, "Alice"), (2, "Bob")], ["id", "name"])

# From Pandas
df = spark.createDataFrame(pandas_df)

# Empty DataFrame with schema
from pyspark.sql.types import *
schema = StructType([StructField("id", IntegerType()), StructField("name", StringType())])
df = spark.createDataFrame([], schema)

# Range DataFrame (for testing)
df = spark.range(1000000).toDF("id")
```

### **Basic DataFrame Info**

```python
df.show(5)                    # Display first 5 rows
df.printSchema()              # Show schema
df.count()                    # Row count
df.columns                    # Column names list
df.dtypes                     # Column types
df.describe().show()          # Summary statistics
df.head(3)                    # First 3 rows as list
```

### **Column Operations**

```python
from pyspark.sql.functions import col, lit

# Select columns
df.select("col1", "col2")
df.select(col("col1"), col("col2").alias("renamed"))

# Add/modify columns
df.withColumn("new_col", col("old_col") * 2)
df.withColumn("literal_col", lit("constant_value"))

# Drop columns
df.drop("col1", "col2")

# Rename columns
df.withColumnRenamed("old_name", "new_name")
```

---

## 🔧 Data I/O Operations

### **Reading Data**

```python
# CSV
df = spark.read.option("header", "true").option("inferSchema", "true").csv("path/to/file.csv")

# Parquet (recommended)
df = spark.read.parquet("path/to/file.parquet")

# JSON
df = spark.read.json("path/to/file.json")

# Delta Lake
df = spark.read.format("delta").load("path/to/delta-table")

# JDBC Database
df = spark.read.format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/db") \
    .option("dbtable", "table_name") \
    .option("user", "username") \
    .option("password", "password") \
    .load()
```

### **Writing Data**

```python
# Write modes: "overwrite", "append", "ignore", "error"

# Parquet
df.write.mode("overwrite").parquet("output/path")

# CSV
df.write.mode("overwrite").option("header", "true").csv("output/path")

# Delta Lake
df.write.format("delta").mode("overwrite").save("output/path")

# Partitioned write
df.write.partitionBy("year", "month").parquet("output/path")

# Database
df.write.format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/db") \
    .option("dbtable", "table_name") \
    .mode("overwrite") \
    .save()
```

---

## 🔄 Transformations & Actions

### **Common Transformations** (Lazy)

```python
# Filter
df.filter(col("age") > 25)
df.filter("age > 25")  # SQL string
df.where(col("status") == "active")

# Select & Project
df.select("name", "age")
df.select(col("name"), (col("salary") * 1.1).alias("new_salary"))

# Add columns
df.withColumn("full_name", concat(col("first_name"), lit(" "), col("last_name")))

# Sort
df.orderBy("age")
df.orderBy(col("age").desc())

# Distinct
df.distinct()
df.dropDuplicates(["name", "email"])

# Sample
df.sample(0.1)  # 10% sample
```

### **Common Actions** (Eager)

```python
# Collect data
df.collect()          # All rows (careful with large data!)
df.take(5)           # First 5 rows
df.first()           # First row
df.head(3)           # First 3 rows

# Counts
df.count()           # Total rows
df.agg(countDistinct("user_id")).collect()[0][0]  # Distinct count

# Write operations
df.show()
df.write.parquet("path")
```

---

## 📊 Aggregations & GroupBy

### **Basic Aggregations**

```python
from pyspark.sql.functions import *

# Simple aggregations
df.agg(
    count("*").alias("total_rows"),
    sum("amount").alias("total_amount"),
    avg("amount").alias("avg_amount"),
    min("amount").alias("min_amount"),
    max("amount").alias("max_amount")
)

# GroupBy aggregations
df.groupBy("category") \
  .agg(
      count("*").alias("count"),
      sum("amount").alias("total"),
      avg("amount").alias("average")
  )
```

### **Advanced Aggregations**

```python
# Multiple grouping columns
df.groupBy("region", "category").agg(sum("sales"))

# Conditional aggregations
df.groupBy("category") \
  .agg(
      sum(when(col("amount") > 100, col("amount")).otherwise(0)).alias("high_value_sum"),
      count(when(col("status") == "completed", 1)).alias("completed_count")
  )

# Collect aggregations
df.groupBy("user_id") \
  .agg(collect_list("product_id").alias("purchased_products"))

# Statistical functions
df.agg(
    variance("amount").alias("variance"),
    stddev("amount").alias("std_dev"),
    skewness("amount").alias("skewness"),
    kurtosis("amount").alias("kurtosis")
)
```

---

## 🪟 Window Functions

### **Window Specification**

```python
from pyspark.sql.window import Window

# Basic window
window = Window.partitionBy("department").orderBy("salary")

# Window with frame
window_frame = Window.partitionBy("user_id") \
                     .orderBy("timestamp") \
                     .rowsBetween(Window.unboundedPreceding, Window.currentRow)
```

### **Ranking Functions**

```python
# Ranking
df.withColumn("rank", rank().over(window))
df.withColumn("dense_rank", dense_rank().over(window))
df.withColumn("row_number", row_number().over(window))
df.withColumn("percent_rank", percent_rank().over(window))

# Top N per group
df.withColumn("rank", row_number().over(window)) \
  .filter(col("rank") <= 3)
```

### **Analytical Functions**

```python
# Lead/Lag
df.withColumn("previous_value", lag("amount", 1).over(window))
df.withColumn("next_value", lead("amount", 1).over(window))

# First/Last
df.withColumn("first_in_group", first("amount").over(window))
df.withColumn("last_in_group", last("amount").over(window))

# Running totals
window_running = Window.partitionBy("user_id") \
                       .orderBy("timestamp") \
                       .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df.withColumn("running_total", sum("amount").over(window_running))
df.withColumn("running_avg", avg("amount").over(window_running))
```

---

## 🔗 Joins & Unions

### **Join Types**

```python
# Inner join (default)
df1.join(df2, "common_key")
df1.join(df2, df1.key == df2.key)

# Other join types
df1.join(df2, "key", "left")      # Left outer
df1.join(df2, "key", "right")     # Right outer  
df1.join(df2, "key", "outer")     # Full outer
df1.join(df2, "key", "left_semi") # Left semi
df1.join(df2, "key", "left_anti") # Left anti
```

### **Join Optimizations**

```python
# Broadcast join (for small tables)
from pyspark.sql.functions import broadcast
large_df.join(broadcast(small_df), "key")

# Multiple conditions
df1.join(df2, (df1.key1 == df2.key1) & (df1.key2 == df2.key2))
```

### **Unions**

```python
# Union (remove duplicates)
df1.union(df2)

# Union all (keep duplicates) 
df1.unionAll(df2)  # Deprecated
df1.union(df2)     # Same as unionAll in newer versions
```

---

## ⚡ Performance Optimization

### **Caching & Persistence**

```python
from pyspark import StorageLevel

# Simple caching
df.cache()                                    # MEMORY_AND_DISK
df.persist()                                  # MEMORY_AND_DISK

# Specific storage levels
df.persist(StorageLevel.MEMORY_ONLY)         # Memory only
df.persist(StorageLevel.MEMORY_AND_DISK_SER) # Serialized
df.persist(StorageLevel.DISK_ONLY)           # Disk only

# Unpersist when done
df.unpersist()
```

### **Partitioning**

```python
# Check current partitions
df.rdd.getNumPartitions()

# Repartition (full shuffle)
df.repartition(200)                    # Number of partitions
df.repartition("key")                  # By column
df.repartition(10, "key")             # Both

# Coalesce (reduce partitions, no shuffle)
df.coalesce(50)

# Optimal partition size: 128MB - 1GB per partition
```

### **Query Optimization**

```python
# Predicate pushdown (apply filters early)
df.filter(col("date") >= "2024-01-01") \
  .select("user_id", "amount") \
  .groupBy("user_id").sum("amount")

# Column pruning (select only needed columns)
df.select("needed_col1", "needed_col2") \
  .filter(col("status") == "active")

# Broadcast hint for joins
df1.hint("broadcast").join(df2, "key")
```

---

## 🎛️ Spark Configuration

### **Common Configuration Settings**

```python
# Set configuration
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "200")
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")

# Get configuration
spark.conf.get("spark.sql.shuffle.partitions")

# Show all configurations
for key, value in spark.sparkContext.getConf().getAll():
    print(f"{key}: {value}")
```

### **Performance Configurations**

```python
# Adaptive Query Execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Memory settings
spark.conf.set("spark.executor.memory", "4g")
spark.conf.set("spark.driver.memory", "2g")
spark.conf.set("spark.executor.cores", "4")

# Shuffle optimization
spark.conf.set("spark.sql.shuffle.partitions", "200")
spark.conf.set("spark.sql.adaptive.shuffle.targetPostShuffleInputSize", "64MB")
```

---

## 🐛 Debugging & Troubleshooting

### **Query Plan Analysis**

```python
# Show execution plan
df.explain()                    # Physical plan
df.explain(True)               # Extended plan
df.explain("extended")         # All plans
df.explain("formatted")        # Formatted plan

# Check for common issues
plan_string = df._jdf.queryExecution().toString()
if "CartesianProduct" in plan_string:
    print("⚠️ Cartesian product detected!")
if "Exchange" in plan_string:
    print("🔄 Shuffle operations present")
```

### **Performance Debugging**

```python
import time

# Measure execution time
start_time = time.time()
result = df.count()
end_time = time.time()
print(f"Execution time: {end_time - start_time:.2f} seconds")

# Check partition distribution
partition_counts = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
print(f"Partition sizes: {partition_counts}")
print(f"Max/Min ratio: {max(partition_counts) / min(partition_counts):.2f}")

# Memory usage estimation
row_count = df.count()
column_count = len(df.columns)
estimated_size_mb = (row_count * column_count * 8) / (1024 * 1024)  # Rough estimate
print(f"Estimated size: {estimated_size_mb:.2f} MB")
```

### **Common Error Patterns**

```python
# Handle schema mismatch
try:
    df1.union(df2)
except Exception as e:
    if "schema" in str(e).lower():
        print("Schema mismatch - check column names and types")
        print(f"DF1 schema: {df1.columns}")
        print(f"DF2 schema: {df2.columns}")

# Check for null values
null_counts = df.select([count(when(col(c).isNull(), c)).alias(c) for c in df.columns])
null_counts.show()

# Data quality checks
print(f"Total rows: {df.count()}")
print(f"Distinct rows: {df.distinct().count()}")
print(f"Null values per column:")
df.select([sum(col(c).isNull().cast("int")).alias(c) for c in df.columns]).show()
```

---

## 📈 Monitoring & Metrics

### **Spark UI Navigation**

```python
# Access Spark UI at: http://driver-node:4040
print(f"Spark UI: http://localhost:4040")
print(f"Application ID: {spark.sparkContext.applicationId}")
print(f"Application Name: {spark.sparkContext.appName}")

# Get application status
status = spark.sparkContext.statusTracker()
print(f"Active jobs: {len(status.getActiveJobIds())}")
print(f"Active stages: {len(status.getActiveStageIds())}")
```

### **Performance Metrics**

```python
# DataFrame metrics
print(f"Partitions: {df.rdd.getNumPartitions()}")
print(f"Is cached: {df.is_cached}")
print(f"Storage level: {df.storageLevel}")

# Execution metrics (after action)
start_time = time.time()
count = df.count()
execution_time = time.time() - start_time
print(f"Rows: {count:,}")
print(f"Execution time: {execution_time:.2f}s")
print(f"Rows/second: {count/execution_time:.0f}")
```

---

## 🧪 Testing Patterns

### **Unit Testing Setup**

```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import *

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder \
        .appName("test") \
        .master("local[2]") \
        .config("spark.sql.shuffle.partitions", "2") \
        .getOrCreate()

def test_dataframe_transformation(spark):
    # Create test data
    test_data = [(1, "Alice"), (2, "Bob")]
    test_df = spark.createDataFrame(test_data, ["id", "name"])
    
    # Apply transformation
    result_df = test_df.filter(col("id") > 1)
    
    # Assert results
    assert result_df.count() == 1
    assert result_df.collect()[0]["name"] == "Bob"
```

### **Data Quality Tests**

```python
def assert_dataframe_equal(df1, df2):
    """Assert two DataFrames are equal"""
    assert df1.schema == df2.schema
    assert df1.count() == df2.count()
    assert df1.collect() == df2.collect()

def assert_no_nulls(df, columns):
    """Assert no null values in specified columns"""
    for col_name in columns:
        null_count = df.filter(col(col_name).isNull()).count()
        assert null_count == 0, f"Found {null_count} nulls in {col_name}"

def assert_positive_values(df, column):
    """Assert all values in column are positive"""
    negative_count = df.filter(col(column) <= 0).count()
    assert negative_count == 0, f"Found {negative_count} non-positive values"
```

---

## 🌊 Streaming Essentials

### **Basic Streaming Setup**

```python
# Read from Kafka
streaming_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "topic_name") \
    .option("startingOffsets", "latest") \
    .load()

# Parse JSON from Kafka
parsed_df = streaming_df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

# Basic aggregation with watermark
windowed_df = parsed_df \
    .withWatermark("timestamp", "10 minutes") \
    .groupBy(window(col("timestamp"), "5 minutes")) \
    .count()
```

### **Streaming Outputs**

```python
# Console output (for testing)
query = windowed_df.writeStream \
    .outputMode("update") \
    .format("console") \
    .trigger(processingTime="30 seconds") \
    .start()

# File output
query = windowed_df.writeStream \
    .outputMode("append") \
    .format("parquet") \
    .option("path", "/output/path") \
    .option("checkpointLocation", "/checkpoint/path") \
    .trigger(processingTime="1 minute") \
    .start()

# Kafka output
kafka_output = processed_df.select(
    col("key").cast("string"),
    to_json(struct("*")).alias("value")
).writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "output_topic") \
    .start()

# Manage queries
query.awaitTermination()  # Wait for termination
query.stop()              # Stop query
```

---

## 💡 Quick Tips & Best Practices

### **Performance Tips**

- Use `.cache()` on DataFrames accessed multiple times
- Apply filters as early as possible in your pipeline
- Use `broadcast()` for small lookup tables (< 100MB)
- Prefer Parquet over CSV for analytical workloads
- Set appropriate number of partitions (aim for 128MB-1GB per partition)

### **Development Tips**

- Use `.limit()` when developing with large datasets
- Always specify schema for streaming and production jobs
- Use descriptive aliases for complex expressions
- Test with small datasets before running on production data

### **Common Gotchas**

- Remember transformations are lazy - add `.show()` or `.count()` to see results
- DataFrame operations create new DataFrames - assign results to variables
- Be careful with `.collect()` on large datasets - it brings all data to driver
- Always unpersist cached DataFrames when done to free memory

---

## 🔗 Related Resources

- **Full Documentation**: [[PySpark MOC (Map of Content)]]
- **Deep Dives**: Individual topic notes in your knowledge system
- **Troubleshooting**: [[PySpark Troubleshooting Guide]]
- **Interview Prep**: [[PySpark Interview Questions]]

---

_Keep this reference handy for daily development! Bookmark the sections you use most frequently._

---

_Last Updated: 2024-08-20_