
#pyspark #performance #optimization #caching #partitioning #skew #tuning #production

## Overview

This note covers comprehensive performance optimization techniques for PySpark applications. From basic caching strategies to advanced skew handling, these techniques are essential for production-ready, high-performance data processing pipelines.

---

## Performance Fundamentals

### Understanding Spark's Execution Model

Before optimizing, understand how Spark executes your code:

1. **Driver Program** - Creates SparkContext, defines transformations
2. **Cluster Manager** - Allocates resources across cluster
3. **Executors** - Run tasks and store data in memory
4. **Tasks** - Units of work sent to executors

### Key Performance Metrics

Monitor these metrics to identify bottlenecks:

```python
# Enable Spark metrics and monitoring
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")

# Check current configuration
print("Current Spark Configuration:")
for key, value in spark.sparkContext.getConf().getAll():
    if "sql" in key or "shuffle" in key or "memory" in key:
        print(f"{key}: {value}")
```

---

## Caching and Persistence

### Storage Levels Deep Dive

Understanding when and how to cache is crucial for performance.

```python
from pyspark import StorageLevel

# Storage level comparison with use cases
storage_levels = {
    StorageLevel.MEMORY_ONLY: {
        "description": "Store in memory only, recompute if lost",
        "use_case": "Fast repeated access, enough memory available",
        "memory_usage": "High",
        "cpu_cost": "Low (if fits in memory)"
    },
    StorageLevel.MEMORY_AND_DISK: {
        "description": "Store in memory, spill to disk if needed",
        "use_case": "Default choice for most scenarios",
        "memory_usage": "Medium",
        "cpu_cost": "Medium"
    },
    StorageLevel.MEMORY_ONLY_SER: {
        "description": "Store in memory serialized",
        "use_case": "Limited memory, can afford serialization cost",
        "memory_usage": "Lower",
        "cpu_cost": "Higher (serialization)"
    },
    StorageLevel.DISK_ONLY: {
        "description": "Store only on disk",
        "use_case": "Large datasets, limited memory",
        "memory_usage": "None",
        "cpu_cost": "High (disk I/O)"
    }
}

# Intelligent caching based on dataset characteristics
def smart_cache(df, estimated_size_gb, access_frequency, memory_available_gb):
    """Intelligently choose cache strategy based on data characteristics"""
    
    if estimated_size_gb <= memory_available_gb * 0.3 and access_frequency >= 3:
        # Small dataset, frequent access
        return df.persist(StorageLevel.MEMORY_ONLY)
    elif estimated_size_gb <= memory_available_gb * 0.6 and access_frequency >= 2:
        # Medium dataset, moderate access
        return df.persist(StorageLevel.MEMORY_AND_DISK)
    elif access_frequency >= 2:
        # Large dataset but frequent access
        return df.persist(StorageLevel.MEMORY_AND_DISK_SER)
    else:
        # Infrequent access, don't cache
        return df

# Example usage
customers_df = spark.read.parquet("customers.parquet")
cached_customers = smart_cache(customers_df, 2.5, 4, 16)  # 2.5GB, 4 accesses, 16GB memory
```

### Cache Management Best Practices

```python
class CacheManager:
    """Manage DataFrame caching lifecycle"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.cached_dfs = {}
        self.cache_stats = {}
    
    def cache_with_tracking(self, df, name, storage_level=StorageLevel.MEMORY_AND_DISK):
        """Cache DataFrame with usage tracking"""
        cached_df = df.persist(storage_level)
        
        # Force materialization and measure
        start_time = time.time()
        count = cached_df.count()
        cache_time = time.time() - start_time
        
        self.cached_dfs[name] = cached_df
        self.cache_stats[name] = {
            "count": count,
            "cache_time": cache_time,
            "access_count": 0,
            "storage_level": storage_level
        }
        
        print(f"Cached {name}: {count:,} rows in {cache_time:.2f}s")
        return cached_df
    
    def access_cached_df(self, name):
        """Access cached DataFrame and track usage"""
        if name in self.cached_dfs:
            self.cache_stats[name]["access_count"] += 1
            return self.cached_dfs[name]
        else:
            raise ValueError(f"DataFrame {name} not found in cache")
    
    def cleanup_unused_cache(self, min_access_threshold=2):
        """Remove DataFrames from cache that aren't being used"""
        to_remove = []
        
        for name, stats in self.cache_stats.items():
            if stats["access_count"] < min_access_threshold:
                print(f"Removing {name} from cache (only {stats['access_count']} accesses)")
                self.cached_dfs[name].unpersist()
                to_remove.append(name)
        
        for name in to_remove:
            del self.cached_dfs[name]
            del self.cache_stats[name]
    
    def cache_summary(self):
        """Print cache usage summary"""
        for name, stats in self.cache_stats.items():
            print(f"{name}: {stats['count']:,} rows, {stats['access_count']} accesses, "
                  f"cached in {stats['cache_time']:.2f}s")

# Usage example
cache_manager = CacheManager(spark)

# Cache commonly used datasets
lookup_tables = cache_manager.cache_with_tracking(
    spark.read.parquet("lookup_tables.parquet"), 
    "lookup_tables",
    StorageLevel.MEMORY_ONLY
)

# Use cached data
result1 = cache_manager.access_cached_df("lookup_tables").filter(col("status") == "active")
result2 = cache_manager.access_cached_df("lookup_tables").filter(col("category") == "premium")

# Clean up unused cache
cache_manager.cleanup_unused_cache()
```

---

## Data Partitioning Strategies

### Understanding Partitioning Impact

