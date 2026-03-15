
#pyspark #window-functions #analytics #ranking #aggregations #data-analysis #advanced

## Overview

Window functions in PySpark perform calculations across a set of rows related to the current row without collapsing the result set like traditional GROUP BY operations. They're essential for advanced analytics, ranking, running totals, and time-series analysis.

---

## Window Function Fundamentals

### What are Window Functions?

Window functions operate on a "window" of rows and return a value for each row based on the group of rows. Unlike aggregate functions that collapse multiple rows into one, window functions maintain the original number of rows.

### Key Components of Window Functions

1. **PARTITION BY**: Divides data into groups (similar to GROUP BY)
2. **ORDER BY**: Defines row ordering within partitions
3. **Frame Specification**: Defines which rows to include in calculation (optional)

### Basic Window Specification

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import *

# Basic window specification
window_spec = Window.partitionBy("department").orderBy("salary")

# Window with frame specification
window_with_frame = Window.partitionBy("department") \
                         .orderBy("timestamp") \
                         .rowsBetween(Window.unboundedPreceding, Window.currentRow)
```

---

## Types of Window Functions

### 1. Ranking Functions

Ranking functions assign ranks to rows within each partition.

#### Row Number

Assigns unique sequential numbers to rows within each partition.

```python
# Sample data for examples
employees = spark.createDataFrame([
    ("Alice", "Engineering", 95000, "2023-01-15"),
    ("Bob", "Engineering", 87000, "2023-02-01"),
    ("Charlie", "Engineering", 92000, "2023-01-20"),
    ("Diana", "Marketing", 78000, "2023-01-10"),
    ("Eve", "Marketing", 82000, "2023-02-05"),
    ("Frank", "Sales", 75000, "2023-01-25"),
    ("Grace", "Sales", 79000, "2023-02-10")
], ["name", "department", "salary", "hire_date"])

# Add row numbers within each department ordered by salary
window_salary = Window.partitionBy("department").orderBy(desc("salary"))

df_with_row_number = employees.withColumn(
    "salary_rank", 
    row_number().over(window_salary)
)

df_with_row_number.show()
# +-------+-----------+------+----------+-----------+
# |   name| department|salary| hire_date|salary_rank|
# +-------+-----------+------+----------+-----------+
# |  Alice|Engineering| 95000|2023-01-15|          1|
# |Charlie|Engineering| 92000|2023-01-20|          2|
# |    Bob|Engineering| 87000|2023-02-01|          3|
# |    Eve|  Marketing| 82000|2023-02-05|          1|
# |  Diana|  Marketing| 78000|2023-01-10|          2|
# |  Grace|      Sales| 79000|2023-02-10|          1|
# |  Frank|      Sales| 75000|2023-01-25|          2|
# +-------+-----------+------+----------+-----------+
```

#### Rank and Dense Rank

Handle ties differently than row_number.

```python
# Create data with salary ties
employees_with_ties = spark.createDataFrame([
    ("Alice", "Engineering", 95000),
    ("Bob", "Engineering", 87000),
    ("Charlie", "Engineering", 87000),  # Same salary as Bob
    ("Diana", "Engineering", 82000)
], ["name", "department", "salary"])

# Compare different ranking functions
df_rankings = employees_with_ties.withColumn("row_number", row_number().over(window_salary)) \
                                .withColumn("rank", rank().over(window_salary)) \
                                .withColumn("dense_rank", dense_rank().over(window_salary))

df_rankings.show()
# +-------+-----------+------+----------+----+----------+
# |   name| department|salary|row_number|rank|dense_rank|
# +-------+-----------+------+----------+----+----------+
# |  Alice|Engineering| 95000|         1|   1|         1|
# |    Bob|Engineering| 87000|         2|   2|         2|
# |Charlie|Engineering| 87000|         3|   2|         2|  # Same rank as Bob
# |  Diana|Engineering| 82000|         4|   4|         3|  # rank skips 3, dense_rank doesn't
# +-------+-----------+------+----------+----+----------+
```

#### Percent Rank and Ntile

Statistical ranking functions.

```python
# Percent rank and ntile
df_statistical_ranks = employees.withColumn(
    "percent_rank", percent_rank().over(window_salary)
).withColumn(
    "quartile", ntile(4).over(window_salary)  # Divide into 4 groups
)

