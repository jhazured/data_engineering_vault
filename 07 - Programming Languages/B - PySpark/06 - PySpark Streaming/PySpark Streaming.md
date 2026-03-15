
#pyspark #streaming #structured-streaming #real-time #kafka #watermarks #windowing #triggers

## Overview

PySpark Structured Streaming provides a scalable and fault-tolerant stream processing engine built on the Spark SQL engine. It treats streaming data as an unbounded table that is continuously appended, allowing you to express streaming computations the same way you would express a batch computation on static data.

---

## Structured Streaming Fundamentals

### Core Concepts

**Structured Streaming** treats streaming data as an unbounded table where new data is continuously appended:

- **Input Table**: Unbounded table representing streaming data
- **Query**: Defines transformations on the input table
- **Result Table**: Output of the query, updated incrementally
- **Output**: What gets written to external storage

### Key Advantages

- **Unified API**: Same DataFrame/SQL API for batch and streaming
- **Fault Tolerance**: Automatic recovery with exactly-once semantics
- **Late Data Handling**: Built-in support for late-arriving data
- **Scalability**: Leverages Spark's distributed computing
- **Integration**: Works with existing Spark ecosystem

### Basic Streaming Setup

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Create Spark session with streaming support
spark = SparkSession.builder \
    .appName("StructuredStreamingApp") \
    .config("spark.sql.adaptive.enabled", "false") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "false") \
    .getOrCreate()

# Define schema for streaming data (recommended for production)
schema = StructType([
    StructField("timestamp", TimestampType(), True),
    StructField("user_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("value", DoubleType(), True),
    StructField("metadata", StringType(), True)
])

# Read from a streaming source
streaming_df = spark \
    .readStream \
    .format("json") \
    .schema(schema) \
    .option("path", "/path/to/streaming/data") \
    .load()

# Basic transformation
processed_df = streaming_df \
    .filter(col("event_type") == "purchase") \
    .select("user_id", "timestamp", "value")

# Write to sink
query = processed_df.writeStream \
    .outputMode("append") \
    .format("console") \
    .trigger(processingTime="10 seconds") \
    .start()

# Wait for termination
query.awaitTermination()
```

---

## Streaming Sources

### File Sources

#### JSON Files

```python
# Read JSON files from a directory
json_stream = spark \
    .readStream \
    .format("json") \
    .schema(schema) \
    .option("path", "/data/streaming/json/") \
    .option("maxFilesPerTrigger", 1) \
    .load()

# With additional options
json_stream_advanced = spark \
    .readStream \
    .format("json") \
    .schema(schema) \
    .option("path", "/data/streaming/json/") \
    .option("maxFilesPerTrigger", 5) \
    .option("latestFirst", "true") \
    .option("fileNameOnly", "true") \
    .load()
```

#### CSV Files

```python
# Read CSV files
csv_stream = spark \
    .readStream \
    .format("csv") \
    .schema(schema) \
    .option("path", "/data/streaming/csv/") \
    .option("header", "true") \
    .option("maxFilesPerTrigger", 1) \
    .load()
```

#### Parquet Files

```python
# Read Parquet files
parquet_stream = spark \
    .readStream \
    .format("parquet") \
    .schema(schema) \
    .option("path", "/data/streaming/parquet/") \
    .load()
```

### Kafka Integration

#### Basic Kafka Source

```python
# Read from Kafka topic
kafka_df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "user_events") \
    .option("startingOffsets", "latest") \
    .load()

# Kafka DataFrame schema
# root
#  |-- key: binary (nullable = true)
#  |-- value: binary (nullable = true)
#  |-- topic: string (nullable = true)
#  |-- partition: integer (nullable = true)
#  |-- offset: long (nullable = true)
#  |-- timestamp: timestamp (nullable = true)
#  |-- timestampType: integer (nullable = true)

# Parse JSON from Kafka value
parsed_df = kafka_df.select(
    col("timestamp").alias("kafka_timestamp"),
    col("partition"),
    col("offset"),
    from_json(col("value").cast("string"), schema).alias("data")
).select("kafka_timestamp", "partition", "offset", "data.*")
```

#### Advanced Kafka Configuration

```python
# Multiple topics with pattern
kafka_multi_topics = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092") \
    .option("subscribePattern", "events_.*") \
    .option("startingOffsets", "earliest") \
    .option("endingOffsets", "latest") \
    .option("failOnDataLoss", "false") \
    .option("maxOffsetsPerTrigger", 10000) \
    .load()

# Kafka with security configuration
kafka_secure = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "secure-broker:9093") \
    .option("subscribe", "secure_topic") \
    .option("kafka.security.protocol", "SASL_SSL") \
    .option("kafka.sasl.mechanism", "PLAIN") \
    .option("kafka.sasl.jaas.config", 
            "org.apache.kafka.common.security.plain.PlainLoginModule required "
            "username='user' password='password';") \
    .load()

# Specific partition and offset range
kafka_specific = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("assign", '{"topic1":[0,1,2]}') \
    .option("startingOffsets", '{"topic1":{"0":23,"1":-2,"2":-1}}') \
    .load()
```

### Socket Source (for testing)

```python
# Read from socket (testing only - not fault-tolerant)
socket_df = spark \
    .readStream \
    .format("socket") \
    .option("host", "localhost") \
    .option("port", 9999) \
    .load()

# Process socket data
socket_processed = socket_df \
    .select(split(col("value"), " ").alias("words")) \
    .select(explode(col("words")).alias("word")) \
    .filter(col("word") != "")
```

### Rate Source (for testing)

```python
# Generate test data at specified rate
rate_stream = spark \
    .readStream \
    .format("rate") \
    .option("rowsPerSecond", 100) \
    .option("rampUpTime", "10s") \
    .option("numPartitions", 4) \
    .load()

# Rate source schema:
# root
#  |-- timestamp: timestamp (nullable = true)
#  |-- value: long (nullable = true)

