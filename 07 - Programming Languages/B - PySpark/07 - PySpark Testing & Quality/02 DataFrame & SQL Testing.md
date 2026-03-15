
#pyspark #testing #dataframe #sql #schema-validation #data-transformation #window-functions #joins #aggregations #udf-testing #spark-sql
### Schema Validation Testing

**Schema validation** is critical in data engineering because schema changes are one of the most common causes of pipeline failures. Unlike application code where type errors are caught at compile time, schema mismatches in data pipelines often only surface when processing real data, potentially hours into a long-running job.

**Schema Comparison Utilities**

```python
from pyspark.sql.types import StructType, DataType

class SchemaValidator:
    """Utility class for schema validation
    
    Why Schema Validation Matters:
    1. Early Detection: Catch schema issues before processing TB of data
    2. Data Quality: Ensure downstream systems receive expected data structure
    3. Pipeline Reliability: Prevent cascading failures from schema drift
    4. Documentation: Schema tests serve as living documentation
    """
    
    @staticmethod
    def compare_schemas(schema1: StructType, schema2: StructType) -> bool:
        """Compare two schemas for equality
        
        Comparison Strategy:
        - Field count must match (no missing/extra columns)
        - Field names must match exactly (case-sensitive)
        - Data types must be identical (Int vs Long matters!)
        - Nullability must match (prevents runtime NullPointerExceptions)
        """
        if len(schema1.fields) != len(schema2.fields):
            return False
        
        for field1, field2 in zip(schema1.fields, schema2.fields):
            if (field1.name != field2.name or 
                field1.dataType != field2.dataType or 
                field1.nullable != field2.nullable):
                return False
        
        return True
    
    @staticmethod
    def validate_required_columns(df, required_columns: list):
        """Validate that required columns exist
        
        Use Case: API contracts between data teams
        - Upstream team promises certain columns will always exist
        - Downstream team validates this promise in tests
        - Catches breaking changes before they reach production
        """
        missing_columns = set(required_columns) - set(df.columns)
        if missing_columns:
            raise ValueError(f"Missing required columns: {missing_columns}")
        return True
    
    @staticmethod
    def get_schema_diff(schema1: StructType, schema2: StructType) -> dict:
        """Get detailed differences between schemas
        
        Returns human-readable diff for debugging schema mismatches
        """
        diff = {
            "missing_fields": [],
            "extra_fields": [],
            "type_mismatches": [],
            "nullability_mismatches": []
        }
        
        schema1_fields = {f.name: f for f in schema1.fields}
        schema2_fields = {f.name: f for f in schema2.fields}
        
        # Find missing and extra fields
        diff["missing_fields"] = list(set(schema1_fields.keys()) - set(schema2_fields.keys()))
        diff["extra_fields"] = list(set(schema2_fields.keys()) - set(schema1_fields.keys()))
        
        # Find type and nullability mismatches
        common_fields = set(schema1_fields.keys()) & set(schema2_fields.keys())
        for field_name in common_fields:
            field1, field2 = schema1_fields[field_name], schema2_fields[field_name]
            
            if field1.dataType != field2.dataType:
                diff["type_mismatches"].append({
                    "field": field_name,
                    "expected": str(field1.dataType),
                    "actual": str(field2.dataType)
                })
            
            if field1.nullable != field2.nullable:
                diff["nullability_mismatches"].append({
                    "field": field_name,
                    "expected_nullable": field1.nullable,
                    "actual_nullable": field2.nullable
                })
        
        return diff

def test_schema_validation(spark):
    """Test schema validation functionality
    
    Testing Strategy:
    1. Define expected schema as "contract"
    2. Create test DataFrame matching contract
    3. Validate contract is enforced
    4. Test that violations are caught
    """
    from pyspark.sql.types import StringType, IntegerType
    
    # Expected schema (this would be your API contract)
    expected_schema = StructType([
        StructField("id", IntegerType(), False),  # Not nullable
        StructField("name", StringType(), True)   # Nullable
    ])
    
    # Create DataFrame with expected schema
    data = [(1, "Alice"), (2, "Bob")]
    df = spark.createDataFrame(data, expected_schema)
    
    # Validate schema matches expectation
    assert SchemaValidator.compare_schemas(df.schema, expected_schema)
    assert SchemaValidator.validate_required_columns(df, ["id", "name"])
    
    # Test schema mismatch detection
    wrong_schema = StructType([
        StructField("id", StringType(), False),  # Wrong type!
        StructField("name", StringType(), True)
    ])
    
    diff = SchemaValidator.get_schema_diff(expected_schema, wrong_schema)
    assert len(diff["type_mismatches"]) == 1
    assert diff["type_mismatches"][0]["field"] == "id"

def test_schema_evolution_compatibility(spark):
    """Test backward compatibility with schema evolution
    
    Real-world Scenario:
    - Data source adds new optional column
    - Should not break existing downstream consumers
    - But removal of columns should be detected
    """
    # Original schema
    v1_schema = StructType([
        StructField("id", IntegerType(), False),
        StructField("name", StringType(), True)
    ])
    
    # Evolved schema with additional column (backward compatible)
    v2_schema = StructType([
        StructField("id", IntegerType(), False),
        StructField("name", StringType(), True),
        StructField("email", StringType(), True)  # New optional field
    ])
    
    # Test that v2 is backward compatible with v1
    validator = SchemaValidator()
    
    # Create DataFrames
    v1_data = [(1, "Alice"), (2, "Bob")]
    v2_data = [(1, "Alice", "alice@example.com"), (2, "Bob", None)]
    
    v1_df = spark.createDataFrame(v1_data, v1_schema)
    v2_df = spark.createDataFrame(v2_data, v2_schema)
    
    # v2 should contain all required v1 columns
    assert validator.validate_required_columns(v2_df, ["id", "name"])
    
    # But v1 should not contain v2 columns
    with pytest.raises(ValueError):
        validator.validate_required_columns(v1_df, ["id", "name", "email"])
```