df_statistical_ranks.select("name", "department", "salary", "percent_rank", "quartile").show()
```

### 2. Analytical Functions

#### Lead and Lag

Access values from subsequent or previous rows.

```python
# Time-ordered window for lead/lag operations
window_time = Window.partitionBy("customer_id").orderBy("timestamp")

# Sample time series data
transactions = spark.createDataFrame([
    ("C001", "2024-01-01 10:00:00", 100.0),
    ("C001", "2024-01-02 14:30:00", 150.0),
    ("C001", "2024-01-03 09:15:00", 200.0),
    ("C001", "2024-01-04 16:45:00", 75.0),
    ("C002", "2024-01-01 11:00:00", 250.0),
    ("C002", "2024-01-02 13:20:00", 180.0)
], ["customer_id", "timestamp", "amount"])

# Convert timestamp to proper format
transactions = transactions.withColumn("timestamp", 
    to_timestamp(col("timestamp"), "yyyy-MM-dd HH:mm:ss"))

# Add lead and lag values
df_lead_lag = transactions.withColumn(
    "previous_amount", lag("amount", 1).over(window_time)
).withColumn(
    "next_amount", lead("amount", 1).over(window_time)
).withColumn(
    "amount_change", col("amount") - lag("amount", 1).over(window_time)
)

df_lead_lag.show()
# +-----------+-------------------+------+---------------+-----------+-------------+
# |customer_id|          timestamp|amount|previous_amount|next_amount|amount_change|
# +-----------+-------------------+------+---------------+-----------+-------------+
# |       C001|2024-01-01 10:00:00| 100.0|           null|      150.0|         null|
# |       C001|2024-01-02 14:30:00| 150.0|          100.0|      200.0|         50.0|
# |       C001|2024-01-03 09:15:00| 200.0|          150.0|       75.0|         50.0|
# |       C001|2024-01-04 16:45:00|  75.0|          200.0|       null|       -125.0|
# |       C002|2024-01-01 11:00:00| 250.0|           null|      180.0|         null|
# |       C002|2024-01-02 13:20:00| 180.0|          250.0|       null|        -70.0|
# +-----------+-------------------+------+---------------+-----------+-------------+
```

#### First and Last Values

Get first/last values in the window.

```python
# Get first and last values within each partition
df_first_last = transactions.withColumn(
    "first_amount", first("amount").over(window_time)
).withColumn(
    "last_amount", last("amount").over(window_time)
).withColumn(
    "first_transaction_date", first("timestamp").over(window_time)
).withColumn(
    "last_transaction_date", last("timestamp").over(window_time)
)

df_first_last.select("customer_id", "timestamp", "amount", 
                     "first_amount", "last_amount").show()
```

### 3. Aggregate Functions with Windows

Unlike regular aggregates, window aggregates don't collapse rows.

#### Running Totals and Cumulative Calculations

```python
# Running totals and cumulative calculations
window_unbounded = Window.partitionBy("customer_id") \
                         .orderBy("timestamp") \
                         .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df_running_totals = transactions.withColumn(
    "running_total", sum("amount").over(window_unbounded)
).withColumn(
    "running_average", avg("amount").over(window_unbounded)
).withColumn(
    "transaction_count", count("*").over(window_unbounded)
).withColumn(
    "running_max", max("amount").over(window_unbounded)
).withColumn(
    "running_min", min("amount").over(window_unbounded)
)

df_running_totals.select("customer_id", "timestamp", "amount", 
                        "running_total", "running_average", "transaction_count").show()