# Generate realistic test data
test_events = rate_stream \
    .withColumn("user_id", concat(lit("user_"), (col("value") % 1000).cast("string"))) \
    .withColumn("event_type", 
        when(col("value") % 4 == 0, "login")
        .when(col("value") % 4 == 1, "view")
        .when(col("value") % 4 == 2, "click")
        .otherwise("purchase")) \
    .withColumn("amount", 
        when(col("event_type") == "purchase", rand() * 100)
        .otherwise(lit(0.0)))
```

---

## Windowing Operations

### Time-based Windows

#### Tumbling Windows

```python
# Non-overlapping windows
tumbling_window_df = streaming_df \
    .groupBy(
        window(col("timestamp"), "10 minutes"),
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        sum("value").alias("total_value"),
        avg("value").alias("avg_value")
    )

# Custom tumbling window
custom_tumbling = streaming_df \
    .groupBy(
        window(col("timestamp"), "5 minutes"),
        col("user_id")
    ) \
    .agg(
        countDistinct("event_type").alias("unique_events"),
        collect_list("event_type").alias("event_sequence")
    )
```

#### Sliding Windows

```python
# Overlapping windows
sliding_window_df = streaming_df \
    .groupBy(
        window(col("timestamp"), "10 minutes", "5 minutes"),  # window duration, slide interval
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        sum("value").alias("total_value")
    )

# Multiple sliding windows
multi_sliding = streaming_df \
    .groupBy(
        window(col("timestamp"), "1 hour", "15 minutes")
    ) \
    .agg(
        count("*").alias("hourly_count"),
        avg("value").alias("hourly_avg")
    )
```

#### Session Windows (using custom logic)

```python
# Session windows with timeout-based grouping
def create_session_windows(df, session_timeout="30 minutes"):
    """Create session windows based on inactivity timeout"""
    
    from pyspark.sql.window import Window
    
    # Sort by user and timestamp
    window_spec = Window.partitionBy("user_id").orderBy("timestamp")
    
    # Calculate time difference from previous event
    df_with_gaps = df.withColumn(
        "time_diff",
        col("timestamp").cast("long") - lag("timestamp").over(window_spec).cast("long")
    )
    
    # Mark session boundaries (gaps > timeout)
    timeout_seconds = 30 * 60  # 30 minutes in seconds
    df_with_sessions = df_with_gaps.withColumn(
        "new_session",
        when(col("time_diff") > timeout_seconds, 1).otherwise(0)
    )
    
    # Create session ID using cumulative sum
    df_with_session_id = df_with_sessions.withColumn(
        "session_id",
        concat(
            col("user_id"),
            lit("_"),
            sum("new_session").over(
                Window.partitionBy("user_id")
                      .orderBy("timestamp")
                      .rowsBetween(Window.unboundedPreceding, Window.currentRow)
            ).cast("string")
        )
    )
    
    return df_with_session_id

# Apply session windowing
session_df = create_session_windows(streaming_df)

# Aggregate by session
session_aggregated = session_df \
    .groupBy("session_id", "user_id") \
    .agg(
        min("timestamp").alias("session_start"),
        max("timestamp").alias("session_end"),
        count("*").alias("events_in_session"),
        sum("value").alias("session_value")
    ) \
    .withColumn("session_duration", 
                col("session_end").cast("long") - col("session_start").cast("long"))
```

---

## Watermarks and Late Data Handling

### Understanding Watermarks

Watermarks define how long to wait for late-arriving data before considering a time window complete.

```python
# Basic watermark configuration
watermarked_df = streaming_df \
    .withWatermark("timestamp", "10 minutes")

# Windowed aggregation with watermark
windowed_with_watermark = watermarked_df \
    .groupBy(
        window(col("timestamp"), "5 minutes"),
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        sum("value").alias("total_value")
    )
```

### Advanced Watermark Strategies

```python
# Conservative watermark (wait longer for late data)
conservative_watermark = streaming_df \
    .withWatermark("timestamp", "1 hour") \
    .groupBy(
        window(col("timestamp"), "15 minutes"),
        col("user_id")
    ) \
    .agg(count("*").alias("user_events"))

# Aggressive watermark (faster processing, may miss very late data)
aggressive_watermark = streaming_df \
    .withWatermark("timestamp", "1 minute") \
    .groupBy(
        window(col("timestamp"), "5 minutes")
    ) \
    .agg(count("*").alias("total_events"))

# Multiple watermarks for different use cases
def create_tiered_watermarks(df):
    """Create different aggregations with appropriate watermarks"""
    
    # Real-time alerts (aggressive watermark)
    alerts = df \
        .withWatermark("timestamp", "30 seconds") \
        .filter(col("value") > 1000) \
        .groupBy(window(col("timestamp"), "1 minute")) \
        .agg(count("*").alias("high_value_events"))
    
    # Business metrics (balanced watermark)
    metrics = df \
        .withWatermark("timestamp", "5 minutes") \
        .groupBy(
            window(col("timestamp"), "15 minutes"),
            col("event_type")
        ) \
        .agg(
            count("*").alias("count"),
            avg("value").alias("avg_value")
        )
    
    # Historical analysis (conservative watermark)
    historical = df \
        .withWatermark("timestamp", "1 hour") \
        .groupBy(
            window(col("timestamp"), "1 hour"),
            col("user_id")
        ) \
        .agg(
            sum("value").alias("hourly_total"),
            countDistinct("event_type").alias("unique_events")
        )
    
    return alerts, metrics, historical

# Apply tiered watermarks
alert_stream, metrics_stream, historical_stream = create_tiered_watermarks(streaming_df)
```

### Handling Very Late Data

```python
# Separate processing for late data
def handle_late_data(df, watermark_threshold="10 minutes"):
    """Handle late-arriving data separately"""
    
    # Current time for comparison
    current_time = current_timestamp()
    
    # Separate on-time and late data
    on_time_data = df.filter(
        col("timestamp") >= (current_time - expr(f"INTERVAL {watermark_threshold}"))
    )
    
    late_data = df.filter(
        col("timestamp") < (current_time - expr(f"INTERVAL {watermark_threshold}"))
    )
    
    # Process on-time data normally
    on_time_aggregated = on_time_data \
        .withWatermark("timestamp", watermark_threshold) \
        .groupBy(window(col("timestamp"), "5 minutes")) \
        .agg(count("*").alias("on_time_count"))
    
    # Store late data for batch reprocessing
    late_data_stored = late_data \
        .withColumn("processing_time", current_timestamp()) \
        .withColumn("lateness", 
                   (current_timestamp().cast("long") - col("timestamp").cast("long")))
    
    return on_time_aggregated, late_data_stored

