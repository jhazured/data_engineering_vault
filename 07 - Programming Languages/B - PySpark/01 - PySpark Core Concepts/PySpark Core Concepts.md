
#pyspark #spark #core-concepts #fundamentals #data-engineering #big-data

## Overview

This note covers the fundamental concepts that form the foundation of PySpark knowledge. Understanding these concepts is crucial before diving into advanced operations and optimizations.

---

## What is PySpark?

**PySpark** is the Python API for Apache Spark, an open-source distributed computing system designed for large-scale data processing and analytics.

### Key Advantages over Traditional Python

- **Scalability**: Handle datasets from gigabytes to petabytes by distributing computation across clusters
- **High Performance**: 10-100x faster processing through in-memory computing and parallel execution
- **Fault Tolerance**: Automatic recovery from node failures using RDD lineage information
- **Integration**: Seamless integration with Hadoop ecosystem (HDFS, Hive, HBase) and cloud storage (S3, ADLS, GCS)
- **Unified Platform**: Single framework for batch processing, streaming, ML, and graph processing
- **Lazy Evaluation**: Optimizes entire execution pipeline before running any computation

### Traditional Python vs PySpark Comparison

```python
# Traditional Python (single-threaded)
import pandas as pd
df = pd.read_csv("large_file.csv")  # Loads entire file into memory
result = df.groupby("category").sum()  # Limited by single machine memory

# PySpark (distributed)
df = spark.read.csv("large_file.csv", header=True)  # Distributed across cluster
result = df.groupBy("category").sum()  # Processed in parallel across nodes
```

---

## SparkSession: The Entry Point

**SparkSession** is the entry point for all Spark functionality in PySpark 2.0+. It encapsulates SparkContext, SQLContext, and HiveContext into a single unified API.

### Creating a SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MyDataApplication") \
    .master("local[*]") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()
```

### SparkSession Responsibilities

- **DataFrame Operations**: Creating and manipulating DataFrames
- **SQL Interface**: Executing SQL queries on distributed data
- **Configuration Management**: Setting Spark properties and cluster configurations
- **Resource Management**: Managing SparkContext lifecycle and cluster resources
- **Catalog Access**: Interacting with metadata catalogs (Hive, Delta Lake)
- **UDF Registration**: Registering user-defined functions for SQL use

---

## Spark DAG (Directed Acyclic Graph)

A **DAG** in Spark represents the logical execution plan of transformations applied to RDDs or DataFrames.

### Key Characteristics

- **Directed**: Each node represents a transformation with dependencies flowing in one direction
- **Acyclic**: No loops or cycles exist, ensuring finite execution paths
- **Lazy**: Built during transformation definitions, executed only when actions are called

### DAG Optimization Features

The Catalyst optimizer applies several optimizations to the DAG:

- **Pipeline Optimization**: Combines narrow transformations into single stages
- **Predicate Pushdown**: Moves filter operations closer to data sources
- **Column Pruning**: Eliminates unnecessary columns early in the pipeline
- **Constant Folding**: Pre-computes constant expressions
- **Join Reordering**: Optimizes join order based on statistics

### DAG in Action Example

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

---

## RDDs vs DataFrames vs Datasets

Understanding when to use each data structure is crucial for optimal performance.

### Comparison Table

|Feature|RDD|DataFrame|Dataset|
|---|---|---|---|
|**API Level**|Low-level|High-level|High-level|
|**Schema**|No schema|Schema-aware|Schema + compile-time type safety|
|**Optimization**|Manual|Catalyst optimizer|Catalyst optimizer|
|**Type Safety**|Runtime|Runtime|Compile-time (Scala/Java only)|
|**Performance**|Slower|Fast|Fastest|
|**Memory Usage**|Higher|Lower (columnar)|Lower (columnar)|
|**Language Support**|All|All|Scala/Java only|

### When to Use Each

#### RDDs: Unstructured Data or Complex Transformations

```python
# Use RDDs when working with unstructured data
text_rdd = spark.sparkContext.textFile("logs.txt")
parsed_rdd = text_rdd.map(lambda line: parse_complex_format(line))
filtered_rdd = parsed_rdd.filter(lambda record: custom_business_logic(record))
```

**Use RDDs when:**

- Working with unstructured data (raw text, binary data)
- Need fine-grained control over data distribution
- Manipulating data in ways not expressible in SQL
- Working with data that doesn't have a natural schema

#### DataFrames: Structured Data with SQL-like Operations (Recommended)

```python
# Use DataFrames for structured/semi-structured data
df = spark.read.json("user_events.json")
result = df.select("user_id", "timestamp", "event_type") \
           .filter(df.timestamp > "2024-01-01") \
           .groupBy("event_type").count()