# +-----------+-------------------+------+-------------+---------------+-----------------+
# |customer_id|          timestamp|amount|running_total|running_average|transaction_count|
# +-----------+-------------------+------+-------------+---------------+-----------------+
# |       C001|2024-01-01 10:00:00| 100.0|        100.0|          100.0|                1|
# |       C001|2024-01-02 14:30:00| 150.0|        250.0|          125.0|                2|
# |       C001|2024-01-03 09:15:00| 200.0|        450.0|          150.0|                3|
# |       C001|2024-01-04 16:45:00|  75.0|        525.0|         131.25|                4|
# |       C002|2024-01-01 11:00:00| 250.0|        250.0|          250.0|                1|
# |       C002|2024-01-02 13:20:00| 180.0|        430.0|          215.0|                2|
# +-----------+-------------------+------+-------------+---------------+-----------------+
```

#### Moving Averages and Rolling Windows

```python
# Moving averages with different window sizes
window_moving_3 = Window.partitionBy("customer_id") \
                        .orderBy("timestamp") \
                        .rowsBetween(-2, Window.currentRow)  # 3-day moving window

window_moving_7 = Window.partitionBy("customer_id") \
                        .orderBy("timestamp") \
                        .rowsBetween(-6, Window.currentRow)  # 7-day moving window

df_moving_averages = transactions.withColumn(
    "moving_avg_3", avg("amount").over(window_moving_3)
).withColumn(
    "moving_avg_7", avg("amount").over(window_moving_7)
).withColumn(
    "moving_sum_3", sum("amount").over(window_moving_3)
).withColumn(
    "moving_max_3", max("amount").over(window_moving_3)
)

df_moving_averages.select("customer_id", "timestamp", "amount", 
                         "moving_avg_3", "moving_sum_3").show()
```

---

## Advanced Window Frame Specifications

### Frame Types

Window frames define which rows to include in the calculation relative to the current row.

#### Row-based Frames

```python
# Different row-based frame specifications

# Current row only
window_current = Window.partitionBy("customer_id") \
                      .orderBy("timestamp") \
                      .rowsBetween(Window.currentRow, Window.currentRow)

# Previous 2 rows + current row
window_prev_2 = Window.partitionBy("customer_id") \
                     .orderBy("timestamp") \
                     .rowsBetween(-2, Window.currentRow)

# Current row + next 2 rows
window_next_2 = Window.partitionBy("customer_id") \
                     .orderBy("timestamp") \
                     .rowsBetween(Window.currentRow, 2)

# Centered window: previous 1 + current + next 1
window_centered = Window.partitionBy("customer_id") \
                        .orderBy("timestamp") \
                        .rowsBetween(-1, 1)

# Full partition (all rows)
window_full = Window.partitionBy("customer_id") \
                    .rowsBetween(Window.unboundedPreceding, Window.unboundedFollowing)
```

#### Range-based Frames

```python
# Range-based frames work with actual values rather than row positions
# Useful for time-series data

# Create hourly transaction data
hourly_data = spark.createDataFrame([
    ("C001", "2024-01-01 10:00:00", 100.0),
    ("C001", "2024-01-01 11:00:00", 150.0),
    ("C001", "2024-01-01 12:00:00", 200.0),
    ("C001", "2024-01-01 13:00:00", 175.0),
    ("C001", "2024-01-01 14:00:00", 125.0)
], ["customer_id", "timestamp", "amount"])

hourly_data = hourly_data.withColumn("timestamp", 
    to_timestamp(col("timestamp"), "yyyy-MM-dd HH:mm:ss"))

# Add hour of day for range-based window
hourly_data = hourly_data.withColumn("hour", hour("timestamp"))

# Range-based window: current hour ± 1 hour
window_range = Window.partitionBy("customer_id") \
                     .orderBy("hour") \
                     .rangeBetween(-1, 1)

df_range_window = hourly_data.withColumn(
    "avg_surrounding_hours", avg("amount").over(window_range)
)