# Apply late data handling
on_time_stream, late_data_stream = handle_late_data(streaming_df)

# Write late data to separate sink for analysis
late_data_query = late_data_stream.writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("path", "/data/late_arrivals") \
    .option("checkpointLocation", "/checkpoints/late_data") \
    .trigger(processingTime="1 minute") \
    .start()
```

---

## Output Modes and Triggers

### Output Modes

#### Append Mode

```python
# Append mode - only new rows (default for most operations)
append_query = windowed_aggregated \
    .writeStream \
    .outputMode("append") \
    .format("console") \
    .trigger(processingTime="30 seconds") \
    .start()

# Append mode with file sink
append_to_files = processed_df \
    .writeStream \
    .outputMode("append") \
    .format("parquet") \
    .option("path", "/data/streaming_output") \
    .option("checkpointLocation", "/checkpoints/append_mode") \
    .partitionBy("date") \
    .trigger(processingTime="1 minute") \
    .start()
```

#### Update Mode

```python
# Update mode - only updated rows (for aggregations)
update_query = windowed_aggregated \
    .writeStream \
    .outputMode("update") \
    .format("delta") \
    .option("path", "/data/streaming_updates") \
    .option("checkpointLocation", "/checkpoints/update_mode") \
    .trigger(processingTime="30 seconds") \
    .start()

# Update mode with memory sink for debugging
update_memory = windowed_aggregated \
    .writeStream \
    .outputMode("update") \
    .format("memory") \
    .queryName("windowed_updates") \
    .trigger(processingTime="10 seconds") \
    .start()

# Query the memory sink
spark.sql("SELECT * FROM windowed_updates").show()
```

#### Complete Mode

```python
# Complete mode - entire result table (memory intensive)
complete_query = streaming_df \
    .groupBy("event_type") \
    .count() \
    .writeStream \
    .outputMode("complete") \
    .format("console") \
    .trigger(processingTime="1 minute") \
    .start()

# Complete mode with limit for large results
limited_complete = streaming_df \
    .groupBy("user_id") \
    .agg(count("*").alias("event_count")) \
    .orderBy(desc("event_count")) \
    .limit(100) \
    .writeStream \
    .outputMode("complete") \
    .format("console") \
    .trigger(processingTime="30 seconds") \
    .start()
```

### Trigger Types

#### Processing Time Triggers

```python
# Fixed interval processing
fixed_interval = processed_df \
    .writeStream \
    .trigger(processingTime="30 seconds") \
    .format("console") \
    .start()

# Longer intervals for batch-like processing
batch_like = aggregated_df \
    .writeStream \
    .trigger(processingTime="5 minutes") \
    .format("delta") \
    .option("path", "/data/batch_streaming") \
    .start()
```

#### Once Trigger

```python
# Process available data once and stop
once_trigger = processed_df \
    .writeStream \
    .trigger(once=True) \
    .format("parquet") \
    .option("path", "/data/one_time_processing") \
    .start()

# Wait for completion
once_trigger.awaitTermination()
```

#### Continuous Processing (Experimental)

```python
# Ultra-low latency processing (experimental)
continuous_query = processed_df \
    .writeStream \
    .trigger(continuous="1 second") \
    .format("console") \
    .start()

# Note: Continuous processing has limitations:
# - Limited operations supported
# - No fault-tolerance guarantees
# - Experimental feature
```

#### Available Now Trigger

```python
# Process all available data and stop (Spark 3.3+)
available_now = processed_df \
    .writeStream \
    .trigger(availableNow=True) \
    .format("delta") \
    .option("path", "/data/available_now_processing") \
    .start()

available_now.awaitTermination()
```

---

## Streaming Sinks

### File Sinks

#### Parquet Sink

```python
# Write to Parquet files
parquet_sink = processed_df \
    .writeStream \
    .outputMode("append") \
    .format("parquet") \
    .option("path", "/data/streaming/parquet_output") \
    .option("checkpointLocation", "/checkpoints/parquet_sink") \
    .partitionBy("year", "month", "day") \
    .trigger(processingTime="2 minutes") \
    .start()
```

#### Delta Lake Sink

```python
# Write to Delta Lake
delta_sink = processed_df \
    .writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("path", "/data/streaming/delta_output") \
    .option("checkpointLocation", "/checkpoints/delta_sink") \
    .partitionBy("event_type") \
    .trigger(processingTime="1 minute") \
    .start()

# Delta sink with optimization
optimized_delta = aggregated_df \
    .writeStream \
    .outputMode("update") \
    .format("delta") \
    .option("path", "/data/streaming/delta_aggregated") \
    .option("checkpointLocation", "/checkpoints/delta_agg") \
    .option("mergeSchema", "true") \
    .trigger(processingTime="30 seconds") \
    .start()
```

### Kafka Sink

```python
# Write back to Kafka
kafka_sink_df = processed_df \
    .select(
        col("user_id").cast("string").alias("key"),
        to_json(struct("*")).alias("value")
    )

kafka_sink = kafka_sink_df \
    .writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "processed_events") \
    .option("checkpointLocation", "/checkpoints/kafka_sink") \
    .trigger(processingTime="10 seconds") \
    .start()

# Kafka sink with partitioning
partitioned_kafka_sink = processed_df \
    .select(
        col("event_type").alias("key"),
        to_json(struct("*")).alias("value"),
        (hash(col("user_id")) % 10).alias("partition")
    ) \
    .writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "partitioned_events") \
    .trigger(processingTime="5 seconds") \
    .start()
```

### Database Sinks

#### JDBC Sink (Batch function)

```python
def write_to_database(df, epoch_id):
    """Custom sink function to write to database"""
    
    df.write \
      .format("jdbc") \
      .option("url", "jdbc:postgresql://localhost:5432/streaming_db") \
      .option("dbtable", "streaming_results") \
      .option("user", "username") \
      .option("password", "password") \
      .mode("append") \
      .save()