```

**Use DataFrames when:**

- Working with structured or semi-structured data
- Want automatic optimizations from Catalyst
- Need SQL-like operations
- Interoperability with other Spark components (ML, Streaming)

#### Datasets: Type Safety with Performance (Scala/Java Only)

Datasets provide compile-time type safety while maintaining the performance benefits of DataFrames. Only available in Scala and Java.

---

## Transformations vs Actions

Understanding the difference between transformations and actions is fundamental to understanding Spark's lazy evaluation.

### Transformations (Lazy Operations)

Transformations create new RDDs/DataFrames from existing ones but don't execute immediately.

**Characteristics:**

- Create new RDDs/DataFrames from existing ones
- Build computation graph (DAG) without executing
- Enable optimization across entire pipeline
- Return new DataFrame/RDD objects

**Common Transformations:**

```python
# All these are transformations - no computation happens yet
df_customers = spark.read.parquet("customers.parquet")
df_filtered = df_customers.filter(df_customers.age > 25)        # Transformation
df_selected = df_filtered.select("customer_id", "name", "age")  # Transformation
df_grouped = df_selected.groupBy("age").count()                 # Transformation
```

### Actions (Eager Operations)

Actions trigger execution of the entire transformation pipeline.

**Characteristics:**

- Trigger execution of entire transformation pipeline
- Return results to driver or write to external storage
- Force DAG optimization and execution
- Actual computation happens

**Common Actions:**

```python
# These actions trigger execution of all previous transformations
df_grouped.show()           # Display results - Action
count = df_grouped.count()  # Return count to driver - Action
df_grouped.write.parquet("age_distribution.parquet")  # Write to storage - Action
```

### Lazy Evaluation Benefits

- **Optimization**: Catalyst optimizer sees entire pipeline and applies optimizations
- **Efficiency**: Eliminates intermediate results that aren't needed
- **Resource Management**: Delays resource allocation until necessary
- **Pipeline Fusion**: Combines operations to minimize data movement

### Example: Lazy Evaluation in Practice

```python
# Build transformation pipeline (lazy)
df = spark.read.csv("large_dataset.csv", header=True)
filtered_df = df.filter(df.amount > 1000)
grouped_df = filtered_df.groupBy("category").sum("amount")

# At this point, no data has been read or processed!
# Spark has only built the execution plan

# Trigger execution with action
result = grouped_df.collect()  # NOW the entire pipeline executes
```

---

## Narrow vs Wide Transformations

Understanding transformation types helps predict performance characteristics and optimize workflows.

### Narrow Transformations

**Characteristics:**

- Each input partition contributes to at most one output partition
- No data shuffling across network
- Can be pipelined together for efficiency
- Executed in the same stage

**Examples:**

```python
# All these operations don't require shuffling
df_narrow = df.filter(col("amount") > 100) \           # Filter
              .select("customer_id", "amount", "date") \  # Select/Project
              .withColumn("tax", col("amount") * 0.1) \   # Map/Transform
              .union(other_df) \                          # Union
              .drop("temp_column")                        # Drop
```

### Wide Transformations

**Characteristics:**

- Input partitions contribute to multiple output partitions
- Require shuffling data across network
- Create stage boundaries in execution plan
- More expensive operations

**Examples:**

```python
# These operations require shuffling
df_wide = df.groupBy("customer_id").agg(sum("amount")) \  # GroupBy/Aggregation
            .join(other_df, "customer_id") \               # Join
            .orderBy("customer_id") \                      # Sort
            .distinct()                                    # Distinct
```

### Performance Implications

- **Narrow transformations** are fast and can be chained efficiently
- **Wide transformations** involve network shuffling and are more expensive
- Understanding this helps in optimizing query performance

---

## Key Takeaways

1. **SparkSession** is your entry point - configure it properly for your use case
2. **DataFrames** are recommended for most use cases due to Catalyst optimization
3. **Lazy evaluation** allows Spark to optimize entire pipelines before execution
4. **DAG optimization** happens automatically but understanding it helps write better code
5. **Narrow transformations** are fast; **wide transformations** require careful consideration
6. **Actions trigger execution** - be mindful of when and how often you call them

---

## Related Notes

- [[PySpark Data Operations]] - Building on these concepts for data manipulation
- [[PySpark Performance Optimization]] - Advanced optimization using these fundamentals
- [[PySpark Architecture & System Design]] - How these concepts fit into larger systems
- [[PySpark Functions & Methods]] - Specific functions that implement these concepts

---

## Tags

#pyspark #spark #fundamentals #dag #lazy-evaluation #dataframes #rdds #transformations #actions #core-concepts

---

_Last Updated: 2024-08-20_