df_range_window.select("customer_id", "hour", "amount", "avg_surrounding_hours").show()
```

### Dynamic Frame Specifications

```python
def create_dynamic_window(partition_cols, order_cols, window_size):
    """Create a dynamic window specification"""
    return Window.partitionBy(*partition_cols) \
                 .orderBy(*order_cols) \
                 .rowsBetween(-window_size + 1, Window.currentRow)

# Usage
dynamic_window_5 = create_dynamic_window(["customer_id"], ["timestamp"], 5)
dynamic_window_10 = create_dynamic_window(["customer_id"], ["timestamp"], 10)

df_dynamic = transactions.withColumn(
    "moving_avg_5", avg("amount").over(dynamic_window_5)
).withColumn(
    "moving_avg_10", avg("amount").over(dynamic_window_10)
)
```

---

## Practical Use Cases and Patterns

### 1. Time Series Analysis

#### Calculating Period-over-Period Changes

```python
# Month-over-month growth calculation
monthly_sales = spark.createDataFrame([
    ("2024-01", 10000),
    ("2024-02", 12000),
    ("2024-03", 11500),
    ("2024-04", 13200),
    ("2024-05", 14800)
], ["month", "sales"])

window_monthly = Window.orderBy("month")

df_growth = monthly_sales.withColumn(
    "previous_month_sales", lag("sales", 1).over(window_monthly)
).withColumn(
    "mom_change", col("sales") - lag("sales", 1).over(window_monthly)
).withColumn(
    "mom_growth_rate", 
    (col("sales") - lag("sales", 1).over(window_monthly)) / lag("sales", 1).over(window_monthly) * 100
)

df_growth.show()
```

#### Trend Analysis and Smoothing

```python
# Calculate trends using linear regression over windows
def calculate_trend(values_col, periods=3):
    """Calculate trend direction over specified periods"""
    window_trend = Window.orderBy("timestamp") \
                         .rowsBetween(-periods + 1, Window.currentRow)
    
    # Simple trend calculation using first and last values in window
    first_val = first(values_col).over(window_trend)
    last_val = last(values_col).over(window_trend)
    trend = (last_val - first_val) / periods
    
    return trend

# Apply trend calculation
df_with_trend = transactions.withColumn(
    "trend_3_period", calculate_trend(col("amount"), 3)
).withColumn(
    "trend_direction",
    when(col("trend_3_period") > 0, "Increasing")
    .when(col("trend_3_period") < 0, "Decreasing")
    .otherwise("Stable")
)
```

### 2. Ranking and Top-N Analysis

#### Top N per Group

```python
# Find top 3 highest-paid employees per department
top_n_window = Window.partitionBy("department").orderBy(desc("salary"))

top_employees = employees.withColumn("rank", row_number().over(top_n_window)) \
                        .filter(col("rank") <= 3)

top_employees.show()
```

#### Percentile Analysis

```python
# Calculate salary percentiles within each department
percentile_window = Window.partitionBy("department")

df_percentiles = employees.withColumn(
    "salary_percentile", percent_rank().over(Window.partitionBy("department").orderBy("salary"))
).withColumn(
    "salary_quartile", ntile(4).over(Window.partitionBy("department").orderBy("salary"))
).withColumn(
    "is_top_10_percent", col("salary_percentile") >= 0.9
)

df_percentiles.show()
```

### 3. Gap and Island Analysis

#### Finding Consecutive Sequences

```python
# Find consecutive login days for users
user_logins = spark.createDataFrame([
    ("user1", "2024-01-01"),
    ("user1", "2024-01-02"),
    ("user1", "2024-01-03"),
    ("user1", "2024-01-05"),  # Gap here
    ("user1", "2024-01-06"),
    ("user1", "2024-01-07")
], ["user_id", "login_date"])

user_logins = user_logins.withColumn("login_date", to_date(col("login_date")))

# Create sequence groups
window_user = Window.partitionBy("user_id").orderBy("login_date")

