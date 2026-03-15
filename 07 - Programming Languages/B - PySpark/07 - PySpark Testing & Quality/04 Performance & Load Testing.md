
#pyspark #testing #performance #benchmarking #scalability #memory-testing #load-testing #optimization #resource-monitoring #throughput #latency


**Performance testing** in PySpark is fundamentally different from traditional application performance testing. You're not just testing CPU and memory usage - you're testing distributed computing patterns, data locality, serialization overhead, and cluster resource utilization. Poor performance in PySpark often stems from inefficient data distribution, excessive shuffling, or suboptimal join strategies rather than algorithmic complexity.

### Benchmark Testing Framework

**Performance Testing Strategy:** The key to effective PySpark performance testing is establishing baselines and understanding the performance characteristics of different operations. Unlike web application testing where you measure response times, PySpark performance testing focuses on throughput, resource utilization, and scalability patterns.

**Performance Testing Utilities**

```python
import time
from typing import Callable, Any, Dict
from contextlib import contextmanager
import psutil
from pyspark.sql import DataFrame

class PerformanceTester:
    """Performance testing utilities for PySpark operations
    
    Performance Testing Philosophy:
    1. Baseline Establishment: Know normal performance before optimizing
    2. Comparative Analysis: Test different approaches to same problem
    3. Resource Monitoring: Track CPU, memory, network, and disk usage
    4. Scalability Testing: Understand how performance changes with data size
    5. Regression Detection: Catch performance degradations early
    
    What Makes PySpark Performance Different:
    - Network I/O often dominates (shuffles, broadcasts)
    - Memory usage patterns are complex (caching, spilling)
    - Parallelism doesn't always mean better performance
    - Small changes can have massive performance impacts
    """
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.baseline_metrics = {}
    
    @staticmethod
    @contextmanager
    def timer():
        """Context manager for timing operations
        
        Usage Pattern:
        with PerformanceTester.timer() as timer:
            result = expensive_operation()
        print(f"Operation took {timer.elapsed_time:.2f} seconds")
        """
        start_time = time.time()
        timer_obj = type('Timer', (), {})()
        yield timer_obj
        end_time = time.time()
        timer_obj.elapsed_time = end_time - start_time

    @staticmethod
    def benchmark_transformation(df: DataFrame, transformation_func: Callable, 
                                iterations: int = 3, cache_input: bool = True) -> Dict[str, float]:
        """Benchmark a transformation function
        
        Parameters:
        - df: Input DataFrame 
        - transformation_func: Function that takes DataFrame and returns DataFrame
        - iterations: Number of test runs (more iterations = more reliable results)
        - cache_input: Whether to cache input DataFrame (eliminates upstream computation noise)
        
        Returns:
        - Dictionary with timing statistics
        
        Why Multiple Iterations:
        - First run often slower due to JVM warmup and optimization
        - Subsequent runs show steady-state performance
        - Statistical analysis requires multiple samples
        """
        if cache_input:
            df.cache()
            df.count()  # Force caching
        
        execution_times = []
        
        for i in range(iterations):
            # Clear any previous caching of results
            if hasattr(transformation_func, '_clear_cache'):
                transformation_func._clear_cache()
            
            start_time = time.time()
            result = transformation_func(df)
            
            # Force execution - transformations are lazy!
            if hasattr(result, 'count'):
                record_count = result.count()
            else:
                record_count = len(result) if hasattr(result, '__len__') else None
            
            end_time = time.time()
            execution_time = end_time - start_time
            execution_times.append(execution_time)
            
            # Log progress for long-running tests
            if execution_time > 10:  # More than 10 seconds
                print(f"Iteration {i+1}/{iterations} completed in {execution_time:.2f}s")
        
        return {
            "min_time": min(execution_times),
            "max_time": max(execution_times),
            "avg_time": sum(execution_times) / len(execution_times),
            "median_time": sorted(execution_times)[len(execution_times)//2],
            "total_time": sum(execution_times),
            "std_deviation": (sum((t - sum(execution_times)/len(execution_times))**2 for t in execution_times) / len(execution_times))**0.5,
            "record_count": record_count,
            "records_per_second": record_count / (sum(execution_times) / len(execution_times)) if record_count else None
        }
    
    def compare_strategies(self, df: DataFrame, strategies: Dict[str, Callable]) -> Dict[str, Dict]:
        """Compare multiple implementation strategies
        
        Usage:
        strategies = {
            "broadcast_join": lambda df: df.join(broadcast(small_df), "key"),
            "regular_join": lambda df: df.join(small_df, "key"),
            "bucketed_join": lambda df: df.join(bucketed_df, "key")
        }
        results = tester.compare_strategies(large_df, strategies)
        """
        results = {}
        
        for strategy_name, strategy_func in strategies.items():
            print(f"Testing strategy: {strategy_name}")
            results[strategy_name] = self.benchmark_transformation(df, strategy_func)
            
            # Add relative performance metrics
            if len(results) > 1:
                baseline_time = list(results.values())[0]["avg_time"]
                current_time = results[strategy_name]["avg_time"]
                results[strategy_name]["relative_performance"] = baseline_time / current_time
                results[strategy_name]["speedup"] = f"{baseline_time / current_time:.2f}x"
        
        return results

def test_join_performance_strategies(spark):
    """Test join operation performance with different strategies
    
    Join Performance Factors:
    1. Data Size: Small table vs large table join patterns
    2. Join Strategy: Broadcast vs shuffle vs bucket joins
    3. Data Distribution: Even vs skewed key distributions
    4. Memory Settings: Impact of executor memory on join performance
    
    Real-world Application:
    - Choose optimal join strategy based on data characteristics
    - Detect when data growth invalidates previous optimizations
    - Validate that performance optimizations actually work
    """
    # Create realistically sized test datasets
    large_df = spark.range(1000000).select(
        col("id").alias("user_id"),
        (rand() * 100).cast("int").alias("score"),
        concat(lit("user_"), col("id")).alias("username")
    )
    
    # Small lookup table (perfect for broadcast join)
    small_df = spark.range(1000).select(
        col("id").alias("user_id"),
        concat(lit("type_"), (col("id") % 10)).alias("user_type"),
        (rand() * 1000).cast("int").alias("priority")
    )
    
    # Medium table (broadcast vs shuffle decision point)
    medium_df = spark.range(100000).select(
        col("id").alias("user_id"),
        concat(lit("region_"), (col("id") % 50)).alias("region")
    )
    
    tester = PerformanceTester(spark)
    
    # Test different join strategies
    join_strategies = {
        "broadcast_small": lambda df: df.join(broadcast(small_df), "user_id"),
        "regular_small": lambda df: df.join(small_df, "user_id"),
        "broadcast_medium": lambda df: df.join(broadcast(medium_df), "user_id"),
        "regular_medium": lambda df: df.join(medium_df, "user_id")
    }
    
    # Benchmark all strategies
    results = tester.compare_strategies(large_df, join_strategies)
    
    # Performance assertions based on expected patterns
    broadcast_small_time = results["broadcast_small"]["avg_time"]
    regular_small_time = results["regular_small"]["avg_time"]
    
    # Broadcast should be faster for small tables
    assert broadcast_small_time < regular_small_time, \
        f"Broadcast join ({broadcast_small_time:.2f}s) should be faster than regular join ({regular_small_time:.2f}s) for small tables"
    
    # Test resource efficiency 
    broadcast_medium_time = results["broadcast_medium"]["avg_time"]
    regular_medium_time = results["regular_medium"]["avg_time"]
    
    # For medium tables, broadcast might be slower due to memory pressure
    if broadcast_medium_time > regular_medium_time * 1.5:
        print(f"WARNING: Broadcast join is {broadcast_medium_time/regular_medium_time:.1f}x slower for medium table")
    
    # Print performance summary
    print("\nJoin Performance Summary:")
    for strategy, metrics in results.items():
        print(f"{strategy}: {metrics['avg_time']:.2f}s ({metrics.get('speedup', '1.0x')})")

def test_aggregation_performance_patterns(spark):
    """Test aggregation performance with different data patterns
    
    Aggregation Performance Factors:
    1. Cardinality: High vs low cardinality grouping keys
    2. Data Skew: Even vs skewed distributions
    3. Aggregation Complexity: Simple sum vs complex statistical functions
    4. Memory Pressure: Large groups that don't fit in memory
    """
    
    # High cardinality data (many small groups)
    high_cardinality_df = spark.range(1000000).select(
        col("id").alias("group_key"),  # 1M unique groups
        (rand() * 1000).alias("value1"),
        (rand() * 1000).alias("value2")
    )
    
    # Low cardinality data (few large groups)
    low_cardinality_df = spark.range(1000000).select(
        (col("id") % 100).alias("group_key"),  # 100 groups of 10K records each
        (rand() * 1000).alias("value1"),
        (rand() * 1000).alias("value2")
    )
    
    # Skewed data (80/20 distribution)
    skewed_df = spark.range(1000000).select(
        F.when(rand() < 0.8, lit(1))  # 80% of data in group 1
         .otherwise((col("id") % 999) + 2)  # 20% spread across 999 other groups
         .alias("group_key"),
        (rand() * 1000).alias("value1"),
        (rand() * 1000).alias("value2")
    )
    
    def simple_aggregation(df):
        """Simple aggregation - should be fast"""
        return df.groupBy("group_key").agg(F.sum("value1").alias("sum_value1"))
    
    def complex_aggregation(df):
        """Complex aggregation - more CPU intensive"""
        return df.groupBy("group_key").agg(
            F.sum("value1").alias("sum_value1"),
            F.avg("value2").alias("avg_value2"),
            F.stddev("value1").alias("stddev_value1"),
            F.count("*").alias("record_count"),
            F.min("value1").alias("min_value1"),
            F.max("value2").alias("max_value2")
        )
    
    tester = PerformanceTester(spark)
    
    # Test different data patterns with simple aggregation
    data_pattern_results = {}
    for pattern_name, df in [("high_cardinality", high_cardinality_df), 
                           ("low_cardinality", low_cardinality_df),
                           ("skewed", skewed_df)]:
        print(f"Testing {pattern_name} data pattern...")
        data_pattern_results[pattern_name] = tester.benchmark_transformation(df, simple_aggregation)
    
    # Test aggregation complexity with low cardinality data
    complexity_results = {}
    for complexity_name, agg_func in [("simple", simple_aggregation), 
                                    ("complex", complex_aggregation)]:
        print(f"Testing {complexity_name} aggregation...")
        complexity_results[complexity_name] = tester.benchmark_transformation(low_cardinality_df, agg_func)
    
    # Performance expectations and assertions
    high_card_time = data_pattern_results["high_cardinality"]["avg_time"]
    low_card_time = data_pattern_results["low_cardinality"]["avg_time"]
    skewed_time = data_pattern_results["skewed"]["avg_time"]
    
    # Low cardinality should generally be faster (fewer groups to track)
    # But this can vary based on cluster configuration
    print(f"\nAggregation Performance by Data Pattern:")
    print(f"High cardinality: {high_card_time:.2f}s")
    print(f"Low cardinality: {low_card_time:.2f}s") 
    print(f"Skewed data: {skewed_time:.2f}s")
    
    # Complex aggregations should be slower
    simple_time = complexity_results["simple"]["avg_time"]
    complex_time = complexity_results["complex"]["avg_time"]
    
    assert complex_time > simple_time, \
        f"Complex aggregation ({complex_time:.2f}s) should be slower than simple ({simple_time:.2f}s)"
    
    complexity_overhead = complex_time / simple_time
    print(f"\nComplexity overhead: {complexity_overhead:.1f}x")
    
    # If complexity overhead is > 5x, might indicate inefficient query planning
    if complexity_overhead > 5:
        print("WARNING: High complexity overhead detected - consider query optimization")

def test_caching_performance_impact(spark):
    """Test the performance impact of caching strategies
    
    Caching Considerations:
    1. Cache vs Recompute Trade-off: When is caching worth the memory?
    2. Storage Levels: Memory vs disk vs serialized vs deserialized
    3. Cache Eviction: What happens when cache fills up?
    4. Iterative Algorithms: Caching intermediate results
    """
    
    # Create expensive-to-compute dataset
    base_df = spark.range(500000).select(
        col("id"),
        (rand() * 1000).alias("value"),
        concat(lit("category_"), (col("id") % 100)).alias("category")
    )
    
    # Expensive transformation that we'll reuse
    def expensive_transformation(df):
        """Simulate expensive transformation with multiple operations"""
        return df.withColumn("computed_value", 
                           F.sqrt(col("value")) * F.sin(col("value")) + F.cos(col("value"))
                  ).withColumn("value_rank",
                           F.row_number().over(Window.orderBy(col("value")))
                  ).filter(col("computed_value") > 10)
    
    expensive_df = expensive_transformation(base_df)
    
    tester = PerformanceTester(spark)
    
    # Test 1: No caching - recompute every time
    def multiple_operations_no_cache(df):
        result1 = df.groupBy("category").count()
        result2 = df.agg(F.avg("computed_value"))
        result3 = df.filter(col("value_rank") < 1000).count()
        return result1.count() + result2.collect()[0][0] + result3
    
    no_cache_results = tester.benchmark_transformation(expensive_df, multiple_operations_no_cache, iterations=2)
    
    # Test 2: With caching - cache expensive intermediate result
    expensive_df.cache()
    expensive_df.count()  # Force caching
    
    cached_results = tester.benchmark_transformation(expensive_df, multiple_operations_no_cache, iterations=2)
    
    # Test 3: Different storage levels
    from pyspark import StorageLevel
    
    expensive_df.unpersist()  # Clear previous cache
    
    # Test memory-only vs memory-and-disk
    storage_level_results = {}
    
    for level_name, storage_level in [
        ("memory_only", StorageLevel.MEMORY_ONLY),
        ("memory_and_disk", StorageLevel.MEMORY_AND_DISK),
        ("disk_only", StorageLevel.DISK_ONLY)
    ]:
        expensive_df.unpersist()
        expensive_df.persist(storage_level)
        expensive_df.count()  # Force caching
        
        results = tester.benchmark_transformation(expensive_df, multiple_operations_no_cache, iterations=1)
        storage_level_results[level_name] = results
        
        print(f"{level_name}: {results['avg_time']:.2f}s")
    
    # Performance assertions
    cache_speedup = no_cache_results["avg_time"] / cached_results["avg_time"]
    
    print(f"\nCaching Performance Analysis:")
    print(f"No cache: {no_cache_results['avg_time']:.2f}s")
    print(f"With cache: {cached_results['avg_time']:.2f}s")
    print(f"Speedup: {cache_speedup:.1f}x")
    
    # Caching should provide speedup for multiple operations
    assert cache_speedup > 1.5, f"Caching should provide significant speedup, got {cache_speedup:.1f}x"
    
    # Memory-only should be fastest (if it fits)
    memory_time = storage_level_results["memory_only"]["avg_time"]
    disk_time = storage_level_results["disk_only"]["avg_time"]
    
    if memory_time < disk_time:
        print(f"Memory storage is {disk_time/memory_time:.1f}x faster than disk")
    else:
        print("WARNING: Memory storage not faster than disk - possible memory pressure")
    
    # Clean up
    expensive_df.unpersist()
```

