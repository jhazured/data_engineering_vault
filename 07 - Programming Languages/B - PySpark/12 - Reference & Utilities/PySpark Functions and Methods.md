
## Table of Contents

1. [Data Transformation Functions]
    - 1.1 [map(), flatMap(), and mapPartitions()]
2. [User-Defined Functions (UDFs)]
    - 2.1 [UDF Implementation and Performance]
3. [Data Retrieval and Display Methods]
    - 3.1 [collect(), take(), first(), and show()]
4. [DataFrame API vs SQL Functions]
    - 4.1 [Built-in Functions vs SQL Functions]
5. [Aggregation and Grouping Operations]
    - 5.1 [groupBy(), agg(), and pivot()]
6. [Column Operations and Transformations]
    - 6.1 [select(), withColumn(), and withColumnRenamed()]
7. [Data Filtering Techniques]
    - 7.1 [filter(), where(), and Condition Chaining]
8. [Complex Data Structures]
    - 8.1 [Nested Data Handling with explode(), struct(), and Arrays]
9. [Data Redistribution and Partitioning]
    - 9.1 [coalesce() vs repartition()]
    - 9.2 [Custom Partitioning Logic]
10. [Summary and Best Practices]

---

## 1. Data Transformation Functions

### 1.1 map(), flatMap(), and mapPartitions()

**Q: What are the differences between `map()`, `flatMap()`, and `mapPartitions()` in PySpark? When would you use each?**

**Answer:**

**map():**

- Transforms each element in an RDD using a function
- Returns one output element for each input element (1:1 mapping)
- Applied element by element

```python
rdd = sc.parallelize([1, 2, 3, 4])
mapped_rdd = rdd.map(lambda x: x * 2)  # [2, 4, 6, 8]
```

**flatMap():**

- Transforms each element and flattens the results
- Can return 0, 1, or multiple elements for each input (1:N mapping)
- Useful for splitting strings, expanding collections

```python
rdd = sc.parallelize(["hello world", "spark python"])
flat_rdd = rdd.flatMap(lambda x: x.split(" "))  # ["hello", "world", "spark", "python"]
```

**mapPartitions():**

- Applies function to entire partitions rather than individual elements
- More efficient for operations requiring setup/teardown (like database connections)
- Function receives an iterator of partition data

```python
def process_partition(iterator):
    # Setup code (e.g., database connection)
    result = []
    for record in iterator:
        # Process each record
        result.append(record * 2)
    # Cleanup code
    return iter(result)

rdd = sc.parallelize([1, 2, 3, 4], 2)
partition_rdd = rdd.mapPartitions(process_partition)
```

**When to use:**

- **map()**: Simple element-wise transformations
- **flatMap()**: When you need to flatten nested structures or split elements
- **mapPartitions()**: When you need to amortize setup costs across multiple elements or need partition-level operations

---

## 2. User-Defined Functions (UDFs)

### 2.1 UDF Implementation and Performance

**Q: How do you use User Defined Functions (UDFs) in PySpark? What are the performance implications and alternatives?**

**Answer:**

**Using UDFs:**

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType, IntegerType

# Define UDF
def categorize_age(age):
    if age < 18:
        return "Minor"
    elif age < 65:
        return "Adult"
    else:
        return "Senior"

# Register UDF
age_category_udf = udf(categorize_age, StringType())

# Use in DataFrame
df = spark.createDataFrame([(25,), (15,), (70,)], ["age"])
result = df.withColumn("category", age_category_udf(df.age))
```

**Performance Implications:**

- **Serialization overhead**: Data must be serialized/deserialized between JVM and Python
- **No Catalyst optimization**: UDFs are black boxes to Spark's optimizer
- **Memory overhead**: Additional memory for Python processes
- **Slower execution**: Can be 10-100x slower than built-in functions

**Alternatives:**

**Built-in functions:** Always prefer when available

```python
from pyspark.sql.functions import when, col

df.withColumn("category", 
    when(col("age") < 18, "Minor")
    .when(col("age") < 65, "Adult")
    .otherwise("Senior"))