**Production Benefits:**

- **Early Warning**: Catch schema changes before they break downstream jobs
- **Documentation**: Tests serve as executable contracts between teams
- **Debugging**: Detailed diffs help quickly identify what changed
- **Automation**: Can be integrated into CI/CD to block incompatible changes

### Data Transformation Testing

**Data transformation testing** is where most PySpark bugs hide. Unlike simple function testing, transformations involve complex SQL logic, window functions, and aggregations that can fail in subtle ways. The key is testing not just the happy path, but also edge cases like empty partitions, null values, and boundary conditions that commonly occur in real data.

**Testing Complex Transformations**

```python
def test_complex_aggregation(spark):
    """Test complex aggregation logic
    
    Why This Test Pattern Works:
    1. Controlled Input: We know exactly what data goes in
    2. Expected Output: We can calculate expected results manually
    3. Edge Cases: Includes multiple categories, dates, and stores
    4. Realistic Complexity: Tests groupBy + multiple aggregations + conditional logic
    
    What This Test Catches:
    - Wrong aggregation functions (sum vs avg)
    - Incorrect grouping logic
    - Window function boundary errors
    - Null handling issues in aggregations
    """
    from pyspark.sql import functions as F
    
    # Create test data with known patterns
    sales_data = [
        ("2023-01-01", "Electronics", 1000, "A"),  # Store A, day 1
        ("2023-01-01", "Electronics", 1500, "B"),  # Store B, day 1 
        ("2023-01-02", "Clothing", 800, "A"),      # Store A, day 2, different category
        ("2023-01-02", "Electronics", 1200, "A"),  # Store A, day 2, electronics again
        ("2023-01-02", "Electronics", 0, "C"),     # Edge case: zero amount
        ("2023-01-03", "Electronics", None, "A")   # Edge case: null amount
    ]
    
    schema = ["date", "category", "amount", "store"]
    df = spark.createDataFrame(sales_data, schema)
    
    # Apply complex transformation that might exist in production
    result = df.groupBy("date", "category") \
               .agg(
                   F.sum("amount").alias("total_sales"),
                   F.count("store").alias("transaction_count"),
                   F.avg("amount").alias("avg_transaction"),
                   F.countDistinct("store").alias("unique_stores")
               ) \
               .withColumn("sales_tier", 
                          F.when(F.col("total_sales") > 2000, "High")
                           .when(F.col("total_sales") > 1000, "Medium")
                           .otherwise("Low"))
    
    # Validate specific results we can calculate manually
    electronics_2023_01_01 = result.filter(
        (F.col("date") == "2023-01-01") & 
        (F.col("category") == "Electronics")
    ).collect()[0]
    
    # Manual calculation: 1000 + 1500 = 2500
    assert electronics_2023_01_01.total_sales == 2500
    # Manual calculation: 2 transactions 
    assert electronics_2023_01_01.transaction_count == 2
    # Manual calculation: (1000 + 1500) / 2 = 1250
    assert electronics_2023_01_01.avg_transaction == 1250.0
    # Manual calculation: stores A and B = 2 unique stores
    assert electronics_2023_01_01.unique_stores == 2
    # Manual calculation: 2500 > 2000 = "High"
    assert electronics_2023_01_01.sales_tier == "High"
    
    # Test edge case: date with null amounts
    electronics_2023_01_03 = result.filter(
        (F.col("date") == "2023-01-03") & 
        (F.col("category") == "Electronics")
    ).collect()
    
    # Spark sum() ignores nulls, so sum of [None] = None, not 0
    if electronics_2023_01_03:  # Row might not exist if all nulls filtered
        assert electronics_2023_01_03[0].total_sales is None
    
    # Test boundary condition for sales tier logic
    clothing_result = result.filter(F.col("category") == "Clothing").collect()[0]
    assert clothing_result.total_sales == 800  # Only one transaction
    assert clothing_result.sales_tier == "Low"  # 800 <= 1000

def test_window_function_logic(spark):
    """Test window functions which are notoriously tricky to get right
    
    Window Function Gotchas:
    - Partition boundaries: Data in different partitions don't see each other
    - Order by clauses: Wrong ordering gives wrong running totals
    - Frame specifications: unboundedPreceding vs currentRow matters
    - Null handling: nulls can appear first or last depending on ordering
    """
    from pyspark.sql.window import Window
    
    # Sales data with temporal ordering
    daily_sales = [
        ("2023-01-01", 1000),
        ("2023-01-02", 1500), 
        ("2023-01-03", 800),
        ("2023-01-04", 1200),
        ("2023-01-05", 0),     # Edge case: zero sales day
    ]
    
    df = spark.createDataFrame(daily_sales, ["date", "sales"])
    
    # Define window specification for running totals
    window_spec = Window.orderBy("date").rowsBetween(Window.unboundedPreceding, Window.currentRow)
    
    # Apply window function
    result = df.withColumn("running_total", F.sum("sales").over(window_spec)) \
               .withColumn("prev_day_sales", F.lag("sales", 1).over(Window.orderBy("date"))) \
               .withColumn("sales_growth", 
                          F.when(F.col("prev_day_sales").isNull(), None)
                           .otherwise(F.col("sales") - F.col("prev_day_sales")))
    
    # Collect for detailed assertions
    results = result.orderBy("date").collect()
    
    # Test running total calculation
    assert results[0].running_total == 1000        # Day 1: 1000
    assert results[1].running_total == 2500        # Day 2: 1000 + 1500
    assert results[2].running_total == 3300        # Day 3: 2500 + 800
    assert results[3].running_total == 4500        # Day 4: 3300 + 1200
    assert results[4].running_total == 4500        # Day 5: 4500 + 0
    
    # Test lag function (previous day sales)
    assert results[0].prev_day_sales is None       # No previous day
    assert results[1].prev_day_sales == 1000       # Previous = day 1
    assert results[2].prev_day_sales == 1500       # Previous = day 2
    
    # Test derived calculation (growth)
    assert results[0].sales_growth is None         # No previous day
    assert results[1].sales_growth == 500          # 1500 - 1000
    assert results[2].sales_growth == -700         # 800 - 1500 
    assert results[4].sales_growth == -1200        # 0 - 1200
```