```python
def analyze_partitions(df, description=""):
    """Analyze partition distribution"""
    partition_counts = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
    
    print(f"\n=== Partition Analysis: {description} ===")
    print(f"Total partitions: {len(partition_counts)}")
    print(f"Total records: {sum(partition_counts):,}")
    print(f"Average records per partition: {sum(partition_counts) / len(partition_counts):.0f}")
    print(f"Min records per partition: {min(partition_counts):,}")
    print(f"Max records per partition: {max(partition_counts):,}")
    print(f"Partition skew ratio: {max(partition_counts) / min(partition_counts):.2f}")
    
    # Show partition size distribution
    size_ranges = [0, 1000, 10000, 100000, 1000000, float('inf')]
    range_labels = ["0-1K", "1K-10K", "10K-100K", "100K-1M", "1M+"]
    
    for i, (start, end, label) in enumerate(zip(size_ranges[:-1], size_ranges[1:], range_labels)):
        count = sum(1 for size in partition_counts if start <= size < end)
        print(f"Partitions with {label} records: {count}")
    
    return partition_counts

# Example analysis
df_original = spark.read.parquet("large_dataset.parquet")
analyze_partitions(df_original, "Original Data")
```

### Optimal Partitioning Strategies

```python
def optimize_partitions(df, target_partition_size_mb=128, target_partition_count=None):
    """Automatically optimize partition count based on data size"""
    
    # Estimate data size (approximate)
    sample_fraction = 0.01
    sample_df = df.sample(sample_fraction)
    sample_count = sample_df.count()
    
    if sample_count == 0:
        return df
    
    # Estimate total size
    estimated_total_rows = sample_count / sample_fraction
    
    # Estimate optimal partition count
    if target_partition_count is None:
        # Rule of thumb: target 128MB per partition
        # Rough estimate: 1M rows ≈ 100MB for typical datasets
        rows_per_mb = 10000  # Adjust based on your data
        target_rows_per_partition = target_partition_size_mb * rows_per_mb
        optimal_partitions = max(1, int(estimated_total_rows / target_rows_per_partition))
        
        # Ensure we don't exceed cluster capacity
        max_partitions = spark.sparkContext.defaultParallelism * 3
        optimal_partitions = min(optimal_partitions, max_partitions)
    else:
        optimal_partitions = target_partition_count
    
    current_partitions = df.rdd.getNumPartitions()
    
    print(f"Current partitions: {current_partitions}")
    print(f"Estimated total rows: {estimated_total_rows:,.0f}")
    print(f"Recommended partitions: {optimal_partitions}")
    
    if optimal_partitions < current_partitions:
        return df.coalesce(optimal_partitions)
    elif optimal_partitions > current_partitions:
        return df.repartition(optimal_partitions)
    else:
        return df

# Column-based partitioning for better query performance
def optimize_column_partitioning(df, partition_column, max_partitions=200):
    """Optimize partitioning based on column cardinality"""
    
    # Analyze column cardinality
    distinct_values = df.select(partition_column).distinct().count()
    print(f"Distinct values in {partition_column}: {distinct_values}")
    
    if distinct_values <= max_partitions:
        # Low cardinality - partition by the column
        print(f"Partitioning by {partition_column}")
        return df.repartition(col(partition_column))
    else:
        # High cardinality - use hash partitioning
        optimal_partitions = min(distinct_values, max_partitions)
        print(f"Hash partitioning into {optimal_partitions} partitions")
        return df.repartition(optimal_partitions, col(partition_column))

# Usage examples
df_optimized = optimize_partitions(df_original)
analyze_partitions(df_optimized, "After Optimization")

# For datasets with natural partition keys
df_by_date = optimize_column_partitioning(df_optimized, "date")
```

### Range Partitioning for Sorted Data

```python
def range_partition_by_date(df, date_column, num_partitions=None):
    """Partition data by date ranges for better sort performance"""
    
    # Get date range
    date_stats = df.agg(
        min(date_column).alias("min_date"),
        max(date_column).alias("max_date")
    ).collect()[0]
    
    min_date = date_stats["min_date"]
    max_date = date_stats["max_date"]
    
    if num_partitions is None:
        # Calculate partitions based on date range
        days_diff = (max_date - min_date).days
        num_partitions = max(1, min(days_diff // 7, 200))  # Weekly partitions, max 200
    
    print(f"Date range: {min_date} to {max_date}")
    print(f"Creating {num_partitions} range partitions")
    
    return df.repartitionByRange(num_partitions, col(date_column))

# Usage
df_range_partitioned = range_partition_by_date(df_optimized, "transaction_date")
```

---

## Broadcast Variables and Joins

### Smart Broadcast Join Optimization

```python
def optimize_broadcast_joins(large_df, small_df, join_key, size_threshold_mb=100):
    """Intelligently decide whether to use broadcast join"""
    
    # Estimate size of smaller DataFrame
    small_sample = small_df.sample(0.1)
    small_sample_count = small_sample.count()
    
    if small_sample_count == 0:
        estimated_size_mb = 0
    else:
        # Rough estimation (adjust based on your data)
        estimated_size_mb = (small_sample_count / 0.1) * 0.001  # Very rough estimate
    
    print(f"Estimated size of smaller DataFrame: {estimated_size_mb:.2f} MB")
    
    if estimated_size_mb <= size_threshold_mb:
        print("Using broadcast join")
        return large_df.join(broadcast(small_df), join_key)
    else:
        print("Using sort-merge join")
        return large_df.join(small_df, join_key)

# Configure broadcast threshold
def configure_broadcast_settings(spark_session, threshold_mb=100):
    """Configure optimal broadcast settings"""
    
    # Set broadcast threshold
    spark_session.conf.set("spark.sql.autoBroadcastJoinThreshold", f"{threshold_mb}MB")
    
    # Enable adaptive query execution
    spark_session.conf.set("spark.sql.adaptive.enabled", "true")
    spark_session.conf.set("spark.sql.adaptive.localShuffleReader.enabled", "true")
    
    # Configure broadcast timeout
    spark_session.conf.set("spark.sql.broadcastTimeout", "600")  # 10 minutes
    
    print(f"Configured broadcast threshold: {threshold_mb}MB")

configure_broadcast_settings(spark, 200)
```

### Broadcast Variables for Lookup Tables