```

**Pandas UDFs (Vectorized UDFs):** Better performance for complex operations

```python
from pyspark.sql.functions import pandas_udf
import pandas as pd

@pandas_udf(returnType=StringType())
def vectorized_categorize(ages: pd.Series) -> pd.Series:
    return ages.apply(lambda x: "Minor" if x < 18 else "Adult" if x < 65 else "Senior")
```

**SQL expressions:** Use native SQL functions when possible

---

## 3. Data Retrieval and Display Methods

### 3.1 collect(), take(), first(), and show()

**Q: Explain the difference between `collect()`, `take()`, `first()`, and `show()` methods. When is each appropriate?**

**Answer:**

**collect():**

- Returns all DataFrame rows as a list of Row objects to the driver
- Brings entire dataset to driver memory

```python
all_data = df.collect()  # Returns List[Row]
```

**take(n):**

- Returns first n rows as a list of Row objects to the driver
- More memory-efficient than collect() for large datasets

```python
first_10 = df.take(10)  # Returns List[Row] with 10 elements
```

**first():**

- Returns the first row as a Row object
- Equivalent to `take(1)[0]` but more convenient

```python
first_row = df.first()  # Returns single Row object
```

**show(n=20, truncate=True):**

- Displays n rows in tabular format (doesn't return data)
- For exploration and debugging only

```python
df.show(5)  # Prints table to console
```

**When to use:**

- **collect()**: Only for small datasets that fit in driver memory, final results
- **take(n)**: When you need to examine a few rows programmatically
- **first()**: When you need just one row (checking if DataFrame is empty, getting sample)
- **show()**: For data exploration, debugging, and quick inspection during development

⚠️ **Warning:** Never use `collect()` on large datasets as it can cause OutOfMemory errors.

---

## 4. DataFrame API vs SQL Functions

### 4.1 Built-in Functions vs SQL Functions

**Q: What are DataFrame built-in functions vs SQL functions in PySpark? Provide examples of when to use each approach.**

**Answer:**

**DataFrame Built-in Functions:**

- Functions from `pyspark.sql.functions` module
- Programmatic approach with method chaining
- Better for complex data pipelines and when building reusable code

```python
from pyspark.sql.functions import col, sum, avg, max, when, regexp_replace

result = df.select(
    col("name"),
    when(col("age") > 18, "Adult").otherwise("Minor").alias("category"),
    regexp_replace(col("phone"), r"(\d{3})(\d{3})(\d{4})", r"($1) $2-$3").alias("formatted_phone")
).groupBy("category").agg(
    avg("age").alias("avg_age"),
    sum("salary").alias("total_salary")
)
```

**SQL Functions:**

- Use `spark.sql()` with SQL syntax
- More familiar to SQL developers
- Better for complex analytical queries

```python
df.createOrReplaceTempView("people")

result = spark.sql("""
    SELECT 
        category,
        AVG(age) as avg_age,
        SUM(salary) as total_salary
    FROM (
        SELECT 
            name,
            CASE WHEN age > 18 THEN 'Adult' ELSE 'Minor' END as category,
            REGEXP_REPLACE(phone, '(\\d{3})(\\d{3})(\\d{4})', '($1) $2-($3)') as formatted_phone,
            age,
            salary
        FROM people
    ) 
    GROUP BY category
""")
```

**When to use each:**

**DataFrame Functions:**

- Building reusable data processing pipelines
- When you need strong typing and IDE support
- Complex transformations with method chaining
- When integrating with other Python libraries

**SQL Functions:**

- Complex analytical queries with multiple joins and subqueries
- When working with analysts familiar with SQL
- Ad-hoc data exploration
- Porting existing SQL queries to Spark

---

## 5. Aggregation and Grouping Operations

### 5.1 groupBy(), agg(), and pivot()

**Q: How do you implement and optimize aggregate functions like `groupBy()`, `agg()`, and `pivot()` in PySpark?**

**Answer:**

**Basic Aggregations:**

```python
from pyspark.sql.functions import sum, avg, count, max, min, stddev

# Simple groupBy with agg
result = df.groupBy("department").agg(
    avg("salary").alias("avg_salary"),
    sum("salary").alias("total_salary"),
    count("employee_id").alias("employee_count")
)