**Benefits of This Testing Approach:**

- **Manual Verification**: Expected results can be calculated by hand, making tests self-documenting
- **Edge Case Coverage**: Includes nulls, zeros, and boundary conditions that break production systems
- **Realistic Complexity**: Tests actual transformation patterns used in data pipelines
- **Debugging Friendly**: When tests fail, it's easy to see what went wrong and why

### Custom Function Testing

**User Defined Functions (UDFs)** are often necessary for complex business logic that can't be expressed with built-in Spark functions. However, UDFs are also the most common source of performance problems and runtime errors in PySpark applications. Thorough testing of UDFs is critical because they break Spark's optimization capabilities and require serialization across the cluster.

**Testing User Defined Functions (UDFs)**

python

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType, IntegerType

# Define UDF with comprehensive business logic
@udf(returnType=StringType())
def categorize_age(age):
    """Categorize age into groups
    
    Business Rules:
    - Minor: < 18 (legal drinking age consideration)
    - Adult: 18-64 (working age population)
    - Senior: 65+ (retirement age)
    - Unknown: null values (data quality issue)
    
    Why UDF Instead of Built-in Functions:
    - Complex business rules that change frequently
    - Multiple conditional branches that are hard to read with when/otherwise
    - Need for consistent logic across multiple pipelines
    """
    if age is None:
        return "Unknown"
    elif age < 0:  # Invalid data
        return "Invalid"
    elif age < 18:
        return "Minor"
    elif age <= 64:  # Note: inclusive of 64
        return "Adult"
    else:
        return "Senior"

