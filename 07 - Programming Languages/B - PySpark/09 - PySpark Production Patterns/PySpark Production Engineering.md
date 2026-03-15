
#pyspark #production #engineering #testing #logging #monitoring #deployment #error-handling #reliability

## Overview

This note covers essential production engineering practices for PySpark applications, including error handling, testing strategies, logging, monitoring, deployment patterns, and reliability engineering. These practices are crucial for building robust, maintainable, and scalable data pipelines.

---

## Error Handling and Resilience

### Structured Error Handling Framework

```python
import logging
import traceback
from datetime import datetime
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Dict, Any
import json

class ErrorSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class ErrorCategory(Enum):
    DATA_QUALITY = "data_quality"
    SCHEMA_MISMATCH = "schema_mismatch"
    RESOURCE_EXHAUSTION = "resource_exhaustion"
    NETWORK_FAILURE = "network_failure"
    CONFIGURATION_ERROR = "configuration_error"
    BUSINESS_LOGIC = "business_logic"
    UNKNOWN = "unknown"

@dataclass
class SparkError:
    """Structured error information"""
    error_id: str
    timestamp: datetime
    severity: ErrorSeverity
    category: ErrorCategory
    message: str
    details: Dict[str, Any]
    stack_trace: Optional[str] = None
    recovery_action: Optional[str] = None

class SparkErrorHandler:
    """Comprehensive error handling for Spark applications"""
    
    def __init__(self, app_name: str, enable_alerts: bool = True):
        self.app_name = app_name
        self.enable_alerts = enable_alerts
        self.error_log = []
        self.setup_logging()
    
    def setup_logging(self):
        """Setup structured logging"""
        self.logger = logging.getLogger(f"spark.{self.app_name}")
        self.logger.setLevel(logging.INFO)
        
        # Console handler with structured format
        console_handler = logging.StreamHandler()
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        console_handler.setFormatter(formatter)
        self.logger.addHandler(console_handler)
        
        # File handler for error logs
        file_handler = logging.FileHandler(f"/tmp/spark_{self.app_name}_errors.log")
        file_handler.setFormatter(formatter)
        file_handler.setLevel(logging.ERROR)
        self.logger.addHandler(file_handler)
    
    def handle_error(self, exception: Exception, context: Dict[str, Any] = None) -> SparkError:
        """Handle errors with structured logging and recovery"""
        
        error_id = f"{self.app_name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}_{len(self.error_log)}"
        
        # Categorize error
        category = self._categorize_error(exception)
        severity = self._determine_severity(exception, category)
        
        # Create structured error
        spark_error = SparkError(
            error_id=error_id,
            timestamp=datetime.now(),
            severity=severity,
            category=category,
            message=str(exception),
            details=context or {},
            stack_trace=traceback.format_exc(),
            recovery_action=self._suggest_recovery_action(category, exception)
        )
        
        # Log error
        self._log_error(spark_error)
        
        # Store for analysis
        self.error_log.append(spark_error)
        
        # Handle based on severity
        if severity in [ErrorSeverity.HIGH, ErrorSeverity.CRITICAL]:
            self._handle_critical_error(spark_error)
        
        return spark_error
    
    def _categorize_error(self, exception: Exception) -> ErrorCategory:
        """Categorize errors for better handling"""
        
        error_msg = str(exception).lower()
        error_type = type(exception).__name__
        
        if "schema" in error_msg or "column" in error_msg:
            return ErrorCategory.SCHEMA_MISMATCH
        elif "memory" in error_msg or "disk" in error_msg or "space" in error_msg:
            return ErrorCategory.RESOURCE_EXHAUSTION
        elif "connection" in error_msg or "network" in error_msg or "timeout" in error_msg:
            return ErrorCategory.NETWORK_FAILURE
        elif "config" in error_msg or "property" in error_msg:
            return ErrorCategory.CONFIGURATION_ERROR
        elif error_type in ["ValueError", "AssertionError"]:
            return ErrorCategory.DATA_QUALITY
        else:
            return ErrorCategory.UNKNOWN
    
    def _determine_severity(self, exception: Exception, category: ErrorCategory) -> ErrorSeverity:
        """Determine error severity"""
        
        critical_categories = [ErrorCategory.RESOURCE_EXHAUSTION, ErrorCategory.CONFIGURATION_ERROR]
        high_categories = [ErrorCategory.SCHEMA_MISMATCH, ErrorCategory.NETWORK_FAILURE]
        
        if category in critical_categories:
            return ErrorSeverity.CRITICAL
        elif category in high_categories:
            return ErrorSeverity.HIGH
        elif category == ErrorCategory.DATA_QUALITY:
            return ErrorSeverity.MEDIUM
        else:
            return ErrorSeverity.LOW
    
    def _suggest_recovery_action(self, category: ErrorCategory, exception: Exception) -> str:
        """Suggest recovery actions based on error category"""
        
        suggestions = {
            ErrorCategory.SCHEMA_MISMATCH: "Check schema compatibility, update schema mapping",
            ErrorCategory.RESOURCE_EXHAUSTION: "Increase memory/disk allocation, optimize partition size",
            ErrorCategory.NETWORK_FAILURE: "Retry with exponential backoff, check network connectivity",
            ErrorCategory.CONFIGURATION_ERROR: "Verify configuration parameters, check environment setup",
            ErrorCategory.DATA_QUALITY: "Implement data validation, add data cleaning steps",
            ErrorCategory.BUSINESS_LOGIC: "Review business logic implementation",
            ErrorCategory.UNKNOWN: "Review logs and stack trace for debugging"
        }
        
        return suggestions.get(category, "Contact support team for assistance")
    
    def _log_error(self, error: SparkError):
        """Log error with appropriate level"""
        
        error_msg = f"[{error.error_id}] {error.message}"
        error_details = {
            "error_id": error.error_id,
            "category": error.category.value,
            "severity": error.severity.value,
            "details": error.details,
            "recovery_action": error.recovery_action
        }
        
        if error.severity == ErrorSeverity.CRITICAL:
            self.logger.critical(f"{error_msg} | Details: {json.dumps(error_details)}")
        elif error.severity == ErrorSeverity.HIGH:
            self.logger.error(f"{error_msg} | Details: {json.dumps(error_details)}")
        elif error.severity == ErrorSeverity.MEDIUM:
            self.logger.warning(f"{error_msg} | Details: {json.dumps(error_details)}")
        else:
            self.logger.info(f"{error_msg} | Details: {json.dumps(error_details)}")
    
    def _handle_critical_error(self, error: SparkError):
        """Handle critical errors with immediate attention"""
        
        if self.enable_alerts:
            # In production, this would send alerts via email, Slack, PagerDuty, etc.
            print(f"🚨 CRITICAL ERROR ALERT: {error.error_id}")
            print(f"   Application: {self.app_name}")
            print(f"   Message: {error.message}")
            print(f"   Recovery Action: {error.recovery_action}")
    
    def get_error_summary(self) -> Dict[str, Any]:
        """Get summary of errors for monitoring"""
        
        if not self.error_log:
            return {"total_errors": 0}
        
        error_counts = {}
        severity_counts = {}
        category_counts = {}
        
        for error in self.error_log:
            # Count by severity
            severity = error.severity.value
            severity_counts[severity] = severity_counts.get(severity, 0) + 1
            
            # Count by category
            category = error.category.value
            category_counts[category] = category_counts.get(category, 0) + 1
        
        return {
            "total_errors": len(self.error_log),
            "by_severity": severity_counts,
            "by_category": category_counts,
            "latest_error": self.error_log[-1].error_id if self.error_log else None
        }

# Usage example
error_handler = SparkErrorHandler("data_pipeline_v1")

def safe_spark_operation(operation_func, *args, **kwargs):
    """Wrapper for safe Spark operations"""
    try:
        return operation_func(*args, **kwargs)
    except Exception as e:
        context = {
            "operation": operation_func.__name__,
            "args": str(args)[:100],  # Truncate for logging
            "kwargs": str(kwargs)[:100]
        }
        error = error_handler.handle_error(e, context)
        
        # Decide whether to re-raise based on severity
        if error.severity == ErrorSeverity.CRITICAL:
            raise
        else:
            return None

# Example usage
def risky_data_operation(df):
    """Example operation that might fail"""
    return df.select("non_existent_column").count()

# Safe execution
result = safe_spark_operation(risky_data_operation, sample_df)
```