# Multiple grouping columns
result = df.groupBy("department", "location").agg(
    max("salary").alias("max_salary"),
    min("salary").alias("min_salary")
)
```

**Advanced Aggregations:**

```python
from pyspark.sql.functions import collect_list, collect_set, first, last

# Collecting values
result = df.groupBy("department").agg(
    collect_list("employee_name").alias("all_employees"),
    collect_set("skill").alias("unique_skills")
)

# Custom aggregations
from pyspark.sql.functions import expr
result = df.groupBy("department").agg(
    expr("percentile_approx(salary, 0.5)").alias("median_salary"),
    expr("count(distinct employee_id)").alias("unique_employees")
)
```

**Pivot Operations:**

```python
# Basic pivot
pivot_df = df.groupBy("department").pivot("year").sum("sales")

# Pivot with specific values (more efficient)
pivot_df = df.groupBy("department").pivot("year", [2022, 2023, 2024]).sum("sales")

# Multiple aggregations in pivot
pivot_df = df.groupBy("department").pivot("year").agg(
    sum("sales").alias("total_sales"),
    avg("sales").alias("avg_sales")
)
```

**Optimization Strategies:**

**Pre-aggregate when possible:**

```python
# Instead of multiple groupBy operations
df.groupBy("dept").agg(sum("sales"), avg("sales"), count("*"))
```

**Use specific pivot values:**

```python
# More efficient - Spark knows exact columns to create
df.groupBy("dept").pivot("year", [2022, 2023]).sum("sales")
```

**Push down filters:**

```python
# Filter before aggregation
df.filter(col("active") == True).groupBy("dept").sum("sales")
```

**Optimize partitioning:**

```python
# Repartition by groupBy key if data is skewed
df.repartition("department").groupBy("department").sum("sales")
```

---

## 6. Column Operations and Transformations

### 6.1 select(), withColumn(), and withColumnRenamed()

**Q: Explain the `select()`, `withColumn()`, and `withColumnRenamed()` methods. How do they differ in performance?**

**Answer:**

**select():**

- Selects specific columns from DataFrame
- Can include transformations and new column creation
- Creates a new DataFrame with only specified columns

```python
# Basic column selection
df.select("name", "age", "salary")

# With transformations
df.select(
    col("name"),
    (col("salary") * 1.1).alias("new_salary"),
    when(col("age") > 30, "Senior").otherwise("Junior").alias("level")
)
```

**withColumn():**

- Adds a new column or replaces an existing column
- Keeps all existing columns
- Can reference other columns in the transformation

```python
# Add new column
df.withColumn("bonus", col("salary") * 0.1)

# Replace existing column
df.withColumn("salary", col("salary") * 1.05)

# Chain multiple withColumn operations
df.withColumn("full_name", concat(col("first_name"), lit(" "), col("last_name"))) \
  .withColumn("age_group", when(col("age") > 30, "Senior").otherwise("Junior"))
```

**withColumnRenamed():**

- Renames an existing column
- Simple metadata operation
- Most efficient for column renaming

```python
# Rename single column
df.withColumnRenamed("old_name", "new_name")

# Chain multiple renames
df.withColumnRenamed("emp_id", "employee_id") \
  .withColumnRenamed("emp_name", "employee_name")
```

**Performance Differences:**

- **withColumnRenamed()**: Most efficient - only changes metadata, no data processing
- **select()**: Moderate - may involve column pruning and projection optimization
- **withColumn()**: Can be expensive if transformations are complex

**Performance Tips:**

```python
# More efficient - single select with all transformations
df.select(
    col("name"),
    col("age"),
    (col("salary") * 1.1).alias("new_salary"),
    when(col("age") > 30, "Senior").otherwise("Junior").alias("level")
)

# Less efficient - multiple withColumn operations
df.withColumn("new_salary", col("salary") * 1.1) \
  .withColumn("level", when(col("age") > 30, "Senior").otherwise("Junior"))