**Performance Testing Best Practices:**

- **Realistic Data Sizes**: Test with production-scale data volumes
- **Multiple Iterations**: Account for JVM warmup and optimization effects
- **Resource Monitoring**: Track CPU, memory, and network utilization
- **Comparative Analysis**: Always test multiple approaches to the same problem
- **Baseline Establishment**: Know current performance before optimizing
- **Regression Detection**: Integrate performance tests into CI/CD pipelines

### Memory Usage Testing

**Memory management** in PySpark is complex because you're dealing with both JVM heap memory and Python process memory, distributed across multiple nodes. Memory issues often manifest as performance degradation, job failures, or unpredictable behavior rather than simple out-of-memory errors.

**Memory Monitoring Utilities**

````python
import psutil
import gc
from pyspark.sql import SparkSession
from typing import Dict, Callable, Any

class MemoryMonitor:
    """Monitor memory usage during PySpark operations
    
    Memory Complexity in PySpark:
    1. JVM Memory: Spark driver and executor heap memory
    2. Python Memory: Python worker process memory for UDFs/pandas operations
    3. Off-heap Memory: Direct memory for network buffers, cached data
    4. Disk Spill: When memory is insufficient, Spark spills to disk
    
    Memory Issues to Watch For:
    - OutOfMemoryError: JVM heap exhausted
    - Memory pressure: Excessive garbage collection
    - Spilling: Performance degradation from disk I/O
    - Memory leaks: Gradual memory growth over time
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.initial_memory = self._get_memory_stats()
        self.measurements = []
    
    def _get_memory_stats(self) -> Dict[str, float]:
        """Get comprehensive memory statistics"""
        # System memory
        vm = psutil.virtual_memory()
        process = psutil.Process()
        
        # Python memory
        try:
            import resource
            max_rss = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / 1024 / 1024  # Convert to MB
        except:
            max_rss = 0
        
        stats = {
            # System level
            "system_total_gb": vm.total / (1024**3),
            "system_available_gb": vm.available / (1024**3),
            "system_used_gb": vm.used / (1024**3),
            "system_percent": vm.percent,
            
            # Process level
            "process_rss_gb": process.memory_info().rss / (1024**3),  # Resident Set Size
            "process_vms_gb": process.memory_info().vms / (1024**3),  # Virtual Memory Size
            "process_percent": process.memory_percent(),
            
            # Python level
            "python_objects": len(gc.get_objects()),
            "max_rss_gb": max_rss / 1024,
        }
        
        # Try to get Spark memory info if available
        try:
            spark_context = self.spark.sparkContext
            status_tracker = spark_context.statusTracker()
            
            # Executor memory info
            executor_infos = status_tracker.getExecutorInfos()
            
            total_executor_memory = 0
            total_memory_used = 0
            
            for executor in executor_infos:
                if hasattr(executor, 'maxMemory'):
                    total_executor_memory += executor.maxMemory
                if hasattr(executor, 'memoryUsed'):
                    total_memory_used += executor.memoryUsed
            
            stats.update({
                "spark_total_executor_memory_gb": total_executor_memory / (1024**3),
                "spark_used_executor_memory_gb": total_memory_used / (1024**3),
                "spark_executor_count": len(executor_infos)
            })
        except Exception as e:
            # Spark memory info not available
            pass
        
        return stats
    
    def get_current_usage(self) -> Dict[str, float]:
        """Get current memory usage statistics"""
        current_stats = self._get_memory_stats()
        
        # Calculate deltas from initial state
        for key in current_stats:
            if key in self.initial_memory:
                delta_key = f"delta_{key}"
                current_stats[delta_key] = current_stats[key] - self.initial_memory[key]
        
        return current_stats
    
    def monitor_operation(self, operation_func: Callable, df, 
                         sample_interval: float = 1.0) -> Dict[str, Any]:
        """Monitor memory usage during an operation
        
        Parameters:
        - operation_func: Function to monitor
        - df: Input DataFrame
        - sample_interval: How often to sample memory (seconds)
        
        Returns:
        - Dictionary with before/after stats and operation results
        """
        import threading
        import time
        
        # Memory sampling thread
        memory_samples = []
        sampling_active = threading.Event()
        sampling_active.set()
        
        def sample_memory():
            while sampling_active.is_set():
                memory_samples.append({
                    'timestamp': time.time(),
                    'stats': self._get_memory_stats()
                })
                time.sleep(sample_interval)
        
        # Start memory monitoring
        before_stats = self.get_current_usage()
        
        sampling_thread = threading.Thread(target=sample_memory)
        sampling_thread.start()
        
        # Execute operation
        start_time = time.time()
        try:
            result = operation_func(df)
            # Force execution for DataFrames
            if hasattr(result, 'count'):
                record_count = result.count()
            elif hasattr(result, 'collect'):
                collected_result = result.collect()
                record_count = len(collected_result)
            else:
                record_count = None
            
            operation_success = True
            operation_error = None
            
        except Exception as e:
            operation_success = False
            operation_error = str(e)
            record_count = None
            result = None
        
        end_time = time.time()
        
        # Stop memory monitoring
        sampling_active.clear()
        sampling_thread.join()
        
        after_stats = self.get_current_usage()
        
        # Analyze memory samples
        if memory_samples:
            peak_memory = max(sample['stats']['process_rss_gb'] for sample in memory_samples)
            min_memory = min(sample['stats']['process_rss_gb'] for sample in memory_samples)
            avg_memory = sum(sample['stats']['process_rss_gb'] for sample in memory_samples) / len(memory_samples)
        else:
            peak_memory = after_stats['process_rss_gb']
            min_memory = before_stats['process_rss_gb']
            avg_memory = (before_stats['process_rss_gb'] + after_stats['process_rss_gb']) / 2
        
        return {
            "before": before_stats,
            "after": after_stats,
            "memory_increase_gb": after_stats["process_rss_gb"] - before_stats["process_rss_gb"],
            "peak_memory_gb": peak_memory,
            "min_memory_gb": min_memory,
            "avg_memory_gb": avg_memory,
            "memory_samples": memory_samples,
            "execution_time": end_time - start_time,
            "operation_success": operation_success,
            "operation_error": operation_error,
            "record_count": record_count
        }

def test_memory_usage_aggregation(spark):
    """Test memory usage for large aggregations
    
    Memory Patterns in Aggregations:
    1. Hash tables for grouping keys grow with cardinality
    2. Large groups may spill to disk when memory is full
    3. Complex aggregation functions require more memory per group
    4. Skewed data can cause memory hotspots on specific executors
    """
    
    # Create large dataset that will challenge memory
    large_df = spark.range(10000000).select(
        (col("id") % 1000).alias("group_id"),  # 1000 groups
        (rand() * 1000).alias("value1"),
        (rand() * 1000).alias("value2"),
        concat(lit("data_"), col("id")).alias("text_data")  # Adds memory pressure
    )
    
    monitor = MemoryMonitor(spark)
    
    # Test memory usage for different aggregation complexities
    def simple_aggregation(df):
        """Simple aggregation - minimal memory per group"""
        return df.groupBy("group_id").agg(
            F.sum("value1").alias("sum_value1"),
            F.count("*").alias("count")
        )
    
    def complex_aggregation(df):
        """Complex aggregation - more memory per group"""
        return df.groupBy("group_id").agg(
            F.sum("value1").alias("sum_value1"),
            F.avg("value2").alias("avg_value2"),
            F.stddev("value1").alias("stddev_value1"),
            F.count("*").alias("count"),
            F.collect_list("text_data").alias("all_text")  # Memory intensive!
        )
    
    # Test simple aggregation memory usage
    simple_memory_stats = monitor.monitor_operation(simple_aggregation, large_df)
    
    # Test complex aggregation memory usage  
    complex_memory_stats = monitor.monitor_operation(complex_aggregation, large_df)
    
    # Memory usage assertions
    simple_memory_increase = simple_memory_stats["memory_increase_gb"]
    complex_memory_increase = complex_memory_stats["memory_increase_gb"]
    
    print(f"Simple aggregation memory increase: {simple_memory_increase:.2f} GB")
    print(f"Complex aggregation memory increase: {complex_memory_increase:.2f} GB")
    print(f"Simple peak memory: {simple_memory_stats['peak_memory_gb']:.2f} GB")
    print(f"Complex peak memory: {complex_memory_stats['peak_memory_gb']:.2f} GB")
    
    # Complex aggregation should use more memory
    assert complex_memory_increase > simple_memory_increase, \
        "Complex aggregation should use more memory than simple aggregation"
    
    # Check for memory pressure indicators
    if complex_memory_increase > 2.0:  # More than 2GB increase
        print("WARNING: High memory usage detected - consider data partitioning or optimization")
    
    # Verify operations completed successfully
    assert simple_memory_stats["operation_success"], f"Simple aggregation failed: {simple_memory_stats['operation_error']}"
    assert complex_memory_stats["operation_success"], f"Complex aggregation failed: {complex_memory_stats['operation_error']}"

def test_memory_leak_detection(spark):
    """Test for memory leaks in iterative operations
    
    Memory Leak Patterns:
    1. Cached DataFrames not properly unpersisted
    2. Accumulating intermediate results in driver
    3. Python object references not released
    4. Broadcast variables not cleaned up
    """
    
    monitor = MemoryMonitor(spark)
    initial_stats = monitor.get_current_usage()
    
    # Simulate iterative ML training or batch processing
    base_df = spark.range(100000).select(
        col("id"),
        (rand() * 100).alias("feature1"),
        (rand() * 100).alias("feature2")
    )
    
    memory_progression = []
    
    # Run multiple iterations to detect gradual memory growth
    for iteration in range(10):
        iteration_start_memory = monitor.get_current_usage()
        
        # Simulate processing that might leak memory
        processed_df = base_df.withColumn(
            f"computed_{iteration}",
            col("feature1") * col("feature2") + lit(iteration)
        ).cache()  # Potential leak if not unpersisted
        
        # Force computation
        result_count = processed_df.count()
        
        # Clean up (this should prevent leaks)
        processed_df.unpersist()
        
        iteration_end_memory = monitor.get_current_usage()
        
        memory_progression.append({
            "iteration": iteration,
            "memory_gb": iteration_end_memory["process_rss_gb"],
            "memory_delta": iteration_end_memory["process_rss_gb"] - iteration_start_memory["process_rss_gb"]
        })
        
        print(f"Iteration {iteration}: {iteration_end_memory['process_rss_gb']:.2f} GB")
    
    final_stats = monitor.get_current_usage()
    
    # Analyze memory trend
    first_iteration_memory = memory_progression[0]["memory_gb"]
    last_iteration_memory = memory_progression[-1]["memory_gb"]
    memory_growth = last_iteration_memory - first_iteration_memory
    
    print(f"\nMemory Leak Analysis:")
    print(f"Initial memory: {first_iteration_memory:.2f} GB")
    print(f"Final memory: {last_iteration_memory:.2f} GB")
    print(f"Total growth: {memory_growth:.2f} GB")
    
    # Check for concerning memory growth patterns
    if memory_growth > 0.5:  # More than 500MB growth
        print("WARNING: Potential memory leak detected")
        
        # Analyze growth pattern
        growth_per_iteration = [
            memory_progression[i]["memory_gb"] - memory_progression[i-1]["memory_gb"] 
            for i in range(1, len(memory_progression))
        ]
        
        avg_growth = sum(growth_per_iteration) / len(growth_per_iteration)
        print(f"Average growth per iteration: {avg_growth*1000:.1f} MB")
        
        if avg_growth > 0.05:### Data Completeness Tests

**Data completeness validation** is fundamental to data quality. Incomplete data can silently corrupt business metrics, cause ML models to perform poorly, and lead to incorrect business decisions. Unlike application bugs that usually fail fast, data quality issues often go unnoticed until they've propagated through multiple downstream systems.

**Comprehensive Completeness Validation**
```python
class DataQualityValidator:
    """Data quality validation utilities
    
    Completeness vs. Accuracy:
    - Completeness: Is the data present? (not null/empty)
    - Accuracy: Is the data correct? (valid values, correct format)
    
    Why Completeness Matters:
    1. Business Impact: Missing customer emails = lost revenue opportunities
    2. Analytical Impact: Missing data skews statistical analysis
    3. Compliance Impact: Some fields are legally required to be complete
    4. Operational Impact: Downstream systems may fail on missing data
    """
    
    @staticmethod
    def check_completeness(df, required_columns: list, threshold: float = 0.95):
        """Check data completeness for required columns
        
        Parameters:
        - df: DataFrame to validate
        - required_columns: Columns that must meet completeness threshold
        - threshold: Minimum completeness ratio (0.95 = 95% non-null)
        
        Returns:
        - Dictionary with completeness statistics for each column
        
        Why Use Thresholds Instead of 100%:
        - Real data is rarely 100% complete
        - Some missing data may be legitimate (e.g., optional fields)
        - Thresholds allow for data quality SLAs
        """
        total_rows = df.count()
        completeness_report = {}
        
        for column in required_columns:
            if column not in df.columns:
                completeness_report[column] = {"error": "Column not found"}
                continue
                
            # Count non-null values (Spark considers empty strings as non-null)
            non_null_count = df.filter(col(column).isNotNull()).count()
            
            # Count non-empty values (for string columns)
            non_empty_count = df.filter(
                col(column).isNotNull() & (col(column) != "")
            ).count()
            
            completeness_ratio = non_null_count / total_rows if total_rows > 0 else 0
            non_empty_ratio = non_empty_count / total_rows if total_rows > 0 else 0
            
            completeness_report[column] = {
                "completeness_ratio": completeness_ratio,
                "non_empty_ratio": non_empty_ratio,  # More strict for strings
                "passes_threshold": completeness_ratio >= threshold,
                "null_count": total_rows - non_null_count,
                "empty_count": total_rows - non_empty_count,
                "total_count": total_rows
            }
        
        return completeness_report
    
    @staticmethod
    def check_critical_completeness(df, critical_columns: list):
        """Check 100% completeness for business-critical columns
        
        Critical Columns Examples:
        - Primary keys (must never be null)
        - Foreign keys (referential integrity)
        - Legal compliance fields (customer consent, audit trails)
        - Financial amounts (revenue, costs)
        """
        issues = []
        
        for column in critical_columns:
            if column not in df.columns:
                issues.append(f"Critical column '{column}' missing from dataset")
                continue
            
            null_count = df.filter(col(column).isNull()).count()
            if null_count > 0:
                issues.append(f"Critical column '{column}' has {null_count} null values")
        
        return issues

def test_data_completeness(spark):
    """Test data completeness validation with realistic scenarios
    
    Test Scenarios:
    1. Columns with different completeness levels
    2. Missing columns (schema issues)
    3. Empty strings vs null values
    4. Threshold-based validation
    """
    # Create test data with varying completeness patterns
    test_data = [
        (1, "Alice", 25, "alice@email.com"),      # Complete row
        (2, "Bob", None, "bob@email.com"),        # Missing age
        (3, "Charlie", 35, None),                 # Missing email
        (4, None, 40, "dave@email.com"),          # Missing name
        (5, "", 28, ""),                          # Empty strings
        (6, "Frank", 32, "frank@email.com")       # Complete row
    ]
    
    schema = ["id", "name", "age", "email"]
    df = spark.createDataFrame(test_data, schema)
    
    # Validate completeness with different thresholds
    validator = DataQualityValidator()
    
    # Test with 80% threshold (should pass for most columns)
    report_80 = validator.check_completeness(df, ["id", "name", "age", "email"], 0.8)
    
    # Assertions for 80% threshold
    assert report_80["id"]["passes_threshold"] == True     # 100% complete (6/6)
    assert report_80["name"]["passes_threshold"] == True   # 83% complete (5/6) 
    assert report_80["age"]["passes_threshold"] == True    # 83% complete (5/6)
    assert report_80["email"]["passes_threshold"] == True  # 83% complete (5/6)
    
    # Test with 90% threshold (should fail for some columns) 
    report_90 = validator.check_completeness(df, ["id", "name", "age", "email"], 0.9)
    
    assert report_90["id"]["passes_threshold"] == True     # 100% >= 90%
    assert report_90["name"]["passes_threshold"] == False  # 83% < 90%
    assert report_90["age"]["passes_threshold"] == False   # 83% < 90%
    assert report_90["email"]["passes_threshold"] == False # 83% < 90%
    
    # Test critical column validation
    critical_issues = validator.check_critical_completeness(df, ["id"])
    assert len(critical_issues) == 0  # ID column is 100% complete
    
    critical_issues_strict = validator.check_critical_completeness(df, ["name", "age"])
    assert len(critical_issues_strict) == 2  # Both have nulls
    
    # Test string completeness (empty strings)
    name_report = report_80["name"]
    assert name_report["completeness_ratio"] > name_report["non_empty_ratio"]  # Empty string counted as null

def test_completeness_by_partition(spark):
    """Test completeness validation across data partitions
    
    Why Partition-Level Testing Matters:
    - Data quality issues often affect specific partitions (dates, regions, sources)
    - Helps identify systematic data collection problems
    - Enables targeted data quality remediation
    """
    # Create data with partition-specific quality issues
    partitioned_data = [
        # Good partition (2023-01-01)
        ("2023-01-01", 1, "Alice", "alice@email.com"),
        ("2023-01-01", 2, "Bob", "bob@email.com"),
        
        # Degraded partition (2023-01-02) - missing emails
        ("2023-01-02", 3, "Charlie", None),
        ("2023-01-02", 4, "David", None),
        
        # Bad partition (2023-01-03) - missing names and emails  
        ("2023-01-03", 5, None, None),
        ("2023-01-03", 6, None, None)
    ]
    
    schema = ["date", "id", "name", "email"]
    df = spark.createDataFrame(partitioned_data, schema)
    
    # Check completeness by partition
    validator = DataQualityValidator()
    
    for date in ["2023-01-01", "2023-01-02", "2023-01-03"]:
        partition_df = df.filter(col("date") == date)
        report = validator.check_completeness(partition_df, ["name", "email"], 0.8)
        
        if date == "2023-01-01":
            # Good partition should pass
            assert report["name"]["passes_threshold"] == True
            assert report["email"]["passes_threshold"] == True
        elif date == "2023-01-02":
            # Degraded partition - names OK, emails fail
            assert report["name"]["passes_threshold"] == True
            assert report["email"]["passes_threshold"] == False
        else:  # 2023-01-03
            # Bad partition - both fail
            assert report["name"]["passes_threshold"] == False
            assert report["email"]["passes_threshold"] == False

def test_temporal_completeness_trends(spark):
    """Test completeness trends over time
    
    Business Scenario:
    - Data quality may degrade over time due to system changes
    - New data sources may have different completeness patterns
    - Seasonal effects may impact data collection
    """
    from datetime import datetime, timedelta
    
    # Generate time-series data with decreasing quality
    base_date = datetime(2023, 1, 1)
    time_data = []
    
    for day in range(30):  # 30 days of data
        current_date = base_date + timedelta(days=day)
        date_str = current_date.strftime("%Y-%m-%d")
        
        # Simulate degrading data quality over time
        completeness_rate = max(0.5, 1.0 - (day * 0.02))  # Starts at 100%, degrades 2% per day
        
        for record in range(10):  # 10 records per day
            record_id = day * 10 + record
            # Randomly null fields based on completeness rate
            name = f"User_{record_id}" if random.random() < completeness_rate else None
            email = f"user{record_id}@email.com" if random.random() < completeness_rate else None
            
            time_data.append((date_str, record_id, name, email))
    
    time_df = spark.createDataFrame(time_data, ["date", "id", "name", "email"])
    
    # Calculate completeness by week
    validator = DataQualityValidator()
    weekly_completeness = []
    
    for week in range(4):  # 4 weeks
        week_start = base_date + timedelta(days=week * 7)
        week_end = week_start + timedelta(days=6)
        
        week_df = time_df.filter(
            (col# PySpark Testing & Quality

#pyspark #testing #data-quality #performance #ci-cd #monitoring #fundamentals #data-engineering

## Overview
This comprehensive guide covers testing strategies and quality assurance practices for PySpark applications. From unit testing basics to production monitoring, these practices ensure reliable, maintainable, and high-quality data pipelines.

---

## Core Testing Philosophy for Distributed Systems

**Testing distributed systems** like PySpark requires fundamentally different approaches than traditional single-threaded applications. Unlike testing a simple Python function that runs predictably on one machine, PySpark operations are executed across multiple nodes with varying execution orders and resource constraints.

**Why Traditional Testing Approaches Fail:**
- **Non-deterministic execution order** across partitions - data in partition 1 might be processed before or after partition 2
- **Resource dependencies** on cluster configurations - tests might pass with 4GB executor memory but fail with 2GB
- **Data distribution complexity** across multiple nodes - data skew can cause some partitions to process much more data
- **Network and I/O failures** in distributed environments - temporary network issues can cause spurious test failures

**Real-world Example:**
```python
# This test might be flaky in distributed environments
def bad_test_example(spark):
    df = spark.range(1000).repartition(4)
    # Assumes specific partition ordering - WRONG!
    first_partition_data = df.mapPartitions(lambda x: [list(x)]).collect()[0]
    assert first_partition_data[0] == 0  # May fail randomly
````

### Key Testing Principles

**Deterministic Results**: Your tests should produce the same results whether you run them on a single-node cluster or a 100-node cluster. This means focusing on final outcomes rather than intermediate execution details.

**Resource Independence**: Tests shouldn't fail just because you're running with different memory settings or CPU counts. Design tests that work within reasonable resource constraints.

**Isolation**: Each test should clean up after itself and not leave data or configurations that affect subsequent tests. This is especially important with shared SparkContext instances.

**Realistic Data**: Use test data that reflects real-world scenarios including edge cases like null values, empty partitions, and data skew patterns that you'll encounter in production.
