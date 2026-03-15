
#pyspark #data-operations #data-io #joins #missing-data #transformations #data-engineering

## Overview

This note covers essential data operations in PySpark, including reading and writing data, handling missing values, performing joins, and common data transformations. These are the day-to-day operations you'll use most frequently in PySpark.

---

## Reading Data into PySpark

### File-based Sources

#### CSV Files

Good for data exchange and human-readable formats, but slower than binary formats.

```python
# Basic CSV reading
df_csv = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv("data.csv")

# Advanced CSV options
df_csv_advanced = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .option("multiline", "true") \
    .option("escape", '"') \
    .option("delimiter", ",") \
    .option("nullValue", "NULL") \
    .csv("complex_data.csv")
```

#### Parquet Files

Optimized for analytics workloads - columnar storage format.

```python
# Simple parquet read
df_parquet = spark.read.parquet("data.parquet")

# Reading partitioned parquet
df_partitioned = spark.read.parquet("partitioned_data/year=2024/month=*/")

# Reading with specific columns (column pruning)
df_columns = spark.read.parquet("data.parquet").select("col1", "col2", "col3")
```

#### JSON Files

Good for semi-structured data with nested objects.

```python
# Basic JSON reading
df_json = spark.read.json("data.json")

# Multi-line JSON
df_multiline_json = spark.read \
    .option("multiline", "true") \
    .json("multiline_data.json")

# JSON with schema enforcement
from pyspark.sql.types import *
json_schema = StructType([
    StructField("id", StringType(), True),
    StructField("name", StringType(), True),
    StructField("data", StructType([
        StructField("value", DoubleType(), True),
        StructField("timestamp", TimestampType(), True)
    ]), True)
])

df_json_schema = spark.read.schema(json_schema).json("data.json")
```

#### Delta Lake

Modern data lake format with ACID transactions and time travel.

```python
# Read Delta table
df_delta = spark.read.format("delta").load("delta-table")

# Time travel - read historical version
df_historical = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-01") \
    .load("delta-table")

# Read specific version
df_version = spark.read.format("delta") \
    .option("versionAsOf", 0) \
    .load("delta-table")
```

### Database Sources

#### JDBC Connections

```python
# PostgreSQL connection
df_postgres = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/database") \
    .option("dbtable", "sales") \
    .option("user", "username") \
    .option("password", "password") \
    .option("driver", "org.postgresql.Driver") \
    .load()

# Optimized JDBC reading with partitioning
df_partitioned_jdbc = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/database") \
    .option("dbtable", "large_table") \
    .option("user", "username") \
    .option("password", "password") \
    .option("numPartitions", "10") \
    .option("partitionColumn", "id") \
    .option("lowerBound", "1") \
    .option("upperBound", "1000000") \
    .load()

# Custom SQL query
df_custom_query = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://host:5432/database") \
    .option("query", "SELECT * FROM sales WHERE date >= '2024-01-01'") \
    .option("user", "username") \
    .option("password", "password") \
    .load()
```

### Performance Considerations for Reading

- **Parquet**: Best for analytical workloads, 10x faster than CSV
- **Delta Lake**: Best for data lakes requiring ACID properties
- **Partitioning**: Use partitioned datasets for better query performance
- **Schema Inference**: Disable for large CSV files to improve performance
- **Column Pruning**: Select only needed columns early in the pipeline

```python
# Performance optimization example
df_optimized = spark.read \
    .option("maxPartitionBytes", "128MB") \
    .parquet("large_dataset") \
    .select("needed_col1", "needed_col2") \
    .filter(col("date") >= "2024-01-01")
```

---

## Writing Data from PySpark

### File Formats

#### Writing CSV

```python
# Basic CSV write
df.write \
    .mode("overwrite") \
    .option("header", "true") \
    .csv("output_data.csv")

# CSV with custom options
df.write \
    .mode("append") \
    .option("header", "true") \
    .option("delimiter", "|") \
    .option("nullValue", "NULL") \
    .csv("custom_output.csv")
```

#### Writing Parquet

```python
# Basic parquet write
df.write.mode("overwrite").parquet("output_data.parquet")

# Partitioned parquet write
df.write \
    .mode("overwrite") \
    .partitionBy("year", "month") \
    .parquet("partitioned_output")

# Parquet with compression
df.write \
    .mode("overwrite") \
    .option("compression", "snappy") \
    .parquet("compressed_output.parquet")
```

#### Writing Delta

```python
# Basic Delta write
df.write.format("delta").mode("overwrite").save("delta_output")

# Delta with merge operation
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "delta_table_path")
delta_table.alias("old_data") \
    .merge(new_df.alias("new_data"), "old_data.id = new_data.id") \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()
```