```python
class BroadcastManager:
    """Manage broadcast variables for lookup operations"""
    
    def __init__(self, spark_context):
        self.sc = spark_context
        self.broadcast_vars = {}
    
    def create_lookup_broadcast(self, df, key_col, value_col, name):
        """Create broadcast variable from DataFrame for lookup operations"""
        
        # Convert DataFrame to dictionary
        lookup_dict = df.select(key_col, value_col) \
                        .rdd \
                        .map(lambda row: (row[0], row[1])) \
                        .collectAsMap()
        
        # Create broadcast variable
        broadcast_var = self.sc.broadcast(lookup_dict)
        self.broadcast_vars[name] = broadcast_var
        
        print(f"Created broadcast lookup '{name}' with {len(lookup_dict)} entries")
        return broadcast_var
    
    def get_broadcast(self, name):
        """Get broadcast variable by name"""
        return self.broadcast_vars.get(name)
    
    def cleanup_broadcast(self, name):
        """Clean up specific broadcast variable"""
        if name in self.broadcast_vars:
            self.broadcast_vars[name].destroy()
            del self.broadcast_vars[name]
            print(f"Cleaned up broadcast variable '{name}'")
    
    def cleanup_all(self):
        """Clean up all broadcast variables"""
        for name in list(self.broadcast_vars.keys()):
            self.cleanup_broadcast(name)

# Usage example
broadcast_manager = BroadcastManager(spark.sparkContext)

# Create lookup table
country_lookup = spark.createDataFrame([
    ("US", "United States"),
    ("CA", "Canada"),
    ("GB", "United Kingdom")
], ["country_code", "country_name"])

# Create broadcast variable
country_broadcast = broadcast_manager.create_lookup_broadcast(
    country_lookup, "country_code", "country_name", "countries"
)

# Use in UDF
def lookup_country_name(country_code):
    country_dict = country_broadcast.value
    return country_dict.get(country_code, "Unknown")

lookup_country_udf = udf(lookup_country_name, StringType())

# Apply lookup
df_with_country = df.withColumn("country_name", lookup_country_udf(col("country_code")))
```

---

## Data Skew Handling

### Skew Detection and Analysis

```python
def detect_data_skew(df, partition_col, skew_threshold=10.0):
    """Detect data skew in partitioning column"""
    
    # Analyze distribution
    distribution = df.groupBy(partition_col) \
                     .agg(count("*").alias("record_count")) \
                     .orderBy(desc("record_count"))
    
    stats = distribution.agg(
        avg("record_count").alias("avg_count"),
        min("record_count").alias("min_count"),
        max("record_count").alias("max_count"),
        stddev("record_count").alias("stddev_count")
    ).collect()[0]
    
    skew_ratio = stats["max_count"] / stats["avg_count"] if stats["avg_count"] > 0 else 0
    
    print(f"\n=== Skew Analysis for column '{partition_col}' ===")
    print(f"Average records per key: {stats['avg_count']:.0f}")
    print(f"Max records per key: {stats['max_count']:,}")
    print(f"Min records per key: {stats['min_count']:,}")
    print(f"Standard deviation: {stats['stddev_count']:.0f}")
    print(f"Skew ratio (max/avg): {skew_ratio:.2f}")
    
    if skew_ratio > skew_threshold:
        print(f"⚠️  HIGH SKEW DETECTED! Ratio {skew_ratio:.2f} > threshold {skew_threshold}")
        
        # Show top skewed keys
        print("\nTop 10 most skewed keys:")
        distribution.show(10)
        
        return True, distribution
    else:
        print("✅ Skew within acceptable limits")
        return False, distribution

# Partition-level skew detection
def detect_partition_skew(df, skew_threshold=5.0):
    """Detect skew at partition level"""
    
    partition_counts = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
    
    if not partition_counts:
        return False
    
    avg_count = sum(partition_counts) / len(partition_counts)
    max_count = max(partition_counts)
    skew_ratio = max_count / avg_count if avg_count > 0 else 0
    
    print(f"\n=== Partition Skew Analysis ===")
    print(f"Average records per partition: {avg_count:.0f}")
    print(f"Max records per partition: {max_count:,}")
    print(f"Partition skew ratio: {skew_ratio:.2f}")
    
    if skew_ratio > skew_threshold:
        print(f"⚠️  PARTITION SKEW DETECTED! Ratio {skew_ratio:.2f} > threshold {skew_threshold}")
        return True
    else:
        print("✅ Partitions well balanced")
        return False

# Usage
has_skew, skew_distribution = detect_data_skew(df, "customer_id")
has_partition_skew = detect_partition_skew(df)
```

### Advanced Skew Mitigation Techniques