# Use foreach batch for database writes
database_sink = processed_df \
    .writeStream \
    .outputMode("append") \
    .foreachBatch(write_to_database) \
    .trigger(processingTime="1 minute") \
    .start()
```

#### Custom Sink with ForeachWriter

```python
class DatabaseWriter:
    """Custom writer for streaming to database"""
    
    def __init__(self, connection_properties):
        self.connection_properties = connection_properties
        self.connection = None
    
    def open(self, partition_id, epoch_id):
        """Open connection for this partition and epoch"""
        try:
            import psycopg2
            self.connection = psycopg2.connect(**self.connection_properties)
            return True
        except Exception as e:
            print(f"Failed to open connection: {e}")
            return False
    
    def process(self, row):
        """Process each row"""
        try:
            cursor = self.connection.cursor()
            cursor.execute(
                "INSERT INTO streaming_events (user_id, event_type, value, timestamp) VALUES (%s, %s, %s, %s)",
                (row.user_id, row.event_type, row.value, row.timestamp)
            )
            cursor.close()
        except Exception as e:
            print(f"Failed to process row: {e}")
    
    def close(self, error):
        """Close connection"""
        if self.connection:
            if error:
                self.connection.rollback()
            else:
                self.connection.commit()
            self.connection.close()

# Use custom writer
db_properties = {
    "host": "localhost",
    "database": "streaming_db",
    "user": "username",
    "password": "password"
}

custom_sink = processed_df \
    .writeStream \
    .foreach(DatabaseWriter(db_properties)) \
    .trigger(processingTime="30 seconds") \
    .start()
```

---

## Stream-Stream and Stream-Static Joins

### Stream-Static Joins

```python
# Load static data
static_user_data = spark.read \
    .format("delta") \
    .load("/data/static/user_profiles")

# Join streaming data with static data
enriched_stream = streaming_df \
    .join(static_user_data, "user_id", "left") \
    .select(
        "user_id", "timestamp", "event_type", "value",
        "user_name", "user_segment", "registration_date"
    )

# Stream-static join with broadcast
from pyspark.sql.functions import broadcast

broadcast_enriched = streaming_df \
    .join(broadcast(static_user_data), "user_id", "left")
```

### Stream-Stream Joins

#### Inner Join with Watermarks

```python
# Two streaming DataFrames
impressions = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "impressions") \
    .load() \
    .select(from_json(col("value").cast("string"), impression_schema).alias("data")) \
    .select("data.*") \
    .withWatermark("timestamp", "10 minutes")

clicks = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "clicks") \
    .load() \
    .select(from_json(col("value").cast("string"), click_schema).alias("data")) \
    .select("data.*") \
    .withWatermark("timestamp", "5 minutes")

# Stream-stream inner join
impression_clicks = impressions \
    .join(
        clicks,
        expr("""
            impressions.ad_id = clicks.ad_id AND
            clicks.timestamp >= impressions.timestamp AND
            clicks.timestamp <= impressions.timestamp + interval 1 hour
        """)
    )
```

#### Left Join with Time Bounds

```python
# Left join to capture impressions without clicks
impression_attribution = impressions \
    .join(
        clicks,
        expr("""
            impressions.ad_id = clicks.ad_id AND
            clicks.timestamp >= impressions.timestamp AND
            clicks.timestamp <= impressions.timestamp + interval 30 minutes
        """),
        "left"
    ) \
    .select(
        impressions.ad_id,
        impressions.timestamp.alias("impression_time"),
        clicks.timestamp.alias("click_time"),
        when(clicks.timestamp.isNotNull(), 1).otherwise(0).alias("converted")
    )
```

### Complex Stream Joins with Multiple Conditions

```python
# Complex join with multiple conditions and time windows
attribution_stream = impressions.alias("imp") \
    .join(
        clicks.alias("clk"),
        expr("""
            imp.ad_id = clk.ad_id AND
            imp.user_id = clk.user_id AND
            clk.timestamp >= imp.timestamp AND
            clk.timestamp <= imp.timestamp + interval 2 hours AND
            imp.campaign_id = clk.campaign_id
        """),
        "left"
    ) \
    .select(
        col("imp.ad_id"),
        col("imp.user_id"),
        col("imp.campaign_id"),
        col("imp.timestamp").alias("impression_time"),
        col("clk.timestamp").alias("click_time"),
        (col("clk.timestamp").cast("long") - col("imp.timestamp").cast("long")).alias("time_to_click"),
        when(col("clk.timestamp").isNotNull(), "converted").otherwise("not_converted").alias("status")
    )
```

---

## Advanced Streaming Patterns

### Sessionization

```python
def sessionize_events(df, session_timeout_minutes=30):
    """Sessionize user events based on inactivity timeout"""
    
    from pyspark.sql.window import Window
    
    # Window spec for time-ordered events per user
    window_spec = Window.partitionBy("user_id").orderBy("timestamp")
    
    # Calculate time gaps between consecutive events
    df_with_gaps = df.withColumn(
        "time_gap_seconds",
        (col("timestamp").cast("long") - 
         lag("timestamp").over(window_spec).cast("long"))
    )
    
    # Mark session boundaries (first event or gap > timeout)
    timeout_seconds = session_timeout_minutes * 60
    df_with_boundaries = df_with_gaps.withColumn(
        "is_session_start",
        when(col("time_gap_seconds").isNull(), 1)  # First event
        .when(col("time_gap_seconds") > timeout_seconds, 1)  # Gap > timeout
        .otherwise(0)
    )
    
    # Create session IDs using cumulative sum of session starts
    df_sessionized = df_with_boundaries.withColumn(
        "session_id",
        concat(
            col("user_id"),
            lit("_"),
            sum("is_session_start").over(
                Window.partitionBy("user_id")
                      .orderBy("timestamp")
                      .rowsBetween(Window.unboundedPreceding, Window.currentRow)
            ).cast("string")
        )
    )
    
    return df_sessionized

# Apply sessionization
sessionized_events = sessionize_events(streaming_df, session_timeout_minutes=30)