### Write Modes

```python
# Overwrite - replace existing data
df.write.mode("overwrite").parquet("data")

# Append - add to existing data
df.write.mode("append").parquet("data")

# Ignore - do nothing if data exists
df.write.mode("ignore").parquet("data")

# Error (default) - throw error if data exists
df.write.mode("error").parquet("data")
```

---

## Handling Missing Data

### 1. Dropping Missing Values

```python
# Drop rows with any null values
df_clean = df.dropna(how="any")

# Drop rows only if all values are null
df_clean = df.dropna(how="all")

# Drop rows with nulls in specific columns
df_clean = df.dropna(subset=["customer_id", "amount"])

# Drop rows with nulls in at least 2 columns
df_clean = df.dropna(thresh=2)

# Drop columns with high null percentage
def drop_high_null_columns(df, threshold=0.5):
    """Drop columns where null percentage exceeds threshold"""
    total_rows = df.count()
    columns_to_keep = []
    
    for col_name in df.columns:
        null_count = df.filter(col(col_name).isNull()).count()
        null_percentage = null_count / total_rows
        
        if null_percentage <= threshold:
            columns_to_keep.append(col_name)
    
    return df.select(*columns_to_keep)

df_filtered_columns = drop_high_null_columns(df, 0.3)  # Drop columns >30% null
```

### 2. Filling Missing Values

```python
# Fill with constant values
df_filled = df.fillna({
    "age": 0, 
    "name": "Unknown", 
    "salary": -1,
    "is_active": False
})

# Fill with different strategies per column
df_filled = df.fillna("Unknown", subset=["name", "description"]) \
             .fillna(0, subset=["age", "salary"])

# Forward fill (use previous non-null value)
from pyspark.sql.window import Window
from pyspark.sql.functions import last, col, when, isnan, isnull

window = Window.partitionBy("customer_id").orderBy("timestamp") \
               .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df_ffill = df.withColumn("amount_filled", 
    last(col("amount"), ignorenulls=True).over(window))

# Backward fill (use next non-null value)
window_bfill = Window.partitionBy("customer_id").orderBy("timestamp") \
                     .rowsBetween(Window.currentRow, Window.unboundedFollowing)

df_bfill = df.withColumn("amount_bfilled", 
    first(col("amount"), ignorenulls=True).over(window_bfill))
```

### 3. Statistical Imputation

```python
from pyspark.ml.feature import Imputer

# Impute with mean, median, or mode
imputer = Imputer(
    strategy="median",  # or "mean", "mode"
    inputCols=["age", "income", "score"],
    outputCols=["age_imputed", "income_imputed", "score_imputed"]
)

model = imputer.fit(df)
df_imputed = model.transform(df)

# Custom imputation function
def impute_with_group_median(df, impute_col, group_cols):
    """Impute missing values with median by group"""
    from pyspark.sql.functions import median
    
    # Calculate median by group
    median_by_group = df.filter(col(impute_col).isNotNull()) \
                        .groupBy(*group_cols) \
                        .agg(median(impute_col).alias("median_value"))
    
    # Join back and fill nulls
    df_with_median = df.join(median_by_group, group_cols, "left")
    
    return df_with_median.withColumn(
        impute_col,
        when(col(impute_col).isNull(), col("median_value"))
        .otherwise(col(impute_col))
    ).drop("median_value")

# Usage
df_group_imputed = impute_with_group_median(df, "salary", ["department", "level"])
```

### 4. Detecting and Handling Outliers

```python
def detect_outliers_iqr(df, column_name, factor=1.5):
    """Detect outliers using IQR method"""
    
    # Calculate Q1, Q3, and IQR
    quantiles = df.approxQuantile(column_name, [0.25, 0.75], 0.05)
    Q1, Q3 = quantiles[0], quantiles[1]
    IQR = Q3 - Q1
    
    # Define outlier bounds
    lower_bound = Q1 - factor * IQR
    upper_bound = Q3 + factor * IQR
    
    # Mark outliers
    df_with_outliers = df.withColumn(
        f"{column_name}_is_outlier",
        (col(column_name) < lower_bound) | (col(column_name) > upper_bound)
    )
    
    return df_with_outliers, lower_bound, upper_bound

# Usage
df_outliers, lower, upper = detect_outliers_iqr(df, "amount")
print(f"Outlier bounds: {lower} to {upper}")

# Remove outliers
df_no_outliers = df_outliers.filter(~col("amount_is_outlier"))
```

---

## Join Operations

### Join Types