```python
def apply_salting_technique(df, skewed_column, salt_range=100, operation="aggregation"):
    """Apply salting technique to handle data skew"""
    
    print(f"Applying salting with range {salt_range} to column '{skewed_column}'")
    
    # Add salt to the skewed column
    salted_df = df.withColumn(
        "salted_key",
        concat(col(skewed_column), lit("_"), (rand() * salt_range).cast("int"))
    )
    
    if operation == "aggregation":
        # Perform operations on salted data
        intermediate_result = salted_df.groupBy("salted_key") \
                                     .agg(
                                         sum("amount").alias("total_amount"),
                                         count("*").alias("record_count"),
                                         avg("amount").alias("avg_amount")
                                     )
        
        # Remove salt and aggregate final results
        final_result = intermediate_result.withColumn(
            "original_key",
            regexp_replace(col("salted_key"), "_\\d+$", "")
        ).groupBy("original_key") \
         .agg(
             sum("total_amount").alias("final_total"),
             sum("record_count").alias("final_count"),
             avg("avg_amount").alias("final_avg")
         )
        
        return final_result
    
    elif operation == "join":
        # For joins, salt both sides
        return salted_df
    
    else:
        return salted_df

def handle_skewed_joins(large_df, small_df, join_key, skew_threshold=10.0):
    """Handle skewed joins using multiple strategies"""
    
    # Detect skew in join key
    has_skew, skew_dist = detect_data_skew(large_df, join_key, skew_threshold)
    
    if not has_skew:
        # No skew, use regular join
        return large_df.join(small_df, join_key)
    
    # Strategy 1: Broadcast join if small table is small enough
    small_count = small_df.count()
    if small_count < 10000:  # Configurable threshold
        print("Using broadcast join to handle skew")
        return large_df.join(broadcast(small_df), join_key)
    
    # Strategy 2: Skewed join with salting
    print("Using salted join to handle skew")
    
    # Get top skewed keys
    top_skewed = skew_dist.limit(10).select(join_key).rdd.map(lambda row: row[0]).collect()
    
    # Separate skewed and non-skewed data
    skewed_large = large_df.filter(col(join_key).isin(top_skewed))
    non_skewed_large = large_df.filter(~col(join_key).isin(top_skewed))
    
    skewed_small = small_df.filter(col(join_key).isin(top_skewed))
    non_skewed_small = small_df.filter(~col(join_key).isin(top_skewed))
    
    # Handle non-skewed data with regular join
    non_skewed_result = non_skewed_large.join(non_skewed_small, join_key)
    
    # Handle skewed data with broadcast (since we isolated hot keys)
    skewed_result = skewed_large.join(broadcast(skewed_small), join_key)
    
    # Union results
    return non_skewed_result.union(skewed_result)

# Usage
# For aggregations
result_salted = apply_salting_technique(skewed_df, "customer_id", salt_range=50)

# For joins
result_join = handle_skewed_joins(large_df, small_df, "customer_id")
```

### Custom Partitioner for Skew Handling

```python
def create_custom_partitioner(df, partition_column, num_partitions=200):
    """Create custom partitioner to handle skew"""
    
    # Analyze key distribution
    key_distribution = df.groupBy(partition_column) \
                         .agg(count("*").alias("count")) \
                         .orderBy(desc("count"))
    
    total_records = df.count()
    target_records_per_partition = total_records / num_partitions
    
    # Identify heavy keys that need special handling
    heavy_keys = key_distribution.filter(
        col("count") > target_records_per_partition * 2
    ).select(partition_column).rdd.map(lambda row: row[0]).collect()
    
    print(f"Found {len(heavy_keys)} heavy keys that need special partitioning")
    
    if heavy_keys:
        # Apply salting to heavy keys
        df_processed = df.withColumn(
            "partition_key",
            when(col(partition_column).isin(heavy_keys),
                 concat(col(partition_column), lit("_"), (rand() * 10).cast("int")))
            .otherwise(col(partition_column))
        )
        
        return df_processed.repartition(num_partitions, col("partition_key"))
    else:
        return df.repartition(num_partitions, col(partition_column))

# Usage
df_custom_partitioned = create_custom_partitioner(df, "customer_id", 200)
```

---

## Memory Management and Garbage Collection

### Memory Configuration Optimization

```python
def optimize_memory_settings(spark_session, executor_memory="4g", driver_memory="2g"):
    """Configure optimal memory settings"""
    
    # Memory configuration
    memory_configs = {
        "spark.executor.memory": executor_memory,
        "spark.driver.memory": driver_memory,
        "spark.executor.memoryFraction": "0.8",  # Deprecated in Spark 2.0+
        "spark.sql.execution.arrow.pyspark.enabled": "true",  # Arrow optimization
        "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
        
        # Storage memory management
        "spark.storage.memoryFraction": "0.5",  # Deprecated
        "spark.storage.unrollFraction": "0.2",  # Deprecated
        
        # New unified memory manager (Spark 2.0+)
        "spark.storage.storageFraction": "0.5",
        
        # Garbage collection tuning
        "spark.executor.extraJavaOptions": 
            "-XX:+UseG1GC -XX:+UnlockDiagnosticVMOptions -XX:+G1PrintRegionRememberSetInfo "
            "-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCApplicationStoppedTime",
        
        # Off-heap storage (if available)
        "spark.memory.offHeap.enabled": "false",  # Enable if you have off-heap memory
        "spark.memory.offHeap.size": "1g",
    }
    
    for key, value in memory_configs.items():
        try:
            spark_session.conf.set(key, value)
            print(f"Set {key}: {value}")
        except Exception as e:
            print(f"Could not set {key}: {e}")

# Monitor memory usage
def monitor_memory_usage(df, operation_name):
    """Monitor memory usage during operations"""
    
    # Get current storage level info
    print(f"\n=== Memory Usage: {operation_name} ===")
    
    # Force materialization and measure
    start_time = time.time()
    count = df.count()
    end_time = time.time()
    
    print(f"Operation completed: {count:,} records in {end_time - start_time:.2f}s")
    
    # Check cached tables
    cached_tables = spark.catalog.cacheTable.__doc__  # This is just for demo
    print("Currently cached tables:")
    for table_name in spark.catalog.listTables():
        if hasattr(table_name, 'isTemporary') and table_name.isTemporary:
            print(f"  - {table_name.name}")

# Usage
optimize_memory_settings(spark)
```

### Efficient Data Serialization

```python
def optimize_serialization(spark_session):
    """Configure optimal serialization settings"""
    
    serialization_configs = {
        # Use Kryo serializer (faster than Java serializer)
        "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
        "spark.kryo.unsafe": "true",
        "spark.kryo.optimize": "true",
        
        # Increase Kryo buffer sizes for large objects
        "spark.kryoserializer.buffer": "64k",
        "spark.kryoserializer.buffer.max": "64m",
        
        # Enable compression
        "spark.rdd.compress": "true",
        "spark.shuffle.compress": "true",
        "spark.shuffle.spill.compress": "true",
        "spark.broadcast.compress": "true",
        
        # Choose compression codec (snappy is fastest, lz4 is good balance)
        "spark.io.compression.codec": "snappy",  # or "lz4", "lzf"
    }
    
    for key, value in serialization_configs.items():
        spark_session.conf.set(key, value)
        print(f"Set {key}: {value}")

optimize_serialization(spark)
```