```

---

## 7. Data Filtering Techniques

### 7.1 filter(), where(), and Condition Chaining

**Q: What are the different ways to filter data in PySpark (`filter()`, `where()`, condition chaining)? Which is most efficient?**

**Answer:**

**Basic Filtering Methods:**

**filter() method:**

```python
# Using column expressions
df.filter(col("age") > 25)
df.filter(col("department") == "Engineering")

# Using SQL string expressions
df.filter("age > 25 AND department = 'Engineering'")
```

**where() method:**

```python
# Identical to filter() - they are aliases
df.where(col("age") > 25)
df.where("age > 25 AND department = 'Engineering'")
```

**Complex Filtering Approaches:**

**Condition Chaining:**

```python
from pyspark.sql.functions import col

# Method chaining
filtered_df = df.filter(col("age") > 25) \
                .filter(col("department") == "Engineering") \
                .filter(col("salary") > 50000)

# Single filter with multiple conditions
filtered_df = df.filter(
    (col("age") > 25) & 
    (col("department") == "Engineering") & 
    (col("salary") > 50000)
)
```

**Using SQL expressions:**

```python
df.filter("age > 25 AND department = 'Engineering' AND salary > 50000")
```

**Complex conditions:**

```python
from pyspark.sql.functions import isnan, isnull

# Multiple conditions with OR
df.filter((col("status") == "active") | (col("status") == "pending"))

# Using isin()
df.filter(col("department").isin(["Engineering", "Data", "Product"]))

# Null handling
df.filter(col("email").isNotNull())
df.filter(~isnan(col("score")))

# Pattern matching
df.filter(col("name").like("John%"))
df.filter(col("email").rlike(r".*@company\.com$"))
```

**Performance Comparison:**

|Method|Performance|Efficiency|
|---|---|---|
|Single filter with combined conditions|Most Efficient|⭐⭐⭐|
|SQL string expressions|Very Efficient|⭐⭐⭐|
|Chained filters|Less Efficient|⭐⭐|

**Examples:**

```python
# Most efficient
df.filter((col("age") > 25) & (col("dept") == "Eng") & (col("salary") > 50000))

# Very efficient
df.filter("age > 25 AND dept = 'Eng' AND salary > 50000")

# Less efficient
df.filter(col("age") > 25).filter(col("dept") == "Eng").filter(col("salary") > 50000)
```

**Performance Optimization Tips:**

**Push filters early:**

```python
# Good - filter early in the pipeline
df.filter(col("active") == True).select("name", "salary").groupBy("dept").sum("salary")
```

**Use partition pruning:**

```python
# If data is partitioned by date
df.filter(col("date") >= "2024-01-01")  # Prunes partitions
```

**Optimize for data types:**

```python
# Use appropriate data types for better performance
df.filter(col("age") > lit(25))  # lit() ensures type consistency
```

---

## 8. Complex Data Structures

### 8.1 Nested Data Handling with explode(), struct(), and Arrays

**Q: How do you handle complex nested data structures using functions like `explode()`, `struct()`, and array functions?**

**Answer:**

**Working with Arrays:**

```python
from pyspark.sql.functions import explode, explode_outer, posexplode, array, array_contains, size, sort_array

# Sample data with arrays
data = [("Alice", ["Java", "Python", "Scala"]), ("Bob", ["Python", "R"])]
df = spark.createDataFrame(data, ["name", "skills"])

# Explode array into separate rows
df.select("name", explode("skills").alias("skill")).show()

# Explode with nulls (explode_outer)
df.select("name", explode_outer("skills").alias("skill")).show()

# Explode with position
df.select("name", posexplode("skills").alias("pos", "skill")).show()

# Array functions
df.select("name", size("skills").alias("skill_count")).show()
df.select("name", sort_array("skills").alias("sorted_skills")).show()
df.filter(array_contains("skills", "Python")).show()
```

**Working with Structs:**

```python
from pyspark.sql.functions import struct, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# Creating structs
df = spark.createDataFrame([("Alice", 25, "Engineer"), ("Bob", 30, "Manager")], 
                          ["name", "age", "role"])

# Create struct column
df_with_struct = df.select("name", struct("age", "role").alias("info"))

# Access struct fields
df_with_struct.select("name", col("info.age"), col("info.role")).show()