```python
# Sample DataFrames for examples
customers = spark.createDataFrame([
    (1, "Alice", "alice@email.com"),
    (2, "Bob", "bob@email.com"),
    (3, "Charlie", "charlie@email.com")
], ["customer_id", "name", "email"])

orders = spark.createDataFrame([
    (101, 1, 100.0),
    (102, 2, 150.0),
    (103, 1, 200.0),
    (104, 4, 75.0)  # customer_id 4 doesn't exist in customers
], ["order_id", "customer_id", "amount"])
```

#### Inner Join

Only matching records from both sides.

```python
# Inner join - only customers who have orders
inner_result = customers.join(orders, "customer_id", "inner")
inner_result.show()
# +------------+-------+---------------+--------+------+
# |customer_id |name   |email          |order_id|amount|
# +------------+-------+---------------+--------+------+
# |1           |Alice  |alice@email.com|101     |100.0 |
# |1           |Alice  |alice@email.com|103     |200.0 |
# |2           |Bob    |bob@email.com  |102     |150.0 |
# +------------+-------+---------------+--------+------+
```

#### Left Outer Join

All records from left, matching from right.

```python
# Left join - all customers, with their orders if they exist
left_result = customers.join(orders, "customer_id", "left")
left_result.show()
# +------------+-------+-----------------+--------+------+
# |customer_id |name   |email            |order_id|amount|
# +------------+-------+-----------------+--------+------+
# |1           |Alice  |alice@email.com  |101     |100.0 |
# |1           |Alice  |alice@email.com  |103     |200.0 |
# |2           |Bob    |bob@email.com    |102     |150.0 |
# |3           |Charlie|charlie@email.com|null    |null  |
# +------------+-------+-----------------+--------+------+
```

#### Right Outer Join

All records from right, matching from left.

```python
# Right join - all orders, with customer info if customer exists
right_result = customers.join(orders, "customer_id", "right")
right_result.show()
```

#### Full Outer Join

All records from both sides.

```python
# Full outer join - all customers and all orders
full_result = customers.join(orders, "customer_id", "outer")
full_result.show()
```

#### Semi Join

Left records that have matches in right (no right columns).

```python
# Left semi join - customers who have orders (no order details)
semi_result = customers.join(orders, "customer_id", "left_semi")
semi_result.show()
# +------------+-----+---------------+
# |customer_id |name |email          |
# +------------+-----+---------------+
# |1           |Alice|alice@email.com|
# |2           |Bob  |bob@email.com  |
# +------------+-----+---------------+
```

#### Anti Join

Left records that don't have matches in right.

```python
# Left anti join - customers who have no orders
anti_result = customers.join(orders, "customer_id", "left_anti")
anti_result.show()
# +------------+-------+-----------------+
# |customer_id |name   |email            |
# +------------+-------+-----------------+
# |3           |Charlie|charlie@email.com|
# +------------+-------+-----------------+
```

### Complex Join Conditions

```python
# Multiple join conditions
complex_join = df1.join(
    df2,
    (df1.customer_id == df2.customer_id) & 
    (df1.date >= df2.start_date) & 
    (df1.date <= df2.end_date),
    "inner"
)

# Join with different column names
df_renamed_join = customers.join(
    orders.withColumnRenamed("customer_id", "cust_id"),
    customers.customer_id == orders.cust_id,
    "inner"
)
```

### Join Performance Optimization

#### Broadcast Joins

For small tables (< 10MB by default).

```python
from pyspark.sql.functions import broadcast

# Automatically broadcast small table
result = large_df.join(broadcast(small_df), "key")

# Configure broadcast threshold
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")

# Force broadcast for specific table
result = orders.join(broadcast(customers), "customer_id")
```

#### Bucketed Joins

For repeated joins on the same keys.

```python
# Pre-bucket tables for efficient joins
customers.write \
    .bucketBy(10, "customer_id") \
    .mode("overwrite") \
    .saveAsTable("bucketed_customers")

orders.write \
    .bucketBy(10, "customer_id") \
    .mode("overwrite") \
    .saveAsTable("bucketed_orders")

# Join bucketed tables (no shuffle required)
result = spark.table("bucketed_customers").join(
    spark.table("bucketed_orders"), 
    "customer_id"
)
```

#### Join Hints

```python
# Broadcast hint in SQL
spark.sql("""
    SELECT /*+ BROADCAST(customers) */ *
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
""")

# Merge hint for sort-merge join
spark.sql("""
    SELECT /*+ MERGE(orders, customers) */ *
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
""")
```

---

## Common Data Transformations

### Column Operations