---

## Shuffle Optimization

### Minimize Shuffle Operations

```python
def optimize_shuffle_operations(spark_session):
    """Configure optimal shuffle settings"""
    
    shuffle_configs = {
        # Shuffle partitions (very important!)
        "spark.sql.shuffle.partitions": "200",  # Adjust based on data size
        
        # Shuffle behavior
        "spark.sql.adaptive.shuffle.targetPostShuffleInputSize": "64MB",
        "spark.sql.adaptive.skewJoin.skewedPartitionFactor": "5",
        "spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes": "64MB",
        
        # Shuffle compression
        "spark.shuffle.compress": "true",
        "spark.shuffle.spill.compress": "true",
        
        # Shuffle file consolidation
        "spark.shuffle.consolidateFiles": "true",
        
        # Network timeout
        "spark.network.timeout": "300s",
        "spark.shuffle.io.connectionTimeout": "300s",
    }
    
    for key, value in shuffle_configs.items():
        spark_session.conf.set(key, value)

def reduce_shuffle_with_bucketing(df1, df2, join_key, bucket_count=200):
    """Use bucketing to reduce shuffle in repeated joins"""
    
    # Write bucketed tables
    df1.write \
       .mode("overwrite") \
       .bucketBy(bucket_count, join_key) \
       .sortBy(join_key) \
       .saveAsTable("bucketed_table1")
    
    df2.write \
       .mode("overwrite") \
       .bucketBy(bucket_count, join_key) \
       .sortBy(join_key) \
       .saveAsTable("bucketed_table2")
    
    # Join bucketed tables (no shuffle required)
    bucketed_df1 = spark.table("bucketed_table1")
    bucketed_df2 = spark.table("bucketed_table2")
    
    return bucketed_df1.join(bucketed_df2, join_key)

def analyze_shuffle_impact(df, operation_description=""):
    """Analyze shuffle impact of operations"""
    
    print(f"\n=== Shuffle Analysis: {operation_description} ===")
    
    # Get execution plan
    explain_string = df._jdf.queryExecution().toString()
    
    # Count shuffle operations
    shuffle_count = explain_string.count("Exchange")
    sort_count = explain_string.count("Sort")
    
    print(f"Shuffle operations (Exchange): {shuffle_count}")
    print(f"Sort operations: {sort_count}")
    
    if shuffle_count > 0:
        print("⚠️  This operation involves data shuffling")
        print("Consider:")
        print("  - Pre-partitioning data by join/group keys")
        print("  - Using broadcast joins for small tables")
        print("  - Bucketing for repeated operations")
    else:
        print("✅ No shuffle operations detected")
    
    return shuffle_count

# Usage
optimize_shuffle_operations(spark)

# Analyze before optimization
shuffle_count = analyze_shuffle_impact(
    df.groupBy("customer_id").agg(sum("amount")), 
    "GroupBy operation"
)

# Use bucketing for repeated operations
bucketed_result = reduce_shuffle_with_bucketing(df1, df2, "customer_id")
```

---

## Query Optimization Techniques

### Predicate Pushdown and Column Pruning

```python
def optimize_query_execution(df, filters=None, selected_columns=None):
    """Apply query optimizations: predicate pushdown and column pruning"""
    
    optimized_df = df
    
    # Apply filters early (predicate pushdown)
    if filters:
        for filter_expr in filters:
            optimized_df = optimized_df.filter(filter_expr)
            print(f"Applied filter: {filter_expr}")
    
    # Select only needed columns (column pruning)
    if selected_columns:
        optimized_df = optimized_df.select(*selected_columns)
        print(f"Selected columns: {selected_columns}")
    
    return optimized_df

# Example: Reading with optimizations
def read_with_optimizations(file_path, filters=None, columns=None):
    """Read data with built-in optimizations"""
    
    reader = spark.read.parquet(file_path)
    
    # Apply schema if available (avoid schema inference)
    # reader = reader.schema(predefined_schema)
    
    df = reader.load()
    
    return optimize_query_execution(df, filters, columns)

# Usage
optimized_df = read_with_optimizations(
    "large_dataset.parquet",
    filters=[col("date") >= "2024-01-01", col("status") == "active"],
    columns=["customer_id", "amount", "date", "status"]
)

def optimize_joins_order(dfs_with_sizes):
    """Optimize join order based on data sizes"""
    
    # Sort by size (smallest first)
    sorted_dfs = sorted(dfs_with_sizes, key=lambda x: x[1])
    
    print("Optimized join order (smallest to largest):")
    for name, size in sorted_dfs:
        print(f"  {name}: {size:,} records")
    
    # Perform joins in optimal order
    result = sorted_dfs[0][0]  # Start with smallest
    
    for df_name, df_size in sorted_dfs[1:]:
        df = next(df for name, df in dfs_with_sizes if name == df_name and df.count() == df_size)
        result = result.join(df, "join_key")
        print(f"Joined {df_name}")
    
    return result

# Example usage
dfs_with_sizes = [
    ("customers", 1000000),
    ("orders", 5000000), 
    ("products", 10000),
    ("categories", 50)
]

# Convert to actual DataFrames (example)
df_list = [(name, spark.range(size).toDF("id").withColumn("join_key", col("id"))) 
           for name, size in dfs_with_sizes]

optimized_join_result = optimize_joins_order(df_list)
```

### Adaptive Query Execution (AQE)