# Create nested struct
df.select("name", 
    struct(
        col("age"),
        struct(col("role"), lit("Engineering").alias("department")).alias("job_info")
    ).alias("person_info")
).show()
```

**Complex Nested Operations:**

```python
from pyspark.sql.functions import collect_list, map_keys, map_values, create_map

# Working with maps
df_map = spark.createDataFrame([
    ("Alice", {"skill1": "Java", "skill2": "Python"}),
    ("Bob", {"skill1": "Python", "skill2": "R"})
], ["name", "skill_map"])

# Access map operations
df_map.select("name", map_keys("skill_map")).show()
df_map.select("name", map_values("skill_map")).show()

# Complex nested array of structs
from pyspark.sql.types import ArrayType

nested_data = [
    ("Alice", [{"skill": "Java", "level": 5}, {"skill": "Python", "level": 4}]),
    ("Bob", [{"skill": "Python", "level": 3}, {"skill": "R", "level": 4}])
]

schema = StructType([
    StructField("name", StringType()),
    StructField("skills", ArrayType(StructType([
        StructField("skill", StringType()),
        StructField("level", IntegerType())
    ])))
])

df_nested = spark.createDataFrame(nested_data, schema)

# Explode and access nested struct fields
df_nested.select("name", explode("skills").alias("skill_info")) \
         .select("name", col("skill_info.skill"), col("skill_info.level")) \
         .show()
```

**Advanced Nested Data Processing:**

```python
from pyspark.sql.functions import transform, filter as array_filter, exists, aggregate

# Transform array elements (Spark 2.4+)
df.select("name", transform("skills", lambda x: upper(x)).alias("upper_skills"))

# Filter array elements
df.select("name", array_filter("skills", lambda x: x.contains("Python")).alias("python_skills"))

# Check if array contains elements matching condition
df.select("name", exists("skills", lambda x: x == "Python").alias("has_python"))

# Aggregate array elements
df.select("name", 
    aggregate("skills", lit(""), lambda acc, x: concat(acc, lit(","), x)).alias("skills_concat")
)
```

---

## 9. Data Redistribution and Partitioning

### 9.1 coalesce() vs repartition()

**Q: Explain `coalesce()` vs `repartition()` methods. When would you use each for data redistribution?**

**Answer:**

**coalesce():**

- Reduces the number of partitions by merging adjacent partitions
- Does not trigger a full shuffle (more efficient)
- Can only reduce partitions, cannot increase them
- Maintains data locality when possible

```python
# Reduce partitions from current number to 5
df_coalesced = df.coalesce(5)

# Check current partition count
print(f"Original partitions: {df.rdd.getNumPartitions()}")
print(f"After coalesce: {df_coalesced.rdd.getNumPartitions()}")
```

**repartition():**

- Can increase or decrease the number of partitions
- Triggers a full shuffle operation (more expensive)
- Distributes data evenly across all partitions
- Can repartition by column values for better data organization

```python
# Repartition to exactly 10 partitions
df_repartitioned = df.repartition(10)

# Repartition by column (hash partitioning)
df_by_dept = df.repartition("department")
df_by_multiple = df.repartition("department", "location")