def test_age_categorization_udf(spark):
    """Test age categorization UDF with comprehensive edge cases
    
    Testing Strategy:
    1. Test each business rule boundary (17/18, 64/65)
    2. Test edge cases (null, negative, extreme values)
    3. Test data type handling (int vs float)
    4. Verify UDF works with DataFrame operations
    """
    # Test data including all edge cases
    test_data = [
        (10,),      # Minor
        (17,),      # Minor boundary
        (18,),      # Adult boundary  
        (25,),      # Adult middle
        (64,),      # Adult boundary
        (65,),      # Senior boundary
        (90,),      # Senior
        (None,),    # Null handling
        (-5,),      # Invalid data
        (150,)      # Extreme but valid
    ]
    df = spark.createDataFrame(test_data, ["age"])
    
    # Apply UDF
    result = df.withColumn("age_category", categorize_age("age"))
    
    # Collect and validate results
    results = result.collect()
    
    # Test business rule boundaries specifically
    age_to_category = {row.age: row.age_category for row in results}
    
    assert age_to_category[10] == "Minor"
    assert age_to_category[17] == "Minor"       # Boundary test
    assert age_to_category[18] == "Adult"       # Boundary test  
    assert age_to_category[64] == "Adult"       # Boundary test
    assert age_to_category[65] == "Senior"      # Boundary test
    assert age_to_category[None] == "Unknown"   # Null handling
    assert age_to_category[-5] == "Invalid"     # Invalid data
    assert age_to_category[150] == "Senior"     # Extreme case

def test_udf_error_handling(spark):
    """Test UDF error handling and data type robustness
    
    UDF Error Scenarios:
    - Wrong data types passed to UDF
    - UDF throws exceptions during processing
    - Large datasets causing serialization issues
    """
    # Test with string data that should be numeric
    invalid_data = [("twenty",), ("30",), (None,)]
    df = spark.createDataFrame(invalid_data, ["age_str"])
    
    # UDF that handles type conversion errors gracefully
    @udf(returnType=StringType())
    def safe_age_categorizer(age_str):
        try:
            if age_str is None:
                return "Unknown"
            age = int(age_str)
            return categorize_age(age)
        except (ValueError, TypeError):
            return "Invalid_Format"
    
    result = df.withColumn("category", safe_age_categorizer("age_str"))
    results = result.collect()
    
    assert results[0].category == "Invalid_Format"  # "twenty"
    assert results[1].category == "Adult"           # "30" 
    assert results[2].category == "Unknown"         # None