# Aggregate by session
session_metrics = sessionized_events \
    .groupBy("session_id", "user_id") \
    .agg(
        min("timestamp").alias("session_start"),
        max("timestamp").alias("session_end"),
        count("*").alias("events_count"),
        countDistinct("event_type").alias("unique_event_types"),
        sum("value").alias("session_value"),
        collect_list("event_type").alias("event_sequence")
    ) \
    .withColumn("session_duration_minutes",
                (col("session_end").cast("long") - col("session_start").cast("long")) / 60)
```

### Real-time Anomaly Detection

```python
def detect_anomalies(df, window_duration="5 minutes", threshold_multiplier=3.0):
    """Real-time anomaly detection using statistical methods"""
    
    # Calculate rolling statistics
    windowed_stats = df \
        .withWatermark("timestamp", "2 minutes") \
        .groupBy(
            window(col("timestamp"), window_duration),
            col("user_id")
        ) \
        .agg(
            count("*").alias("event_count"),
            avg("value").alias("avg_value"),
            stddev("value").alias("stddev_value"),
            min("value").alias("min_value"),
            max("value").alias("max_value")
        )
    
    # Calculate global statistics for comparison
    global_stats = df \
        .withWatermark("timestamp", "10 minutes") \
        .groupBy(window(col("timestamp"), "1 hour")) \
        .agg(
            avg("value").alias("global_avg"),
            stddev("value").alias("global_stddev")
        )
    
    # Join window stats with global stats for anomaly detection
    anomaly_detection = windowed_stats \
        .join(global_stats, 
              windowed_stats.window.start >= global_stats.window.start) \
        .withColumn("z_score", 
                   (col("avg_value") - col("global_avg")) / col("global_stddev")) \
        .withColumn("is_anomaly",
                   abs(col("z_score")) > threshold_multiplier) \
        .withColumn("anomaly_type",
                   when(col("z_score") > threshold_multiplier, "high_value")
                   .when(col("z_score") < -threshold_multiplier, "low_value")
                   .otherwise("normal"))
    
    return anomaly_detection

# Apply anomaly detection
anomaly_stream = detect_anomalies(streaming_df)

# Filter and alert on anomalies
anomaly_alerts = anomaly_stream \
    .filter(col("is_anomaly") == True) \
    .select(
        "user_id", "window", "event_count", "avg_value", 
        "z_score", "anomaly_type", "global_avg"
    )

# Write anomalies to alert system
anomaly_query = anomaly_alerts \
    .writeStream \
    .outputMode("append") \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "anomaly_alerts") \
    .trigger(processingTime="30 seconds") \
    .start()
```

### Real-time Feature Engineering

```python
def create_realtime_features(df):
    """Create real-time features for ML models"""
    
    from pyspark.sql.window import Window
    
    # User-based window for historical features
    user_window = Window.partitionBy("user_id") \
                        .orderBy("timestamp") \
                        .rowsBetween(-10, -1)  # Last 10 events
    
    # Time-based window for recent activity
    time_window_1h = Window.partitionBy("user_id") \
                           .orderBy("timestamp") \
                           .rangeBetween(-3600, 0)  # Last hour
    
    features_df = df \
        .withColumn("hour_of_day", hour("timestamp")) \
        .withColumn("day_of_week", dayofweek("timestamp")) \
        .withColumn("is_weekend", 
                   when(dayofweek("timestamp").isin([1, 7]), 1).otherwise(0)) \
        \
        .withColumn("user_avg_value_historical", 
                   avg("value").over(user_window)) \
        .withColumn("user_event_count_historical", 
                   count("*").over(user_window)) \
        .withColumn("user_last_event_gap_minutes",
                   (col("timestamp").cast("long") - 
                    lag("timestamp").over(Window.partitionBy("user_id").orderBy("timestamp")).cast("long")) / 60) \
        \
        .withColumn("user_events_last_hour",
                   count("*").over(time_window_1h)) \
        .withColumn("user_value_sum_last_hour",
                   sum("value").over(time_window_1h)) \
        \
        .withColumn("value_percentile_rank",
                   percent_rank().over(Window.partitionBy().orderBy("value"))) \
        .withColumn("is_high_value",
                   when(col("value") > 100, 1).otherwise(0))
    
    return features_df

# Apply feature engineering
feature_stream = create_realtime_features(streaming_df)

# Select final feature set for ML
ml_features = feature_stream.select(
    "user_id", "timestamp", "event_type",
    "hour_of_day", "day_of_week", "is_weekend",
    "user_avg_value_historical", "user_event_count_historical",
    "user_last_event_gap_minutes", "user_events_last_hour",
    "value_percentile_rank", "is_high_value"
)
```

### Event Time vs Processing Time

```python
def compare_event_vs_processing_time(df):
    """Compare event time and processing time for monitoring"""
    
    processed_df = df \
        .withColumn("processing_time", current_timestamp()) \
        .withColumn("latency_seconds",
                   col("processing_time").cast("long") - col("timestamp").cast("long")) \
        .withColumn("is_late_data",
                   when(col("latency_seconds") > 300, True).otherwise(False))  # 5 min threshold
    
    # Monitor latency metrics
    latency_metrics = processed_df \
        .withWatermark("timestamp", "10 minutes") \
        .groupBy(window(col("processing_time"), "1 minute")) \
        .agg(
            count("*").alias("total_events"),
            avg("latency_seconds").alias("avg_latency"),
            max("latency_seconds").alias("max_latency"),
            sum(when(col("is_late_data"), 1).otherwise(0)).alias("late_events_count"),
            (sum(when(col("is_late_data"), 1).otherwise(0)) * 100.0 / count("*")).alias("late_events_percentage")
        )
    
    return processed_df, latency_metrics

# Apply event vs processing time analysis
processed_events, latency_monitoring = compare_event_vs_processing_time(streaming_df)

# Write latency metrics for monitoring
latency_query = latency_monitoring \
    .writeStream \
    .outputMode("append") \
    .format("console") \
    .trigger(processingTime="1 minute") \
    .start()