# Combine count and column partitioning
df_optimized = df.repartition(8, "department")
```

**Performance Characteristics:**

|Aspect|coalesce()|repartition()|
|---|---|---|
|Shuffle|No full shuffle|Full shuffle|
|Performance|Faster|Slower|
|Partition increase|Not possible|Possible|
|Data distribution|May be uneven|Even distribution|
|Use case|Reduce output files|Load balancing|

**When to Use coalesce():**

**Before writing to storage (reduce output files):**

```python
# Avoid many small files when writing
df.coalesce(1).write.mode("overwrite").parquet("output_path")
```

**After filtering (when you have fewer records):**

```python
# After filtering, you might have fewer partitions with data
filtered_df = df.filter(col("active") == True)
filtered_df.coalesce(4).write.parquet("filtered_output")
```

**Memory optimization (when partitions are small):**

```python
# Combine small partitions to reduce overhead
df.coalesce(df.rdd.getNumPartitions() // 2)
```

**When to Use repartition():**

**Load balancing (when partitions are skewed):**

```python
# Redistribute skewed data evenly
skewed_df.repartition(10)
```

**Optimizing for downstream operations:**

```python
# Repartition by groupBy key for better performance
df.repartition("department").groupBy("department").sum("salary")
```

**Preparing for joins:**

```python
# Both DataFrames partitioned by join key
df1_partitioned = df1.repartition("employee_id")
df2_partitioned = df2.repartition("employee_id")
result = df1_partitioned.join(df2_partitioned, "employee_id")
```

**Increasing parallelism:**

```python
# Increase partitions for better parallelism on large cluster
df.repartition(100).map(expensive_operation)
```

**Best Practices:**

```python
# Good: Use coalesce before writing
df.filter(col("active") == True) \
  .coalesce(5) \
  .write.mode("overwrite") \
  .parquet("output")

# Good: Repartition by column for better data locality
df.repartition("date", "region") \
  .write.mode("overwrite") \
  .partitionBy("date", "region") \
  .parquet("partitioned_output")

# Good: Balance between parallelism and overhead
optimal_partitions = max(1, df.count() // 100000)  # ~100k records per partition
df.repartition(optimal_partitions)
```

### 9.2 Custom Partitioning Logic

**Q: How do you implement custom partitioning logic using `partitionBy()` and custom partitioner functions?**

**Answer:**

**Basic partitionBy() for Writing:**

```python
# Partition data when writing to storage
df.write.mode("overwrite") \
  .partitionBy("year", "month") \
  .parquet("data/partitioned")

# Multiple level partitioning
df.write.mode("overwrite") \
  .partitionBy("department", "location", "year") \
  .parquet("data/employee_partitioned")

# With specific ordering
df.orderBy("employee_id") \
  .write.mode("overwrite") \
  .partitionBy("department") \
  .parquet("data/ordered_partitioned")
```

**Custom Partitioner for RDDs:**

```python
class CustomPartitioner:
    def __init__(self, num_partitions):
        self.num_partitions = num_partitions
    
    def numPartitions(self):
        return self.num_partitions
    
    def getPartition(self, key):
        # Custom logic to determine partition
        if isinstance(key, str):
            # Hash-based partitioning for strings
            return hash(key) % self.num_partitions
        elif isinstance(key, int):
            # Range-based partitioning for integers
            if key < 1000:
                return 0
            elif key < 10000:
                return 1
            else:
                return 2
        return 0

# Using custom partitioner
rdd = spark.sparkContext.parallelize([("Alice", 1), ("Bob", 5000), ("Charlie", 15000)])
partitioner = CustomPartitioner(3)
partitioned_rdd = rdd.partitionBy(partitioner)
```

**Advanced Custom Partitioning Strategies:**

**Range-based Partitioning:**

```python
def range_partitioner(key, num_partitions):
    """Partition based on key ranges"""
    if key < 100:
        return 0
    elif key < 1000:
        return 1 % num_partitions
    else:
        return 2 % num_partitions

# Apply range partitioning
key_value_rdd = spark.sparkContext.parallelize([(50, "data1"), (500, "data2"), (1500, "data3")])
range_partitioned = key_value_rdd.partitionBy(3, range_partitioner)
```

**Geographic Partitioning:**

```python
class GeographicPartitioner:
    def __init__(self, num_partitions):
        self.num_partitions = num_partitions
        self.regions = {
            "US": 0, "EU": 1, "ASIA": 2, "OTHER": 3
        }
    
    def numPartitions(self):
        return self.num_partitions
    
    def getPartition(self, key):
        # Extract region from key (assuming key format: "REGION_COUNTRY")
        region = key.split("_")[0] if "_" in key else "OTHER"
        return self.regions.get(region, 3) % self.num_partitions

# Usage
location_data = [("US_NY", "data1"), ("EU_UK", "data2"), ("ASIA_JP", "data3")]
geo_rdd = spark.sparkContext.parallelize(location_data)
geo_partitioner = GeographicPartitioner(4)
geo_partitioned = geo_rdd.partitionBy(geo_partitioner)
```

**DataFrame Custom Partitioning Techniques:**

**Using repartition() with custom column logic:**

```python
from pyspark.sql.functions import when, col, hash

# Create custom partition column
df_with_partition = df.withColumn("custom_partition",
    when(col("age") < 25, 0)
    .when(col("age") < 50, 1)
    .otherwise(2)
)

# Repartition using the custom column
custom_partitioned = df_with_partition.repartition("custom_partition")
```

**Hash-based custom partitioning:**

```python
from pyspark.sql.functions import hash, abs

# Custom hash partitioning
num_partitions = 10
df_hash_partitioned = df.withColumn("partition_id", 
    abs(hash(col("employee_id"))) % num_partitions
).repartition(col("partition_id"))
```

**Partition Optimization Strategies:**

```python
# 1. Check partition distribution
def analyze_partitions(df):
    partition_counts = df.rdd.mapPartitions(lambda iterator: [sum(1 for _ in iterator)]).collect()
    print(f"Partition distribution: {partition_counts}")
    return partition_counts

# 2. Optimize partition size
def optimize_partition_size(df, target_size_mb=128):
    """Optimize partitions to target size"""
    total_size_mb = df.rdd.map(lambda row: len(str(row))).sum() / (1024 * 1024)
    optimal_partitions = max(1, int(total_size_mb / target_size_mb))
    return df.repartition(optimal_partitions)

# 3. Custom skew handling
def handle_skewed_partitions(df, skew_column, num_partitions=None):
    """Handle skewed data by adding salt"""
    import random
    
    if num_partitions is None:
        num_partitions = df.rdd.getNumPartitions()
    
    # Add random salt to skewed keys
    salted_df = df.withColumn("salt", 
        when(col(skew_column) == "popular_value", 
             lit(random.randint(0, num_partitions-1)))
        .otherwise(lit(0))
    )
    
    # Create composite key for partitioning
    composite_key_df = salted_df.withColumn("partition_key",
        concat(col(skew_column), lit("_"), col("salt"))
    )
    
    return composite_key_df.repartition("partition_key").drop("salt", "partition_key")
```

**Best Practices for Custom Partitioning:**

**Consider data locality:**

```python
# Partition by columns used in joins or groupBy operations
df.repartition("department").write.partitionBy("department").parquet("output")
```

**Balance partition sizes:**

```python
# Aim for 100MB-200MB per partition
def calculate_optimal_partitions(df, target_mb=150):
    estimated_size_mb = df.count() * 0.001  # Rough estimation
    return max(1, int(estimated_size_mb / target_mb))
```

**Monitor partition skew:**

```python
# Check for skewed partitions
def detect_skew(df):
    partition_sizes = df.rdd.mapPartitions(lambda it: [sum(1 for _ in it)]).collect()
    max_size = max(partition_sizes)
    min_size = min(partition_sizes)
    skew_ratio = max_size / min_size if min_size > 0 else float('inf')
    print(f"Skew ratio: {skew_ratio:.2f}")
    return skew_ratio > 3  # Consider skewed if ratio > 3
```

---

## Summary and Best Practices

### Key Performance Tips

- **Prefer built-in functions over UDFs** for better performance and Catalyst optimization
- **Use single filters with combined conditions** instead of chaining multiple filters
- **Choose coalesce() for reducing partitions** and repartition() for load balancing
- **Leverage column-based partitioning** for better query performance
- **Monitor partition sizes** and avoid skewed data distribution
- **Push filters early** in your data pipeline to reduce data movement
- **Use appropriate data types** and avoid unnecessary data conversions

### Common Anti-patterns to Avoid

- Using `collect()` on large datasets
- Chaining multiple `withColumn()` operations instead of using `select()`
- Creating UDFs when built-in functions are available
- Not considering partition strategies for large datasets
- Ignoring data skew in partitions
- Using `repartition()` when `coalesce()` would suffice

### Interview Tips

- **Always explain the why** behind your choice of functions
- **Discuss performance implications** and trade-offs
- **Provide concrete examples** with code snippets
- **Mention optimization strategies** and best practices
- **Show understanding of Spark's internal mechanisms** (Catalyst optimizer, shuffle operations)
- **Be prepared to discuss real-world scenarios** and how you would handle them