```python
from pyspark.sql.functions import *

# Add new columns
df_with_new_cols = df.withColumn("full_name", concat(col("first_name"), lit(" "), col("last_name"))) \
                     .withColumn("age_group", 
                         when(col("age") < 18, "Minor")
                         .when(col("age") < 65, "Adult")
                         .otherwise("Senior")) \
                     .withColumn("created_date", current_date())

# Rename columns
df_renamed = df.withColumnRenamed("old_name", "new_name")

# Drop columns
df_dropped = df.drop("unwanted_col1", "unwanted_col2")

# Select and reorder columns
df_selected = df.select("col1", "col3", "col2", "col4")
```

### Filtering and Conditional Logic

```python
# Basic filtering
df_filtered = df.filter(col("age") > 18)
df_filtered = df.filter((col("age") > 18) & (col("status") == "active"))

# String operations
df_string_filtered = df.filter(col("name").startswith("A")) \
                       .filter(col("email").contains("@gmail.com")) \
                       .filter(col("description").isNotNull())

# Complex conditional logic
df_conditional = df.withColumn("risk_category",
    when(col("credit_score") >= 750, "Low")
    .when(col("credit_score") >= 650, "Medium")
    .when(col("credit_score") >= 550, "High")
    .otherwise("Very High")
)
```

### Aggregations and Grouping

```python
# Basic aggregations
df_agg = df.groupBy("department") \
           .agg(
               count("*").alias("employee_count"),
               avg("salary").alias("avg_salary"),
               max("salary").alias("max_salary"),
               min("salary").alias("min_salary"),
               sum("bonus").alias("total_bonus")
           )

# Multiple grouping columns
df_multi_group = df.groupBy("department", "level") \
                   .agg(avg("salary").alias("avg_salary"))

# Pivot operations
df_pivot = df.groupBy("department") \
             .pivot("level") \
             .agg(avg("salary"))

# Unpivot operations (stack)
df_unpivot = df.select("id", 
    expr("stack(3, 'salary', salary, 'bonus', bonus, 'commission', commission) as (metric, value)")
)
```

### Date and Time Operations

```python
from pyspark.sql.functions import *

# Date parsing and formatting
df_dates = df.withColumn("parsed_date", to_date(col("date_string"), "yyyy-MM-dd")) \
             .withColumn("formatted_date", date_format(col("date_col"), "MM/dd/yyyy")) \
             .withColumn("year", year(col("date_col"))) \
             .withColumn("month", month(col("date_col"))) \
             .withColumn("day_of_week", dayofweek(col("date_col")))

# Date arithmetic
df_date_calc = df.withColumn("days_ago", datediff(current_date(), col("date_col"))) \
                 .withColumn("future_date", date_add(col("date_col"), 30)) \
                 .withColumn("past_date", date_sub(col("date_col"), 7))

# Timestamp operations
df_timestamps = df.withColumn("current_timestamp", current_timestamp()) \
                  .withColumn("hour", hour(col("timestamp_col"))) \
                  .withColumn("unix_timestamp", unix_timestamp(col("timestamp_col")))
```

---

## Performance Tips for Data Operations

### 1. Optimize Reading Operations

```python
# Read only needed columns
df = spark.read.parquet("large_dataset").select("col1", "col2", "col3")

# Use predicate pushdown
df = spark.read.parquet("partitioned_data") \
              .filter(col("year") == 2024) \
              .filter(col("month") >= 6)

# Specify schema to avoid inference
schema = StructType([...])
df = spark.read.schema(schema).csv("data.csv")
```

### 2. Optimize Joins

```python
# Use broadcast joins for small lookup tables
result = large_table.join(broadcast(small_lookup), "key")

# Pre-filter before joining
filtered_df1 = df1.filter(col("date") >= "2024-01-01")
filtered_df2 = df2.filter(col("status") == "active")
result = filtered_df1.join(filtered_df2, "key")
```

### 3. Optimize Aggregations

```python
# Use approximate functions for large datasets
df.agg(approx_count_distinct("user_id", 0.05).alias("unique_users"))

# Pre-aggregate before complex operations
daily_agg = df.groupBy("date", "category").agg(sum("amount").alias("daily_total"))
final_result = daily_agg.groupBy("category").agg(avg("daily_total"))
```

---

## Related Notes

- [[PySpark Core Concepts]] - Fundamental concepts these operations build upon
- [[PySpark Performance Optimization]] - Advanced optimization techniques
- [[PySpark Window Functions]] - Advanced analytical operations
- [[PySpark Functions & Methods]] - Complete reference of available functions

---

## Tags

#pyspark #data-operations #joins #missing-data #data-io #transformations #performance #data-engineering

---

_Last Updated: 2024-08-20_