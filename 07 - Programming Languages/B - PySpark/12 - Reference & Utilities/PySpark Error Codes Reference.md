
#pyspark #errors #troubleshooting #debugging #reference #solutions #production

---

## 🎯 Purpose

Comprehensive reference for PySpark errors, exceptions, and their solutions. Organized by error type with copy-paste solutions and prevention strategies. Perfect for quick debugging and production incident resolution.

---

## 📋 Error Categories Index

- [[#💥 Runtime Errors]]
- [[#🧠 Memory & Resource Errors]]
- [[#📊 Data & Schema Errors]]
- [[#🔗 Join & Shuffle Errors]]
- [[#📁 File & I/O Errors]]
- [[#🌊 Streaming Errors]]
- [[#⚙️ Configuration Errors]]
- [[#🔌 Network & Connection Errors]]
- [[#🐍 Python & UDF Errors]]
- [[#🏗️ SQL & Catalyst Errors]]

---

## 💥 Runtime Errors

### **AnalysisException**

#### **Column Not Found**

```
AnalysisException: Column 'column_name' does not exist
```

**Common Causes:**

- Typo in column name
- Column was dropped in previous transformation
- Case sensitivity issues

**Solutions:**

```python
# Check available columns
print(df.columns)

# Case-insensitive column access
df.select([col(c) for c in df.columns if c.lower() == "target_column".lower()])

# Safe column selection
def safe_select(df, columns):
    available_cols = df.columns
    valid_cols = [c for c in columns if c in available_cols]
    return df.select(*valid_cols)

# Rename columns to avoid conflicts
df = df.toDF(*[c.lower().replace(" ", "_") for c in df.columns])
```

#### **Cannot Resolve Column**

```
AnalysisException: cannot resolve 'column_name' given input columns
```

**Solution:**

```python
# Check schema after transformations
df.printSchema()

# Use explicit column references
from pyspark.sql.functions import col
df.select(col("table_alias.column_name"))

# Debug pipeline step by step
df1 = df.select("col1", "col2")
print("After select:", df1.columns)
df2 = df1.withColumn("new_col", col("col1") * 2)
print("After withColumn:", df2.columns)
```

### **IllegalArgumentException**

#### **Empty Path**

```
IllegalArgumentException: Path does not exist
```

**Solutions:**

```python
import os
from pathlib import Path

# Check if path exists
def safe_read_parquet(spark, path):
    if os.path.exists(path):
        return spark.read.parquet(path)
    else:
        print(f"Path does not exist: {path}")
        return None

# Handle multiple possible paths
def read_with_fallback(spark, primary_path, fallback_path):
    try:
        return spark.read.parquet(primary_path)
    except:
        print(f"Primary path failed, trying fallback: {fallback_path}")
        return spark.read.parquet(fallback_path)
```

### **UnsupportedOperationException**

#### **Operation Not Supported**

```
UnsupportedOperationException: Operation not supported
```

**Common in:**

- Streaming queries with unsupported operations
- Complex nested operations
- Some SQL functions

**Solutions:**

```python
# Replace unsupported operations
# Instead of: streaming_df.orderBy("timestamp")  # Not supported in streaming
# Use window functions:
from pyspark.sql.window import Window
window = Window.partitionBy("user_id").orderBy("timestamp")
streaming_df.withColumn("rank", row_number().over(window))

# For complex operations, materialize intermediate results
intermediate_df = complex_df.checkpoint()  # Break lineage
result_df = intermediate_df.some_complex_operation()
```

---

## 🧠 Memory & Resource Errors

### **OutOfMemoryError**

#### **Driver Out of Memory**

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

**Solutions:**

```python
# Increase driver memory
spark.conf.set("spark.driver.memory", "8g")
spark.conf.set("spark.driver.maxResultSize", "4g")

# Avoid collecting large datasets
# Instead of: large_df.collect()
# Use: large_df.limit(1000).collect()

# Process data in chunks
def process_in_chunks(df, chunk_size=10000):
    total_rows = df.count()
    for i in range(0, total_rows, chunk_size):
        chunk = df.limit(chunk_size).offset(i)
        yield chunk.collect()

# Use actions that don't return data to driver
large_df.write.parquet("output_path")  # Instead of collect()
```

#### **Executor Out of Memory**

```
ExecutorLostFailure: Lost executor due to java.lang.OutOfMemoryError
```

**Solutions:**

```python
# Increase executor memory
spark.conf.set("spark.executor.memory", "8g")
spark.conf.set("spark.executor.memoryFraction", "0.8")

# Reduce partition size
current_partitions = df.rdd.getNumPartitions()
df = df.repartition(current_partitions * 2)  # Smaller partitions

# Handle skewed data
def handle_memory_skew(df, partition_col):
    # Add salt to skewed keys
    salted_df = df.withColumn("salt", (rand() * 10).cast("int"))
    salted_df = salted_df.withColumn("salted_key", 
                                   concat(col(partition_col), lit("_"), col("salt")))
    return salted_df.repartition("salted_key")

# Increase memory overhead
spark.conf.set("spark.executor.memoryOverhead", "1g")
```

### **GC (Garbage Collection) Issues**

```
WARN TaskSetManager: Lost task due to fetch failure or excessive GC
```

**Solutions:**

```python
# Optimize GC settings
spark.conf.set("spark.executor.extraJavaOptions", 
               "-XX:+UseG1GC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps")

# Reduce object creation
# Use primitive types instead of objects
# Cache frequently used DataFrames
df.cache()

# Tune memory fractions
spark.conf.set("spark.storage.memoryFraction", "0.5")
spark.conf.set("spark.shuffle.memoryFraction", "0.3")
```

---

## 📊 Data & Schema Errors

### **Schema Mismatch Errors**

#### **Union Schema Mismatch**

```
AnalysisException: Union can only be performed on tables with the same number of columns
```

**Solutions:**

```python
def safe_union(df1, df2):
    """Safely union DataFrames with different schemas"""
    
    # Get all columns from both DataFrames
    all_cols = set(df1.columns + df2.columns)
    
    # Add missing columns to each DataFrame
    for col_name in all_cols:
        if col_name not in df1.columns:
            df1 = df1.withColumn(col_name, lit(None))
        if col_name not in df2.columns:
            df2 = df2.withColumn(col_name, lit(None))
    
    # Ensure same column order
    sorted_cols = sorted(all_cols)
    df1 = df1.select(*sorted_cols)
    df2 = df2.select(*sorted_cols)
    
    return df1.union(df2)

# Handle data type mismatches
def align_schemas(df1, df2):
    """Align schemas by casting to common types"""
    
    schema1 = {field.name: field.dataType for field in df1.schema.fields}
    schema2 = {field.name: field.dataType for field in df2.schema.fields}
    
    for col_name in schema1:
        if col_name in schema2 and schema1[col_name] != schema2[col_name]:
            # Cast both to string as common type
            df1 = df1.withColumn(col_name, col(col_name).cast("string"))
            df2 = df2.withColumn(col_name, col(col_name).cast("string"))
    
    return df1, df2
```

#### **Data Type Mismatch**

```
AnalysisException: cannot resolve due to data type mismatch
```

**Solutions:**

```python
# Check and fix data types
def diagnose_schema_issues(df):
    """Diagnose common schema issues"""
    
    print("Schema Information:")
    df.printSchema()
    
    print("\nColumn Types:")
    for col_name, col_type in df.dtypes:
        print(f"  {col_name}: {col_type}")
    
    # Check for mixed types in string columns
    string_cols = [name for name, dtype in df.dtypes if dtype == 'string']
    for col_name in string_cols:
        print(f"\nSample values in {col_name}:")
        df.select(col_name).distinct().show(5)

# Safe type casting
def safe_cast(df, column, target_type):
    """Safely cast column to target type"""
    
    try:
        return df.withColumn(column, col(column).cast(target_type))
    except Exception as e:
        print(f"Failed to cast {column} to {target_type}: {e}")
        
        # Try to identify problematic values
        if target_type in ['int', 'double', 'float']:
            # Find non-numeric values
            non_numeric = df.filter(~col(column).rlike(r'^-?\d+\.?\d*$'))
            print(f"Non-numeric values found:")
            non_numeric.select(column).distinct().show()
        
        return df
```

### **Null Value Errors**

#### **Null Pointer Exception**

```
NullPointerException in user code
```

**Solutions:**

```python
# Handle nulls in UDFs
from pyspark.sql.functions import udf, when, isnan, isnull

def safe_divide_udf(numerator, denominator):
    """Safe division UDF that handles nulls and zeros"""
    if numerator is None or denominator is None or denominator == 0:
        return None
    return float(numerator) / float(denominator)

safe_divide = udf(safe_divide_udf, DoubleType())

# Handle nulls in aggregations
df.agg(
    sum(when(col("amount").isNotNull(), col("amount")).otherwise(0)).alias("total"),
    count(when(col("amount").isNotNull(), 1)).alias("non_null_count")
)

# Null-safe operations
df.withColumn("safe_result", 
              when(col("value").isNull(), lit("N/A"))
              .otherwise(col("value")))
```

---

## 🔗 Join & Shuffle Errors

### **Join Errors**

#### **Cartesian Product Warning**

```
WARN SQL: Cartesian product detected
```

**Solutions:**

```python
# Add proper join conditions
# Instead of: df1.join(df2)  # Creates cartesian product
# Use: df1.join(df2, "common_key")

# For intentional cartesian products
df1.crossJoin(df2)  # Explicit cartesian product

# Check for missing join conditions
def safe_join(df1, df2, join_keys, join_type="inner"):
    """Safe join with validation"""
    
    # Validate join keys exist
    missing_keys_df1 = [k for k in join_keys if k not in df1.columns]
    missing_keys_df2 = [k for k in join_keys if k not in df2.columns]
    
    if missing_keys_df1:
        raise ValueError(f"Join keys missing in df1: {missing_keys_df1}")
    if missing_keys_df2:
        raise ValueError(f"Join keys missing in df2: {missing_keys_df2}")
    
    # Check for null values in join keys
    for key in join_keys:
        null_count_df1 = df1.filter(col(key).isNull()).count()
        null_count_df2 = df2.filter(col(key).isNull()).count()
        
        if null_count_df1 > 0:
            print(f"Warning: {null_count_df1} null values in {key} (df1)")
        if null_count_df2 > 0:
            print(f"Warning: {null_count_df2} null values in {key} (df2)")
    
    return df1.join(df2, join_keys, join_type)
```

#### **Shuffle Hash Join Error**

```
Exception: Not enough memory to build hash table
```

**Solutions:**

```python
# Use broadcast join for smaller table
from pyspark.sql.functions import broadcast
large_df.join(broadcast(small_df), "key")

# Increase shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", "400")

# Use sort-merge join instead
df1.hint("merge").join(df2, "key")

# Check table sizes before joining
def analyze_join_tables(df1, df2, sample_fraction=0.01):
    """Analyze tables before joining"""
    
    count1 = df1.count()
    count2 = df2.count()
    
    print(f"Table 1 rows: {count1:,}")
    print(f"Table 2 rows: {count2:,}")
    
    # Estimate sizes
    if count1 < 1000000 and count2 > count1 * 10:
        print("Recommendation: Use broadcast join")
        return "broadcast"
    elif count1 > 100000000 or count2 > 100000000:
        print("Recommendation: Increase shuffle partitions")
        return "increase_partitions"
    else:
        print("Recommendation: Default join should work")
        return "default"
```

### **Shuffle Errors**

#### **Shuffle Fetch Failed**

```
FetchFailedException: Failed to fetch shuffle block
```

**Solutions:**

```python
# Increase shuffle retry and timeout settings
spark.conf.set("spark.shuffle.io.maxRetries", "5")
spark.conf.set("spark.shuffle.io.retryWait", "30s")
spark.conf.set("spark.network.timeout", "300s")

# Increase shuffle buffer size
spark.conf.set("spark.reducer.maxSizeInFlight", "96m")

# Handle large shuffle blocks
spark.conf.set("spark.sql.adaptive.shuffle.targetPostShuffleInputSize", "64MB")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "128MB")

# Checkpoint before large shuffles
df_checkpointed = df.checkpoint()
result = df_checkpointed.groupBy("key").agg(sum("value"))
```

---

## 📁 File & I/O Errors

### **File System Errors**

#### **File Not Found**

```
FileNotFoundException: Path does not exist
```

**Solutions:**

```python
import os
from pathlib import Path

def robust_file_reader(spark, path, format="parquet"):
    """Robust file reader with error handling"""
    
    if not os.path.exists(path):
        print(f"Path does not exist: {path}")
        
        # Try common variations
        variations = [
            path.rstrip('/'),
            path + '/',
            path.replace('\\', '/'),
            path.replace('/', '\\')
        ]
        
        for variation in variations:
            if os.path.exists(variation):
                print(f"Found alternative path: {variation}")
                path = variation
                break
        else:
            raise FileNotFoundError(f"No valid path found for: {path}")
    
    try:
        if format == "parquet":
            return spark.read.parquet(path)
        elif format == "csv":
            return spark.read.option("header", "true").csv(path)
        elif format == "json":
            return spark.read.json(path)
        else:
            raise ValueError(f"Unsupported format: {format}")
            
    except Exception as e:
        print(f"Error reading file: {e}")
        raise

# Handle S3/cloud storage errors
def safe_s3_read(spark, s3_path):
    """Safe S3 reading with retry logic"""
    
    max_retries = 3
    for attempt in range(max_retries):
        try:
            return spark.read.parquet(s3_path)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
```

#### **Permission Denied**

```
AccessDeniedException: Permission denied
```

**Solutions:**

```python
# Check and set proper permissions
import subprocess
import os

def fix_permissions(path):
    """Fix common permission issues"""
    
    try:
        # Make directory writable
        os.chmod(path, 0o755)
        print(f"Fixed permissions for: {path}")
    except Exception as e:
        print(f"Could not fix permissions: {e}")

# For HDFS
def check_hdfs_permissions(path):
    """Check HDFS permissions"""
    
    try:
        result = subprocess.run(['hdfs', 'dfs', '-ls', path], 
                              capture_output=True, text=True)
        print(f"HDFS permissions for {path}:")
        print(result.stdout)
    except Exception as e:
        print(f"Could not check HDFS permissions: {e}")
```

### **Serialization Errors**

#### **Serialization Exception**

```
NotSerializableException: Task not serializable
```

**Solutions:**

```python
# Make functions serializable
class SerializableClass:
    def __init__(self, value):
        self.value = value
    
    def process(self, x):
        return x * self.value

# Use broadcast variables for large objects
large_dict = {"key1": "value1", "key2": "value2"}
broadcast_dict = spark.sparkContext.broadcast(large_dict)

def process_with_broadcast(row):
    lookup = broadcast_dict.value
    return lookup.get(row.key, "default")

# Avoid capturing large objects in closures
# Instead of:
# large_object = some_large_data
# rdd.map(lambda x: process(x, large_object))  # large_object gets serialized

# Use:
def create_processor(large_object):
    def processor(x):
        return process(x, large_object)
    return processor

processor = create_processor(large_object)
rdd.map(processor)
```

---

## 🌊 Streaming Errors

### **Streaming Query Errors**

#### **Concurrent Stream Writers**

```
ConcurrentModificationException: Multiple streaming queries writing to same path
```

**Solutions:**

```python
# Use unique checkpoint locations
import uuid

def create_unique_checkpoint_path(base_path):
    """Create unique checkpoint path"""
    unique_id = str(uuid.uuid4())[:8]
    return f"{base_path}/checkpoint_{unique_id}"

# Proper streaming query management
class StreamingQueryManager:
    def __init__(self):
        self.active_queries = {}
    
    def start_query(self, query_name, streaming_df, output_path):
        if query_name in self.active_queries:
            print(f"Stopping existing query: {query_name}")
            self.active_queries[query_name].stop()
        
        checkpoint_path = create_unique_checkpoint_path("/tmp/checkpoints")
        
        query = streaming_df.writeStream \
            .format("parquet") \
            .option("path", output_path) \
            .option("checkpointLocation", checkpoint_path) \
            .start()
        
        self.active_queries[query_name] = query
        return query
    
    def stop_all_queries(self):
        for name, query in self.active_queries.items():
            query.stop()
        self.active_queries.clear()
```

#### **Watermark Issues**

```
AnalysisException: Append output mode not supported when there are streaming aggregations
```

**Solutions:**

```python
# Fix output mode for aggregations
# Instead of:
# .outputMode("append")  # Not supported for aggregations without watermarks

# Use:
streaming_df.withWatermark("timestamp", "10 minutes") \
    .groupBy(window(col("timestamp"), "5 minutes")) \
    .count() \
    .writeStream \
    .outputMode("update")  # or "complete"

# Handle late data properly
def setup_watermark_query(streaming_df, watermark_duration="10 minutes"):
    """Setup streaming query with proper watermark"""
    
    return streaming_df \
        .withWatermark("event_time", watermark_duration) \
        .groupBy(
            window(col("event_time"), "5 minutes"),
            col("user_id")
        ) \
        .agg(count("*").alias("event_count")) \
        .writeStream \
        .outputMode("append")  # Now supported with watermark
```

---

## ⚙️ Configuration Errors

### **Configuration Issues**

#### **Invalid Configuration**

```
IllegalArgumentException: Invalid value for configuration
```

**Solutions:**

```python
# Validate configuration before setting
def safe_set_config(spark, key, value):
    """Safely set Spark configuration"""
    
    valid_configs = {
        "spark.sql.shuffle.partitions": (int, 1, 10000),
        "spark.executor.memory": (str, None, None),
        "spark.sql.adaptive.enabled": (bool, None, None)
    }
    
    if key in valid_configs:
        expected_type, min_val, max_val = valid_configs[key]
        
        if expected_type == int:
            try:
                int_value = int(value)
                if min_val and int_value < min_val:
                    raise ValueError(f"Value too small: {int_value} < {min_val}")
                if max_val and int_value > max_val:
                    raise ValueError(f"Value too large: {int_value} > {max_val}")
                value = str(int_value)
            except ValueError as e:
                print(f"Invalid integer value for {key}: {value}")
                return False
        
        elif expected_type == bool:
            if str(value).lower() not in ['true', 'false']:
                print(f"Invalid boolean value for {key}: {value}")
                return False
    
    try:
        spark.conf.set(key, value)
        print(f"Set {key} = {value}")
        return True
    except Exception as e:
        print(f"Failed to set {key}: {e}")
        return False

# Get optimal configurations
def get_optimal_configs(cluster_size, executor_memory_gb=4):
    """Get optimal configurations based on cluster"""
    
    return {
        "spark.sql.shuffle.partitions": str(cluster_size * 50),
        "spark.executor.memory": f"{executor_memory_gb}g",
        "spark.executor.cores": "4",
        "spark.sql.adaptive.enabled": "true",
        "spark.sql.adaptive.coalescePartitions.enabled": "true"
    }
```

---

## 🔌 Network & Connection Errors

### **Database Connection Errors**

#### **JDBC Connection Failed**

```
SQLException: Connection refused
```

**Solutions:**

```python
import time

def robust_jdbc_connection(spark, jdbc_url, table, properties, max_retries=3):
    """Robust JDBC connection with retry logic"""
    
    for attempt in range(max_retries):
        try:
            df = spark.read \
                .format("jdbc") \
                .option("url", jdbc_url) \
                .option("dbtable", table) \
                .option("user", properties.get("user")) \
                .option("password", properties.get("password")) \
                .option("driver", properties.get("driver")) \
                .load()
            
            # Test connection by counting
            count = df.count()
            print(f"Successfully connected, rows: {count}")
            return df
            
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff

# Connection pool settings
def optimize_jdbc_settings():
    """Optimize JDBC connection settings"""
    
    return {
        "numPartitions": "10",
        "fetchsize": "10000",
        "batchsize": "10000",
        "queryTimeout": "300",
        "connectionTimeout": "60"
    }
```

### **Kafka Connection Errors**

#### **Kafka Bootstrap Servers Unreachable**

```
KafkaException: Failed to resolve bootstrap servers
```

**Solutions:**

```python
def test_kafka_connection(bootstrap_servers):
    """Test Kafka connection before streaming"""
    
    try:
        # Simple test read
        test_df = spark.read \
            .format("kafka") \
            .option("kafka.bootstrap.servers", bootstrap_servers) \
            .option("subscribe", "test-topic") \
            .option("startingOffsets", "earliest") \
            .option("endingOffsets", "latest") \
            .load()
        
        count = test_df.count()
        print(f"Kafka connection successful, test messages: {count}")
        return True
        
    except Exception as e:
        print(f"Kafka connection failed: {e}")
        return False

# Robust Kafka streaming setup
def create_robust_kafka_stream(spark, bootstrap_servers, topic):
    """Create robust Kafka stream with error handling"""
    
    if not test_kafka_connection(bootstrap_servers):
        raise ConnectionError("Cannot connect to Kafka")
    
    return spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", bootstrap_servers) \
        .option("subscribe", topic) \
        .option("startingOffsets", "latest") \
        .option("failOnDataLoss", "false") \
        .option("maxOffsetsPerTrigger", "10000") \
        .load()
```

---

## 🐍 Python & UDF Errors

### **UDF Errors**

#### **Python Worker Failed**

```
Py4JJavaError: Python worker failed to connect back
```

**Solutions:**

```python
# Increase timeouts
spark.conf.set("spark.sql.pyspark.jvmStacktrace.enabled", "true")
spark.conf.set("spark.python.worker.reuse", "true")

# Optimize UDF performance
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import DoubleType
import pandas as pd

# Use Pandas UDF instead of regular UDF
@pandas_udf(returnType=DoubleType())
def optimized_calculation(values: pd.Series) -> pd.Series:
    """Vectorized calculation using Pandas UDF"""
    return values * 2 + 1

# Handle exceptions in UDFs
def safe_udf_wrapper(func):
    """Wrapper to handle exceptions in UDFs"""
    
    def wrapper(*args):
        try:
            return func(*args)
        except Exception as e:
            print(f"UDF error: {e}")
            return None
    
    return wrapper

@udf(returnType=StringType())
@safe_udf_wrapper
def risky_processing(value):
    """UDF that might fail"""
    if value is None:
        return "NULL"
    return str(value).upper()
```

#### **Pickle Serialization Error**

```
PicklingError: Could not serialize object
```

**Solutions:**

```python
# Use simpler data structures in UDFs
# Instead of complex objects, use basic types
def create_serializable_function(config_dict):
    """Create serializable function with configuration"""
    
    def process_value(value):
        # Use only serializable objects
        multiplier = config_dict.get("multiplier", 1)
        return value * multiplier
    
    return udf(process_value, DoubleType())

# Use broadcast variables for large lookup data
lookup_data = {"A": 1, "B": 2, "C": 3}
broadcast_lookup = spark.sparkContext.broadcast(lookup_data)

@udf(returnType=IntegerType())
def lookup_value(key):
    """Use broadcast variable in UDF"""
    return broadcast_lookup.value.get(key, 0)
```

---

## 🏗️ SQL & Catalyst Errors

### **SQL Parsing Errors**

#### **ParseException**

```
ParseException: Syntax error in SQL statement
```

**Solutions:**

```python
# Validate SQL before execution
def safe_sql_execution(spark, sql_query):
    """Safely execute SQL with validation"""
    
    try:
        # Try to create a plan without executing
        plan = spark.sql(sql_query).explain(extended=True)
        print("SQL syntax is valid")
        
        return spark.sql(sql_query)
        
    except Exception as e:
        print(f"SQL error: {e}")
        
        # Common fixes
        fixes = [
            "Check for unmatched quotes or parentheses",
            "Verify table and column names exist",
            "Check for reserved keywords",
            "Validate join conditions"
        ]
        
        print("Common fixes:")
        for fix in fixes:
            print(f"  - {fix}")
        
        raise

# Escape special characters in SQL
def escape_sql_identifier(identifier):
    """Escape SQL identifiers that might be reserved words"""
    return f"`{identifier}`"

def build_safe_sql(table_name, columns, conditions=None):
    """Build SQL with proper escaping"""
    
    escaped_table = escape_sql_identifier(table_name)
    escaped_columns = [escape_sql_identifier(col) for col in columns]
    
    sql = f"SELECT {', '.join(escaped_columns)} FROM {escaped_table}"
    
    if conditions:
        sql += f" WHERE {conditions}"
    
    return sql
```

---

## 🚨 Emergency Debugging Procedures

### **Production Incident Response**

#### **Quick Diagnosis Checklist**

```python
def emergency_diagnosis(spark, df=None):
    """Quick diagnosis for production issues"""
    
    print("=== EMERGENCY SPARK DIAGNOSIS ===")
    
    # 1. Check Spark Context
    try:
        print(f"✅ SparkContext active: {not spark.sparkContext._jsc.sc().isStopped()}")
        print(f"✅ Application ID: {spark.sparkContext.applicationId}")
        print(f"✅ Master: {spark.sparkContext.master}")
    except Exception as e:
        print(f"❌ SparkContext issue: {e}")
    
    # 2. Check configuration
    critical_configs = [
        "spark.sql.shuffle.partitions",
        "spark.executor.memory", 
        "spark.driver.memory",
        "spark.sql.adaptive.enabled"
    ]
    
    print("\n--- Critical Configurations ---")
    for config in critical_configs:
        try:
            value = spark.conf.get(config)
            print(f"✅ {config}: {value}")
        except:
            print(f"❌ {config}: NOT SET")
    
    #
```