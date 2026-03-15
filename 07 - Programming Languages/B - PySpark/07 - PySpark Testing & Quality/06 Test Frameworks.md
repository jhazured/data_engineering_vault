
#pyspark #testing #frameworks #pytest #spark-testing-base #great-expectations #test-utilities #automation #ci-cd #test-tools #testing-libraries

## 7.6 Testing Frameworks & Tools

**Testing frameworks and tools** provide the infrastructure needed to write, execute, and maintain comprehensive test suites for PySpark applications. The right combination of tools can dramatically improve test reliability, reduce maintenance overhead, and provide better insights into test failures.

### pytest with PySpark

**pytest integration** with PySpark requires careful management of SparkSession lifecycle, test isolation, and resource cleanup. The framework's fixture system is particularly well-suited for managing the complex setup and teardown requirements of distributed testing.

**Advanced pytest Configuration**

````python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import *
import tempfile
import shutil
from pathlib import Path
import logging
from typing import Generator, Dict, Any

# Configure logging for tests
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class PySparkTestConfig:
    """Centralized configuration for PySpark testing
    
    Configuration Management Strategy:
    1. Environment-specific settings (local vs CI vs cluster)
    2. Resource allocation based on test type
    3. Feature flags for optional components
    4. Performance tuning for test execution
    """
    
    @staticmethod
    def get_test_spark_config() -> Dict[str, str]:
        """Get optimized Spark configuration for testing"""
        return {
            # Core settings
            "spark.app.name": "PySpark-Test-Suite",
            "spark.master": "local[*]",
            
            # Memory settings optimized for tests
            "spark.driver.memory": "2g",
            "spark.executor.memory": "1g",
            "spark.driver.maxResultSize": "1g",
            
            # Performance settings for faster tests
            "spark.sql.shuffle.partitions": "4",  # Reduced from default 200
            "spark.sql.adaptive.enabled": "false",  # Consistent test results
            "spark.sql.adaptive.coalescePartitions.enabled": "false",
            
            # Serialization for better performance
            "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
            "spark.kryo.unsafe": "true",
            
            # Disable unnecessary features for tests
            "spark.ui.enabled": "false",
            "spark.sql.execution.arrow.pyspark.enabled": "true",
            
            # Checkpoint and temporary directories
            "spark.sql.streaming.checkpointLocation": "/tmp/spark-checkpoints",
            "spark.local.dir": "/tmp/spark-local",
            
            # Logging configuration
            "spark.sql.execution.arrow.maxRecordsPerBatch": "1000"
        }

@pytest.fixture(scope="session")
def spark_session() -> Generator[SparkSession, None, None]:
    """Session-scoped SparkSession for efficient test execution
    
    Session Scope Benefits:
    - Single SparkSession creation per test session (major performance gain)
    - Shared executor JVMs across tests
    - Consistent Spark configuration throughout test suite
    - Reduced test execution time from ~5min to ~30sec for typical suites
    
    Important: Tests must be designed to not interfere with each other
    """
    config = PySparkTestConfig.get_test_spark_config()
    
    builder = SparkSession.builder
    for key, value in config.items():
        builder = builder.config(key, value)
    
    spark = builder.getOrCreate()
    
    # Set log level to reduce noise during tests
    spark.sparkContext.setLogLevel("WARN")
    
    logger.info(f"Created SparkSession with {spark.sparkContext.defaultParallelism} cores")
    
    yield spark
    
    # Cleanup
    spark.stop()
    logger.info("SparkSession stopped")

@pytest.fixture
def spark(spark_session) -> SparkSession:
    """Function-scoped alias for SparkSession
    
    This fixture provides a clean interface for individual tests while
    reusing the session-scoped SparkSession for performance.
    """
    return spark_session

@pytest.fixture
def temp_dir() -> Generator[Path, None, None]:
    """Temporary directory for test files"""
    temp_path = Path(tempfile.mkdtemp(prefix="pyspark_test_"))
    try:
        yield temp_path
    finally:
        shutil.rmtree(temp_path, ignore_errors=True)

@pytest.fixture
def sample_dataframe(spark) -> "DataFrame":
    """Standard test DataFrame for common test scenarios"""
    data = [
        (1, "Alice", 25, "Engineer", 75000.0, "2020-01-15"),
        (2, "Bob", 30, "Manager", 85000.0, "2019-03-22"),
        (3, "Charlie", 35, "Director", 95000.0, "2018-07-10"),
        (4, "Diana", 28, "Engineer", 70000.0, "2021-02-01"),
        (5, "Eve", 32, "Manager", 80000.0, "2020-11-30")
    ]
    
    schema = StructType([
        StructField("id", IntegerType(), False),
        StructField("name", StringType(), False),
        StructField("age", IntegerType(), False),
        StructField("title", StringType(), False),
        StructField("salary", DoubleType(), False),
        StructField("hire_date", StringType(), False)
    ])
    
    return spark.createDataFrame(data, schema)