```

---

## Error Handling and Fault Tolerance

### Checkpoint Management

```python
def configure_checkpointing(query_name, checkpoint_base_path="/checkpoints"):
    """Configure robust checkpointing"""
    
    checkpoint_path = f"{checkpoint_base_path}/{query_name}"
    
    # Checkpoint configuration options
    checkpoint_config = {
        "checkpointLocation": checkpoint_path,
        "minBatchesToRetain": "100",  # Keep more batches for recovery
    }
    
    return checkpoint_config

# Use structured checkpointing
checkpoint_config = configure_checkpointing("user_events_aggregation")

robust_query = processed_df \
    .writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("path", "/data/streaming/user_events") \
    .options(**checkpoint_config) \
    .trigger(processingTime="1 minute") \
    .start()
```

### Error Recovery Strategies

```python
def create_fault_tolerant_stream(spark, source_config, sink_config):
    """Create fault-tolerant streaming query with error handling"""
    
    # Configure Spark for fault tolerance
    spark.conf.set("spark.sql.streaming.forceDeleteTempCheckpointLocation", "true")
    spark.conf.set("spark.sql.adaptive.enabled", "false")  # Disable for streaming
    spark.conf.set("spark.sql.streaming.checkpointLocation.deleteOnStop", "false")
    
    try:
        # Read stream with error handling
        stream_df = spark \
            .readStream \
            .format(source_config["format"]) \
            .options(**source_config["options"]) \
            .load()
        
        # Add error handling columns
        safe_stream = stream_df \
            .withColumn("processing_timestamp", current_timestamp()) \
            .withColumn("source_file", input_file_name())
        
        # Create query with comprehensive error handling
        query = safe_stream \
            .writeStream \
            .outputMode(sink_config["output_mode"]) \
            .format(sink_config["format"]) \
            .options(**sink_config["options"]) \
            .trigger(processingTime=sink_config["trigger"]) \
            .start()
        
        return query
        
    except Exception as e:
        print(f"Failed to create streaming query: {e}")
        # Implement retry logic or fallback strategy
        raise

# Example configuration
source_config = {
    "format": "kafka",
    "options": {
        "kafka.bootstrap.servers": "localhost:9092",
        "subscribe": "user_events",
        "startingOffsets": "latest",
        "failOnDataLoss": "false",  # Continue on data loss
        "maxOffsetsPerTrigger": "10000"
    }
}

sink_config = {
    "format": "delta",
    "output_mode": "append",
    "options": {
        "path": "/data/streaming/processed_events",
        "checkpointLocation": "/checkpoints/processed_events"
    },
    "trigger": "30 seconds"
}

# Create fault-tolerant query
ft_query = create_fault_tolerant_stream(spark, source_config, sink_config)
```

### Dead Letter Queue Pattern

```python
def process_with_dead_letter_queue(df):
    """Process data with dead letter queue for failed records"""
    
    # Define processing function that can fail
    def safe_process_record(record):
        try:
            # Complex processing logic here
            processed = {
                "user_id": record.user_id,
                "processed_value": record.value * 2,
                "status": "success",
                "error": None
            }
            return processed
        except Exception as e:
            return {
                "user_id": record.get("user_id", "unknown"),
                "processed_value": None,
                "status": "failed",
                "error": str(e)
            }
    
    # Apply processing with error handling
    processed_df = df.withColumn(
        "processing_result",
        udf(safe_process_record, StructType([
            StructField("user_id", StringType(), True),
            StructField("processed_value", DoubleType(), True),
            StructField("status", StringType(), True),
            StructField("error", StringType(), True)
        ]))(struct("*"))
    )
    
    # Separate successful and failed records
    success_df = processed_df \
        .filter(col("processing_result.status") == "success") \
        .select("*", col("processing_result.processed_value").alias("final_value"))
    
    failed_df = processed_df \
        .filter(col("processing_result.status") == "failed") \
        .select("*", col("processing_result.error").alias("error_message"))
    
    return success_df, failed_df

# Apply dead letter queue pattern
success_stream, dead_letter_stream = process_with_dead_letter_queue(streaming_df)

# Write successful records to main sink
success_query = success_stream \
    .writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("path", "/data/streaming/success") \
    .option("checkpointLocation", "/checkpoints/success") \
    .start()

# Write failed records to dead letter queue
dlq_query = dead_letter_stream \
    .writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("path", "/data/streaming/dead_letter_queue") \
    .option("checkpointLocation", "/checkpoints/dlq") \
    .start()
```

---

## Performance Optimization

### Streaming Performance Tuning

```python
def optimize_streaming_performance(spark):
    """Configure Spark for optimal streaming performance"""
    
    # Streaming-specific optimizations
    streaming_configs = {
        # Disable adaptive query execution for streaming
        "spark.sql.adaptive.enabled": "false",
        "spark.sql.adaptive.coalescePartitions.enabled": "false",
        
        # Optimize shuffle for streaming
        "spark.sql.shuffle.partitions": "200",  # Adjust based on cluster size
        "spark.sql.streaming.metricsEnabled": "true",
        
        # Optimize Kafka consumer
        "spark.sql.streaming.kafka.consumer.pollTimeoutMs": "512",
        "spark.sql.streaming.kafka.consumer.cache.capacity": "64",
        
        # Memory optimizations
        "spark.sql.streaming.stateStore.compressionCodec": "lz4",
        "spark.sql.streaming.stateStore.maintenanceInterval": "60s",
        
        # Checkpoint optimizations
        "spark.sql.streaming.checkpointFileManagerClass": 
            "org.apache.spark.sql.execution.streaming.state.RocksDBCheckpointFileManager",
        
        # State store optimizations
        "spark.sql.streaming.stateStore.providerClass": 
            "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
    }
    
    for key, value in streaming_configs.items():
        spark.conf.set(key, value)
    
    print("Streaming performance optimizations applied")

# Apply optimizations
optimize_streaming_performance(spark)
```

### Partition Optimization for Streaming

```python
def optimize_streaming_partitions(df, target_partition_size_mb=64):
    """Optimize partitions for streaming workloads"""
    
    # Calculate optimal partitions based on throughput
    current_partitions = df.rdd.getNumPartitions()
    
    # For streaming, typically want smaller partitions for lower latency
    # but not too small to avoid overhead
    
    # Repartition if needed
    if current_partitions > 1000:  # Too many partitions
        optimized_df = df.coalesce(200)
        print(f"Coalesced from {current_partitions} to 200 partitions")
    elif current_partitions < 10:  # Too few partitions
        optimized_df = df.repartition(50)
        print(f"Repartitioned from {current_partitions} to 50 partitions")
    else:
        optimized_df = df
        print(f"Partition count ({current_partitions}) is optimal")
    
    return optimized_df