```python
def configure_adaptive_query_execution(spark_session):
    """Configure Adaptive Query Execution for automatic optimization"""
    
    aqe_configs = {
        # Enable AQE
        "spark.sql.adaptive.enabled": "true",
        
        # Coalesce partitions
        "spark.sql.adaptive.coalescePartitions.enabled": "true",
        "spark.sql.adaptive.coalescePartitions.minPartitionNum": "1",
        "spark.sql.adaptive.advisoryPartitionSizeInBytes": "64MB",
        
        # Skew join optimization
        "spark.sql.adaptive.skewJoin.enabled": "true",
        "spark.sql.adaptive.skewJoin.skewedPartitionFactor": "5",
        "spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes": "64MB",
        
        # Dynamic join optimization
        "spark.sql.adaptive.localShuffleReader.enabled": "true",
        
        # Runtime filter pushdown
        "spark.sql.optimizer.runtime.bloomFilter.enabled": "true",
        "spark.sql.optimizer.runtime.bloomFilter.applicationSideScanSizeThreshold": "10GB",
    }
    
    for key, value in aqe_configs.items():
        spark_session.conf.set(key, value)
        print(f"AQE: Set {key}: {value}")

# Enable AQE optimizations
configure_adaptive_query_execution(spark)

def monitor_aqe_optimizations(df, description=""):
    """Monitor AQE optimizations applied"""
    
    print(f"\n=== AQE Monitoring: {description} ===")
    
    # Force execution to see AQE in action
    start_time = time.time()
    result = df.count()
    end_time = time.time()
    
    print(f"Query executed: {result:,} records in {end_time - start_time:.2f}s")
    
    # Check execution plan
    print("\nFinal execution plan:")
    df.explain(extended=False)
    
    return result

# Usage
aqe_result = monitor_aqe_optimizations(
    df.groupBy("category").agg(sum("amount")),
    "Aggregation with AQE"
)
```

---

## Storage Format Optimization

### Choosing Optimal File Formats

```python
def benchmark_file_formats(df, base_path="/tmp/format_test"):
    """Benchmark different file formats for read/write performance"""
    
    formats = {
        "parquet": {"compression": "snappy"},
        "delta": {"compression": "snappy"},
        "csv": {"compression": "gzip"},
        "json": {"compression": "gzip"}
    }
    
    results = {}
    
    for format_name, options in formats.items():
        print(f"\n=== Testing {format_name.upper()} format ===")
        
        # Write performance
        write_start = time.time()
        writer = df.write.mode("overwrite")
        
        if format_name == "delta":
            writer.format("delta").save(f"{base_path}/{format_name}")
        elif format_name in ["csv", "json"]:
            writer.option("compression", options["compression"]) \
                  .format(format_name) \
                  .option("header", "true") \
                  .save(f"{base_path}/{format_name}")
        else:  # parquet
            writer.option("compression", options["compression"]) \
                  .parquet(f"{base_path}/{format_name}")
        
        write_time = time.time() - write_start
        
        # Read performance
        read_start = time.time()
        if format_name == "delta":
            read_df = spark.read.format("delta").load(f"{base_path}/{format_name}")
        elif format_name == "csv":
            read_df = spark.read.option("header", "true") \
                                .option("inferSchema", "true") \
                                .csv(f"{base_path}/{format_name}")
        elif format_name == "json":
            read_df = spark.read.json(f"{base_path}/{format_name}")
        else:  # parquet
            read_df = spark.read.parquet(f"{base_path}/{format_name}")
        
        count = read_df.count()
        read_time = time.time() - read_start
        
        # File size (approximate)
        try:
            file_info = spark.read.format("binaryFile").load(f"{base_path}/{format_name}")
            total_size = file_info.agg(sum("length")).collect()[0][0] / (1024*1024)  # MB
        except:
            total_size = "N/A"
        
        results[format_name] = {
            "write_time": write_time,
            "read_time": read_time,
            "file_size_mb": total_size,
            "records": count
        }
        
        print(f"Write time: {write_time:.2f}s")
        print(f"Read time: {read_time:.2f}s")
        print(f"File size: {total_size} MB")
    
    # Summary
    print(f"\n=== FORMAT COMPARISON SUMMARY ===")
    print(f"{'Format':<10} {'Write(s)':<10} {'Read(s)':<10} {'Size(MB)':<12} {'Records':<10}")
    print("-" * 60)
    
    for format_name, metrics in results.items():
        print(f"{format_name:<10} {metrics['write_time']:<10.2f} "
              f"{metrics['read_time']:<10.2f} {metrics['file_size_mb']:<12} "
              f"{metrics['records']:<10,}")
    
    return results

# Usage
# format_results = benchmark_file_formats(sample_df)
```

### Partitioning Strategies for Storage

```python
def optimize_storage_partitioning(df, partition_columns, base_path, max_files_per_partition=1000):
    """Optimize storage partitioning strategy"""
    
    print(f"Analyzing partitioning strategy for columns: {partition_columns}")
    
    # Analyze partition cardinality
    for col_name in partition_columns:
        distinct_count = df.select(col_name).distinct().count()
        print(f"  {col_name}: {distinct_count} distinct values")
        
        if distinct_count > max_files_per_partition:
            print(f"  ⚠️  Warning: {col_name} has high cardinality ({distinct_count})")
            print(f"     Consider using a different partitioning strategy")
    
    # Check partition size distribution
    partition_stats = df.groupBy(*partition_columns).count()
    size_stats = partition_stats.agg(
        avg("count").alias("avg_size"),
        min("count").alias("min_size"),
        max("count").alias("max_size"),
        stddev("count").alias("stddev_size")
    ).collect()[0]
    
    print(f"\nPartition size distribution:")
    print(f"  Average: {size_stats['avg_size']:.0f} records")
    print(f"  Min: {size_stats['min_size']:,} records")
    print(f"  Max: {size_stats['max_size']:,} records")
    print(f"  Std Dev: {size_stats['stddev_size']:.0f}")
    
    skew_ratio = size_stats['max_size'] / size_stats['avg_size'] if size_stats['avg_size'] > 0 else 0
    
    if skew_ratio > 10:
        print(f"  ⚠️  High partition skew detected (ratio: {skew_ratio:.2f})")
        print(f"     Consider adding salt to partition keys")
    
    # Write with optimal settings
    df.write \
      .mode("overwrite") \
      .partitionBy(*partition_columns) \
      .option("maxRecordsPerFile", "1000000") \
      .parquet(base_path)
    
    print(f"\nData written to {base_path} with partitioning by {partition_columns}")

def dynamic_partition_pruning_demo(spark_session):
    """Demonstrate dynamic partition pruning benefits"""
    
    # Enable dynamic partition pruning
    spark_session.conf.set("spark.sql.optimizer.dynamicPartitionPruning.enabled", "true")
    spark_session.conf.set("spark.sql.optimizer.dynamicPartitionPruning.useStats", "true")
    
    print("Dynamic partition pruning enabled")
    
    # Example query that benefits from DPP
    query = """
    SELECT sales.*, products.category
    FROM sales 
    JOIN products ON sales.product_id = products.product_id
    WHERE products.category = 'Electronics'
    """
    
    print(f"Query with DPP: {query}")
    print("This will automatically prune partitions in 'sales' table based on 'products' filter")

# Usage
# optimize_storage_partitioning(df, ["year", "month"], "/data/partitioned_sales")
dynamic_partition_pruning_demo(spark)
```