# Testing UDF performance
def test_udf_performance(spark, benchmark_data):
    """Test UDF performance vs built-in functions
    
    Performance Testing Strategy:
    1. Create large enough dataset to see performance differences
    2. Compare UDF vs equivalent built-in functions
    3. Measure execution time for both approaches
    4. Assert performance is within acceptable bounds
    
    Why This Matters:
    - UDFs are 5-100x slower than built-in functions
    - UDFs prevent Spark SQL optimizations
    - UDFs require Python/JVM serialization overhead
    """
    import time
    
    # Create larger dataset for meaningful performance comparison
    if benchmark_data is None:
        benchmark_data = spark.range(100000).select(
            (F.rand() * 100).cast("int").alias("age")
        )
    
    # Test UDF performance
    start_time = time.time()
    udf_result = benchmark_data.withColumn("category_udf", categorize_age("age"))
    udf_count = udf_result.count()  # Trigger execution
    udf_time = time.time() - start_time
    
    # Test equivalent built-in function performance
    start_time = time.time()
    builtin_result = benchmark_data.withColumn(
        "category_builtin",
        F.when(F.col("age").isNull(), "Unknown")
         .when(F.col("age") < 0, "Invalid")
         .when(F.col("age") < 18, "Minor")
         .when(F.col("age") <= 64, "Adult")
         .otherwise("Senior")
    )
    builtin_count = builtin_result.count()  # Trigger execution
    builtin_time = time.time() - start_time
    
    # Verify both produce same results (data quality check)
    assert udf_count == builtin_count
    
    # Performance assertion - UDF should be slower but not excessively
    performance_ratio = udf_time / builtin_time
    assert performance_ratio < 10  # UDF should be < 10x slower for this simple logic
    
    print(f"UDF time: {udf_time:.2f}s, Built-in time: {builtin_time:.2f}s")
    print(f"Performance ratio: {performance_ratio:.1f}x")
    
    # If UDF is more than 5x slower, consider refactoring to built-in functions
    if performance_ratio > 5:
        print("WARNING: UDF is significantly slower, consider using built-in functions")

def test_udf_with_complex_types(spark):
    """Test UDF with complex data types (arrays, structs)
    
    Advanced UDF Scenarios:
    - Processing array columns
    - Working with nested struct data
    - Returning complex types from UDFs
    """
    from pyspark.sql.types import ArrayType, StructType, StructField
    
    # UDF that processes array data
    @udf(returnType=IntegerType())
    def count_valid_scores(scores_array):
        """Count non-null scores in array"""
        if scores_array is None:
            return 0
        return len([score for score in scores_array if score is not None])
    
    # Test data with arrays
    array_data = [
        ([85, 90, 78],),           # All valid scores
        ([85, None, 78],),         # One null score
        (None,),                   # Null array
        ([],)                      # Empty array
    ]
    
    array_schema = StructType([
        StructField("scores", ArrayType(IntegerType(), True), True)
    ])
    
    df = spark.createDataFrame(array_data, array_schema)
    result = df.withColumn("valid_count", count_valid_scores("scores"))
    
    results = result.collect()
    assert results[0].valid_count == 3  # All valid
    assert results[1].valid_count == 2  # One null
    assert results[2].valid_count == 0  # Null array
    assert results[3].valid_count == 0  # Empty array
```

**Key UDF Testing Principles:**

- **Boundary Testing**: Test every conditional boundary in your UDF logic
- **Error Handling**: UDFs should gracefully handle invalid inputs and data type issues
- **Performance Awareness**: Always benchmark UDFs against built-in alternatives
- **Complex Types**: Test UDFs with arrays, structs, and nested data if your pipeline uses them
- **Null Safety**: Ensure UDFs handle null inputs correctly (Spark passes nulls frequently)```

---