df_sequences = user_logins.withColumn(
    "row_num", row_number().over(window_user)
).withColumn(
    "date_rank", datediff(col("login_date"), lit("2024-01-01"))
).withColumn(
    "group_id", col("date_rank") - col("row_num")
)

# Find consecutive login streaks
streak_analysis = df_sequences.groupBy("user_id", "group_id") \
    .agg(
        min("login_date").alias("streak_start"),
        max("login_date").alias("streak_end"),
        count("*").alias("streak_length")
    )

streak_analysis.show()
```

### 4. Statistical Analysis

#### Outlier Detection using Z-Score

```python
# Calculate z-scores using window functions
stats_window = Window.partitionBy("department")

df_with_zscore = employees.withColumn(
    "dept_avg_salary", avg("salary").over(stats_window)
).withColumn(
    "dept_stddev_salary", stddev("salary").over(stats_window)
).withColumn(
    "salary_zscore", 
    (col("salary") - col("dept_avg_salary")) / col("dept_stddev_salary")
).withColumn(
    "is_outlier", abs(col("salary_zscore")) > 2.0  # |z-score| > 2
)

df_with_zscore.select("name", "department", "salary", "salary_zscore", "is_outlier").show()
```

#### Running Statistics and Control Charts

```python
# Calculate control chart statistics
control_window = Window.orderBy("timestamp") \
                       .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df_control_chart = transactions.withColumn(
    "cumulative_mean", avg("amount").over(control_window)
).withColumn(
    "cumulative_stddev", stddev("amount").over(control_window)
).withColumn(
    "upper_control_limit", col("cumulative_mean") + 3 * col("cumulative_stddev")
).withColumn(
    "lower_control_limit", col("cumulative_mean") - 3 * col("cumulative_stddev")
).withColumn(
    "out_of_control", 
    (col("amount") > col("upper_control_limit")) | 
    (col("amount") < col("lower_control_limit"))
)
```

---

## Performance Optimization for Window Functions

### 1. Partition Pruning

```python
# Optimize by reducing partition size
# Bad: Large partitions
large_partition_window = Window.partitionBy("country").orderBy("timestamp")

# Better: Smaller, more focused partitions
optimized_window = Window.partitionBy("country", "region", "city").orderBy("timestamp")
```

### 2. Minimize Window Recalculation

```python
# Reuse window specifications
base_window = Window.partitionBy("customer_id").orderBy("timestamp")

# Apply multiple functions to the same window
df_optimized = transactions.withColumn("row_num", row_number().over(base_window)) \
                          .withColumn("lag_amount", lag("amount", 1).over(base_window)) \
                          .withColumn("lead_amount", lead("amount", 1).over(base_window))

# Instead of creating separate windows for each function
```

### 3. Use Appropriate Frame Specifications

```python
# Use bounded frames when possible instead of unbounded
# Unbounded (potentially expensive)
unbounded_window = Window.partitionBy("customer_id") \
                         .orderBy("timestamp") \
                         .rowsBetween(Window.unboundedPreceding, Window.unboundedFollowing)

# Bounded (more efficient)
bounded_window = Window.partitionBy("customer_id") \
                       .orderBy("timestamp") \
                       .rowsBetween(-30, Window.currentRow)  # Last 30 rows
```

### 4. Consider Alternative Approaches

```python
# Sometimes self-joins or aggregations might be more efficient
# Window function approach
window_sum = Window.partitionBy("department")
df_window = employees.withColumn("dept_total_salary", sum("salary").over(window_sum))

# Alternative aggregation approach
dept_totals = employees.groupBy("department").agg(sum("salary").alias("dept_total_salary"))
df_join = employees.join(dept_totals, "department")

# Choose based on your specific use case and data size
```

---

## Common Window Function Patterns

### 1. Custom Window Function Utilities

```python
def create_lag_features(df, partition_cols, order_cols, value_col, lags):
    """Create multiple lag features efficiently"""
    window_spec = Window.partitionBy(*partition_cols).orderBy(*order_cols)
    
    for lag_period in lags:
        df = df.withColumn(f"{value_col}_lag_{lag_period}", 
                          lag(value_col, lag_period).over(window_spec))
    return df