---

## Monitoring and Profiling

### Performance Monitoring Framework

```python
import time
from contextlib import contextmanager

class SparkPerformanceMonitor:
    """Comprehensive performance monitoring for Spark operations"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.metrics = {}
    
    @contextmanager
    def monitor_operation(self, operation_name):
        """Context manager to monitor Spark operations"""
        
        print(f"\n🚀 Starting operation: {operation_name}")
        start_time = time.time()
        
        # Get initial metrics
        initial_metrics = self._get_spark_metrics()
        
        try:
            yield self
        finally:
            end_time = time.time()
            duration = end_time - start_time
            
            # Get final metrics
            final_metrics = self._get_spark_metrics()
            
            # Calculate deltas
            metrics_delta = self._calculate_metrics_delta(initial_metrics, final_metrics)
            
            # Store results
            self.metrics[operation_name] = {
                "duration": duration,
                "metrics_delta": metrics_delta,
                "timestamp": start_time
            }
            
            print(f"✅ Completed operation: {operation_name}")
            print(f"   Duration: {duration:.2f}s")
            self._print_metrics_delta(metrics_delta)
    
    def _get_spark_metrics(self):
        """Get current Spark metrics"""
        try:
            # Get executor metrics
            status = self.spark.sparkContext.statusTracker()
            executor_infos = status.getExecutorInfos()
            
            total_cores = sum(exec_info.totalCores for exec_info in executor_infos)
            total_memory = sum(exec_info.maxMemory for exec_info in executor_infos)
            
            return {
                "total_cores": total_cores,
                "total_memory": total_memory,
                "active_executors": len(executor_infos)
            }
        except Exception as e:
            print(f"Could not get metrics: {e}")
            return {}
    
    def _calculate_metrics_delta(self, initial, final):
        """Calculate difference in metrics"""
        delta = {}
        for key in initial:
            if key in final:
                delta[key] = final[key] - initial[key]
        return delta
    
    def _print_metrics_delta(self, delta):
        """Print metrics changes"""
        if delta:
            print("   Metrics changes:")
            for key, value in delta.items():
                if value != 0:
                    print(f"     {key}: {value:+}")
    
    def performance_summary(self):
        """Print performance summary of all monitored operations"""
        
        print(f"\n📊 PERFORMANCE SUMMARY")
        print("=" * 60)
        print(f"{'Operation':<30} {'Duration(s)':<12} {'Status':<10}")
        print("-" * 60)
        
        total_time = 0
        for op_name, metrics in self.metrics.items():
            duration = metrics["duration"]
            total_time += duration
            status = "✅ OK" if duration < 60 else "⚠️  SLOW"
            
            print(f"{op_name:<30} {duration:<12.2f} {status:<10}")
        
        print("-" * 60)
        print(f"{'TOTAL':<30} {total_time:<12.2f}")
        
        # Identify bottlenecks
        if self.metrics:
            slowest_op = max(self.metrics.items(), key=lambda x: x[1]["duration"])
            print(f"\n🐌 Slowest operation: {slowest_op[0]} ({slowest_op[1]['duration']:.2f}s)")

# Usage example
monitor = SparkPerformanceMonitor(spark)

# Monitor data loading
with monitor.monitor_operation("Data Loading"):
    df = spark.read.parquet("large_dataset.parquet")
    count = df.count()
    print(f"Loaded {count:,} records")

# Monitor aggregation
with monitor.monitor_operation("Aggregation"):
    result = df.groupBy("category").agg(sum("amount")).collect()
    print(f"Aggregated into {len(result)} groups")

# Monitor join operation
with monitor.monitor_operation("Join Operation"):
    lookup_df = spark.read.parquet("lookup_table.parquet")
    joined = df.join(lookup_df, "key")
    final_count = joined.count()
    print(f"Join result: {final_count:,} records")

# Print summary
monitor.performance_summary()
```

### Query Plan Analysis

