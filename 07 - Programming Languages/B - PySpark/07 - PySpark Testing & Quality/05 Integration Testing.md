
#pyspark #testing #integration #end-to-end #external-systems #databases #apis #streaming #kafka #pipelines #cross-environment #system-testing

## 7.5 Integration Testing

**Integration testing** in PySpark validates that your data pipeline components work together correctly in realistic environments. Unlike unit tests that focus on individual transformations, integration tests verify end-to-end data flows, external system connections, and cross-service interactions that mirror production scenarios.

### End-to-End Pipeline Testing

**End-to-end pipeline testing** ensures that data flows correctly from source systems through all transformation stages to final destinations. This is critical because many issues only surface when components interact - schema mismatches, data format incompatibilities, and timing dependencies that work in isolation but fail in integrated environments.

**Full Data Flow Validation**

````python
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