# Apply partition optimization
optimized_stream = optimize_streaming_partitions(streaming_df)
```

### State Store Optimization

```python
def configure_state_store_optimization():
    """Configure state store for optimal performance"""
    
    state_store_configs = {
        # Use RocksDB for better performance with large state
        "spark.sql.streaming.stateStore.providerClass": 
            "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider",
        
        # RocksDB specific configurations
        "spark.sql.streaming.stateStore.rocksdb.compactOnCommit": "true",
        "spark.sql.streaming.stateStore.rocksdb.blockSizeKB": "32",
        "spark.sql.streaming.stateStore.rocksdb.blockCacheSizeMB": "128",
        
        # State cleanup
        "spark.sql.streaming.stateStore.maintenanceInterval": "60s",
        
        # Compression
        "spark.sql.streaming.stateStore.compressionCodec": "lz4"
    }
    
    for key, value in state_store_configs.items():
        spark.conf.set(key, value)

# Configure state store
configure_state_store_optimization()

# Example of stateful operation that benefits from optimization
stateful_aggregation = streaming_df \
    .withWatermark("timestamp", "10 minutes") \
    .groupBy("user_id") \
    .agg(
        count("*").alias("total_events"),
        sum("value").alias("total_value"),
        collect_list("event_type").alias("event_types")
    )
```

---

## Monitoring and Observability

### Streaming Metrics Collection

```python
class StreamingMetricsCollector:
    """Collect and monitor streaming metrics"""
    
    def __init__(self, query_name):
        self.query_name = query_name
        self.metrics_history = []
    
    def collect_query_metrics(self, streaming_query):
        """Collect metrics from streaming query"""
        try:
            progress = streaming_query.lastProgress
            
            if progress:
                metrics = {
                    "timestamp": progress.get("timestamp"),
                    "batch_id": progress.get("batchId"),
                    "input_rows_per_second": progress.get("inputRowsPerSecond", 0),
                    "processed_rows_per_second": progress.get("processedRowsPerSecond", 0),
                    "batch_duration_ms": progress.get("durationMs", {}).get("batchDuration", 0),
                    "trigger_execution_ms": progress.get("durationMs", {}).get("triggerExecution", 0),
                    "state_operators": progress.get("stateOperators", []),
                    "sources": progress.get("sources", []),
                    "sink": progress.get("sink", {})
                }
                
                self.metrics_history.append(metrics)
                return metrics
            
        except Exception as e:
            print(f"Error collecting metrics: {e}")
            return None
    
    def get_performance_summary(self, last_n_batches=10):
        """Get performance summary for last N batches"""
        
        if len(self.metrics_history) < last_n_batches:
            recent_metrics = self.metrics_history
        else:
            recent_metrics = self.metrics_history[-last_n_batches:]
        
        if not recent_metrics:
            return {"message": "No metrics available"}
        
        # Calculate averages
        avg_input_rate = sum(m["input_rows_per_second"] for m in recent_metrics) / len(recent_metrics)
        avg_processing_rate = sum(m["processed_rows_per_second"] for m in recent_metrics) / len(recent_metrics)
        avg_batch_duration = sum(m["batch_duration_ms"] for m in recent_metrics) / len(recent_metrics)
        
        # Check for performance issues
        issues = []
        if avg_input_rate > avg_processing_rate:
            issues.append("Input rate exceeds processing rate - may cause backlog")
        
        if avg_batch_duration > 60000:  # 1 minute
            issues.append("High batch duration detected")
        
        return {
            "query_name": self.query_name,
            "batches_analyzed": len(recent_metrics),
            "avg_input_rows_per_second": avg_input_rate,
            "avg_processed_rows_per_second": avg_processing_rate,
            "avg_batch_duration_ms": avg_batch_duration,
            "performance_issues": issues
        }

# Usage
metrics_collector = StreamingMetricsCollector("user_events_processing")

# Monitor query periodically
def monitor_streaming_query(query, collector, interval_seconds=30):
    """Monitor streaming query metrics"""
    import time
    import threading
    
    def monitor_loop():
        while query.isActive:
            metrics = collector.collect_query_metrics(query)
            if metrics:
                print(f"Batch {metrics['batch_id']}: "
                      f"Input rate: {metrics['input_rows_per_second']:.2f} rows/sec, "
                      f"Processing rate: {metrics['processed_rows_per_second']:.2f} rows/sec")
            
            time.sleep(interval_seconds)
    
    monitor_thread = threading.Thread(target=monitor_loop, daemon=True)
    monitor_thread.start()
    return monitor_thread

# Start monitoring
monitor_thread = monitor_streaming_query(processed_query, metrics_collector)