```python
def analyze_query_plan(df, operation_name=""):
    """Analyze and optimize query execution plan"""
    
    print(f"\n🔍 QUERY PLAN ANALYSIS: {operation_name}")
    print("=" * 50)
    
    # Physical plan analysis
    print("📋 Physical Execution Plan:")
    df.explain(mode="extended")
    
    # Check for common performance issues
    plan_string = df._jdf.queryExecution().toString()
    
    issues = []
    
    # Check for expensive operations
    if "CartesianProduct" in plan_string:
        issues.append("❌ Cartesian product detected - may cause performance issues")
    
    if "BroadcastHashJoin" not in plan_string and "join" in plan_string.lower():
        issues.append("⚠️  No broadcast join detected - consider broadcasting smaller tables")
    
    if plan_string.count("Exchange") > 3:
        issues.append(f"⚠️  Multiple shuffle operations ({plan_string.count('Exchange')}) detected")
    
    if "Sort" in plan_string and "Exchange" in plan_string:
        issues.append("⚠️  Sort operation after shuffle - consider pre-sorting")
    
    # Check for optimization opportunities
    if "Filter" in plan_string and "Exchange" in plan_string:
        filter_pos = plan_string.find("Filter")
        exchange_pos = plan_string.find("Exchange")
        if filter_pos > exchange_pos:
            issues.append("💡 Consider pushing filters earlier in the pipeline")
    
    if issues:
        print("\n🚨 POTENTIAL ISSUES DETECTED:")
        for issue in issues:
            print(f"   {issue}")
    else:
        print("\n✅ No obvious performance issues detected")
    
    # Recommendations
    print(f"\n💡 OPTIMIZATION RECOMMENDATIONS:")
    print(f"   • Cache DataFrames that are used multiple times")
    print(f"   • Use broadcast joins for small lookup tables")
    print(f"   • Apply filters as early as possible")
    print(f"   • Consider bucketing for repeated join operations")
    print(f"   • Use columnar formats (Parquet, Delta) for better performance")

# Usage
analyze_query_plan(
    df.join(lookup_df, "key").groupBy("category").agg(sum("amount")),
    "Join and Aggregation"
)
```

---

## Production Performance Patterns

### Incremental Processing Optimization

```python
def optimize_incremental_processing(source_path, target_path, checkpoint_col="updated_at"):
    """Optimize incremental data processing"""
    
    print(f"🔄 Setting up incremental processing")
    print(f"   Source: {source_path}")
    print(f"   Target: {target_path}")
    print(f"   Checkpoint column: {checkpoint_col}")
    
    # Read checkpoint
    try:
        checkpoint_df = spark.read.parquet(f"{target_path}/_checkpoint")
        last_checkpoint = checkpoint_df.agg(max("checkpoint_value")).collect()[0][0]
        print(f"   Last checkpoint: {last_checkpoint}")
    except:
        last_checkpoint = "1900-01-01"
        print(f"   No checkpoint found, starting from: {last_checkpoint}")
    
    # Read incremental data
    incremental_df = spark.read.parquet(source_path) \
                              .filter(col(checkpoint_col) > last_checkpoint)
    
    new_records = incremental_df.count()
    print(f"   New records to process: {new_records:,}")
    
    if new_records == 0:
        print("   ✅ No new data to process")
        return None
    
    # Process incremental data
    processed_df = incremental_df.cache()  # Cache for reuse
    
    # Write processed data
    processed_df.write \
                .mode("append") \
                .partitionBy("year", "month") \
                .parquet(target_path)
    
    # Update checkpoint
    new_checkpoint = processed_df.agg(max(checkpoint_col)).collect()[0][0]
    checkpoint_data = spark.createDataFrame([(new_checkpoint,)], ["checkpoint_value"])
    
    checkpoint_data.write \
                   .mode("overwrite") \
                   .parquet(f"{target_path}/_checkpoint")
    
    print(f"   ✅ Processed {new_records:,} records")
    print(f"   ✅ Updated checkpoint to: {new_checkpoint}")
    
    processed_df.unpersist()  # Clean up cache
    return processed_df

# Usage
# incremental_result = optimize_incremental_processing(
#     "/data/source/transactions",
#     "/data/processed/transactions",
#     "transaction_timestamp"
# )
```

### Auto-scaling Configuration

```python
def configure_autoscaling_spark(spark_session, workload_type="balanced"):
    """Configure Spark for auto-scaling workloads"""
    
    workload_configs = {
        "memory_intensive": {
            "spark.executor.memory": "8g",
            "spark.executor.cores": "2",
            "spark.executor.instances": "10",
            "spark.sql.shuffle.partitions": "400",
            "spark.storage.storageFraction": "0.7"
        },
        "cpu_intensive": {
            "spark.executor.memory": "4g", 
            "spark.executor.cores": "4",
            "spark.executor.instances": "20",
            "spark.sql.shuffle.partitions": "800",
            "spark.storage.storageFraction": "0.3"
        },
        "balanced": {
            "spark.executor.memory": "6g",
            "spark.executor.cores": "3", 
            "spark.executor.instances": "15",
            "spark.sql.shuffle.partitions": "600",
            "spark.storage.storageFraction": "0.5"
        }
    }
    
    config = workload_configs.get(workload_type, workload_configs["balanced"])
    
    # Apply configuration
    for key, value in config.items():
        spark_session.conf.set(key, value)
        print(f"Set {key}: {value}")
    
    # Enable dynamic allocation if available
    dynamic_allocation_configs = {
        "spark.dynamicAllocation.enabled": "true",
        "spark.dynamicAllocation.minExecutors": "2",
        "spark.dynamicAllocation.maxExecutors": "50",
        "spark.dynamicAllocation.initialExecutors": config["spark.executor.instances"],
        "spark.dynamicAllocation.executorIdleTimeout": "60s",
        "spark.dynamicAllocation.schedulerBacklogTimeout": "10s"
    }
    
    for key, value in dynamic_allocation_configs.items():
        try:
            spark_session.conf.set(key, value)
            print(f"Dynamic allocation: {key}: {value}")
        except Exception as e:
            print(f"Could not set {key}: {e}")

# Configure for different workload types
configure_autoscaling_spark(spark, "memory_intensive")
```

---

## Related Notes

- [[PySpark Core Concepts]] - Foundation concepts that these optimizations build upon
- [[PySpark Data Operations]] - Operations that benefit from these optimizations
- [[PySpark Window Functions]] - Advanced operations requiring performance tuning
- [[PySpark Production Engineering]] - Production deployment considerations

---

## Tags

#pyspark #performance #optimization #caching #partitioning #skew #memory #shuffle #monitoring #production

---

_Last Updated: 2024-08-20_