### Circuit Breaker Pattern

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing fast
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    """Circuit breaker pattern for Spark operations"""
    
    def __init__(self, failure_threshold=5, recovery_timeout=60, success_threshold=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        self.lock = Lock()
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker protection"""
        
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                    print(f"🔄 Circuit breaker transitioning to HALF_OPEN")
                else:
                    raise Exception(f"Circuit breaker is OPEN - failing fast. "
                                  f"Next attempt in {self._time_until_reset():.0f}s")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _should_attempt_reset(self):
        """Check if enough time has passed to attempt reset"""
        if self.last_failure_time is None:
            return True
        return time.time() - self.last_failure_time >= self.recovery_timeout
    
    def _time_until_reset(self):
        """Calculate time until next reset attempt"""
        if self.last_failure_time is None:
            return 0
        return self.recovery_timeout - (time.time() - self.last_failure_time)
    
    def _on_success(self):
        """Handle successful operation"""
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.success_threshold:
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
                    self.success_count = 0
                    print(f"✅ Circuit breaker reset to CLOSED")
            elif self.state == CircuitState.CLOSED:
                self.failure_count = 0
    
    def _on_failure(self):
        """Handle failed operation"""
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            self.success_count = 0
            
            if self.state in [CircuitState.CLOSED, CircuitState.HALF_OPEN]:
                if self.failure_count >= self.failure_threshold:
                    self.state = CircuitState.OPEN
                    print(f"🚨 Circuit breaker OPENED after {self.failure_count} failures")
    
    def get_status(self):
        """Get current circuit breaker status"""
        return {
            "state": self.state.value,
            "failure_count": self.failure_count,
            "success_count": self.success_count,
            "time_until_reset": self._time_until_reset() if self.state == CircuitState.OPEN else 0
        }

# Usage with Spark operations
jdbc_circuit_breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=30)

def read_database_with_circuit_breaker(query):
    """Read from database with circuit breaker protection"""
    
    def database_operation():
        return spark.read \
                   .format("jdbc") \
                   .option("url", "jdbc:postgresql://host:5432/db") \
                   .option("query", query) \
                   .option("user", "username") \
                   .option("password", "password") \
                   .load()
    
    return jdbc_circuit_breaker.call(database_operation)

# Example usage
try:
    df = read_database_with_circuit_breaker("SELECT * FROM users LIMIT 1000")
    print(f"Successfully read {df.count()} records")
except Exception as e:
    print(f"Database read failed: {e}")
    print(f"Circuit breaker status: {jdbc_circuit_breaker.get_status()}")
```

### Retry Mechanisms

```python
import random
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1, max_delay=60, 
                      backoff_factor=2, jitter=True, 
                      retryable_exceptions=(Exception,)):
    """Decorator for retrying operations with exponential backoff"""
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    result = func(*args, **kwargs)
                    if attempt > 0:
                        print(f"✅ {func.__name__} succeeded on attempt {attempt + 1}")
                    return result
                    
                except retryable_exceptions as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        print(f"❌ {func.__name__} failed after {max_retries + 1} attempts")
                        raise e
                    
                    # Calculate delay with exponential backoff
                    delay = min(base_delay * (backoff_factor ** attempt), max_delay)
                    
                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay = delay * (0.5 + random.random() * 0.5)
                    
                    print(f"⚠️  {func.__name__} failed on attempt {attempt + 1}, "
                          f"retrying in {delay:.1f}s: {str(e)[:100]}")
                    
                    time.sleep(delay)
            
            raise last_exception
        
        return wrapper
    return decorator

# Specific retry configurations for different operations
@retry_with_backoff(
    max_retries=5, 
    base_delay=2, 
    max_delay=120,
    retryable_exceptions=(ConnectionError, TimeoutError)
)
def read_from_external_service(url):
    """Read data from external service with retry logic"""
    # Simulate external service call
    if random.random() < 0.3:  # 30% failure rate for demo
        raise ConnectionError("Service temporarily unavailable")
    
    return spark.read.json(url)

@retry_with_backoff(
    max_retries=3,
    base_delay=1,
    retryable_exceptions=(Exception,)
)
def write_to_storage(df, path):
    """Write DataFrame to storage with retry logic"""
    df.write.mode("overwrite").parquet(path)
    return True

# Usage
try:
    df = read_from_external_service("s3://bucket/data.json")
    success = write_to_storage(df, "s3://bucket/processed/")
    print("Pipeline completed successfully")
except Exception as e:
    print(f"Pipeline failed: {e}")
```

---

## Testing Strategies

### Comprehensive Testing Framework

```python
import pytest
import pandas as pd
from pyspark.sql import SparkSession
from pyspark.sql.types import *
from pyspark.sql.functions import *
import tempfile
import shutil
from typing import List, Dict, Any

class SparkTestBase:
    """Base class for Spark tests with comprehensive utilities"""
    
    @pytest.fixture(scope="class")
    def spark(self):
        """Create Spark session for testing"""
        return SparkSession.builder \
            .appName("test_session") \
            .master("local[2]") \
            .config("spark.sql.shuffle.partitions", "2") \
            .config("spark.sql.adaptive.enabled", "false") \
            .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
            .getOrCreate()
    
    @pytest.fixture
    def temp_dir(self):
        """Create temporary directory for test files"""
        temp_path = tempfile.mkdtemp()
        yield temp_path
        shutil.rmtree(temp_path)
    
    @pytest.fixture
    def sample_customers(self, spark):
        """Sample customer data for testing"""
        schema = StructType([
            StructField("customer_id", StringType(), False),
            StructField("name", StringType(), True),
            StructField("email", StringType(), True),
            StructField("age", IntegerType(), True),
            StructField("signup_date", DateType(), True),
            StructField("is_active", BooleanType(), True)
        ])
        
        data = [
            ("C001", "John Doe", "john@email.com", 25, "2023-01-15", True),
            ("C002", "Jane Smith", "jane@email.com", 30, "2023-02-01", True),
            ("C003", "Bob Johnson", "bob@email.com", 35, "2023-01-20", False),
            ("C004", "Alice Brown", "alice@email.com", 28, "2023-03-01", True),
            ("C005", "Charlie Wilson", None, None, "2023-01-10", True)  # Test nulls
        ]
        
        return spark.createDataFrame(data, schema)
    
    @pytest.fixture  
    def sample_orders(self, spark):
        """Sample order data for testing"""
        schema = StructType([
            StructField("order_id", StringType(), False),
            StructField("customer_id", StringType(), False),
            StructField("product_id", StringType(), False),
            StructField("amount", DoubleType(), True),
            StructField("order_date", DateType(), True)
        ])
        
        data = [
            ("O001", "C001", "P001", 100.0, "2023-01-16"),
            ("O002", "C002", "P002", 150.0, "2023-02-02"),
            ("O003", "C001", "P001", 200.0, "2023-02-15"),
            ("O004", "C003", "P003", 75.0, "2023-02-10"),
            ("O005", "C004", "P002", 300.0, "2023-03-05")
        ]
        
        return spark.createDataFrame(data, schema)
    
    def assert_dataframes_equal(self, actual_df, expected_df, check_order=False):
        """Compare two DataFrames for equality"""
        
        # Check schemas
        assert actual_df.schema == expected_df.schema, \
            f"Schemas don't match:\nActual: {actual_df.schema}\nExpected: {expected_df.schema}"
        
        # Check row counts
        actual_count = actual_df.count()
        expected_count = expected_df.count()
        assert actual_count == expected_count, \
            f"Row counts don't match: actual={actual_count}, expected={expected_count}"
        
        if actual_count == 0:
            return  # Both are empty
        
        # Convert to Pandas for detailed comparison
        if check_order:
            actual_pd = actual_df.toPandas()
            expected_pd = expected_df.toPandas()
        else:
            # Sort by all columns for order-independent comparison
            sort_cols = actual_df.columns
            actual_pd = actual_df.orderBy(*sort_cols).toPandas()
            expected_pd = expected_df.orderBy(*sort_cols).toPandas()
        
        # Reset index for comparison
        actual_pd = actual_pd.reset_index(drop=True)
        expected_pd = expected_pd.reset_index(drop=True)
        
        try:
            pd.testing.assert_frame_equal(actual_pd, expected_pd, check_dtype=False)
        except AssertionError as e:
            print(f"\nDataFrame comparison failed:")
            print(f"Actual DataFrame:\n{actual_pd}")
            print(f"Expected DataFrame:\n{expected_pd}")
            raise e
    
    def assert_dataframe_properties(self, df, expected_count=None, 
                                  expected_columns=None, non_null_columns=None):
        """Assert various DataFrame properties"""
        
        if expected_count is not None:
            actual_count = df.count()
            assert actual_count == expected_count, \
                f"Expected {expected_count} rows, got {actual_count}"
        
        if expected_columns is not None:
            actual_columns = set(df.columns)
            expected_columns = set(expected_columns)
            assert actual_columns == expected_columns, \
                f"Column mismatch:\nActual: {actual_columns}\nExpected: {expected_columns}"
        
        if non_null_columns:
            for col_name in non_null_columns:
                null_count = df.filter(col(col_name).isNull()).count()
                assert null_count == 0, f"Column '{col_name}' has {null_count} null values"
    
    def create_test_data(self, spark, schema, data_rows):
        """Helper to create test DataFrames"""
        return spark.createDataFrame(data_rows, schema)

class TestDataTransformations(SparkTestBase):
    """Test data transformation functions"""
    
    def test_customer_aggregation(self, spark, sample_customers, sample_orders):
        """Test customer aggregation logic"""
        
        # Function under test
        def calculate_customer_metrics(customers_df, orders_df):
            """Calculate customer lifetime value and order metrics"""
            
            # Calculate order metrics per customer
            order_metrics = orders_df.groupBy("customer_id") \
                .agg(
                    count("order_id").alias("total_orders"),
                    sum("amount").alias("total_spent"),
                    avg("amount").alias("avg_order_value"),
                    max("order_date").alias("last_order_date")
                )
            
            # Join with customer data
            result = customers_df.join(order_metrics, "customer_id", "left") \
                .fillna({
                    "total_orders": 0,
                    "total_spent": 0.0,
                    "avg_order_value": 0.0
                }) \
                .withColumn("customer_segment",
                    when(col("total_spent") >= 250, "Premium")
                    .when(col("total_spent") >= 100, "Regular")
                    .otherwise("Basic")
                )
            
            return result
        
        # Execute function
        result_df = calculate_customer_metrics(sample_customers, sample_orders)
        
        # Assertions
        self.assert_dataframe_properties(
            result_df,
            expected_count=5,
            expected_columns=["customer_id", "name", "email", "age", "signup_date", 
                            "is_active", "total_orders", "total_spent", 
                            "avg_order_value", "last_order_date", "customer_segment"],
            non_null_columns=["customer_id", "customer_segment"]
        )
        
        # Test specific customer metrics
        c001_metrics = result_df.filter(col("customer_id") == "C001").collect()[0]
        assert c001_metrics["total_orders"] == 2
        assert c001_metrics["total_spent"] == 300.0
        assert c001_metrics["customer_segment"] == "Premium"
        
        # Test customer with no orders
        c005_metrics = result_df.filter(col("customer_id") == "C005").collect()[0]
        assert c005_metrics["total_orders"] == 0
        assert c005_metrics["customer_segment"] == "Basic"
    
    def test_data_quality_checks(self, spark, sample_customers):
        """Test data quality validation functions"""
        
        def validate_data_quality(df):
            """Validate data quality and return issues"""
            
            issues = []
            
            # Check for null emails in active customers
            null_email_active = df.filter(
                col("is_active") == True & col("email").isNull()
            ).count()
            
            if null_email_active > 0:
                issues.append(f"Found {null_email_active} active customers with null emails")
            
            # Check for invalid email formats
            invalid_emails = df.filter(
                col("email").isNotNull() & 
                ~col("email").rlike(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
            ).count()
            
            if invalid_emails > 0:
                issues.append(f"Found {invalid_emails} customers with invalid email formats")
            
            # Check for reasonable age ranges
            invalid_ages = df.filter(
                col("age").isNotNull() & 
                (col("age") < 0 | col("age") > 120)
            ).count()
            
            if invalid_ages > 0:
                issues.append(f"Found {invalid_ages} customers with invalid ages")
            
            return issues
        
        # Test with valid data
        issues = validate_data_quality(sample_customers)
        assert len(issues) == 1  # Only C005 has null email but is active
        assert "null emails" in issues[0]
        
        # Test with invalid data
        invalid_data = sample_customers.union(
            spark.createDataFrame([
                ("C006", "Invalid User", "invalid-email", 150, "2023-01-01", True)
            ], sample_customers.schema)
        )
        
        issues = validate_data_quality(invalid_data)
        assert len(issues) == 3  # null email, invalid email, invalid age

class TestPerformanceScenarios(SparkTestBase):
    """Test performance-related scenarios"""
    
    def test_large_dataset_processing(self, spark, temp_dir):
        """Test processing of larger datasets"""
        
        # Generate larger test dataset
        large_df = spark.range(10000).toDF("id") \
            .withColumn("category", (col("id") % 10).cast("string")) \
            .withColumn("value", (col("id") * 1.5).cast("double")) \
            .withColumn("is_valid", col("id") % 7 != 0)  # Some invalid records
        
        # Write to temporary location
        test_path = f"{temp_dir}/large_test_data"
        large_df.write.mode("overwrite").parquet(test_path)
        
        # Read and process
        def process_large_dataset(input_path):
            df = spark.read.parquet(input_path)
            
            return df.filter(col("is_valid") == True) \
                     .groupBy("category") \
                     .agg(
                         count("id").alias("count"),
                         avg("value").alias("avg_value"),
                         max("value").alias("max_value")
                     ) \
                     .orderBy("category")
        
        result_df = process_large_dataset(test_path)
        
        # Validate results
        self.assert_dataframe_properties(
            result_df,
            expected_count=10,  # 10 categories
            expected_columns=["category", "count", "avg_value", "max_value"]
        )
        
        # Check performance metrics
        assert result_df.count() == 10
        
        # Verify aggregation correctness for one category
        category_0_result = result_df.filter(col("category") == "0").collect()[0]
        assert category_0_result["count"] > 0  # Should have valid records

class TestErrorScenarios(SparkTestBase):
    """Test error handling scenarios"""
    
    def test_schema_mismatch_handling(self, spark):
        """Test handling of schema mismatches"""
        
        def merge_dataframes_safely(df1, df2):
            """Safely merge DataFrames with potential schema differences"""
            
            try:
                # Try direct union first
                return df1.union(df2)
            except Exception as e:
                if "schema" in str(e).lower():
                    # Handle schema mismatch
                    all_columns = set(df1.columns) | set(df2.columns)
                    
                    # Add missing columns with null values
                    for col_name in all_columns:
                        if col_name not in df1.columns:
                            df1 = df1.withColumn(col_name, lit(None))
                        if col_name not in df2.columns:
                            df2 = df2.withColumn(col_name, lit(None))
                    
                    # Align column order
                    sorted_columns = sorted(all_columns)
                    df1 = df1.select(*sorted_columns)
                    df2 = df2.select(*sorted_columns)
                    
                    return df1.union(df2)
                else:
                    raise
        
        # Create DataFrames with different schemas
        df1 = spark.createDataFrame([
            ("1", "Alice", 25)
        ], ["id", "name", "age"])
        
        df2 = spark.createDataFrame([
            ("2", "Bob", "Engineer")
        ], ["id", "name", "job"])
        
        # Test safe merge
        result_df = merge_dataframes_safely(df1, df2)
        
        # Validate result
        self.assert_dataframe_properties(
            result_df,
            expected_count=2,
            expected_columns=["age", "id", "job", "name"]  # Sorted alphabetically
        )
        
        # Check that missing values are filled with null
        result_data = result_df.orderBy("id").collect()
        assert result_data[0]["job"] is None  # Alice has no job
        assert result_data[1]["age"] is None  # Bob has no age

# Usage examples for running tests
if __name__ == "__main__":
    # Run specific test class
    pytest.main(["-v", "TestDataTransformations"])
    
    # Run all tests with coverage
    pytest.main(["-v", "--cov=src", "--cov-report=html"])
```

### Property-Based Testing

```python
from hypothesis import given, strategies as st
from hypothesis.extra.pandas import data_frames, columns
import hypothesis.extra.numpy as npst

class PropertyBasedSparkTests(SparkTestBase):
    """Property-based testing for Spark transformations"""
    
    @given(
        values=st.lists(
            st.integers(min_value=0, max_value=1000), 
            min_size=1, 
            max_size=100
        )
    )
    def test_aggregation_properties(self, spark, values):
        """Test that aggregation functions satisfy mathematical properties"""
        
        # Create DataFrame from generated data
        df = spark.createDataFrame([(v,) for v in values], ["value"])
        
        # Calculate aggregations
        result = df.agg(
            sum("value").alias("total"),
            count("value").alias("count"),
            avg("value").alias("average")
        ).collect()[0]
        
        # Property: sum should equal average * count
        expected_total = result["average"] * result["count"]
        assert abs(result["total"] - expected_total) < 0.001, \
            f"Sum property violated: {result['total']} != {expected_total}"
        
        # Property: count should equal input length
        assert result["count"] == len(values)
        
        # Property: average should be within min/max bounds
        if values:
            assert min(values) <= result["average"] <= max(values)
    
    @given(
        partition_count=st.integers(min_value=1, max_value=10),
        data_size=st.integers(min_value=100, max_value=1000)
    )
    def test_partition_invariants(self, spark, partition_count, data_size):
        """Test that repartitioning preserves data integrity"""
        
        # Create test data
        original_df = spark.range(data_size).toDF("id")
        
        # Repartition
        repartitioned_df = original_df.repartition(partition_count)
        
        # Property: row count should be preserved
        assert original_df.count() == repartitioned_df.count()
        
        # Property: all values should be preserved
        original_sum = original_df.agg(sum("id")).collect()[0][0]
        repartitioned_sum = repartitioned_df.agg(sum("id")).collect()[0][0]
        assert original_sum == repartitioned_sum
        
        # Property: partition count should match (approximately)
        actual_partitions = repartitioned_df.rdd.getNumPartitions()
        assert actual_partitions == partition_count

def generate_spark_dataframe_strategy(spark, schema, max_rows=100):
    """Generate strategy for creating Spark DataFrames"""
    
    def create_df(data):
        if not data:
            return spark.createDataFrame([], schema)
        return spark.createDataFrame(data, schema)
    
    # Create strategy based on schema
    strategies = []
    for field in schema.fields:
        if field.dataType == StringType():
            strategies.append(st.text(min_size=0, max_size=50))
        elif field.dataType == IntegerType():
            strategies.append(st.integers(min_value=-1000, max_value=1000))
        elif field.dataType == DoubleType():
            strategies.append(st.floats(min_value=-1000.0, max_value=1000.0, allow_nan=False))
        elif field.dataType == BooleanType():
            strategies.append(st.booleans())
        else:
            strategies.append(st.none())
    
    return st.lists(
        st.tuples(*strategies),
        min_size=0,
        max_size=max_rows
    ).map(create_df)
```

---

## Logging and Monitoring

### Structured Logging Framework

```python
import logging
import json
import time
from datetime import datetime
from contextlib import contextmanager
from typing import Dict, Any, Optional

class SparkLogger:
    """Structured logging for Spark applications"""
    
    def __init__(self, app_name: str, log_level: str = "INFO", 
                 enable_metrics: bool = True, enable_structured: bool = True):
        self.app_name = app_name
        self.enable_metrics = enable_metrics
        self.enable_structured = enable_structured
        self.setup_logging(log_level)
        self.metrics = {}
        self.operation_stack = []
    
    def setup_logging(self, log_level: str):
        """Setup logging configuration"""
        
        # Create logger
        self.logger = logging.getLogger(f"spark.{self.app_name}")
        self.logger.setLevel(getattr(logging, log_level.upper()))
        
        # Clear existing handlers
        self.logger.handlers.clear()
        
        if self.enable_structured:
            # Structured JSON formatter
            class StructuredFormatter(logging.Formatter):
                def format(self, record):
                    log_entry = {
                        "timestamp": datetime.utcnow().isoformat(),
                        "level": record.levelname,
                        "logger": record.name,
                        "message": record.getMessage(),
                        "module": record.module,
                        "function": record.funcName,
                        "line": record.lineno
                    }
                    
                    # Add custom fields if present
                    if hasattr(record, 'custom_fields'):
                        log_entry.update(record.custom_fields)
                    
                    return json.dumps(log_entry)
            
            formatter = StructuredFormatter()
        else:
            # Standard formatter
            formatter = logging.Formatter(
                '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
            )
        
        # Console handler
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        self.logger.addHandler(console_handler)
        
        # File handler
        file_handler = logging.FileHandler(f"/tmp/spark_{self.app_name}.log")
        file_handler.setFormatter(formatter)
        self.logger.addHandler(file_handler)
    
    def log_with_context(self, level: str, message: str, **kwargs):
        """Log with additional context"""
        
        extra_fields = {
            "app_name": self.app_name,
            "operation_stack": self.operation_stack.copy(),
            **kwargs
        }
        
        # Create log record with custom fields
        log_record = self.logger.makeRecord(
            self.logger.name, getattr(logging, level.upper()),
            "", 0, message, (), None
        )
        log_record.custom_fields = extra_fields
        
        self.logger.handle(log_record)
    
    def info(self, message: str, **kwargs):
        """Log info message with context"""
        self.log_with_context("INFO", message, **kwargs)
    
    def warning(self, message: str, **kwargs):
        """Log warning message with context"""
        self.log_with_context("WARNING", message, **kwargs)
    
    def error(self, message: str, **kwargs):
        """Log error message with context"""
        self.log_with_context("ERROR", message, **kwargs)
    
    def debug(self, message: str, **kwargs):
        """Log debug message with context"""
        self.log_with_context("DEBUG", message, **kwargs)
    
    @contextmanager
    def operation_context(self, operation_name: str, **context):
        """Context manager for tracking operations"""
        
        operation_id = f"{operation_name}_{int(time.time())}"
        start_time = time.time()
        
        # Push operation to stack
        self.operation_stack.append(operation_id)
        
        self.info(f"Starting operation: {operation_name}", 
                 operation_id=operation_id, **context)
        
        try:
            yield operation_id
        except Exception as e:
            duration = time.time() - start_time
            self.error(f"Operation failed: {operation_name}", 
                      operation_id=operation_id,
                      duration_seconds=duration,
                      error=str(e),
                      **context)
            raise
        finally:
            duration = time.time() - start_time
            self.operation_stack.pop()
            
            self.info(f"Completed operation: {operation_name}",
                     operation_id=operation_id,
                     duration_seconds=duration,
                     **context)
            
            # Store metrics if enabled
            if self.enable_metrics:
                self.metrics[operation_id] = {
                    "operation": operation_name,
                    "duration": duration,
                    "timestamp": start_time,
                    "context": context
                }
    
    def log_dataframe_info(self, df, operation_name: str, sample_data: bool = False):
        """Log DataFrame information"""
        
        try:
            count = df.count()
            schema_info = {field.name: str(field.dataType) for field in df.schema.fields}
            
            log_data = {
                "operation": operation_name,
                "record_count": count,
                "column_count": len(df.columns),
                "columns": df.columns,
                "schema": schema_info,
                "partitions": df.rdd.getNumPartitions()
            }
            
            if sample_data and count > 0:
                sample_rows = df.limit(3).collect()
                log_data["sample_data"] = [row.asDict() for row in sample_rows]
            
            self.info(f"DataFrame info: {operation_name}", **log_data)
            
        except Exception as e:
            self.error(f"Failed to log DataFrame info for {operation_name}: {str(e)}")
    
    def log_performance_metrics(self, df, operation_name: str):
        """Log performance metrics for DataFrame operations"""
        
        start_time = time.time()
        
        try:
            # Get basic metrics
            count = df.count()
            partitions = df.rdd.getNumPartitions()
            
            # Get partition distribution
            partition_sizes = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
            
            duration = time.time() - start_time
            
            metrics = {
                "operation": operation_name,
                "record_count": count,
                "partition_count": partitions,
                "avg_partition_size": sum(partition_sizes) / len(partition_sizes) if partition_sizes else 0,
                "max_partition_size": max(partition_sizes) if partition_sizes else 0,
                "min_partition_size": min(partition_sizes) if partition_sizes else 0,
                "skew_ratio": max(partition_sizes) / (sum(partition_sizes) / len(partition_sizes)) if partition_sizes else 0,
                "execution_time_seconds": duration
            }
            
            self.info(f"Performance metrics: {operation_name}", **metrics)
            return metrics
            
        except Exception as e:
            self.error(f"Failed to collect performance metrics for {operation_name}: {str(e)}")
            return None
    
    def get_metrics_summary(self) -> Dict[str, Any]:
        """Get summary of collected metrics"""
        
        if not self.metrics:
            return {"message": "No metrics collected"}
        
        total_operations = len(self.metrics)
        total_duration = sum(m["duration"] for m in self.metrics.values())
        avg_duration = total_duration / total_operations
        
        # Find slowest operations
        sorted_ops = sorted(self.metrics.items(), 
                          key=lambda x: x[1]["duration"], 
                          reverse=True)
        
        return {
            "total_operations": total_operations,
            "total_duration_seconds": total_duration,
            "average_duration_seconds": avg_duration,
            "slowest_operations": [
                {
                    "operation": op[1]["operation"],
                    "duration_seconds": op[1]["duration"],
                    "operation_id": op[0]
                }
                for op in sorted_ops[:5]
            ]
        }

# Usage example
logger = SparkLogger("data_pipeline", log_level="INFO", enable_structured=True)

# Log operations with context
with logger.operation_context("data_loading", source="customers.parquet", target_table="customers"):
    customers_df = spark.read.parquet("customers.parquet")
    logger.log_dataframe_info(customers_df, "loaded_customers", sample_data=True)

with logger.operation_context("data_transformation", transformation="customer_metrics"):
    # Perform transformations
    result_df = customers_df.filter(col("is_active") == True)
    logger.log_performance_metrics(result_df, "active_customers_filter")

# Get metrics summary
summary = logger.get_metrics_summary()
logger.info("Pipeline metrics summary", **summary)
```

### Application Performance Monitoring

```python
import psutil
import time
from threading import Thread, Event
from collections import deque
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class PerformanceSnapshot:
    """Performance metrics at a point in time"""
    timestamp: float
    cpu_percent: float
    memory_percent: float
    memory_used_gb: float
    disk_io_read_mb: float
    disk_io_write_mb: float
    network_sent_mb: float
    network_recv_mb: float

class SparkApplicationMonitor:
    """Monitor Spark application performance metrics"""
    
    def __init__(self, sample_interval: int = 30, max_samples: int = 1000):
        self.sample_interval = sample_interval
        self.max_samples = max_samples
        self.samples = deque(maxlen=max_samples)
        self.monitoring = False
        self.monitor_thread = None
        self.stop_event = Event()
        
        # Initialize baseline metrics
        self.baseline_disk_io = psutil.disk_io_counters()
        self.baseline_network = psutil.net_io_counters()
    
    def start_monitoring(self):
        """Start background monitoring"""
        if self.monitoring:
            return
        
        self.monitoring = True
        self.stop_event.clear()
        self.monitor_thread = Thread(target=self._monitor_loop, daemon=True)
        self.monitor_thread.start()
        print(f"📊 Started performance monitoring (interval: {self.sample_interval}s)")
    
    def stop_monitoring(self):
        """Stop background monitoring"""
        if not self.monitoring:
            return
        
        self.monitoring = False
        self.stop_event.set()
        if self.monitor_thread:
            self.monitor_thread.join(timeout=5)
        print("⏹️  Stopped performance monitoring")
    
    def _monitor_loop(self):
        """Background monitoring loop"""
        while not self.stop_event.wait(self.sample_interval):
            try:
                snapshot = self._take_snapshot()
                self.samples.append(snapshot)
            except Exception as e:
                print(f"Error collecting performance metrics: {e}")
    
    def _take_snapshot(self) -> PerformanceSnapshot:
        """Take a performance snapshot"""
        
        # CPU and memory
        cpu_percent = psutil.cpu_percent(interval=1)
        memory = psutil.virtual_memory()
        
        # Disk I/O
        current_disk_io = psutil.disk_io_counters()
        disk_read_mb = (current_disk_io.read_bytes - self.baseline_disk_io.read_bytes) / (1024 * 1024)
        disk_write_mb = (current_disk_io.write_bytes - self.baseline_disk_io.write_bytes) / (1024 * 1024)
        
        # Network I/O
        current_network = psutil.net_io_counters()
        network_sent_mb = (current_network.bytes_sent - self.baseline_network.bytes_sent) / (1024 * 1024)
        network_recv_mb = (current_network.bytes_recv - self.baseline_network.bytes_recv) / (1024 * 1024)
        
        return PerformanceSnapshot(
            timestamp=time.time(),
            cpu_percent=cpu_percent,
            memory_percent=memory.percent,
            memory_used_gb=memory.used / (1024**3),
            disk_io_read_mb=disk_read_mb,
            disk_io_write_mb=disk_write_mb,
            network_sent_mb=network_sent_mb,
            network_recv_mb=network_recv_mb
        )
    
    def get_current_metrics(self) -> Dict[str, float]:
        """Get current performance metrics"""
        snapshot = self._take_snapshot()
        return {
            "cpu_percent": snapshot.cpu_percent,
            "memory_percent": snapshot.memory_percent,
            "memory_used_gb": snapshot.memory_used_gb,
            "disk_read_mb": snapshot.disk_io_read_mb,
            "disk_write_mb": snapshot.disk_io_write_mb,
            "network_sent_mb": snapshot.network_sent_mb,
            "network_recv_mb": snapshot.network_recv_mb
        }
    
    def get_performance_summary(self, last_n_minutes: int = 30) -> Dict[str, Any]:
        """Get performance summary for the last N minutes"""
        
        if not self.samples:
            return {"message": "No performance data available"}
        
        # Filter samples for the specified time period
        cutoff_time = time.time() - (last_n_minutes * 60)
        recent_samples = [s for s in self.samples if s.timestamp >= cutoff_time]
        
        if not recent_samples:
            return {"message": f"No data available for last {last_n_minutes} minutes"}
        
        # Calculate statistics
        cpu_values = [s.cpu_percent for s in recent_samples]
        memory_values = [s.memory_percent for s in recent_samples]
        
        summary = {
            "time_period_minutes": last_n_minutes,
            "sample_count": len(recent_samples),
            "cpu_stats": {
                "avg": sum(cpu_values) / len(cpu_values),
                "max": max(cpu_values),
                "min": min(cpu_values)
            },
            "memory_stats": {
                "avg": sum(memory_values) / len(memory_values),
                "max": max(memory_values),
                "min": min(memory_values),
                "current_gb": recent_samples[-1].memory_used_gb
            },
            "disk_io": {
                "total_read_mb": recent_samples[-1].disk_io_read_mb,
                "total_write_mb": recent_samples[-1].disk_io_write_mb
            },
            "network_io": {
                "total_sent_mb": recent_samples[-1].network_sent_mb,
                "total_recv_mb": recent_samples[-1].network_recv_mb
            }
        }
        
        # Detect performance issues
        issues = []
        if summary["cpu_stats"]["avg"] > 80:
            issues.append("High average CPU usage")
        if summary["memory_stats"]["avg"] > 85:
            issues.append("High average memory usage")
        if summary["cpu_stats"]["max"] > 95:
            issues.append("CPU spikes detected")
        
        summary["performance_issues"] = issues
        
        return summary
    
    def alert_on_thresholds(self, cpu_threshold: float = 90, 
                          memory_threshold: float = 90) -> List[str]:
        """Check for threshold violations and return alerts"""
        
        current = self.get_current_metrics()
        alerts = []
        
        if current["cpu_percent"] > cpu_threshold:
            alerts.append(f"🚨 High CPU usage: {current['cpu_percent']:.1f}% > {cpu_threshold}%")
        
        if current["memory_percent"] > memory_threshold:
            alerts.append(f"🚨 High memory usage: {current['memory_percent']:.1f}% > {memory_threshold}%")
        
        return alerts

# Usage example
monitor = SparkApplicationMonitor(sample_interval=10)

# Start monitoring
monitor.start_monitoring()

# Run your Spark operations
try:
    # Your Spark code here
    df = spark.read.parquet("large_dataset.parquet")
    result = df.groupBy("category").count().collect()
    
    # Check for performance issues
    alerts = monitor.alert_on_thresholds(cpu_threshold=80, memory_threshold=85)
    for alert in alerts:
        print(alert)
    
finally:
    # Stop monitoring and get summary
    monitor.stop_monitoring()
    summary = monitor.get_performance_summary(last_n_minutes=15)
    print(f"Performance Summary: {json.dumps(summary, indent=2)}")
```

---

## Deployment Strategies

### Environment Configuration Management

```python
import os
import yaml
import json
from typing import Dict, Any, Optional
from dataclasses import dataclass, asdict
from enum import Enum

class Environment(Enum):
    DEVELOPMENT = "development"
    TESTING = "testing"
    STAGING = "staging"
    PRODUCTION = "production"

@dataclass
class SparkConfig:
    """Spark configuration for different environments"""
    app_name: str
    master: str
    executor_memory: str
    executor_cores: str
    executor_instances: str
    driver_memory: str
    sql_shuffle_partitions: str
    adaptive_enabled: bool
    serializer: str
    
    # Storage configurations
    checkpoint_location: str
    warehouse_dir: str
    
    # Security configurations
    ssl_enabled: bool = False
    kerberos_enabled: bool = False
    
    # Performance configurations
    dynamic_allocation: bool = False
    max_executors: Optional[str] = None
    
    def to_spark_conf(self) -> Dict[str, str]:
        """Convert to Spark configuration dictionary"""
        config = {
            "spark.app.name": self.app_name,
            "spark.master": self.master,
            "spark.executor.memory": self.executor_memory,
            "spark.executor.cores": self.executor_cores,
            "spark.executor.instances": self.executor_instances,
            "spark.driver.memory": self.driver_memory,
            "spark.sql.shuffle.partitions": self.sql_shuffle_partitions,
            "spark.sql.adaptive.enabled": str(self.adaptive_enabled).lower(),
            "spark.serializer": self.serializer,
            "spark.sql.warehouse.dir": self.warehouse_dir
        }
        
        if self.checkpoint_location:
            config["spark.sql.streaming.checkpointLocation"] = self.checkpoint_location
        
        if self.dynamic_allocation:
            config["spark.dynamicAllocation.enabled"] = "true"
            if self.max_executors:
                config["spark.dynamicAllocation.maxExecutors"] = self.max_executors
        
        return config

class ConfigurationManager:
    """Manage configuration across different environments"""
    
    def __init__(self, config_path: str = "config"):
        self.config_path = config_path
        self.configs = {}
        self.load_configurations()
    
    def load_configurations(self):
        """Load configurations for all environments"""
        
        # Default configurations
        base_config = {
            "development": SparkConfig(
                app_name="dev_data_pipeline",
                master="local[*]",
                executor_memory="2g",
                executor_cores="2",
                executor_instances="2",
                driver_memory="1g",
                sql_shuffle_partitions="200",
                adaptive_enabled=True,
                serializer="org.apache.spark.serializer.KryoSerializer",
                checkpoint_location="/tmp/spark-checkpoints",
                warehouse_dir="/tmp/spark-warehouse"
            ),
            "testing": SparkConfig(
                app_name="test_data_pipeline",
                master="local[2]",
                executor_memory="1g",
                executor_cores="1",
                executor_instances="1",
                driver_memory="512m",
                sql_shuffle_partitions="2",
                adaptive_enabled=False,  # Disable for deterministic tests
                serializer="org.apache.spark.serializer.KryoSerializer",
                checkpoint_location="/tmp/test-checkpoints",
                warehouse_dir="/tmp/test-warehouse"
            ),
            "staging": SparkConfig(
                app_name="staging_data_pipeline",
                master="yarn",
                executor_memory="8g",
                executor_cores="4",
                executor_instances="10",
                driver_memory="4g",
                sql_shuffle_partitions="800",
                adaptive_enabled=True,
                serializer="org.apache.spark.serializer.KryoSerializer",
                checkpoint_location="s3a://staging-bucket/checkpoints",
                warehouse_dir="s3a://staging-bucket/warehouse",
                dynamic_allocation=True,
                max_executors="50"
            ),
            "production": SparkConfig(
                app_name="prod_data_pipeline",
                master="yarn",
                executor_memory="16g",
                executor_cores="5",
                executor_instances="20",
                driver_memory="8g",
                sql_shuffle_partitions="1000",
                adaptive_enabled=True,
                serializer="org.apache.spark.serializer.KryoSerializer",
                checkpoint_location="s3a://prod-bucket/checkpoints",
                warehouse_dir="s3a://prod-bucket/warehouse",
                ssl_enabled=True,
                kerberos_enabled=True,
                dynamic_allocation=True,
                max_executors="100"
            )
        }
        
        self.configs = base_config
        
        # Load from files if they exist
        for env in Environment:
            config_file = f"{self.config_path}/{env.value}.yaml"
            if os.path.exists(config_file):
                self.load_config_from_file(env.value, config_file)
    
    def load_config_from_file(self, env: str, file_path: str):
        """Load configuration from YAML file"""
        try:
            with open(file_path, 'r') as f:
                config_data = yaml.safe_load(f)
                
            # Convert to SparkConfig object
            self.configs[env] = SparkConfig(**config_data)
            print(f"Loaded configuration for {env} from {file_path}")
        except Exception as e:
            print(f"Failed to load configuration from {file_path}: {e}")
    
    def get_config(self, environment: str) -> SparkConfig:
        """Get configuration for specified environment"""
        if environment not in self.configs:
            raise ValueError(f"Configuration not found for environment: {environment}")
        
        return self.configs[environment]
    
    def get_spark_session(self, environment: str):
        """Create Spark session with environment-specific configuration"""
        config = self.get_config(environment)
        spark_conf = config.to_spark_conf()
        
        builder = SparkSession.builder
        
        # Apply all configurations
        for key, value in spark_conf.items():
            builder = builder.config(key, value)
        
        return builder.getOrCreate()
    
    def validate_environment(self, environment: str) -> bool:
        """Validate environment configuration"""
        try:
            config = self.get_config(environment)
            
            # Check required fields
            required_fields = ['app_name', 'master', 'executor_memory']
            for field in required_fields:
                if not getattr(config, field):
                    print(f"❌ Missing required field: {field}")
                    return False
            
            # Environment-specific validations
            if environment == "production":
                if config.master == "local[*]":
                    print("❌ Production should not use local master")
                    return False
                
                if not config.ssl_enabled:
                    print("⚠️  SSL not enabled in production")
            
            print(f"✅ Configuration valid for {environment}")
            return True
            
        except Exception as e:
            print(f"❌ Configuration validation failed: {e}")
            return False

# Usage example
config_manager = ConfigurationManager()

# Get environment from environment variable or default to development
current_env = os.getenv("SPARK_ENV", "development")
print(f"Running in {current_env} environment")

# Validate configuration
if config_manager.validate_environment(current_env):
    # Create Spark session with environment-specific config
    spark = config_manager.get_spark_session(current_env)
    print(f"Spark session created for {current_env}")
else:
    print(f"Invalid configuration for {current_env}")
```

### Containerized Deployment

```python
# Dockerfile template generation
def generate_dockerfile(python_version="3.9", spark_version="3.4.0"):
    """Generate Dockerfile for Spark application"""
    
    dockerfile_content = f"""
# Multi-stage Dockerfile for Spark application
FROM python:{python_version}-slim as builder

# Install system dependencies
RUN apt-get update && apt-get install -y \\
    gcc \\
    g++ \\
    openjdk-11-jdk \\
    curl \\
    && rm -rf /var/lib/apt/lists/*

# Set Java environment
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production stage
FROM python:{python_version}-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \\
    openjdk-11-jre-headless \\
    curl \\
    && rm -rf /var/lib/apt/lists/*

# Set Java environment
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

# Copy Python dependencies from builder
COPY --from=builder /usr/local/lib/python{python_version}/site-packages /usr/local/lib/python{python_version}/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Create app user
RUN useradd --create-home --shell /bin/bash spark
USER spark
WORKDIR /home/spark

# Copy application code
COPY --chown=spark:spark src/ ./src/
COPY --chown=spark:spark config/ ./config/
COPY --chown=spark:spark scripts/ ./scripts/

# Set environment variables
ENV PYTHONPATH=/home/spark/src
ENV SPARK_HOME=/usr/local/lib/python{python_version}/site-packages/pyspark

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \\
    CMD curl -f http://localhost:4040 || exit 1

# Default command
CMD ["python", "src/main.py"]
"""
    
    return dockerfile_content.strip()

# Docker Compose for development
def generate_docker_compose():
    """Generate docker-compose.yml for local development"""
    
    compose_content = """
version: '3.8'

services:
  spark-master:
    image: bitnami/spark:3.4.0
    environment:
      - SPARK_MODE=master
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    ports:
      - "8080:8080"
      - "7077:7077"
    volumes:
      - ./data:/opt/bitnami/spark/data
      - ./src:/opt/bitnami/spark/work
    networks:
      - spark-network

  spark-worker-1:
    image: bitnami/spark:3.4.0
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2G
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    depends_on:
      - spark-master
    volumes:
      - ./data:/opt/bitnami/spark/data
      - ./src:/opt/bitnami/spark/work
    networks:
      - spark-network

  spark-worker-2:
    image: bitnami/spark:3.4.0
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2G
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    depends_on:
      - spark-master
    volumes:
      - ./data:/opt/bitnami/spark/data
      - ./src:/opt/bitnami/spark/work
    networks:
      - spark-network

  jupyter:
    build: .
    environment:
      - SPARK_MASTER_URL=spark://spark-master:7077
      - JUPYTER_ENABLE_LAB=yes
    ports:
      - "8888:8888"
    depends_on:
      - spark-master
    volumes:
      - ./notebooks:/home/spark/notebooks
      - ./data:/home/spark/data
      - ./src:/home/spark/src
    networks:
      - spark-network
    command: jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root

  app:
    build: .
    environment:
      - SPARK_ENV=development
      - SPARK_MASTER_URL=spark://spark-master:7077
    depends_on:
      - spark-master
      - spark-worker-1
      - spark-worker-2
    volumes:
      - ./src:/home/spark/src
      - ./config:/home/spark/config
      - ./data:/home/spark/data
    networks:
      - spark-network

networks:
  spark-network:
    driver: bridge

volumes:
  spark-data:
"""
    
    return compose_content.strip()

# Kubernetes deployment manifests
def generate_kubernetes_manifests():
    """Generate Kubernetes manifests for Spark application"""
    
    # ConfigMap for application configuration
    configmap = """
apiVersion: v1
kind: ConfigMap
metadata:
  name: spark-app-config
  namespace: spark
data:
  spark-defaults.conf: |
    spark.sql.adaptive.enabled=true
    spark.sql.adaptive.coalescePartitions.enabled=true
    spark.sql.adaptive.skewJoin.enabled=true
    spark.serializer=org.apache.spark.serializer.KryoSerializer
    spark.sql.execution.arrow.pyspark.enabled=true
  log4j.properties: |
    log4j.rootLogger=INFO, console
    log4j.appender.console=org.apache.log4j.ConsoleAppender
    log4j.appender.console.target=System.err
    log4j.appender.console.layout=org.apache.log4j.PatternLayout
    log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
"""
    
    # Deployment for Spark application
    deployment = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-app
  namespace: spark
  labels:
    app: spark-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spark-app
  template:
    metadata:
      labels:
        app: spark-app
    spec:
      serviceAccountName: spark-service-account
      containers:
      - name: spark-app
        image: your-registry/spark-app:latest
        imagePullPolicy: Always
        env:
        - name: SPARK_ENV
          value: "production"
        - name: SPARK_MASTER_URL
          value: "k8s://https://kubernetes.default.svc:443"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
        volumeMounts:
        - name: spark-config
          mountPath: /opt/spark/conf
          readOnly: true
        - name: app-logs
          mountPath: /var/log/spark
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: spark-config
        configMap:
          name: spark-app-config
      - name: app-logs
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: spark-app-service
  namespace: spark
spec:
  selector:
    app: spark-app
  ports:
  - name: web-ui
    port: 4040
    targetPort: 4040
  - name: health
    port: 8080
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark-service-account
  namespace: spark
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: spark-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spark-role-binding
subjects:
- kind: ServiceAccount
  name: spark-service-account
  namespace: spark
roleRef:
  kind: ClusterRole
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
"""
    
    return {"configmap": configmap.strip(), "deployment": deployment.strip()}

# Usage
dockerfile = generate_dockerfile()
docker_compose = generate_docker_compose()
k8s_manifests = generate_kubernetes_manifests()

print("Generated deployment files:")
print("- Dockerfile")
print("- docker-compose.yml") 
print("- Kubernetes manifests")
```

### CI/CD Pipeline Configuration

```python
def generate_github_actions_workflow():
    """Generate GitHub Actions workflow for CI/CD"""
    
    workflow = """
name: Spark Application CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/spark-app

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.8, 3.9, '3.10']
    
    services:
      spark:
        image: bitnami/spark:3.4.0
        env:
          SPARK_MODE: master
        ports:
          - 7077:7077
          - 8080:8080
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Lint with flake8
      run: |
        flake8 src/ tests/ --max-line-length=100 --exclude=__pycache__
    
    - name: Type check with mypy
      run: |
        mypy src/ --ignore-missing-imports
    
    - name: Run unit tests
      run: |
        pytest tests/unit/ -v --cov=src --cov-report=xml
    
    - name: Run integration tests
      run: |
        pytest tests/integration/ -v --spark-master=spark://localhost:7077
      env:
        SPARK_ENV: testing
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security scan
      uses: pypa/gh-action-pip-audit@v1.0.8
      with:
        inputs: requirements.txt
    
    - name: Run Bandit security scan
      run: |
        pip install bandit
        bandit -r src/ -f json -o bandit-report.json
    
    - name: Upload security scan results
      uses: actions/upload-artifact@v3
      with:
        name: security-scan-results
        path: bandit-report.json

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'
    
    - name: Set up Kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Deploy to staging
      run: |
        export KUBECONFIG=kubeconfig
        kubectl set image deployment/spark-app spark-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n spark-staging
        kubectl rollout status deployment/spark-app -n spark-staging --timeout=300s
    
    - name: Run smoke tests
      run: |
        python scripts/smoke_tests.py --environment=staging
      env:
        SPARK_ENV: staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'
    
    - name: Set up Kubeconfig
      run: |
        echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
    
    - name: Deploy to production
      run: |
        export KUBECONFIG=kubeconfig
        kubectl set image deployment/spark-app spark-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n spark-prod
        kubectl rollout status deployment/spark-app -n spark-prod --timeout=600s
    
    - name: Run production health checks
      run: |
        python scripts/health_checks.py --environment=production
      env:
        SPARK_ENV: production
    
    - name: Notify deployment success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: "🚀 Spark application deployed to production successfully!"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      if: success()
    
    - name: Notify deployment failure
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: "❌ Spark application deployment to production failed!"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      if: failure()
"""
    
    return workflow.strip()

def generate_jenkins_pipeline():
    """Generate Jenkins pipeline for CI/CD"""
    
    pipeline = """
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'your-registry.com'
        IMAGE_NAME = 'spark-app'
        SPARK_ENV = 'testing'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install -r requirements-dev.txt
                '''
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'flake8 src/ tests/ --max-line-length=100'
                    }
                }
                stage('Type Check') {
                    steps {
                        sh 'mypy src/ --ignore-missing-imports'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'bandit -r src/ -f json -o bandit-report.json'
                        archiveArtifacts artifacts: 'bandit-report.json', fingerprint: true
                    }
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'pytest tests/unit/ -v --cov=src --cov-report=xml --junitxml=unit-test-results.xml'
                    }
                    post {
                        always {
                            junit 'unit-test-results.xml'
                            publishCoverage adapters: [coberturaAdapter('coverage.xml')], sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh '''
                            docker run -d --name spark-test -p 7077:7077 bitnami/spark:3.4.0
                            sleep 30
                            pytest tests/integration/ -v --spark-master=spark://localhost:7077 --junitxml=integration-test-results.xml
                        '''
                    }
                    post {
                        always {
                            sh 'docker stop spark-test || true'
                            sh 'docker rm spark-test || true'
                            junit 'integration-test-results.xml'
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        kubectl set image deployment/spark-app spark-app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} -n spark-staging
                        kubectl rollout status deployment/spark-app -n spark-staging --timeout=300s
                    """
                }
            }
        }
        
        stage('Smoke Tests') {
            when {
                branch 'main'
            }
            steps {
                sh 'python scripts/smoke_tests.py --environment=staging'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitterParameter "DEPLOYER"
            }
            steps {
                script {
                    sh """
                        kubectl set image deployment/spark-app spark-app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} -n spark-prod
                        kubectl rollout status deployment/spark-app -n spark-prod --timeout=600s
                    """
                }
            }
        }
        
        stage('Production Health Check') {
            when {
                branch 'main'
            }
            steps {
                sh 'python scripts/health_checks.py --environment=production'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ Spark application deployed successfully to production (Build: ${BUILD_NUMBER})"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ Spark application deployment failed (Build: ${BUILD_NUMBER})"
            )
        }
    }
}
"""
    
    return pipeline.strip()

# Generate CI/CD configurations
github_workflow = generate_github_actions_workflow()
jenkins_pipeline = generate_jenkins_pipeline()

print("Generated CI/CD configurations:")
print("- GitHub Actions workflow")
print("- Jenkins pipeline")
```

---

## Health Checks and Monitoring

### Application Health Checks

```python
from flask import Flask, jsonify
import threading
import time
from datetime import datetime, timedelta
from typing import Dict, Any, List
import requests

class HealthCheckServer:
    """HTTP server for health and readiness checks"""
    
    def __init__(self, port: int = 8080, spark_session=None):
        self.port = port
        self.spark = spark_session
        self.app = Flask(__name__)
        self.setup_routes()
        self.health_status = {
            "spark_session": False,
            "database_connection": False,
            "storage_access": False,
            "last_check": None
        }
        
        # Start background health monitoring
        self.monitor_thread = threading.Thread(target=self._monitor_loop, daemon=True)
        self.monitor_thread.start()
    
    def setup_routes(self):
        """Setup Flask routes for health checks"""
        
        @self.app.route('/health')
        def health():
            """Liveness probe - basic health check"""
            try:
                health_data = {
                    "status": "healthy",
                    "timestamp": datetime.utcnow().isoformat(),
                    "uptime_seconds": self._get_uptime(),
                    "version": "1.0.0"
                }
                return jsonify(health_data), 200
            except Exception as e:
                return jsonify({
                    "status": "unhealthy",
                    "error": str(e),
                    "timestamp": datetime.utcnow().isoformat()
                }), 500
        
        @self.app.route('/ready')
        def ready():
            """Readiness probe - check if ready to serve traffic"""
            try:
                readiness_checks = self._perform_readiness_checks()
                
                all_ready = all(readiness_checks.values())
                status_code = 200 if all_ready else 503
                
                return jsonify({
                    "ready": all_ready,
                    "checks": readiness_checks,
                    "timestamp": datetime.utcnow().isoformat()
                }), status_code
                
            except Exception as e:
                return jsonify({
                    "ready": False,
                    "error": str(e),
                    "timestamp": datetime.utcnow().isoformat()
                }), 503
        
        @self.app.route('/metrics')
        def metrics():
            """Expose application metrics"""
            try:
                metrics_data = self._collect_metrics()
                return jsonify(metrics_data), 200
            except Exception as e:
                return jsonify({
                    "error": str(e),
                    "timestamp": datetime.utcnow().isoformat()
                }), 500
    
    def _get_uptime(self) -> float:
        """Get application uptime in seconds"""
        if not hasattr(self, 'start_time'):
            self.start_time = time.time()
        return time.time() - self.start_time
    
    def _perform_readiness_checks(self) -> Dict[str, bool]:
        """Perform readiness checks"""
        checks = {}
        
        # Check Spark session
        checks["spark_session"] = self._check_spark_session()
        
        # Check database connectivity
        checks["database"] = self._check_database_connection()
        
        # Check storage access
        checks["storage"] = self._check_storage_access()
        
        # Check external dependencies
        checks["external_services"] = self._check_external_services()
        
        return checks
    
    def _check_spark_session(self) -> bool:
        """Check if Spark session is healthy"""
        try:
            if self.spark is None:
                return False
            
            # Simple Spark operation to verify health
            test_df = self.spark.range(1)
            count = test_df.count()
            return count == 1
        except Exception:
            return False
    
    def _check_database_connection(self) -> bool:
        """Check database connectivity"""
        try:
            # Example database health check
            # Replace with your actual database connection logic
            if self.spark:
                test_query = "(SELECT 1 as test) as test_table"
                df = self.spark.read.format("jdbc") \
                    .option("url", "jdbc:postgresql://localhost:5432/test") \
                    .option("dbtable", test_query) \
                    .option("user", "test") \
                    .option("password", "test") \
                    .load()
                return df.count() == 1
            return True  # Skip if no Spark session
        except Exception:
            return False
    
    def _check_storage_access(self) -> bool:
        """Check storage accessibility"""
        try:
            # Example storage health check
            # Replace with your actual storage logic
            if self.spark:
                # Try to list files in a known location
                test_path = "/tmp/health_check"
                self.spark.range(1).write.mode("overwrite").parquet(test_path)
                df = self.spark.read.parquet(test_path)
                return df.count() == 1
            return True
        except Exception:
            return False
    
    def _check_external_services(self) -> bool:
        """Check external service dependencies"""
        try:
            # Example external service check
            # Replace with your actual external service endpoints
            external_endpoints = [
                "http://api.example.com/health",
                "http://monitoring.example.com/ping"
            ]
            
            for endpoint in external_endpoints:
                try:
                    response = requests.get(endpoint, timeout=5)
                    if response.status_code != 200:
                        return False
                except:
                    return False
            
            return True
        except Exception:
            return False
    
    def _collect_metrics(self) -> Dict[str, Any]:
        """Collect application metrics"""
        try:
            metrics = {
                "uptime_seconds": self._get_uptime(),
                "timestamp": datetime.utcnow().isoformat(),
                "spark_metrics": self._get_spark_metrics(),
                "system_metrics": self._get_system_metrics(),
                "application_metrics": self._get_application_metrics()
            }
            return metrics
        except Exception as e:
            return {"error": str(e)}
    
    def _get_spark_metrics(self) -> Dict[str, Any]:
        """Get Spark-specific metrics"""
        if not self.spark:
            return {}
        
        try:
            sc = self.spark.sparkContext
            status = sc.statusTracker()
            
            return {
                "application_id": sc.applicationId,
                "application_name": sc.appName,
                "active_jobs": len(status.getActiveJobIds()),
                "active_stages": len(status.getActiveStageIds()),
                "executor_count": len(status.getExecutorInfos())
            }
        except Exception:
            return {}
    
    def _get_system_metrics(self) -> Dict[str, Any]:
        """Get system-level metrics"""
        try:
            import psutil
            
            return {
                "cpu_percent": psutil.cpu_percent(),
                "memory_percent": psutil.virtual_memory().percent,
                "disk_usage_percent": psutil.disk_usage('/').percent
            }
        except Exception:
            return {}
    
    def _get_application_metrics(self) -> Dict[str, Any]:
        """Get application-specific metrics"""
        return {
            "health_check_count": getattr(self, 'health_check_count', 0),
            "last_health_check": self.health_status.get("last_check"),
            "status": self.health_status
        }
    
    def _monitor_loop(self):
        """Background monitoring loop"""
        while True:
            try:
                # Update health status
                self.health_status.update({
                    "spark_session": self._check_spark_session(),
                    "database_connection": self._check_database_connection(),
                    "storage_access": self._check_storage_access(),
                    "last_check": datetime.utcnow().isoformat()
                })
                
                # Increment health check counter
                self.health_check_count = getattr(self, 'health_check_count', 0) + 1
                
                time.sleep(30)  # Check every 30 seconds
            except Exception as e:
                print(f"Health monitoring error: {e}")
                time.sleep(60)  # Wait longer on error
    
    def start(self):
        """Start the health check server"""
        self.app.run(host='0.0.0.0', port=self.port, threaded=True)

# Usage
def start_health_check_server(spark_session, port=8080):
    """Start health check server in background thread"""
    health_server = HealthCheckServer(port=port, spark_session=spark_session)
    
    server_thread = threading.Thread(
        target=health_server.start, 
        daemon=True
    )
    server_thread.start()
    
    print(f"🏥 Health check server started on port {port}")
    print(f"   Health endpoint: http://localhost:{port}/health")
    print(f"   Readiness endpoint: http://localhost:{port}/ready")
    print(f"   Metrics endpoint: http://localhost:{port}/metrics")
    
    return health_server

# Example usage
# health_server = start_health_check_server(spark, port=8080)
```

---

## Related Notes

- [[PySpark Core Concepts]] - Foundation concepts for production applications
- [[PySpark Data Operations]] - Operations that need production-ready error handling
- [[PySpark Window Functions]] - Complex operations requiring robust testing
- [[PySpark Performance Optimization]] - Performance considerations for production
- [[PySpark Security & Governance]] - Security aspects of production deployment

---

## Tags

#pyspark #production #engineering #testing #logging #monitoring #deployment #error-handling #reliability #ci-cd #kubernetes #docker

---

_Last Updated: 2024-08-20_