def create_rolling_features(df, partition_cols, order_cols, value_col, windows):
    """Create multiple rolling window features"""
    result_df = df
    
    for window_size in windows:
        window_spec = Window.partitionBy(*partition_cols) \
                           .orderBy(*order_cols) \
                           .rowsBetween(-window_size + 1, Window.currentRow)
        
        result_df = result_df.withColumn(f"{value_col}_rolling_mean_{window_size}",
                                       avg(value_col).over(window_spec)) \
                           .withColumn(f"{value_col}_rolling_sum_{window_size}",
                                     sum(value_col).over(window_spec))
    return result_df

# Usage
df_with_lags = create_lag_features(transactions, ["customer_id"], ["timestamp"], "amount", [1, 3, 7])
df_with_rolling = create_rolling_features(df_with_lags, ["customer_id"], ["timestamp"], "amount", [3, 7, 30])
```

### 2. Business Logic Patterns

```python
# Customer Lifetime Value calculation
def calculate_clv_features(df):
    """Calculate customer lifetime value features using window functions"""
    customer_window = Window.partitionBy("customer_id").orderBy("transaction_date")
    
    return df.withColumn("days_since_first_purchase",
                        datediff(col("transaction_date"), 
                               first("transaction_date").over(customer_window))) \
           .withColumn("total_spent_to_date",
                      sum("amount").over(
                          Window.partitionBy("customer_id")
                                .orderBy("transaction_date")
                                .rowsBetween(Window.unboundedPreceding, Window.currentRow)
                      )) \
           .withColumn("avg_order_value",
                      avg("amount").over(customer_window)) \
           .withColumn("transaction_frequency",
                      count("*").over(customer_window))

# Cohort analysis
def cohort_analysis(df):
    """Perform cohort analysis using window functions"""
    # Define first purchase month as cohort
    first_purchase_window = Window.partitionBy("customer_id")
    
    return df.withColumn("first_purchase_month",
                        date_trunc("month", first("transaction_date").over(first_purchase_window))) \
           .withColumn("transaction_month",
                      date_trunc("month", col("transaction_date"))) \
           .withColumn("period_number",
                      months_between(col("transaction_month"), col("first_purchase_month")))
```

---

## Error Handling and Best Practices

### 1. Handling Null Values in Windows

```python
# Handle nulls appropriately in window functions
safe_window = Window.partitionBy("customer_id").orderBy("timestamp")

df_safe = transactions.withColumn(
    "safe_lag_amount", 
    coalesce(lag("amount", 1).over(safe_window), lit(0))
).withColumn(
    "safe_moving_avg",
    when(count("amount").over(safe_window) >= 3,
         avg("amount").over(safe_window))
    .otherwise(col("amount"))
)
```

### 2. Performance Monitoring

```python
# Monitor window function performance
def monitor_window_performance(df, operation_name):
    """Monitor performance of window operations"""
    start_time = time.time()
    
    # Force evaluation
    count = df.count()
    
    end_time = time.time()
    print(f"{operation_name}: {count} rows processed in {end_time - start_time:.2f} seconds")
    
    return df

# Usage
df_result = monitor_window_performance(
    df_with_windows.cache(),  # Cache for performance
    "Window function calculations"
)
```

---

## Related Notes

- [[PySpark Core Concepts]] - Understanding DataFrames and transformations
- [[PySpark Data Operations]] - Basic data operations that lead to window functions
- [[PySpark Performance Optimization]] - Optimizing window function performance
- [[PySpark Functions & Methods]] - Complete reference of available functions

---

## Tags

#pyspark #window-functions #analytics #ranking #time-series #aggregations #performance #advanced-analytics

---

_Last Updated: 2024-08-20_