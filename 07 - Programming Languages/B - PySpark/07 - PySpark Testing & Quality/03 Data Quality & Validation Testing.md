
#pyspark #testing #data-quality #validation #completeness #accuracy #business-rules #data-profiling #anomaly-detection #quality-metrics #data-governance

### Data Completeness Tests

**Comprehensive Completeness Validation**

```python
class DataQualityValidator:
    """Data quality validation utilities"""
    
    @staticmethod
    def check_completeness(df, required_columns: list, threshold: float = 0.95):
        """Check data completeness for required columns"""
        total_rows = df.count()
        completeness_report = {}
        
        for column in required_columns:
            if column not in df.columns:
                completeness_report[column] = {"error": "Column not found"}
                continue
                
            non_null_count = df.filter(col(column).isNotNull()).count()
            completeness_ratio = non_null_count / total_rows if total_rows > 0 else 0
            
            completeness_report[column] = {
                "completeness_ratio": completeness_ratio,
                "passes_threshold": completeness_ratio >= threshold,
                "null_count": total_rows - non_null_count,
                "total_count": total_rows
            }
        
        return completeness_report

def test_data_completeness(spark):
    """Test data completeness validation"""
    # Create test data with missing values
    test_data = [
        (1, "Alice", 25, "alice@email.com"),
        (2, "Bob", None, "bob@email.com"),  # Missing age
        (3, "Charlie", 35, None),           # Missing email
        (4, None, 40, "dave@email.com")     # Missing name
    ]
    
    schema = ["id", "name", "age", "email"]
    df = spark.createDataFrame(test_data, schema)
    
    # Validate completeness
    validator = DataQualityValidator()
    report = validator.check_completeness(df, ["id", "name", "age", "email"], 0.8)
    
    # Assertions
    assert report["id"]["passes_threshold"] == True    # 100% complete
    assert report["name"]["passes_threshold"] == False # 75% complete
    assert report["age"]["passes_threshold"] == False  # 75% complete
    assert report["email"]["passes_threshold"] == False # 75% complete
```

### Business Rule Validation

**Complex Business Rules Testing**

```python
def validate_business_rules(df):
    """Validate complex business rules"""
    validations = []
    
    # Rule 1: Age must be between 0 and 150
    age_validation = df.filter(
        (col("age") < 0) | (col("age") > 150)
    ).count()
    validations.append(("age_range", age_validation == 0))
    
    # Rule 2: Email must contain @ symbol
    email_validation = df.filter(
        col("email").rlike("^[^@]+@[^@]+\\.[^@]+$")
    ).count()
    total_emails = df.filter(col("email").isNotNull()).count()
    validations.append(("email_format", email_validation == total_emails))
    
    # Rule 3: ID must be unique
    total_rows = df.count()
    unique_ids = df.select("id").distinct().count()
    validations.append(("id_uniqueness", total_rows == unique_ids))
    
    return validations

def test_business_rules_validation(spark):
    """Test business rules validation"""
    # Valid data
    valid_data = [
        (1, "Alice", 25, "alice@example.com"),
        (2, "Bob", 30, "bob@example.com")
    ]
    
    # Invalid data
    invalid_data = [
        (1, "Alice", -5, "invalid-email"),      # Invalid age and email
        (1, "Bob", 200, "bob@example.com"),     # Duplicate ID, invalid age
        (3, "Charlie", 35, None)                # Missing email
    ]
    
    schema = ["id", "name", "age", "email"]
    
    # Test valid data
    valid_df = spark.createDataFrame(valid_data, schema)
    valid_results = validate_business_rules(valid_df)
    
    for rule_name, is_valid in valid_results:
        assert is_valid, f"Valid data failed rule: {rule_name}"
    
    # Test invalid data
    invalid_df = spark.createDataFrame(invalid_data, schema)
    invalid_results = validate_business_rules(invalid_df)
    
    # Should fail multiple rules
    failed_rules = [rule for rule, is_valid in invalid_results if not is_valid]
    assert len(failed_rules) > 0, "Invalid data passed all rules"
```