# Get performance summary
summary = metrics_collector.get_performance_summary()
print(f"Performance Summary: {summary}")
```

### Custom Streaming Dashboard

```python
def create_streaming_dashboard(queries_list):
    """Create a simple dashboard for multiple streaming queries"""
    
    import time
    from datetime import datetime
    
    def display_dashboard():
        """Display current status of all queries"""
        
        print("\n" + "="*80)
        print(f"STREAMING DASHBOARD - {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print("="*80)
        
        for i, (name, query) in enumerate(queries_list):
            status = "🟢 ACTIVE" if query.isActive else "🔴 STOPPED"
            
            try:
                progress = query.lastProgress
                if progress:
                    batch_id = progress.get("batchId", "N/A")
                    input_rate = progress.get("inputRowsPerSecond", 0)
                    processing_rate = progress.get("processedRowsPerSecond", 0)
                    
                    print(f"{i+1}. {name}")
                    print(f"   Status: {status}")
                    print(f"   Batch ID: {batch_id}")
                    print(f"   Input Rate: {input_rate:.2f} rows/sec")
                    print(f"   Processing Rate: {processing_rate:.2f} rows/sec")
                    
                    if input_rate > 0 and processing_rate > 0:
                        ratio = processing_rate / input_rate
                        health = "🟢 HEALTHY" if ratio >= 0.9 else "🟡 LAGGING" if ratio >= 0.5 else "🔴 FALLING BEHIND"
                        print(f"   Health: {health} (Ratio: {ratio:.2f})")
                else:
                    print(f"{i+1}. {name}")
                    print(f"   Status: {status}")
                    print(f"   Progress: No data available")
                
                print()
                
            except Exception as e:
                print(f"{i+1}. {name}")
                print(f"   Status: {status}")
                print(f"   Error: {e}")
                print()
    
    return display_dashboard

# Create dashboard for multiple queries
queries = [
    ("User Events Processing", user_events_query),
    ("Anomaly Detection", anomaly_query),
    ("Real-time Aggregation", aggregation_query)
]

dashboard = create_streaming_dashboard(queries)

# Display dashboard periodically
def run_dashboard(dashboard_func, interval_seconds=30):
    """Run dashboard in a loop"""
    try:
        while True:
            dashboard_func()
            time.sleep(interval_seconds)
    except KeyboardInterrupt:
        print("\nDashboard stopped.")

# Run dashboard
# run_dashboard(dashboard, 30)
```

---

## Production Best Practices

### Streaming Application Template

```python
class ProductionStreamingApp:
    """Production-ready streaming application template"""
    
    def __init__(self, app_name, configs=None):
        self.app_name = app_name
        self.configs = configs or {}
        self.spark = None
        self.queries = {}
        self.metrics_collectors = {}
        
    def initialize_spark(self):
        """Initialize Spark session with production configurations"""
        
        builder = SparkSession.builder.appName(self.app_name)
        
        # Apply default streaming configurations
        default_configs = {
            "spark.sql.adaptive.enabled": "false",
            "spark.sql.streaming.metricsEnabled": "true",
            "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
            "spark.sql.streaming.stateStore.providerClass": 
                "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
        }
        
        # Merge with custom configs
        all_configs = {**default_configs, **self.configs}
        
        for key, value in all_configs.items():
            builder = builder.config(key, value)
        
        self.spark = builder.getOrCreate()
        print(f"Initialized Spark session for {self.app_name}")
    
    def create_query(self, query_name, source_config, transformation_func, sink_config):
        """Create a streaming query with error handling"""
        
        try:
            # Read from source
            source_df = self.spark.readStream \
                .format(source_config["format"]) \
                .options(**source_config.get("options", {}))
            
            if "schema" in source_config:
                source_df = source_df.schema(source_config["schema"])
            
            stream_df = source_df.load()
            
            # Apply transformation
            transformed_df = transformation_func(stream_df)
            
            # Create sink
            query = transformed_df.writeStream \
                .queryName(query_name) \
                .outputMode(sink_config["output_mode"]) \
                .format(sink_config["format"]) \
                .options(**sink_config.get("options", {})) \
                .trigger(processingTime=sink_config.get("trigger", "30 seconds")) \
                .start()
            
            self.queries[query_name] = query
            self.metrics_collectors[query_name] = StreamingMetricsCollector(query_name)
            
            print(f"Started query: {query_name}")
            return query
            
        except Exception as e:
            print(f"Failed to create query {query_name}: {e}")
            raise
    
    def monitor_queries(self):
        """Monitor all active queries"""
        
        for query_name, query in self.queries.items():
            if query.isActive:
                collector = self.metrics_collectors[query_name]
                collector.collect_query_metrics(query)
    
    def get_application_status(self):
        """Get overall application status"""
        
        total_queries = len(self.queries)
        active_queries = sum(1 for q in self.queries.values() if q.isActive)
        
        status = {
            "application_name": self.app_name,
            "total_queries": total_queries,
            "active_queries": active_queries,
            "spark_context_active": self.spark.sparkContext._jsc.sc().isStopped() == False,
            "queries": {}
        }
        
        for name, query in self.queries.items():
            collector = self.metrics_collectors[name]
            summary = collector.get_performance_summary(5)
            status["queries"][name] = {
                "active": query.isActive,
                "performance": summary
            }
        
        return status
    
    def shutdown(self):
        """Gracefully shutdown the application"""
        
        print("Shutting down streaming application...")
        
        # Stop all queries
        for query_name, query in self.queries.items():
            if query.isActive:
                print(f"Stopping query: {query_name}")
                query.stop()
        
        # Stop Spark session
        if self.spark:
            self.spark.stop()
        
        print("Application shutdown complete")

# Usage example
def user_events_transformation(df):
    """Sample transformation function"""
    return df.filter(col("event_type") == "purchase") \
             .select("user_id", "timestamp", "value")

# Create application
app = ProductionStreamingApp("UserEventsProcessor")
app.initialize_spark()

# Configure sources and sinks
kafka_source = {
    "format": "kafka",
    "options": {
        "kafka.bootstrap.servers": "localhost:9092",
        "subscribe": "user_events",
        "startingOffsets": "latest"
    }
}

delta_sink = {
    "format": "delta",
    "output_mode": "append",
    "options": {
        "path": "/data/processed_events",
        "checkpointLocation": "/checkpoints/processed_events"
    },
    "trigger": "1 minute"
}

# Create query
query = app.create_query(
    "user_events_processing",
    kafka_source,
    user_events_transformation,
    delta_sink
)

# Monitor and manage
try:
    # Keep application running
    query.awaitTermination()
except KeyboardInterrupt:
    app.shutdown()
```

---

## Related Notes

- [[PySpark Core Concepts]] - Foundation concepts for streaming applications
- [[PySpark Data Operations]] - Data manipulation techniques used in streaming
- [[PySpark Window Functions]] - Advanced windowing for streaming analytics
- [[PySpark Performance Optimization]] - Performance tuning for streaming workloads
- [[PySpark Production Engineering]] - Production deployment of streaming applications

---

## Tags

#pyspark #streaming #structured-streaming #real-time #kafka #watermarks #windowing #triggers #fault-tolerance #monitoring

---

_Last Updated: 2024-08-20_