class DataFrameAssertions:
    """Custom assertion methods for DataFrame testing
    
    Assertion Strategy:
    1. Schema validation with detailed error messages
    2. Data validation with row-by-row comparison
    3. Approximate comparisons for floating-point data
    4. Performance-aware assertions for large datasets
    """
    
    @staticmethod
    def assert_dataframe_equal(df1, df2, check_order: bool = True, rtol: float = 1e-5):
        """Assert that two DataFrames are equal
        
        Parameters:
        - df1, df2: DataFrames to compare
        - check_order: Whether row order matters
        - rtol: Relative tolerance for floating-point comparisons
        """
        # Schema comparison
        if df1.schema != df2.schema:
            raise AssertionError(f"Schema mismatch:\nDF1: {df1.schema}\nDF2: {df2.schema}")
        
        # Count comparison
        count1, count2 = df1.count(), df2.count()
        if count1 != count2:
            raise AssertionError(f"Row count mismatch: {count1} != {count2}")
        
        # Data comparison
        if not check_order:
            df1 = df1.orderBy(*df1.columns)
            df2 = df2.orderBy(*df2.columns)
        
        # Collect for detailed comparison (only safe for test datasets)
        if count1 > 10000:
            logger.warning(f"Large DataFrame comparison ({count1} rows) - consider sampling")
        
        rows1 = df1.collect()
        rows2 = df2.collect()
        
        for i, (row1, row2) in enumerate(zip(rows1, rows2)):
            for j, (val1, val2) in enumerate(zip(row1, row2)):
                if isinstance(val1, float) and isinstance(val2, float):
                    if not pytest.approx(val1, rel=rtol) == val2:
                        raise AssertionError(f"Float mismatch at row {i}, col {j}: {val1} != {val2}")
                elif val1 != val2:
                    raise AssertionError(f"Value mismatch at row {i}, col {j}: {# PySpark Testing & Quality - Integration Testing and Production Patterns

#pyspark #testing #integration #ci-cd #production #monitoring #data-engineering

## 7.5 Integration Testing

**Integration testing** in PySpark validates that your data pipeline components work together correctly in realistic environments. Unlike unit tests that focus on individual transformations, integration tests verify end-to-end data flows, external system connections, and cross-service interactions that mirror production scenarios.

### End-to-End Pipeline Testing

**End-to-end pipeline testing** ensures that data flows correctly from source systems through all transformation stages to final destinations. This is critical because many issues only surface when components interact - schema mismatches, data format incompatibilities, and timing dependencies that work in isolation but fail in integrated environments.

**Full Data Flow Validation**
```python
import tempfile
import shutil
from pathlib import Path
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, isnan, isnull
from typing import Dict, List, Any

class PipelineIntegrationTester:
    """End-to-end pipeline testing framework
    
    Integration Testing Philosophy:
    1. Realistic Data Flows: Use actual data formats and volumes
    2. External Dependencies: Test with real (or realistic) external systems
    3. Error Propagation: Verify that errors are handled gracefully
    4. Data Quality: Ensure quality is maintained through the pipeline
    5. Performance: Validate acceptable performance under realistic loads
    
    Why Integration Testing Matters:
    - Unit tests can't catch system-level interactions
    - Schema evolution issues only appear in integrated environments
    - Network timeouts and external system failures need testing
    - Data quality issues compound through pipeline stages
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.temp_dir = None
        self.test_artifacts = []
    
    def setup_test_environment(self):
        """Set up isolated test environment"""
        self.temp_dir = Path(tempfile.mkdtemp(prefix="pyspark_integration_"))
        
        # Create test directory structure
        (self.temp_dir / "input").mkdir()
        (self.temp_dir / "staging").mkdir() 
        (self.temp_dir / "output").mkdir()
        (self.temp_dir / "checkpoints").mkdir()
        
        return self.temp_dir
    
    def cleanup_test_environment(self):
        """Clean up test artifacts"""
        if self.temp_dir and self.temp_dir.exists():
            shutil.rmtree(self.temp_dir)
        
        for artifact in self.test_artifacts:
            if hasattr(artifact, 'unpersist'):
                artifact.unpersist()
    
    def create_realistic_input_data(self, record_count: int = 10000) -> str:
        """Generate realistic input data that mirrors production patterns
        
        Production Data Characteristics:
        - Mixed data types (strings, numbers, dates, nulls)
        - Realistic value distributions 
        - Data quality issues (missing values, invalid formats)
        - Multiple file formats and partitions
        """
        from datetime import datetime, timedelta
        import random
        
        # Generate realistic customer transaction data
        base_date = datetime(2023, 1, 1)
        
        transactions = []
        for i in range(record_count):
            # Realistic data patterns with quality issues
            customer_id = random.randint(1, 10000)
            transaction_date = base_date + timedelta(days=random.randint(0, 365))
            
            # Introduce realistic data quality issues
            amount = round(random.uniform(10.0, 5000.0), 2) if random.random() > 0.02 else None  # 2% missing
            product_category = random.choice(['Electronics', 'Clothing', 'Home', 'Books', 'Sports']) if random.random() > 0.01 else None  # 1% missing
            payment_method = random.choice(['Credit', 'Debit', 'Cash', 'PayPal'])
            
            # Some invalid data
            if random.random() < 0.005:  # 0.5% invalid data
                amount = -abs(amount) if amount else -100.0  # Negative amounts
            
            transactions.append((
                i, customer_id, transaction_date.strftime('%Y-%m-%d'), 
                amount, product_category, payment_method
            ))
        
        # Create DataFrame and save as input
        schema = ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"]
        input_df = self.spark.createDataFrame(transactions, schema)
        
        input_path = str(self.temp_dir / "input" / "transactions")
        input_df.write.mode("overwrite").parquet(input_path)
        
        return input_path

def test_complete_data_pipeline(spark):
    """Test complete data pipeline from ingestion to output
    
    Pipeline Stages:
    1. Data Ingestion: Read from source with validation
    2. Data Cleaning: Handle missing values and invalid data
    3. Business Logic: Apply transformations and calculations
    4. Data Quality: Validate output meets requirements
    5. Data Export: Write to target systems
    
    Integration Points to Test:
    - Schema compatibility between stages
    - Error handling and recovery
    - Performance under realistic data volumes
    - Data quality preservation through transformations
    """
    
    tester = PipelineIntegrationTester(spark)
    temp_dir = tester.setup_test_environment()
    
    try:
        # Stage 1: Create realistic input data
        input_path = tester.create_realistic_input_data(record_count=50000)
        
        # Stage 2: Data Ingestion with validation
        def ingest_data(input_path: str):
            """Ingest data with schema validation and basic quality checks"""
            df = spark.read.parquet(input_path)
            
            # Validate expected schema
            expected_columns = ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"]
            missing_columns = set(expected_columns) - set(df.columns)
            if missing_columns:
                raise ValueError(f"Missing required columns: {missing_columns}")
            
            # Basic data quality validation
            total_records = df.count()
            null_amounts = df.filter(col("amount").isNull()).count()
            
            if null_amounts / total_records > 0.1:  # More than 10% missing amounts
                raise ValueError(f"Too many missing amounts: {null_amounts}/{total_records}")
            
            return df
        
        raw_df = ingest_data(input_path)
        print(f"✓ Ingested {raw_df.count():,} records")
        
        # Stage 3: Data Cleaning
        def clean_data(df):
            """Clean and standardize data"""
            cleaned_df = df.filter(
                # Remove invalid transactions
                (col("amount") > 0) & 
                (col("amount") < 100000) &  # Reasonable upper limit
                col("category").isNotNull() &
                col("customer_id").isNotNull()
            ).withColumn(
                # Standardize categories
                "category_standardized",
                when(col("category").isin(['Electronics', 'Clothing', 'Home', 'Books', 'Sports']), col("category"))
                .otherwise("Other")
            ).withColumn(
                # Add derived fields
                "transaction_month",
                col("transaction_date").substr(1, 7)  # YYYY-MM
            )
            
            return cleaned_df
        
        cleaned_df = clean_data(raw_df)
        
        # Validate cleaning preserved data integrity
        original_count = raw_df.count()
        cleaned_count = cleaned_df.count()
        retention_rate = cleaned_count / original_count
        
        print(f"✓ Cleaned data: {cleaned_count:,} records ({retention_rate:.1%} retention)")
        assert retention_rate > 0.8, f"Too much data lost in cleaning: {retention_rate:.1%} retention"
        
        # Stage 4: Business Logic - Monthly aggregations
        def apply_business_logic(df):
            """Apply business transformations"""
            monthly_summary = df.groupBy("transaction_month", "category_standardized") \
                               .agg(
                                   F.sum("amount").alias("total_amount"),
                                   F.count("*").alias("transaction_count"),
                                   F.avg("amount").alias("avg_transaction"),
                                   F.countDistinct("customer_id").alias("unique_customers")
                               ).withColumn(
                                   "revenue_tier",
                                   when(col("total_amount") > 100000, "High")
                                   .when(col("total_amount") > 50000, "Medium")
                                   .otherwise("Low")
                               )
            
            return monthly_summary
        
        business_df = apply_business_logic(cleaned_df)
        business_count = business_df.count()
        
        print(f"✓ Business logic applied: {business_count} summary records")
        
        # Stage 5: Data Quality Validation
        def validate_output_quality(df):
            """Validate final output meets business requirements"""
            quality_issues = []
            
            # Check for negative amounts
            negative_amounts = df.filter(col("total_amount") < 0).count()
            if negative_amounts > 0:
                quality_issues.append(f"Found {negative_amounts} negative total amounts")
            
            # Check for reasonable transaction counts
            low_transaction_counts = df.filter(col("transaction_count") < 1).count()
            if low_transaction_counts > 0:
                quality_issues.append(f"Found {low_transaction_counts} groups with < 1 transaction")
            
            # Check for data completeness
            null_revenues = df.filter(col("total_amount").isNull()).count()
            if null_revenues > 0:
                quality_issues.append(f"Found {null_revenues} null revenue amounts")
            
            return quality_issues
        
        quality_issues = validate_output_quality(business_df)
        
        if quality_issues:
            for issue in quality_issues:
                print(f"✗ Quality issue: {issue}")
            assert False, f"Data quality validation failed: {quality_issues}"
        else:
            print("✓ Output quality validation passed")
        
        # Stage 6: Data Export
        def export_data(df, output_path: str):
            """Export data with partitioning and compression"""
            df.write.mode("overwrite") \
              .partitionBy("transaction_month") \
              .option("compression", "snappy") \
              .parquet(output_path)
        
        output_path = str(temp_dir / "output" / "monthly_summary")
        export_data(business_df, output_path)
        
        # Validate export integrity
        exported_df = spark.read.parquet(output_path)
        exported_count = exported_df.count()
        
        assert exported_count == business_count, \
            f"Export count mismatch: {exported_count} != {business_count}"
        
        print(f"✓ Data exported successfully: {exported_count} records")
        
        # End-to-end validation
        print(f"\n🎉 Pipeline Integration Test PASSED")
        print(f"   Input: {original_count:,} records")
        print(f"   Output: {exported_count:,} summary records")
        print(f"   Data retention: {retention_rate:.1%}")
        
        return {
            "input_count": original_count,
            "output_count": exported_count,
            "retention_rate": retention_rate,
            "quality_issues": quality_issues
        }
        
    finally:
        tester.cleanup_test_environment()

def test_pipeline_error_handling(spark):
    """Test pipeline behavior under error conditions
    
    Error Scenarios to Test:
    1. Invalid input data formats
    2. Missing required columns
    3. External system failures
    4. Resource exhaustion
    5. Network timeouts
    """
    
    tester = PipelineIntegrationTester(spark)
    temp_dir = tester.setup_test_environment()
    
    try:
        # Test 1: Invalid schema
        invalid_schema_data = [
            (1, "wrong_column", "invalid_data"),  # Wrong column names
            (2, "another_wrong", "more_invalid")
        ]
        
        invalid_df = spark.createDataFrame(invalid_schema_data, ["id", "wrong_col1", "wrong_col2"])
        invalid_path = str(temp_dir / "input" / "invalid_schema")
        invalid_df.write.mode("overwrite").parquet(invalid_path)
        
        # Pipeline should detect and handle schema mismatch
        def robust_data_ingestion(input_path: str):
            """Data ingestion with error handling"""
            try:
                df = spark.read.parquet(input_path)
                
                # Validate schema
                required_columns = ["transaction_id", "customer_id", "amount"]
                missing_columns = set(required_columns) - set(df.columns)
                
                if missing_columns:
                    return {"success": False, "error": f"Missing columns: {missing_columns}"}
                
                return {"success": True, "data": df}
                
            except Exception as e:
                return {"success": False, "error": str(e)}
        
        result = robust_data_ingestion(invalid_path)
        assert not result["success"], "Pipeline should detect invalid schema"
        print(f"✓ Invalid schema detected: {result['error']}")
        
        # Test 2: Corrupted data handling
        # Create data with various corruption patterns
        corrupted_data = [
            (1, 1001, "2023-01-01", "not_a_number", "Electronics", "Credit"),  # Invalid amount
            (2, None, "2023-01-02", 150.50, "Clothing", "Debit"),              # Missing customer_id
            (3, 1003, "invalid_date", 75.25, "Books", "Cash"),                 # Invalid date
            (4, 1004, "2023-01-04", 200.00, None, "PayPal")                    # Missing category
        ]
        
        corrupted_df = spark.createDataFrame(corrupted_data, 
                                           ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"])
        corrupted_path = str(temp_dir / "input" / "corrupted")
        corrupted_df.write.mode("overwrite").parquet(corrupted_path)
        
        def robust_data_cleaning(input_path: str):
            """Data cleaning with comprehensive error handling"""
            try:
                df = spark.read.parquet(input_path)
                
                # Count original records
                original_count = df.count()
                
                # Clean data with detailed error tracking
                cleaned_df = df.filter(
                    col("customer_id").isNotNull() &
                    col("transaction_date").rlike("^[0-9]{4}-[0-9]{2}-[0-9]{2}$") &
                    col("amount").cast("double").isNotNull() &
                    col("category").isNotNull()
                )
                
                cleaned_count = cleaned_df.count()
                rejected_count = original_count - cleaned_count
                
                return {
                    "success": True,
                    "original_count": original_count,
                    "cleaned_count": cleaned_count,
                    "rejected_count": rejected_count,
                    "data": cleaned_df
                }
                
            except Exception as e:
                return {"success": False, "error": str(e)}
        
        cleaning_result = robust_data_cleaning(corrupted_path)
        assert cleaning_result["success"], f"Data cleaning failed: {cleaning_result.get('error')}"
        
        print(f"✓ Data cleaning handled corruption:")
        print(f"   Original: {cleaning_result['original_count']} records") 
        print(f"   Cleaned: {cleaning_result['cleaned_count']} records")
        print(f"   Rejected: {cleaning_result['rejected_count']} records")
        
        # Validate that some records were rejected (corruption was detected)
        assert cleaning_result['rejected_count'] > 0, "Should have rejected some corrupted records"
        
        print("✓ Pipeline error handling tests passed")
        
    finally:
        tester.cleanup_test_environment()

### External System Integration

**External system integration testing** validates connections to databases, file systems, APIs, and message queues that your PySpark pipeline depends on. These tests catch configuration issues, authentication problems, and network connectivity issues that only surface in integrated environments.

**Database Integration Testing**
```python
import os
from contextlib import contextmanager
from typing import Optional, Dict, Any

class DatabaseIntegrationTester:
    """Test database connectivity and operations
    
    Database Integration Challenges:
    1. Connection Management: Timeouts, connection pooling, authentication
    2. Schema Compatibility: Data type mappings between Spark and database
    3. Performance: Large data transfers, query optimization
    4. Transaction Handling: Consistency, rollback capabilities
    5. Error Recovery: Network failures, temporary outages
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.test_connections = {}
    
    @contextmanager
    def database_connection(self, db_config: Dict[str, str]):
        """Context manager for database connections with proper cleanup"""
        connection_name = f"{db_config['host']}:{db_config['database']}"
        
        try:
            # Test basic connectivity
            self.test_database_connectivity(db_config)
            self.test_connections[connection_name] = db_config
            yield db_config
            
        except Exception as e:
            print(f"Database connection failed: {e}")
            raise
        finally:
            # Cleanup test data if needed
            self.cleanup_test_data(db_config)
    
    def test_database_connectivity(self, db_config: Dict[str, str]):
        """Test basic database connectivity"""
        try:
            # Create a simple test DataFrame
            test_df = self.spark.createDataFrame([(1, "test")], ["id", "value"])
            
            # Attempt to read from database (this tests connectivity)
            jdbc_url = f"jdbc:postgresql://{db_config['host']}:{db_config.get('port', 5432)}/{db_config['database']}"
            
            # Test connection by reading system tables or creating a temp table
            test_table_name = "spark_connectivity_test"
            
            # Write test data
            test_df.write.format("jdbc") \
                   .option("url", jdbc_url) \
                   .option("dbtable", test_table_name) \
                   .option("user", db_config["user"]) \
                   .option("password", db_config["password"]) \
                   .option("driver", "org.postgresql.Driver") \
                   .mode("overwrite") \
                   .save()
            
            # Read back test data
            read_df = self.spark.read.format("jdbc") \
                           .option("url", jdbc_url) \
                           .option("dbtable", test_table_name) \
                           .option("user", db_config["user"]) \
                           .option("password", db_config["password"]) \
                           .option("driver", "org.postgresql.Driver") \
                           .load()
            
            result_count = read_df.count()
            assert result_count == 1, f"Connectivity test failed: expected 1 record, got {result_count}"
            
            print(f"✓ Database connectivity verified: {db_config['host']}")
            
        except Exception as e:
            raise ConnectionError(f"Database connectivity test failed: {e}")
    
    def cleanup_test_data(self, db_config: Dict[str, str]):
        """Clean up test data from database"""
        try:
            # In a real implementation, you'd drop test tables here
            pass
        except:
            # Non-critical cleanup failure
            pass

def test_database_read_write_integration(spark):
    """Test complete database read/write integration
    
    Database Integration Scenarios:
    1. Large dataset reads with partitioning
    2. Incremental data loading
    3. Schema evolution handling
    4. Error recovery and retry logic
    """
    
    # Mock database configuration (in real tests, use test database)
    db_config = {
        "host": os.getenv("TEST_DB_HOST", "localhost"),
        "port": os.getenv("TEST_DB_PORT", "5432"),
        "database": os.getenv("TEST_DB_NAME", "test_db"),
        "user": os.getenv("TEST_DB_USER", "test_user"),
        "password": os.getenv("TEST_DB_PASSWORD", "test_password")
    }
    
    # Skip test if database not available
    if not all(db_config.values()):
        print("⚠ Skipping database integration test - no database configuration")
        return
    
    tester = DatabaseIntegrationTester(spark)
    
    try:
        with tester.database_connection(db_config) as db:
            # Test 1: Write large dataset to database
            large_dataset = spark.range(100000).select(
                col("id").alias("customer_id"),
                (rand() * 1000).cast("decimal(10,2)").alias("amount"),
                concat(lit("customer_"), col("id")).alias("customer_name"),
                current_timestamp().alias("created_at")
            )
            
            table_name = "integration_test_customers"
            jdbc_url = f"jdbc:postgresql://{db['host']}:{db.get('port', 5432)}/{db['database']}"
            
            # Write with partitioning for performance
            large_dataset.write.format("jdbc") \
                         .option("url", jdbc_url) \
                         .option("dbtable", table_name) \
                         .option("user", db["user"]) \
                         .option("password", db["password"]) \
                         .option("driver", "org.postgresql.Driver") \
                         .option("numPartitions", "4") \
                         .mode("overwrite") \
                         .save()
            
            print(f"✓ Successfully wrote {large_dataset.count():,} records to database")
            
            # Test 2: Read back with query pushdown
            read_df = spark.read.format("jdbc") \
                           .option("url", jdbc_url) \
                           .option("dbtable", f"(SELECT * FROM {table_name} WHERE amount > 500) as filtered_data") \
                           .option("user", db["user"]) \
                           .option("password", db["password"]) \
                           .option("driver", "org.postgresql.Driver") \
                           .load()
            
            filtered_count = read_df.count()
            total_count = large_dataset.count()
            
            print(f"✓ Successfully read {filtered_count:,} filtered records (from {total_count:,} total)")
            
            # Test 3: Incremental loading simulation
            # Add new data to existing table
            incremental_data = spark.range(100000, 110000).select(
                col("id").alias("customer_id"),
                (rand() * 1000).cast("decimal(10,2)").alias("amount"),
                concat(lit("customer_"), col("id")).alias("customer_name"),
                current_timestamp().alias("created_at")
            )
            
            incremental_data.write.format("jdbc") \
                            .option("url", jdbc_url) \
                            .option("dbtable", table_name) \
                            .option("user", db["user"]) \
                            .option("password", db["password"]) \
                            .option("driver", "org.postgresql.Driver") \
                            .mode("append") \
                            .save()
            
            # Verify incremental load
            final_df = spark.read.format("jdbc") \
                            .option("url", jdbc_url) \
                            .option("dbtable", table_name) \
                            .option("user", db["user"]) \
                            .option("password", db["password"]) \
                            .option("driver", "org.postgresql.Driver") \
                            .load()
            
            final_count = final_df.count()
            expected_count = total_count + incremental_data.count()
            
            assert final_count == expected_count, \
                f"Incremental load failed: expected {expected_count}, got {final_count}"
            
            print(f"✓ Incremental loading successful: {final_count:,} total records")
            
            return {
                "initial_count": total_count,
                "incremental_count": incremental_data.count(),
                "final_count": final_count,
                "filtered_count": filtered_count
            }
            
    except Exception as e:
        print(f"✗ Database integration test failed: {e}")
        raise

def test_file_system_integration(spark):
    """Test file system integration across different storage systems
    
    File System Integration Points:
    1. Local filesystem vs distributed storage (HDFS, S3, ADLS)
    2. File format compatibility (Parquet, Delta, CSV, JSON)
    3. Partitioning and compression strategies
    4. Error handling for missing files and permission issues
    """
    
    import tempfile
    from pathlib import Path
    
    with tempfile.TemporaryDirectory() as temp_dir:
        base_path = Path(temp_dir)
        
        # Create test data with various formats
        test_data = spark.range(10000).select(
            col("id"),
            (col("id") % 100).alias("partition_key"),
            (rand() * 1000).alias("value"),
            current_date().alias("date_partition")
        )
        
        # Test 1: Parquet with partitioning
        parquet_path = base_path / "parquet_data"
        test_data.write.mode("overwrite") \
                .partitionBy("partition_key") \
                .option("compression", "snappy") \
                .parquet(str(parquet_path))
        
        # Verify partitioned read
        parquet_df = spark.read.parquet(str(parquet_path))
        assert parquet_df.count() == test_data.count(), "Parquet read/write count mismatch"
        
        # Test partition pruning
        filtered_df = parquet_df.filter(col("partition_key") < 10)
        assert filtered_df.count() < test_data.count(), "Partition pruning should reduce record count"
        
        print(f"✓ Parquet integration: {parquet_df.count():,} records with partitioning")
        
        # Test 2: Delta Lake integration (if available)
        try:
            delta_path = base_path / "delta_data"
            test_data.write.format("delta") \
                     .mode("overwrite") \
                     .save(str(delta_path))
            
            delta_df = spark.read.format("delta").load(str(delta_path))
            assert delta_df.count() == test_data.count(), "Delta read/write count mismatch"
            
            print(f"✓ Delta Lake integration: {delta_df.count():,} records")
            
        except Exception as e:
            print(f"⚠ Delta Lake not available: {e}")
        
        # Test 3: CSV with schema inference
        csv_path = base_path / "csv_data"
        test_data.coalesce(1).write.mode("overwrite") \
                .option("header", "true") \
                .csv(str(csv_path))
        
        csv_df = spark.read.option("header", "true") \
                      .option("inferSchema", "true") \
                      .csv(str(csv_path))
        
        assert csv_df.count() == test_data.count(), "CSV read/write count mismatch"
        print(f"✓ CSV integration: {csv_df.count():,} records with schema inference")
        
        # Test 4: Error handling for missing files
        try:
            missing_df = spark.read.parquet(str(base_path / "nonexistent_path"))
            missing_df.count()  # Should trigger error
            assert False, "Should have failed reading missing path"
        except Exception:
            print("✓ Missing file error handling works correctly")
        
        return {
            "parquet_count": parquet_df.count(),
            "csv_count": csv_df.count(),
            "filtered_count": filtered_df.count()
        }

### Message Queue Integration

**Message queue integration** is critical for real-time data pipelines. Testing Kafka, Kinesis, or other streaming systems requires validating message production, consumption, error handling, and exactly-once processing semantics.

```python
def test_kafka_integration(spark):
    """Test Kafka integration for streaming data
    
    Kafka Integration Testing:
    1. Producer connectivity and message publishing
    2. Consumer connectivity and message consumption  
    3. Schema registry integration
    4. Error handling and retry logic
    5. Offset management and exactly-once processing
    
    Note: This test requires Kafka to be running
    """
    
    kafka_config = {
        "bootstrap_servers": os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092"),
        "topic": "test_integration_topic",
        "schema_registry_url": os.getenv("SCHEMA_REGISTRY_URL", "http://localhost:8081")
    }
    
    # Skip if Kafka not available
    if not kafka_config["bootstrap_servers"]:
        print("⚠ Skipping Kafka integration test - no Kafka configuration")
        return
    
    try:
        # Test 1: Write streaming data to Kafka
        test_stream_data = spark.range(1000).select(
            col("id").cast("string").alias("key"),
            to_json(struct(
                col("id"),
                current_timestamp().alias("timestamp"),
                (rand() * 100).alias("value")
            )).alias("value")
        )
        
        # Write to Kafka (in real streaming, this would be a streaming write)
        kafka_write_query = test_stream_data.write \
            .format("kafka") \
            .option("kafka.bootstrap.servers", kafka_config["bootstrap_servers"]) \
            .option("topic", kafka_config["topic"]) \
            .mode("append") \
            .save()
        
        print(f"✓ Successfully wrote {test_stream_data.count()} messages to Kafka topic: {kafka_config['topic']}")
        
        # Test 2: Read from Kafka
        kafka_df = spark.read \
            .format("kafka") \
            .option("kafka.bootstrap.servers", kafka_config["bootstrap_servers"]) \
            .option("subscribe", kafka_config["topic"]) \
            .option("startingOffsets", "earliest") \
            .load()
        
        # Parse the JSON messages
        parsed_df = kafka_df.select(
            col("key").cast("string"),
            from_json(col("value").cast("string"), 
                     schema=StructType([
                         StructField("id", LongType()),
                         StructField("timestamp", TimestampType()),
                         StructField("value", DoubleType())
                     ])).alias("data")
        ).select("key", "data.*")
        
        kafka_count = parsed_df.count()
        print(f"✓ Successfully read {kafka_count} messages from Kafka")
        
        # Verify data integrity
        assert kafka_
````

# Parametrized test examples

@pytest.mark.parametrize("input_data,expected_count", [ ([("Alice", 25), ("Bob", 30)], 2), ([("Charlie", 35), ("Diana", 28), ("Eve", 32)], 3), ([], 0) # Edge case: empty dataset ]) def test_parametrized_dataframe_creation(spark, input_data, expected_count): """Example of parametrized testing with different datasets""" if input_data: df = spark.createDataFrame(input_data, ["name", "age"]) else: df = spark.createDataFrame([], StructType([ StructField("name", StringType()), StructField("age", IntegerType()) ]))

```
DataFrameAssertions.assert_dataframe_count(df, expected_count)
```

# Custom markers for test organization

@pytest.mark.unit def test_basic_transformation(spark, sample_dataframe): """Unit test for basic transformation logic""" result = sample_dataframe.withColumn("salary_category", F.when(F.col("salary") >= 90000, "High") .when(F.col("salary") >= 75000, "Medium") .otherwise("Low"))

```
# Verify transformation
high_salary_count = result.filter(F.col("salary_category") == "High").count()
assert high_salary_count == 1  # Only Charlie has >= 90000

DataFrameAssertions.assert_schema_contains(result, {
    "id": "int",
    "name": "string", 
    "salary_category": "string"
})
```

@pytest.mark.integration def test_file_read_write_integration(spark, temp_dir): """Integration test for file I/O operations""" # Create test data test_data = spark.range(1000).select( F.col("id"), (F.rand() * 1000).alias("value"), F.concat(F.lit("item_"), F.col("id")).alias("name") )

```
# Write to multiple formats
parquet_path = temp_dir / "test.parquet"
json_path = temp_dir / "test.json"

test_data.write.mode("overwrite").parquet(str(parquet_path))
test_data.write.mode("overwrite").json(str(json_path))

# Read back and verify
parquet_df = spark.read.parquet(str(parquet_path))
json_df = spark.read.json(str(json_path))

DataFrameAssertions.assert_dataframe_count(parquet_df, 1000)
DataFrameAssertions.assert_dataframe_count(json_df, 1000)

# Verify schema preservation
assert parquet_df.schema == test_data.schema
```

@pytest.mark.performance  
def test_large_dataset_performance(spark): """Performance test for large dataset operations""" import time

```
# Create large dataset
large_df = spark.range(1000000).select(
    F.col("id"),
    (F.col("id") % 1000).alias("group_key"),
    F.rand().alias("value")
)

# Time aggregation operation
start_time = time.time()

result = large_df.groupBy("group_key").agg(
    F.sum("value").alias("sum_value"),
    F.count("*").alias("count"),
    F.avg("value").alias("avg_value")
)

result_count = result.count()
execution_time = time.time() - start_time

# Performance assertions
assert result_count == 1000  # Should have 1000 groups
assert execution_time < 30  # Should complete within 30 seconds

throughput = 1000000 / execution_time
print(f"Throughput: {throughput:,.0f} records/second")
```

# Test configuration and collection hooks

def pytest_configure(config): """pytest configuration hook""" config.addinivalue_line("markers", "unit: Unit tests") config.addinivalue_line("markers", "integration: Integration tests") config.addinivalue_line("markers", "performance: Performance tests") config.addinivalue_line("markers", "slow: Slow-running tests")

def pytest_collection_modifyitems(config, items): """Modify test collection to add markers""" for item in items: # Auto-mark slow tests if "large_dataset" in item.name or "performance" in item.name: item.add_marker(pytest.mark.slow)

```
    # Auto-mark integration tests
    if "integration" in item.name or "database" in item.name:
        item.add_marker(pytest.mark.integration)
```

# Pytest configuration file (pytest.ini or pyproject.toml)

PYTEST_CONFIG = """ [tool.pytest.ini_options] minversion = "6.0" addopts = "-ra -q --tb=short" testpaths = ["tests"] python_files = ["test__.py", "__test.py"] python_classes = ["Test*"] python_functions = ["test_*"]

# Custom markers

markers = [ "unit: Unit tests that run quickly", "integration: Integration tests that require external systems", "performance: Performance tests that may take longer", "slow: Tests that take more than 10 seconds" ]

# Logging configuration

log_cli = true log_cli_level = "INFO" log_cli_format = "%(asctime)s [%(levelname)8s] %(name)s: %(message)s" log_cli_date_format = "%Y-%m-%d %H:%M:%S"

# Test discovery

filterwarnings = [ "ignore::UserWarning", "ignore::DeprecationWarning" ] """

````

### Spark Testing Base

**Spark Testing Base** provides specialized utilities for testing Spark applications, including DataFrame comparison methods, RDD testing helpers, and streaming test utilities that handle the complexities of distributed testing.

```python
from pyspark_test import assert_pyspark_df_equal, assert_rdd_equal
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, lit
import pytest

class SparkTestingBaseIntegration:
    """Integration with Spark Testing Base library
    
    Spark Testing Base Benefits:
    1. Specialized DataFrame comparison with better error messages
    2. RDD testing utilities for low-level operations
    3. Streaming test helpers for temporal data
    4. Performance-optimized comparison algorithms
    5. Built-in support for approximate comparisons
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def test_dataframe_comparison_advanced(self):
        """Advanced DataFrame comparison using Spark Testing Base"""
        # Create test DataFrames
        df1 = self.spark.createDataFrame([
            (1, "Alice", 85.5),
            (2, "Bob", 92.3),
            (3, "Charlie", 78.9)
        ], ["id", "name", "score"])
        
        df2 = self.spark.createDataFrame([
            (1, "Alice", 85.5),
            (2, "Bob", 92.3), 
            (3, "Charlie", 78.9)
        ], ["id", "name", "score"])
        
        # Exact comparison
        assert_pyspark_df_equal(df1, df2)
        
        # Test with floating-point tolerance
        df3 = self.spark.createDataFrame([
            (1, "Alice", 85.51),  # Slightly different
            (2, "Bob", 92.29),    # Slightly different
            (3, "Charlie", 78.91) # Slightly different
        ], ["id", "name", "score"])
        
        # This should pass with tolerance
        assert_pyspark_df_equal(df1, df3, rtol=1e-1)  # 10% tolerance
        
        print("✓ Advanced DataFrame comparison tests passed")
    
    def test_rdd_operations(self):
        """Test RDD operations using Spark Testing Base"""
        # Create test RDD
        rdd1 = self.spark.sparkContext.parallelize([1, 2, 3, 4, 5])
        
        # Transform RDD
        squared_rdd = rdd1.map(lambda x: x * x)
        
        # Expected result
        expected_rdd = self.spark.sparkContext.parallelize([1, 4, 9, 16, 25])
        
        # Compare RDDs
        assert_rdd_equal(squared_rdd, expected_rdd)
        
        print("✓ RDD operation tests passed")
    
    def test_streaming_helpers(self):
        """Test streaming operations with specialized helpers"""
        # Note: This requires additional setup for streaming testing
        # In practice, you'd use the streaming test utilities from Spark Testing Base
        
        # Create streaming DataFrame schema
        from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType
        
        streaming_schema = StructType([
            StructField("id", IntegerType()),
            StructField("event", StringType()),
            StructField("timestamp", TimestampType())
        ])
        
        # Mock streaming data
        streaming_data = [
            (1, "click", "2023-01-01 10:00:00"),
            (2, "view", "2023-01-01 10:01:00"),
            (3, "purchase", "2023-01-01 10:02:00")
        ]
        
        # In real streaming tests, you'd use:
        # - MemoryStream for input
        # - MemorySink for output
        # - Manual clock advancement for time-based operations
        
        print("✓ Streaming test framework ready")

def test_spark_testing_base_integration(spark):
    """Integration test for Spark Testing Base utilities"""
    tester = SparkTestingBaseIntegration(spark)
    
    tester.test_dataframe_comparison_advanced()
    tester.test_rdd_operations() 
    tester.test_streaming_helpers()
````

### Great Expectations Integration

**Great Expectations** provides a comprehensive data quality testing framework that integrates well with PySpark for validation-heavy testing scenarios.

```python
import great_expectations as ge
from great_expectations.dataset import SparkDFDataset
from great_expectations.core import ExpectationSuite
import json

class GreatExpectationsIntegration:
    """Integration with Great Expectations for data quality testing
    
    Great Expectations Benefits:
    1. Declarative data quality expectations
    2. Comprehensive validation suite library
    3. Detailed validation reports
    4. Integration with data profiling
    5. Support for custom business rule expectations
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.context = None
    
    def setup_data_context(self):
        """Set up Great Expectations data context"""
        try:
            from great_expectations import DataContext
            self.context = DataContext()
            print("✓ Great Expectations context initialized")
        except Exception as e:
            print(f"⚠ Great Expectations setup failed: {e}")
            return False
        return True
    
    def create_expectation_suite(self, suite_name: str) -> ExpectationSuite:
        """Create a new expectation suite"""
        if not self.context:
            raise RuntimeError("Data context not initialized")
        
        suite = self.context.create_expectation_suite(suite_name, overwrite_existing=True)
        return suite
    
    def test_customer_data_expectations(self):
        """Test customer data against business expectations"""
        # Create sample customer data
        customer_data = self.spark.createDataFrame([
            (1, "Alice Johnson", "alice@email.com", 25, "Premium", 150000.0),
            (2, "Bob Smith", "bob@email.com", 30, "Standard", 75000.0),
            (3, "Charlie Brown", "charlie@email.com", 35, "Premium", 125000.0),
            (4, "Diana Prince", "diana@email.com", 28, "Standard", 85000.0),
            (5, "Eve Adams", "eve@email.com", 32, "Premium", 95000.0)
        ], ["customer_id", "name", "email", "age", "tier", "lifetime_value"])
        
        # Create Great Expectations dataset
        ge_df = SparkDFDataset(customer_data)
        
        # Define business expectations
        expectations_results = {}
        
        # Expectation 1: Customer ID should be unique
        result1 = ge_df.expect_column_values_to_be_unique("customer_id")
        expectations_results["unique_customer_id"] = result1.success
        
        # Expectation 2: Age should be within reasonable range
        result2 = ge_df.expect_column_values_to_be_between("age", min_value=18, max_value=100)
        expectations_results["age_range"] = result2.success
        
        # Expectation 3: Email should follow valid format
        result3 = ge_df.expect_column_values_to_match_regex("email", r"^[^@]+@[^@]+\.[^@]+$")
        expectations_results["email_format"] = result3.success
        
        # Expectation 4: Tier should be from allowed values
        result4 = ge_df.expect_column_values_to_be_in_set("tier", ["Standard", "Premium", "Enterprise"])
        expectations_results["tier_values"] = result4.success
        
        # Expectation 5: Lifetime value should be positive
        result5 = ge_df.expect_column_values_to_be_between("lifetime_value", min_value=0)
        expectations_results["positive_lifetime_value"] = result5.success
        
        # Expectation 6: Premium customers should have higher lifetime value
        premium_customers = ge_df.filter(ge_df.tier == "Premium")
        result6 = premium_customers.expect_column_values_to_be_between("lifetime_value", min_value=90000)
        expectations_results["premium_value_threshold"] = result6.success
        
        # Print results
        print("Great Expectations Validation Results:")
        for expectation, success in expectations_results.items():
            status = "✓ PASS" if success else "✗ FAIL"
            print(f"  {expectation}: {status}")
        
        # Validate all expectations passed
        all_passed = all(expectations_results.values())
        assert all_passed, f"Some expectations failed: {expectations_results}"
        
        return expectations_results
    
    def test_data_profiling(self):
        """Test automatic data profiling capabilities"""
        # Create dataset with various data quality issues
        problematic_data = self.spark.createDataFrame([
            (1, "Alice", 25, "alice@email.com", 50000.0),
            (2, "Bob", None, "bob@email.com", 60000.0),      # Missing age
            (3, "Charlie", 35, "invalid-email", 70000.0),    # Invalid email
            (4, "Diana", 28, "diana@email.com", None),       # Missing salary
            (5, "Eve", 150, "eve@email.com", 80000.0),       # Unrealistic age
            (6, "Frank", 30, "frank@email.com", -5000.0),    # Negative salary
            (7, None, 32, "ghost@email.com", 75000.0)        # Missing name
        ], ["id", "name", "age", "email", "salary"])
        
        ge_df = SparkDFDataset(problematic_data)
        
        # Profile the dataset
        profile_results = {}
        
        # Check completeness
        profile_results["name_completeness"] = ge_df.expect_column_values_to_not_be_null("name").success
        profile_results["age_completeness"] = ge_df.expect_column_values_to_not_be_null("age").success
        profile_results["salary_completeness"] = ge_df.expect_column_values_to_not_be_null("salary").success
        
        # Check data quality issues
        profile_results["age_realistic"] = ge_df.expect_column_values_to_be_between("age", 0, 120).success
        profile_results["salary_positive"] = ge_df.expect_column_values_to_be_between("salary", 0, 1000000).success
        profile_results["email_format"] = ge_df.expect_column_values_to_match_regex("email", r"^[^@]+@[^@]+\.[^@]+$").success
        
        print("Data Quality Profile:")
        for metric, result in profile_results.items():
            status = "✓ GOOD" if result else "⚠ ISSUE"
            print(f"  {metric}: {status}")
        
        # Expect some issues to be detected
        assert not all(profile_results.values()), "Should detect data quality issues"
        
        return profile_results

def test_great_expectations_integration(spark):
    """Test Great Expectations integration"""
    integrator = GreatExpectationsIntegration(spark)
    
    # Skip if Great Expectations not available
    if not integrator.setup_data_context():
        pytest.skip("Great Expectations not available")
    
    # Test business expectations
    customer_results = integrator.test_customer_data_expectations()
    print(f"Customer data validation: {sum(customer_results.values())}/{len(customer_results)} passed")
    
    # Test data profiling
    profile_results = integrator.test_data_profiling()
    issues_detected = len([r for r in profile_results.values() if not r])
    print(f"Data profiling detected {issues_detected} quality issues")
```

### Custom Testing Utilities

**Custom testing utilities** fill gaps left by general-purpose frameworks, providing domain-specific assertion methods, test data generators, and validation patterns tailored to your specific use cases.

````python
from typing import List, Dict, Any, Optional, Callable
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, isnan, isnull, count, when
import random
from datetime import datetime, timedelta

class CustomPySparkAssertions:
    """Custom assertion methods for domain-specific testing
    
    Custom Utilities Address:
    1. Business-specific validation patterns
    2. Performance-aware assertion methods
    3. Complex data relationship validation
    4. Time-series and temporal data testing
    5. Custom error messages and debugging aids
    """
    
    @staticmethod
    def assert_no_duplicates(df: DataFrame, subset: List[str] = None):
        """Assert DataFrame has no duplicate rows"""
        total_count = df.count()
        
        if subset:
            distinct_count = df.dropDuplicates(subset).count()
            duplicate_count = total_count - distinct_count
            
            if duplicate_count > 0:
                # Show sample duplicates for debugging
                duplicates = df.groupBy(*subset).count().filter(col("count") > 1)
                sample_duplicates = duplicates.limit(5).collect()
                
                raise AssertionError(
                    f"Found {duplicate_count} duplicate rows based on {subset}. "
                    f"Sample duplicates: {sample_duplicates}"
                )
        else:
            distinct_count = df.distinct().count()
            if total_count != distinct_count:
                duplicate_count = total_count - distinct_count
                raise AssertionError(f"Found {duplicate_count} duplicate rows")
    
    @staticmethod
    def assert_referential_integrity(child_df: DataFrame, parent_df: DataFrame, 
                                   child_key: str, parent_key: str):
        """Assert referential integrity between DataFrames"""
        # Find orphaned records
        orphaned = child_df.join(parent_df, child_df[child_key] == parent_df[parent_key], "left_anti")
        orphaned_count = orphaned.count()
        
        if orphaned_count > 0:
            # Show sample orphaned records
            sample_orphaned = orphaned.select(child_key).distinct().limit(5).collect()
            
            raise AssertionError(
                f"Found {orphaned_count} orphaned records. "
                f"Sample orphaned keys: {[row[child_key] for row in sample_orphaned]}"
            )
    
    @staticmethod
    def assert_data_freshness(df: DataFrame, timestamp_col: str, 
                            max_age_hours: int = 24):
        """Assert data is fresh within specified time window"""
        from pyspark.sql.functions import max as max_func, current_timestamp, col
        
        latest_timestamp = df.agg(max_func(timestamp_col).alias("latest")).collect()[0]["latest"]
        
        if latest_timestamp is None:
            raise AssertionError("No timestamp data found")
        
        # Calculate age in hours
        current_time = datetime.now()
        if isinstance(latest_timestamp, str):
            latest_timestamp = datetime.strptime(latest_timestamp, "%Y-%m-%d %H:%M:%S")
        
        age_hours = (current_time - latest_timestamp).total_seconds() / 3600
        
        if age_hours > max_age_hours:
            raise AssertionError(
                f"Data is stale. Latest timestamp: {latest_timestamp}, "
                f"Age: {age_hours:.1f} hours (max allowed: {max_age_hours})"
            )
    
    @staticmethod
    def assert_statistical_properties(df: DataFrame, column: str, 
                                    expected_mean: float = None,
                                    expected_std: float = None,
                                    tolerance: float = 0.1):
        """Assert statistical properties of numeric columns"""
        from pyspark.sql.functions import mean, stddev
        
        stats = df.agg(
            mean(column).alias("mean"),
            stddev(column).alias("stddev")
        ).collect()[0]
        
        actual_mean = stats["mean"]
        actual_std = stats["stddev"]
        
        if expected_mean is not None:
            if abs(actual_mean - expected_mean) > abs(expected_mean * tolerance):
                raise AssertionError(
                    f"Mean outside tolerance: expected {expected_mean}, "
                    f"got {actual_mean} (tolerance: {tolerance})"
                )
        
        if expected_std is not None:
            if abs(actual_std - expected_std) > abs(expected_std * tolerance):
                raise AssertionError(
                    f"Standard deviation outside tolerance: expected {expected_std}, "
                    f"got {actual_std} (tolerance: {tolerance})"
                )

class TestDataFactory:
    """Factory for generating realistic test data
    
    Test Data Generation Strategies:
    1. Realistic distributions and patterns
    2. Configurable data quality issues
    3. Time-series data with seasonality
    4. Hierarchical and relational data
    5. Edge cases and boundary conditions
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
    
    def create_customer_transactions(self, num_customers: int = 1000, 
                                   num_transactions: int = 10000,
                                   start_date: str = "2023-01-01",
                                   end_date: str = "2023-12-31") -> DataFrame:
        """Generate realistic customer transaction data"""
        import random
        from datetime import datetime, timedelta
        
        # Generate customers
        customers = []
        for i in range(1, num_customers + 1):
            tier = random.choices(
                ["Bronze", "Silver", "Gold", "Platinum"],
                weights=[50, 30, 15, 5]
            )[0]
            
            customers.append({
                "customer_id": i,
                "tier": tier,
                "signup_date": self._random_date(start_date, end_date),
                "region": random.choice(["North", "South", "East", "West", "Central"])
            })
        
        customers_df = self.spark.createDataFrame(customers)
        
        # Generate transactions
        transactions = []
        start_dt = datetime.strptime(start_date, "%Y-%m-%d")
        end_dt = datetime.strptime(end_date, "%Y-%m-%d")
        
        for i in range(1, num_transactions + 1):
            customer_id = random.randint(1, num_customers)
            
            # Transaction amount based on customer tier
            base_amount = random.uniform(10, 500)
            tier_multiplier = {"Bronze": 1.0, "Silver": 1.5, "Gold": 2.0, "Platinum": 3.0}
            
            # Get customer tier (simplified - in practice you'd join)
            customer_tier = random.choice(list(tier_multiplier.keys()))
            amount = base_amount * tier_multiplier[customer_tier]
            
            # Add some seasonality (higher spending in Q4)
            transaction_date = self._random_datetime(start_dt, end_dt)
            if transaction_date.month in [11, 12]:  # Q4 boost
                amount *= 1.3
            
            transactions.append({
                "transaction_id": i,
                "customer_id": customer_id,
                "transaction_date": transaction_date.strftime("%Y-%m-%d %H:%M:%S"),
                "amount": round(amount, 2),
                "category": random.choice(["Electronics", "Clothing", "Food", "Travel", "Entertainment"]),
                "payment_method": random.choice(["Credit", "Debit", "Cash", "Digital"])
            })
        
        transactions_df = self.spark.createDataFrame(transactions)
        
        return customers_df, transactions_df
    
    def create_time_series_data(self, start_date: str, end_date: str, 
                              frequency_minutes: int = 15,
                              with_anomalies: bool = True) -> DataFrame:
        """Generate time series data with realistic patterns"""
        from datetime import datetime, timedelta
        import math
        
        start_dt = datetime.strptime(start_date, "%Y-%m-%d")
        end_dt = datetime.strptime(end_date, "%Y-%m-%d")
        
        current_time = start_dt
        time_series_data = []
        
        while current_time <= end_dt:
            # Base value with daily pattern
            hour_of_day = current_time.hour
            daily_pattern = 50 + 30 * math.sin((hour_of_day - 6) * math.pi / 12)
            
            # Weekly pattern (lower on weekends)
            day_of_week = current_time.weekday()
            weekly_multiplier = 0.7 if day_of_week >= 5 else 1.0
            
            # Add noise
            noise = random.uniform(-10, 10)
            value = daily_pattern * weekly_multiplier + noise
            
            # Add anomalies
            if with_anomalies and random.random() < 0.02:  # 2% anomaly rate
                value *= random.uniform(2.0, 5.0)  # Spike
            
            time_series_data.append({
                "timestamp": current_time.strftime("%Y-%m-%d %H:%M:%S"),
                "value": round(value, 2),
                "hour_of_day": hour_of_day,
                "day_of_week": day_of_week,
                "is_weekend": day_of_week >= 5
            })
            
            current_time += timedelta(minutes=frequency_minutes)
        
        return self.spark.createDataFrame(time_series_data)
    
    def _random_date(self, start_date: str, end_date: str) -> str:
        """Generate random date within range"""
        start_dt = datetime.strptime(start_date, "%Y-%m-%d")
        end_dt = datetime.strptime(end_date, "%Y-%m-%d")
        
        delta = end_dt - start_dt
        random_days = random.randint(0, delta.days)
        
        return (start_dt + timedelta(days=random_days)).strftime("%Y-%m-%d")
    
    def _random_datetime(self, start_dt: datetime, end_dt: datetime) -> datetime:
        """Generate random datetime within range"""
        delta = end_dt - start_dt
        random_seconds = random.randint(0, int(delta.total_seconds()))
        
        return start_dt + timedelta(seconds=random_seconds)

def test_custom_utilities_integration(spark):
    """Test custom testing utilities"""
    # Test data factory
    factory = TestDataFactory(spark)
    
    # Generate test data
    customers_df, transactions_df = factory.create_customer_transactions(
        num_customers=100, num_transactions=1000
    )
    
    print(f"Generated {customers_df.count()} customers and {transactions_df.count()} transactions")
    
    # Test custom assertions
    assertions = CustomPySparkAssertions()
    
    # Test no duplicates
    assertions.assert_no_duplicates(customers_df, subset=["customer_id"])
    assertions.assert_no_duplicates(transactions_df, subset=["transaction_id"])
    
    # Test referential integrity
    assertions.assert_referential_integrity(
        transactions_df, customers_df, "customer_id", "customer_id"
    )
    
    # Test statistical properties
    assertions.assert_statistical_properties(
        transactions_df, "amount", 
        expected_mean=150.0, tolerance=0.5  # Allow 50% variation
    )
    
    # Generate time series data
    ts_df = factory.create_time_series_data("2023-01-01", "2023-01-07")
    print(f"Generated {ts_df.count---

## 7.6 Testing Frameworks & Tools

**Testing frameworks and tools** provide the infrastructure needed to write, execute, and maintain comprehensive test suites for PySpark applications. The right combination of tools can dramatically improve test reliability, reduce maintenance overhead, and provide better insights into test failures.

### pytest with PySpark

**pytest integration** with PySpark requires careful management of SparkSession lifecycle, test isolation, and resource cleanup. The framework's fixture system is particularly well-suited for managing the complex setup and teardown requirements of distributed testing.

**Advanced pytest Configuration**
```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.types import *
import tempfile
import shutil
from pathlib import Path
import logging
from typing import Generator, Dict, Any

# Configure logging for tests
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class PySparkTestConfig:
    """Centralized configuration for PySpark testing
    
    Configuration Management Strategy:
    1. Environment-specific settings (local vs CI vs cluster)
    2. Resource allocation based on test type
    3. Feature flags for optional components
    4. Performance tuning for test execution
    """
    
    @staticmethod
    def get_test_spark_config() -> Dict[str, str]:
        """Get optimized Spark configuration for testing"""
        return {
            # Core settings
            "spark.app.name": "PySpark-Test-Suite",
            "spark.master": "local[*]",
            
            # Memory settings optimized for tests
            "spark.driver.memory": "2g",
            "spark.executor.memory": "1g",
            "spark.driver.maxResultSize": "1g",
            
            # Performance settings for faster tests
            "spark.sql.shuffle.partitions": "4",  # Reduced from default 200
            "spark.sql.adaptive.enabled": "false",  # Consistent test results
            "spark.sql.adaptive.coalescePartitions.enabled": "false",
            
            # Serialization for better performance
            "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
            "spark.kryo.unsafe": "true",
            
            # Disable unnecessary features for tests
            "spark.ui.enabled": "false",
            "spark.sql.execution.arrow.pyspark.enabled": "true",
            
            # Checkpoint and temporary directories
            "spark.sql.streaming.checkpointLocation": "/tmp/spark-checkpoints",
            "spark.local.dir": "/tmp/spark-local",
            
            # Logging configuration
            "spark.sql.execution.arrow.maxRecordsPerBatch": "1000"
        }

@pytest.fixture(scope="session")
def spark_session() -> Generator[SparkSession, None, None]:
    """Session-scoped SparkSession for efficient test execution
    
    Session Scope Benefits:
    - Single SparkSession creation per test session (major performance gain)
    - Shared executor JVMs across tests
    - Consistent Spark configuration throughout test suite
    - Reduced test execution time from ~5min to ~30sec for typical suites
    
    Important: Tests must be designed to not interfere with each other
    """
    config = PySparkTestConfig.get_test_spark_config()
    
    builder = SparkSession.builder
    for key, value in config.items():
        builder = builder.config(key, value)
    
    spark = builder.getOrCreate()
    
    # Set log level to reduce noise during tests
    spark.sparkContext.setLogLevel("WARN")
    
    logger.info(f"Created SparkSession with {spark.sparkContext.defaultParallelism} cores")
    
    yield spark
    
    # Cleanup
    spark.stop()
    logger.info("SparkSession stopped")

@pytest.fixture
def spark(spark_session) -> SparkSession:
    """Function-scoped alias for SparkSession
    
    This fixture provides a clean interface for individual tests while
    reusing the session-scoped SparkSession for performance.
    """
    return spark_session

@pytest.fixture
def temp_dir() -> Generator[Path, None, None]:
    """Temporary directory for test files"""
    temp_path = Path(tempfile.mkdtemp(prefix="pyspark_test_"))
    try:
        yield temp_path
    finally:
        shutil.rmtree(temp_path, ignore_errors=True)

@pytest.fixture
def sample_dataframe(spark) -> "DataFrame":
    """Standard test DataFrame for common test scenarios"""
    data = [
        (1, "Alice", 25, "Engineer", 75000.0, "2020-01-15"),
        (2, "Bob", 30, "Manager", 85000.0, "2019-03-22"),
        (3, "Charlie", 35, "Director", 95000.0, "2018-07-10"),
        (4, "Diana", 28, "Engineer", 70000.0, "2021-02-01"),
        (5, "Eve", 32, "Manager", 80000.0, "2020-11-30")
    ]
    
    schema = StructType([
        StructField("id", IntegerType(), False),
        StructField("name", StringType(), False),
        StructField("age", IntegerType(), False),
        StructField("title", StringType(), False),
        StructField("salary", DoubleType(), False),
        StructField("hire_date", StringType(), False)
    ])
    
    return spark.createDataFrame(data, schema)

class DataFrameAssertions:
    """Custom assertion methods for DataFrame testing
    
    Assertion Strategy:
    1. Schema validation with detailed error messages
    2. Data validation with row-by-row comparison
    3. Approximate comparisons for floating-point data
    4. Performance-aware assertions for large datasets
    """
    
    @staticmethod
    def assert_dataframe_equal(df1, df2, check_order: bool = True, rtol: float = 1e-5):
        """Assert that two DataFrames are equal
        
        Parameters:
        - df1, df2: DataFrames to compare
        - check_order: Whether row order matters
        - rtol: Relative tolerance for floating-point comparisons
        """
        # Schema comparison
        if df1.schema != df2.schema:
            raise AssertionError(f"Schema mismatch:\nDF1: {df1.schema}\nDF2: {df2.schema}")
        
        # Count comparison
        count1, count2 = df1.count(), df2.count()
        if count1 != count2:
            raise AssertionError(f"Row count mismatch: {count1} != {count2}")
        
        # Data comparison
        if not check_order:
            df1 = df1.orderBy(*df1.columns)
            df2 = df2.orderBy(*df2.columns)
        
        # Collect for detailed comparison (only safe for test datasets)
        if count1 > 10000:
            logger.warning(f"Large DataFrame comparison ({count1} rows) - consider sampling")
        
        rows1 = df1.collect()
        rows2 = df2.collect()
        
        for i, (row1, row2) in enumerate(zip(rows1, rows2)):
            for j, (val1, val2) in enumerate(zip(row1, row2)):
                if isinstance(val1, float) and isinstance(val2, float):
                    if not pytest.approx(val1, rel=rtol) == val2:
                        raise AssertionError(f"Float mismatch at row {i}, col {j}: {val1} != {val2}")
                elif val1 != val2:
                    raise AssertionError(f"Value mismatch at row {i}, col {j}: {# PySpark Testing & Quality - Integration Testing and Production Patterns

#pyspark #testing #integration #ci-cd #production #monitoring #data-engineering

## 7.5 Integration Testing

**Integration testing** in PySpark validates that your data pipeline components work together correctly in realistic environments. Unlike unit tests that focus on individual transformations, integration tests verify end-to-end data flows, external system connections, and cross-service interactions that mirror production scenarios.

### End-to-End Pipeline Testing

**End-to-end pipeline testing** ensures that data flows correctly from source systems through all transformation stages to final destinations. This is critical because many issues only surface when components interact - schema mismatches, data format incompatibilities, and timing dependencies that work in isolation but fail in integrated environments.

**Full Data Flow Validation**
```python
import tempfile
import shutil
from pathlib import Path
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, when, isnan, isnull
from typing import Dict, List, Any

class PipelineIntegrationTester:
    """End-to-end pipeline testing framework
    
    Integration Testing Philosophy:
    1. Realistic Data Flows: Use actual data formats and volumes
    2. External Dependencies: Test with real (or realistic) external systems
    3. Error Propagation: Verify that errors are handled gracefully
    4. Data Quality: Ensure quality is maintained through the pipeline
    5. Performance: Validate acceptable performance under realistic loads
    
    Why Integration Testing Matters:
    - Unit tests can't catch system-level interactions
    - Schema evolution issues only appear in integrated environments
    - Network timeouts and external system failures need testing
    - Data quality issues compound through pipeline stages
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.temp_dir = None
        self.test_artifacts = []
    
    def setup_test_environment(self):
        """Set up isolated test environment"""
        self.temp_dir = Path(tempfile.mkdtemp(prefix="pyspark_integration_"))
        
        # Create test directory structure
        (self.temp_dir / "input").mkdir()
        (self.temp_dir / "staging").mkdir() 
        (self.temp_dir / "output").mkdir()
        (self.temp_dir / "checkpoints").mkdir()
        
        return self.temp_dir
    
    def cleanup_test_environment(self):
        """Clean up test artifacts"""
        if self.temp_dir and self.temp_dir.exists():
            shutil.rmtree(self.temp_dir)
        
        for artifact in self.test_artifacts:
            if hasattr(artifact, 'unpersist'):
                artifact.unpersist()
    
    def create_realistic_input_data(self, record_count: int = 10000) -> str:
        """Generate realistic input data that mirrors production patterns
        
        Production Data Characteristics:
        - Mixed data types (strings, numbers, dates, nulls)
        - Realistic value distributions 
        - Data quality issues (missing values, invalid formats)
        - Multiple file formats and partitions
        """
        from datetime import datetime, timedelta
        import random
        
        # Generate realistic customer transaction data
        base_date = datetime(2023, 1, 1)
        
        transactions = []
        for i in range(record_count):
            # Realistic data patterns with quality issues
            customer_id = random.randint(1, 10000)
            transaction_date = base_date + timedelta(days=random.randint(0, 365))
            
            # Introduce realistic data quality issues
            amount = round(random.uniform(10.0, 5000.0), 2) if random.random() > 0.02 else None  # 2% missing
            product_category = random.choice(['Electronics', 'Clothing', 'Home', 'Books', 'Sports']) if random.random() > 0.01 else None  # 1% missing
            payment_method = random.choice(['Credit', 'Debit', 'Cash', 'PayPal'])
            
            # Some invalid data
            if random.random() < 0.005:  # 0.5% invalid data
                amount = -abs(amount) if amount else -100.0  # Negative amounts
            
            transactions.append((
                i, customer_id, transaction_date.strftime('%Y-%m-%d'), 
                amount, product_category, payment_method
            ))
        
        # Create DataFrame and save as input
        schema = ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"]
        input_df = self.spark.createDataFrame(transactions, schema)
        
        input_path = str(self.temp_dir / "input" / "transactions")
        input_df.write.mode("overwrite").parquet(input_path)
        
        return input_path

def test_complete_data_pipeline(spark):
    """Test complete data pipeline from ingestion to output
    
    Pipeline Stages:
    1. Data Ingestion: Read from source with validation
    2. Data Cleaning: Handle missing values and invalid data
    3. Business Logic: Apply transformations and calculations
    4. Data Quality: Validate output meets requirements
    5. Data Export: Write to target systems
    
    Integration Points to Test:
    - Schema compatibility between stages
    - Error handling and recovery
    - Performance under realistic data volumes
    - Data quality preservation through transformations
    """
    
    tester = PipelineIntegrationTester(spark)
    temp_dir = tester.setup_test_environment()
    
    try:
        # Stage 1: Create realistic input data
        input_path = tester.create_realistic_input_data(record_count=50000)
        
        # Stage 2: Data Ingestion with validation
        def ingest_data(input_path: str):
            """Ingest data with schema validation and basic quality checks"""
            df = spark.read.parquet(input_path)
            
            # Validate expected schema
            expected_columns = ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"]
            missing_columns = set(expected_columns) - set(df.columns)
            if missing_columns:
                raise ValueError(f"Missing required columns: {missing_columns}")
            
            # Basic data quality validation
            total_records = df.count()
            null_amounts = df.filter(col("amount").isNull()).count()
            
            if null_amounts / total_records > 0.1:  # More than 10% missing amounts
                raise ValueError(f"Too many missing amounts: {null_amounts}/{total_records}")
            
            return df
        
        raw_df = ingest_data(input_path)
        print(f"✓ Ingested {raw_df.count():,} records")
        
        # Stage 3: Data Cleaning
        def clean_data(df):
            """Clean and standardize data"""
            cleaned_df = df.filter(
                # Remove invalid transactions
                (col("amount") > 0) & 
                (col("amount") < 100000) &  # Reasonable upper limit
                col("category").isNotNull() &
                col("customer_id").isNotNull()
            ).withColumn(
                # Standardize categories
                "category_standardized",
                when(col("category").isin(['Electronics', 'Clothing', 'Home', 'Books', 'Sports']), col("category"))
                .otherwise("Other")
            ).withColumn(
                # Add derived fields
                "transaction_month",
                col("transaction_date").substr(1, 7)  # YYYY-MM
            )
            
            return cleaned_df
        
        cleaned_df = clean_data(raw_df)
        
        # Validate cleaning preserved data integrity
        original_count = raw_df.count()
        cleaned_count = cleaned_df.count()
        retention_rate = cleaned_count / original_count
        
        print(f"✓ Cleaned data: {cleaned_count:,} records ({retention_rate:.1%} retention)")
        assert retention_rate > 0.8, f"Too much data lost in cleaning: {retention_rate:.1%} retention"
        
        # Stage 4: Business Logic - Monthly aggregations
        def apply_business_logic(df):
            """Apply business transformations"""
            monthly_summary = df.groupBy("transaction_month", "category_standardized") \
                               .agg(
                                   F.sum("amount").alias("total_amount"),
                                   F.count("*").alias("transaction_count"),
                                   F.avg("amount").alias("avg_transaction"),
                                   F.countDistinct("customer_id").alias("unique_customers")
                               ).withColumn(
                                   "revenue_tier",
                                   when(col("total_amount") > 100000, "High")
                                   .when(col("total_amount") > 50000, "Medium")
                                   .otherwise("Low")
                               )
            
            return monthly_summary
        
        business_df = apply_business_logic(cleaned_df)
        business_count = business_df.count()
        
        print(f"✓ Business logic applied: {business_count} summary records")
        
        # Stage 5: Data Quality Validation
        def validate_output_quality(df):
            """Validate final output meets business requirements"""
            quality_issues = []
            
            # Check for negative amounts
            negative_amounts = df.filter(col("total_amount") < 0).count()
            if negative_amounts > 0:
                quality_issues.append(f"Found {negative_amounts} negative total amounts")
            
            # Check for reasonable transaction counts
            low_transaction_counts = df.filter(col("transaction_count") < 1).count()
            if low_transaction_counts > 0:
                quality_issues.append(f"Found {low_transaction_counts} groups with < 1 transaction")
            
            # Check for data completeness
            null_revenues = df.filter(col("total_amount").isNull()).count()
            if null_revenues > 0:
                quality_issues.append(f"Found {null_revenues} null revenue amounts")
            
            return quality_issues
        
        quality_issues = validate_output_quality(business_df)
        
        if quality_issues:
            for issue in quality_issues:
                print(f"✗ Quality issue: {issue}")
            assert False, f"Data quality validation failed: {quality_issues}"
        else:
            print("✓ Output quality validation passed")
        
        # Stage 6: Data Export
        def export_data(df, output_path: str):
            """Export data with partitioning and compression"""
            df.write.mode("overwrite") \
              .partitionBy("transaction_month") \
              .option("compression", "snappy") \
              .parquet(output_path)
        
        output_path = str(temp_dir / "output" / "monthly_summary")
        export_data(business_df, output_path)
        
        # Validate export integrity
        exported_df = spark.read.parquet(output_path)
        exported_count = exported_df.count()
        
        assert exported_count == business_count, \
            f"Export count mismatch: {exported_count} != {business_count}"
        
        print(f"✓ Data exported successfully: {exported_count} records")
        
        # End-to-end validation
        print(f"\n🎉 Pipeline Integration Test PASSED")
        print(f"   Input: {original_count:,} records")
        print(f"   Output: {exported_count:,} summary records")
        print(f"   Data retention: {retention_rate:.1%}")
        
        return {
            "input_count": original_count,
            "output_count": exported_count,
            "retention_rate": retention_rate,
            "quality_issues": quality_issues
        }
        
    finally:
        tester.cleanup_test_environment()

def test_pipeline_error_handling(spark):
    """Test pipeline behavior under error conditions
    
    Error Scenarios to Test:
    1. Invalid input data formats
    2. Missing required columns
    3. External system failures
    4. Resource exhaustion
    5. Network timeouts
    """
    
    tester = PipelineIntegrationTester(spark)
    temp_dir = tester.setup_test_environment()
    
    try:
        # Test 1: Invalid schema
        invalid_schema_data = [
            (1, "wrong_column", "invalid_data"),  # Wrong column names
            (2, "another_wrong", "more_invalid")
        ]
        
        invalid_df = spark.createDataFrame(invalid_schema_data, ["id", "wrong_col1", "wrong_col2"])
        invalid_path = str(temp_dir / "input" / "invalid_schema")
        invalid_df.write.mode("overwrite").parquet(invalid_path)
        
        # Pipeline should detect and handle schema mismatch
        def robust_data_ingestion(input_path: str):
            """Data ingestion with error handling"""
            try:
                df = spark.read.parquet(input_path)
                
                # Validate schema
                required_columns = ["transaction_id", "customer_id", "amount"]
                missing_columns = set(required_columns) - set(df.columns)
                
                if missing_columns:
                    return {"success": False, "error": f"Missing columns: {missing_columns}"}
                
                return {"success": True, "data": df}
                
            except Exception as e:
                return {"success": False, "error": str(e)}
        
        result = robust_data_ingestion(invalid_path)
        assert not result["success"], "Pipeline should detect invalid schema"
        print(f"✓ Invalid schema detected: {result['error']}")
        
        # Test 2: Corrupted data handling
        # Create data with various corruption patterns
        corrupted_data = [
            (1, 1001, "2023-01-01", "not_a_number", "Electronics", "Credit"),  # Invalid amount
            (2, None, "2023-01-02", 150.50, "Clothing", "Debit"),              # Missing customer_id
            (3, 1003, "invalid_date", 75.25, "Books", "Cash"),                 # Invalid date
            (4, 1004, "2023-01-04", 200.00, None, "PayPal")                    # Missing category
        ]
        
        corrupted_df = spark.createDataFrame(corrupted_data, 
                                           ["transaction_id", "customer_id", "transaction_date", "amount", "category", "payment_method"])
        corrupted_path = str(temp_dir / "input" / "corrupted")
        corrupted_df.write.mode("overwrite").parquet(corrupted_path)
        
        def robust_data_cleaning(input_path: str):
            """Data cleaning with comprehensive error handling"""
            try:
                df = spark.read.parquet(input_path)
                
                # Count original records
                original_count = df.count()
                
                # Clean data with detailed error tracking
                cleaned_df = df.filter(
                    col("customer_id").isNotNull() &
                    col("transaction_date").rlike("^[0-9]{4}-[0-9]{2}-[0-9]{2}$") &
                    col("amount").cast("double").isNotNull() &
                    col("category").isNotNull()
                )
                
                cleaned_count = cleaned_df.count()
                rejected_count = original_count - cleaned_count
                
                return {
                    "success": True,
                    "original_count": original_count,
                    "cleaned_count": cleaned_count,
                    "rejected_count": rejected_count,
                    "data": cleaned_df
                }
                
            except Exception as e:
                return {"success": False, "error": str(e)}
        
        cleaning_result = robust_data_cleaning(corrupted_path)
        assert cleaning_result["success"], f"Data cleaning failed: {cleaning_result.get('error')}"
        
        print(f"✓ Data cleaning handled corruption:")
        print(f"   Original: {cleaning_result['original_count']} records") 
        print(f"   Cleaned: {cleaning_result['cleaned_count']} records")
        print(f"   Rejected: {cleaning_result['rejected_count']} records")
        
        # Validate that some records were rejected (corruption was detected)
        assert cleaning_result['rejected_count'] > 0, "Should have rejected some corrupted records"
        
        print("✓ Pipeline error handling tests passed")
        
    finally:
        tester.cleanup_test_environment()

### External System Integration

**External system integration testing** validates connections to databases, file systems, APIs, and message queues that your PySpark pipeline depends on. These tests catch configuration issues, authentication problems, and network connectivity issues that only surface in integrated environments.

**Database Integration Testing**
```python
import os
from contextlib import contextmanager
from typing import Optional, Dict, Any

class DatabaseIntegrationTester:
    """Test database connectivity and operations
    
    Database Integration Challenges:
    1. Connection Management: Timeouts, connection pooling, authentication
    2. Schema Compatibility: Data type mappings between Spark and database
    3. Performance: Large data transfers, query optimization
    4. Transaction Handling: Consistency, rollback capabilities
    5. Error Recovery: Network failures, temporary outages
    """
    
    def __init__(self, spark: SparkSession):
        self.spark = spark
        self.test_connections = {}
    
    @contextmanager
    def database_connection(self, db_config: Dict[str, str]):
        """Context manager for database connections with proper cleanup"""
        connection_name = f"{db_config['host']}:{db_config['database']}"
        
        try:
            # Test basic connectivity
            self.test_database_connectivity(db_config)
            self.test_connections[connection_name] = db_config
            yield db_config
            
        except Exception as e:
            print(f"Database connection failed: {e}")
            raise
        finally:
            # Cleanup test data if needed
            self.cleanup_test_data(db_config)
    
    def test_database_connectivity(self, db_config: Dict[str, str]):
        """Test basic database connectivity"""
        try:
            # Create a simple test DataFrame
            test_df = self.spark.createDataFrame([(1, "test")], ["id", "value"])
            
            # Attempt to read from database (this tests connectivity)
            jdbc_url = f"jdbc:postgresql://{db_config['host']}:{db_config.get('port', 5432)}/{db_config['database']}"
            
            # Test connection by reading system tables or creating a temp table
            test_table_name = "spark_connectivity_test"
            
            # Write test data
            test_df.write.format("jdbc") \
                   .option("url", jdbc_url) \
                   .option("dbtable", test_table_name) \
                   .option("user", db_config["user"]) \
                   .option("password", db_config["password"]) \
                   .option("driver", "org.postgresql.Driver") \
                   .mode("overwrite") \
                   .save()
            
            # Read back test data
            read_df = self.spark.read.format("jdbc") \
                           .option("url", jdbc_url) \
                           .option("dbtable", test_table_name) \
                           .option("user", db_config["user"]) \
                           .option("password", db_config["password"]) \
                           .option("driver", "org.postgresql.Driver") \
                           .load()
            
            result_count = read_df.count()
            assert result_count == 1, f"Connectivity test failed: expected 1 record, got {result_count}"
            
            print(f"✓ Database connectivity verified: {db_config['host']}")
            
        except Exception as e:
            raise ConnectionError(f"Database connectivity test failed: {e}")
    
    def cleanup_test_data(self, db_config: Dict[str, str]):
        """Clean up test data from database"""
        try:
            # In a real implementation, you'd drop test tables here
            pass
        except:
            # Non-critical cleanup failure
            pass

def test_database_read_write_integration(spark):
    """Test complete database read/write integration
    
    Database Integration Scenarios:
    1. Large dataset reads with partitioning
    2. Incremental data loading
    3. Schema evolution handling
    4. Error recovery and retry logic
    """
    
    # Mock database configuration (in real tests, use test database)
    db_config = {
        "host": os.getenv("TEST_DB_HOST", "localhost"),
        "port": os.getenv("TEST_DB_PORT", "5432"),
        "database": os.getenv("TEST_DB_NAME", "test_db"),
        "user": os.getenv("TEST_DB_USER", "test_user"),
        "password": os.getenv("TEST_DB_PASSWORD", "test_password")
    }
    
    # Skip test if database not available
    if not all(db_config.values()):
        print("⚠ Skipping database integration test - no database configuration")
        return
    
    tester = DatabaseIntegrationTester(spark)
    
    try:
        with tester.database_connection(db_config) as db:
            # Test 1: Write large dataset to database
            large_dataset = spark.range(100000).select(
                col("id").alias("customer_id"),
                (rand() * 1000).cast("decimal(10,2)").alias("amount"),
                concat(lit("customer_"), col("id")).alias("customer_name"),
                current_timestamp().alias("created_at")
            )
            
            table_name = "integration_test_customers"
            jdbc_url = f"jdbc:postgresql://{db['host']}:{db.get('port', 5432)}/{db['database']}"
            
            # Write with partitioning for performance
            large_dataset.write.format("jdbc") \
                         .option("url", jdbc_url) \
                         .option("dbtable", table_name) \
                         .option("user", db["user"]) \
                         .option("password", db["password"]) \
                         .option("driver", "org.postgresql.Driver") \
                         .option("numPartitions", "4") \
                         .mode("overwrite") \
                         .save()
            
            print(f"✓ Successfully wrote {large_dataset.count():,} records to database")
            
            # Test 2: Read back with query pushdown
            read_df = spark.read.format("jdbc") \
                           .option("url", jdbc_url) \
                           .option("dbtable", f"(SELECT * FROM {table_name} WHERE amount > 500) as filtered_data") \
                           .option("user", db["user"]) \
                           .option("password", db["password"]) \
                           .option("driver", "org.postgresql.Driver") \
                           .load()
            
            filtered_count = read_df.count()
            total_count = large_dataset.count()
            
            print(f"✓ Successfully read {filtered_count:,} filtered records (from {total_count:,} total)")
            
            # Test 3: Incremental loading simulation
            # Add new data to existing table
            incremental_data = spark.range(100000, 110000).select(
                col("id").alias("customer_id"),
                (rand() * 1000).cast("decimal(10,2)").alias("amount"),
                concat(lit("customer_"), col("id")).alias("customer_name"),
                current_timestamp().alias("created_at")
            )
            
            incremental_data.write.format("jdbc") \
                            .option("url", jdbc_url) \
                            .option("dbtable", table_name) \
                            .option("user", db["user"]) \
                            .option("password", db["password"]) \
                            .option("driver", "org.postgresql.Driver") \
                            .mode("append") \
                            .save()
            
            # Verify incremental load
            final_df = spark.read.format("jdbc") \
                            .option("url", jdbc_url) \
                            .option("dbtable", table_name) \
                            .option("user", db["user"]) \
                            .option("password", db["password"]) \
                            .option("driver", "org.postgresql.Driver") \
                            .load()
            
            final_count = final_df.count()
            expected_count = total_count + incremental_data.count()
            
            assert final_count == expected_count, \
                f"Incremental load failed: expected {expected_count}, got {final_count}"
            
            print(f"✓ Incremental loading successful: {final_count:,} total records")
            
            return {
                "initial_count": total_count,
                "incremental_count": incremental_data.count(),
                "final_count": final_count,
                "filtered_count": filtered_count
            }
            
    except Exception as e:
        print(f"✗ Database integration test failed: {e}")
        raise

def test_file_system_integration(spark):
    """Test file system integration across different storage systems
    
    File System Integration Points:
    1. Local filesystem vs distributed storage (HDFS, S3, ADLS)
    2. File format compatibility (Parquet, Delta, CSV, JSON)
    3. Partitioning and compression strategies
    4. Error handling for missing files and permission issues
    """
    
    import tempfile
    from pathlib import Path
    
    with tempfile.TemporaryDirectory() as temp_dir:
        base_path = Path(temp_dir)
        
        # Create test data with various formats
        test_data = spark.range(10000).select(
            col("id"),
            (col("id") % 100).alias("partition_key"),
            (rand() * 1000).alias("value"),
            current_date().alias("date_partition")
        )
        
        # Test 1: Parquet with partitioning
        parquet_path = base_path / "parquet_data"
        test_data.write.mode("overwrite") \
                .partitionBy("partition_key") \
                .option("compression", "snappy") \
                .parquet(str(parquet_path))
        
        # Verify partitioned read
        parquet_df = spark.read.parquet(str(parquet_path))
        assert parquet_df.count() == test_data.count(), "Parquet read/write count mismatch"
        
        # Test partition pruning
        filtered_df = parquet_df.filter(col("partition_key") < 10)
        assert filtered_df.count() < test_data.count(), "Partition pruning should reduce record count"
        
        print(f"✓ Parquet integration: {parquet_df.count():,} records with partitioning")
        
        # Test 2: Delta Lake integration (if available)
        try:
            delta_path = base_path / "delta_data"
            test_data.write.format("delta") \
                     .mode("overwrite") \
                     .save(str(delta_path))
            
            delta_df = spark.read.format("delta").load(str(delta_path))
            assert delta_df.count() == test_data.count(), "Delta read/write count mismatch"
            
            print(f"✓ Delta Lake integration: {delta_df.count():,} records")
            
        except Exception as e:
            print(f"⚠ Delta Lake not available: {e}")
        
        # Test 3: CSV with schema inference
        csv_path = base_path / "csv_data"
        test_data.coalesce(1).write.mode("overwrite") \
                .option("header", "true") \
                .csv(str(csv_path))
        
        csv_df = spark.read.option("header", "true") \
                      .option("inferSchema", "true") \
                      .csv(str(csv_path))
        
        assert csv_df.count() == test_data.count(), "CSV read/write count mismatch"
        print(f"✓ CSV integration: {csv_df.count():,} records with schema inference")
        
        # Test 4: Error handling for missing files
        try:
            missing_df = spark.read.parquet(str(base_path / "nonexistent_path"))
            missing_df.count()  # Should trigger error
            assert False, "Should have failed reading missing path"
        except Exception:
            print("✓ Missing file error handling works correctly")
        
        return {
            "parquet_count": parquet_df.count(),
            "csv_count": csv_df.count(),
            "filtered_count": filtered_df.count()
        }

### Message Queue Integration

**Message queue integration** is critical for real-time data pipelines. Testing Kafka, Kinesis, or other streaming systems requires validating message production, consumption, error handling, and exactly-once processing semantics.

```python
def test_kafka_integration(spark):
    """Test Kafka integration for streaming data
    
    Kafka Integration Testing:
    1. Producer connectivity and message publishing
    2. Consumer connectivity and message consumption  
    3. Schema registry integration
    4. Error handling and retry logic
    5. Offset management and exactly-once processing
    
    Note: This test requires Kafka to be running
    """
    
    kafka_config = {
        "bootstrap_servers": os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092"),
        "topic": "test_integration_topic",
        "schema_registry_url": os.getenv("SCHEMA_REGISTRY_URL", "http://localhost:8081")
    }
    
    # Skip if Kafka not available
    if not kafka_config["bootstrap_servers"]:
        print("⚠ Skipping Kafka integration test - no Kafka configuration")
        return
    
    try:
        # Test 1: Write streaming data to Kafka
        test_stream_data = spark.range(1000).select(
            col("id").cast("string").alias("key"),
            to_json(struct(
                col("id"),
                current_timestamp().alias("timestamp"),
                (rand() * 100).alias("value")
            )).alias("value")
        )
        
        # Write to Kafka (in real streaming, this would be a streaming write)
        kafka_write_query = test_stream_data.write \
            .format("kafka") \
            .option("kafka.bootstrap.servers", kafka_config["bootstrap_servers"]) \
            .option("topic", kafka_config["topic"]) \
            .mode("append") \
            .save()
        
        print(f"✓ Successfully wrote {test_stream_data.count()} messages to Kafka topic: {kafka_config['topic']}")
        
        # Test 2: Read from Kafka
        kafka_df = spark.read \
            .format("kafka") \
            .option("kafka.bootstrap.servers", kafka_config["bootstrap_servers"]) \
            .option("subscribe", kafka_config["topic"]) \
            .option("startingOffsets", "earliest") \
            .load()
        
        # Parse the JSON messages
        parsed_df = kafka_df.select(
            col("key").cast("string"),
            from_json(col("value").cast("string"), 
                     schema=StructType([
                         StructField("id", LongType()),
                         StructField("timestamp", TimestampType()),
                         StructField("value", DoubleType())
                     ])).alias("data")
        ).select("key", "data.*")
        
        kafka_count = parsed_df.count()
        print(f"✓ Successfully read {kafka_count} messages from Kafka")
        
        # Verify data integrity
        assert kafka_
````

## Testing Strategy Summary

**Comprehensive PySpark testing** requires a multi-layered approach that addresses the unique challenges of distributed data processing. Unlike traditional application testing, PySpark testing must account for non-deterministic execution, resource dependencies, and complex data quality scenarios.

### Core Testing Layers

**1. Unit Testing Foundation (7.1-7.2)**

- **SparkSession Management**: Session-scoped fixtures for performance
- **Data Transformation Testing**: Isolated testing of business logic
- **Schema Validation**: Early detection of data structure changes
- **Custom Function Testing**: UDF performance and correctness validation

**2. Data Quality Assurance (7.3)**

- **Completeness Testing**: Ensuring data presence and integrity
- **Business Rule Validation**: Domain-specific constraint enforcement
- **Cross-table Validation**: Referential integrity and consistency checks
- **Temporal Quality Monitoring**: Data freshness and trend analysis

**3. Performance & Scalability (7.4)**

- **Benchmark Testing**: Establishing performance baselines
- **Memory Usage Monitoring**: Preventing OOM and leak detection
- **Scalability Validation**: Understanding performance scaling patterns
- **Resource Utilization**: Optimizing cluster resource efficiency

**4. Integration Verification (7.5)**

- **End-to-End Pipeline Testing**: Complete data flow validation
- **External System Integration**: Database, API, and message queue testing
- **Streaming Application Testing**: State management and exactly-once processing
- **Cross-Environment Testing**: Development to production consistency

**5. Framework & Tooling (7.6)**

- **pytest Integration**: Efficient test execution and organization
- **Spark Testing Base**: Specialized DataFrame comparison utilities
- **Great Expectations**: Declarative data quality expectations
- **Custom Testing Utilities**: Domain-specific assertion methods

---

## Key Testing Principles

### Distributed Systems Considerations

**Non-Deterministic Execution**: Tests must produce consistent results regardless of partition count, execution order, or cluster configuration. Focus on final outcomes rather than intermediate execution details.

**Resource Independence**: Design tests that work within reasonable resource constraints without depending on specific memory or CPU allocations.

**Data Locality**: Understand that data distribution across nodes affects performance and test execution patterns.

### Performance-Aware Testing

**Test Data Sizing**: Use appropriately sized datasets for different test categories:

- Unit tests: 1K-10K records for fast feedback
- Integration tests: 100K-1M records for realistic scenarios
- Performance tests: 10M+ records for load validation

**Caching Strategy**: Cache test data appropriately to eliminate upstream computation noise while avoiding memory pressure.

**Resource Monitoring**: Track CPU, memory, and network usage to detect performance regressions and resource leaks.

### Data Quality Focus

**Schema Evolution**: Test backward compatibility and handle schema changes gracefully.

**Edge Case Coverage**: Include null values, empty partitions, data skew, and boundary conditions in test scenarios.

**Business Logic Validation**: Encode domain knowledge as executable tests to catch semantic errors.

---

## Production Readiness Checklist

### Before Deployment

- [ ] **Unit Test Coverage**: >90% coverage for core business logic
- [ ] **Integration Tests**: All external dependencies validated
- [ ] **Performance Baselines**: Established with acceptable thresholds
- [ ] **Data Quality Rules**: Comprehensive validation suite implemented
- [ ] **Error Handling**: Graceful failure modes tested
- [ ] **Resource Limits**: Memory and CPU boundaries validated

### Post-Deployment Monitoring

- [ ] **Data Drift Detection**: Automated monitoring for schema and statistical changes
- [ ] **Quality Metrics**: Continuous data quality measurement
- [ ] **Performance Monitoring**: Throughput and latency tracking
- [ ] **Alert Configuration**: Proactive notification of issues
- [ ] **Dashboard Validation**: Accurate representation of system state

---

## Testing Anti-Patterns to Avoid

### Common Pitfalls

**Over-Reliance on collect()**: Using `.collect()` in tests with large datasets causes driver memory issues. Use `.count()`, sampling, or aggregations instead.

**Non-Deterministic Test Data**: Using random data without seeds leads to flaky tests. Always use deterministic data generation for reproducible results.

**Ignoring Resource Cleanup**: Not unpersisting cached DataFrames or cleaning up temp files causes resource leaks in test suites.

**Testing Implementation Details**: Testing Spark internals instead of business logic makes tests brittle to framework changes.

**Inadequate Error Testing**: Only testing happy paths misses critical failure scenarios that occur in production.

### Performance Anti-Patterns

**Creating New SparkSession Per Test**: Extremely expensive setup/teardown. Use session-scoped fixtures instead.

**Large Shuffle Operations in Tests**: Avoid operations that require significant data movement unless specifically testing performance.

**Synchronous Streaming Tests**: Not using manual clock advancement makes streaming tests slow and unreliable.

---

## Future Considerations

### Emerging Patterns

**Property-Based Testing**: Using libraries like Hypothesis to generate diverse test scenarios automatically.

**Mutation Testing**: Validating test suite quality by introducing code changes and ensuring tests fail appropriately.

**Chaos Engineering**: Deliberately introducing failures to test system resilience and recovery mechanisms.

**ML Pipeline Testing**: Specialized patterns for testing machine learning workflows with PySpark MLlib.

### Technology Evolution

**Delta Lake Integration**: Enhanced testing patterns for ACID transactions and time travel queries.

**Kubernetes Deployment**: Container-based testing strategies for cloud-native PySpark applications.

**Structured Streaming V2**: Advanced testing patterns for continuous processing and exactly-once semantics.

---

## Recommended Tools & Libraries

### Essential Testing Stack

```python
# Core testing framework
pytest>=7.0.0
pytest-xdist>=2.5.0  # Parallel test execution
pytest-cov>=4.0.0    # Coverage reporting
pytest-benchmark>=4.0.0  # Performance testing

# PySpark testing utilities
pyspark-test>=0.2.0   # Spark Testing Base
great-expectations>=0.15.0  # Data quality

# Development utilities
black>=22.0.0         # Code formatting
mypy>=0.991          # Type checking
pre-commit>=2.20.0   # Git hooks
```

### Data Quality Tools

```python
# Data profiling and validation
pandas-profiling>=3.2.0
deequ>=1.0.0         # Amazon's data quality library
evidently>=0.2.0     # ML model and data drift detection

# Monitoring and observability
prometheus-client>=0.14.0
grafana-client>=3.5.0
```

---

## Conclusion

**Effective PySpark testing** is essential for reliable data engineering pipelines. The distributed nature of Spark applications requires specialized testing approaches that go beyond traditional unit testing. By implementing comprehensive testing strategies across all layers - from unit tests to production monitoring - teams can build confidence in their data pipelines and catch issues before they impact business operations.

**Key success factors** include:

- Understanding distributed systems testing challenges
- Implementing performance-aware testing patterns
- Focusing on data quality throughout the pipeline
- Using appropriate tools and frameworks
- Establishing continuous monitoring and validation

**Investment in testing infrastructure** pays dividends through reduced production incidents, faster development cycles, and increased confidence in data quality. The patterns and practices outlined in this guide provide a foundation for building robust, reliable PySpark applications that scale with your organization's data needs.

---

## Tags

#pyspark #testing #data-quality #performance #ci-cd #monitoring #integration #spark #big-data #data-engineering #best-practices #distributed-systems #etl #data-pipelines #quality-assurance

## Related Notes

- [[PySpark Core Concepts]]
- [[PySpark Performance Optimization]]
- [[Data Quality Frameworks]]
- [[Production Data Engineering Patterns]]
- [[Spark SQL Testing Strategies]]
- [[Distributed Systems Testing]]
- [[DataOps and Pipeline Reliability]]