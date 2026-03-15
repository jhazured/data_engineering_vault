#data-streaming #apache-flink #aws-kinesis #stream-processing #real-time

# Apache Flink & AWS Kinesis

Apache Flink is a distributed stream processing framework for stateful computations over unbounded and bounded data streams. AWS Kinesis is a managed platform for collecting, processing, and analysing real-time streaming data. This note covers both systems in depth and compares them against [[Apache Kafka Fundamentals]] and Spark Structured Streaming.

For foundational concepts on event time, watermarks, windowing, and exactly-once semantics, see [[Stream Processing Theory]].

---

## Part 1: Apache Flink

### Architecture

Flink follows a master-worker architecture with clear separation of concerns:

```
                     ┌──────────────────────┐
                     │     JobManager        │
                     │  ─ Job scheduling     │
                     │  ─ Checkpoint coord.  │
                     │  ─ Resource mgmt      │
                     └──────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                   ▼
     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
     │  TaskManager 1 │ │  TaskManager 2 │ │  TaskManager 3 │
     │  ─ Task slots  │ │  ─ Task slots  │ │  ─ Task slots  │
     │  ─ State mgmt  │ │  ─ State mgmt  │ │  ─ State mgmt  │
     │  ─ Network I/O │ │  ─ Network I/O │ │  ─ Network I/O │
     └────────────────┘ └────────────────┘ └────────────────┘
```

| Component | Role |
|-----------|------|
| **JobManager** | Coordinates distributed execution: schedules tasks, triggers checkpoints, handles failover. One per cluster (with optional standby for HA) |
| **TaskManager** | Worker process that executes one or more task slots. Each slot runs a pipeline of operators (subtasks) |
| **Task Slot** | Fixed slice of a TaskManager's resources (memory). Controls parallelism — a job with parallelism 12 needs at least 12 slots |
| **Dispatcher** | Accepts job submissions, launches JobManagers per job (in application mode) |
| **ResourceManager** | Negotiates containers/slots from YARN, Kubernetes, or standalone cluster |

#### Parallelism

Parallelism determines how many instances of each operator run concurrently:

```
Source (parallelism=3)    →    Map (parallelism=3)    →    Sink (parallelism=2)
  ├── subtask 0                 ├── subtask 0               ├── subtask 0
  ├── subtask 1                 ├── subtask 1               └── subtask 1
  └── subtask 2                 └── subtask 2
```

- Set globally (`env.setParallelism(4)`) or per-operator (`.map(...).setParallelism(8)`)
- Max effective parallelism for a source reading from Kafka = number of Kafka partitions
- Downstream operators can have different parallelism — Flink redistributes data via network shuffle

### DataStream API

The core API for building streaming applications:

#### Sources

```python
# Kafka source (most common in production)
source = KafkaSource.builder() \
    .setBootstrapServers("broker:9092") \
    .setTopics("events") \
    .setGroupId("flink-consumer") \
    .setStartingOffsets(KafkaOffsetsInitializer.earliest()) \
    .setValueOnlyDeserializer(SimpleStringSchema()) \
    .build()

stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
```

Other built-in sources: filesystem (bounded/unbounded), socket, collection (testing), custom `SourceFunction`.

#### Transformations

| Transformation | Description | Example |
|---------------|-------------|---------|
| `map()` | One-to-one element mapping | Parse JSON string to object |
| `flat_map()` | One-to-many element mapping | Split line into words |
| `filter()` | Discard elements not matching predicate | Remove invalid records |
| `key_by()` | Partition stream by key (enables keyed state) | Group by customer ID |
| `reduce()` | Rolling aggregation over keyed stream | Running sum of order totals |
| `union()` | Merge multiple streams of the same type | Combine clickstream sources |
| `connect()` | Pair two streams of different types | Enrich events with rules stream |
| `process()` | Low-level access to timestamps, timers, state | Complex event processing |

#### Sinks

```python
# Kafka sink with exactly-once delivery
sink = KafkaSink.builder() \
    .setBootstrapServers("broker:9092") \
    .setRecordSerializer(
        KafkaRecordSerializationSchema.builder()
            .setTopic("output")
            .setValueSerializationSchema(SimpleStringSchema())
            .build()
    ) \
    .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE) \
    .build()

stream.sink_to(sink)
```

Other sinks: JDBC, Elasticsearch, filesystem (Parquet, ORC, CSV), custom `SinkFunction`.

### Windowing

Flink implements all the window types described in [[Stream Processing Theory]], with rich configuration options:

```python
# Tumbling window — fixed, non-overlapping
stream.key_by(lambda e: e.user_id) \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .sum("amount")

# Sliding window — overlapping
stream.key_by(lambda e: e.user_id) \
    .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(2))) \
    .aggregate(MyAggregateFunction())

# Session window — gap-based
stream.key_by(lambda e: e.session_id) \
    .window(EventTimeSessionWindows.with_gap(Time.minutes(30))) \
    .process(MyProcessWindowFunction())

# Global window — requires custom trigger
stream.key_by(lambda e: e.key) \
    .window(GlobalWindows.create()) \
    .trigger(CountTrigger.of(100)) \
    .reduce(MyReduceFunction())
```

| Window Type | Use Case | Key Consideration |
|-------------|----------|-------------------|
| **Tumbling** | Regular aggregations (hourly metrics) | Simple; no overlap means each event belongs to exactly one window |
| **Sliding** | Moving averages, trend detection | Events duplicated across overlapping windows — higher memory/compute |
| **Session** | User activity sessions, clickstream | Dynamic size; gap timeout must match domain behaviour |
| **Global** | Custom windowing logic | Must supply a custom trigger; window never closes otherwise |

#### Window Functions

| Function | When to Use | State |
|----------|-------------|-------|
| `ReduceFunction` | Incremental aggregation (sum, min, max) | Keeps single accumulated value |
| `AggregateFunction` | Incremental with accumulator (avg, percentiles) | Keeps accumulator object |
| `ProcessWindowFunction` | Need access to all elements + metadata (window start/end) | Buffers all elements — use only when necessary |
| `ReduceFunction` + `ProcessWindowFunction` | Incremental aggregation with window metadata | Best of both: incremental compute + metadata access |

### State Management

Flink's state management is a core differentiator — it provides fault-tolerant, scalable state that is co-located with the processing logic.

#### Keyed State

Available only in keyed contexts (after `key_by()`). Each key has its own isolated state:

| State Type | Structure | Use Case |
|------------|-----------|----------|
| `ValueState<T>` | Single value per key | Last seen event, running status |
| `ListState<T>` | List of values per key | Buffered events for a pattern |
| `MapState<K, V>` | Key-value map per key | Feature store per user |
| `ReducingState<T>` | Single value, auto-reduced on add | Running sum without manual merge |
| `AggregatingState<IN, OUT>` | Accumulated value with custom aggregation | Running average |

#### Operator State

Not partitioned by key — each parallel operator instance has its own state:

- `ListState` / `UnionListState` — used by source connectors to track offsets
- Redistributed on rescaling (list state is split/merged, union state is broadcast)

#### State Backends

| Backend | Storage | Performance | Capacity | Use Case |
|---------|---------|-------------|----------|----------|
| **HashMapStateBackend** | JVM heap (Java objects) | Very fast access | Limited by JVM heap | Small state (< a few GB), low latency |
| **EmbeddedRocksDBStateBackend** | RocksDB on local disk + JVM cache | Slower (serialisation overhead) | Terabytes+ | Large state, production default |

```python
# Configure RocksDB state backend
env.set_state_backend(EmbeddedRocksDBStateBackend())
env.get_checkpoint_config().set_checkpoint_storage("s3://bucket/checkpoints")
```

### Checkpointing

Flink implements the Chandy-Lamport distributed snapshot algorithm for consistent checkpoints:

```
Source 1 ──barrier──► Map 1 ──barrier──► Sink 1
Source 2 ──barrier──► Map 2 ──barrier──► Sink 2

Checkpoint N:
  1. JobManager injects barrier N into all sources
  2. Barriers flow through the dataflow graph with the data
  3. When an operator receives barriers from ALL inputs:
     a. Snapshot its state to durable storage
     b. Forward the barrier downstream
  4. When all sinks acknowledge → checkpoint N is complete
```

#### Barrier Alignment

When an operator has multiple inputs, it must **align** barriers:

- **Aligned checkpointing** (default): operator buffers data from fast channels until slow channels deliver their barrier. Guarantees exactly-once but adds latency during checkpoint.
- **Unaligned checkpointing** (Flink 1.11+): operator snapshots in-flight data from fast channels alongside its state. Lower latency under backpressure, at the cost of larger checkpoint sizes.

#### Configuration

```python
env.enable_checkpointing(60000)  # checkpoint every 60 seconds
config = env.get_checkpoint_config()
config.set_checkpointing_mode(CheckpointingMode.EXACTLY_ONCE)
config.set_min_pause_between_checkpoints(30000)
config.set_checkpoint_timeout(600000)
config.set_max_concurrent_checkpoints(1)
config.set_externalized_checkpoint_cleanup(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION
)
```

#### Savepoints vs Checkpoints

| Aspect | Checkpoint | Savepoint |
|--------|-----------|-----------|
| **Purpose** | Automatic fault recovery | Manual, planned operations |
| **Triggered by** | Flink (periodic) | User (CLI or API) |
| **Use case** | Crash recovery | Upgrades, rescaling, migrations |
| **Retention** | Deleted when superseded (unless externalised) | Retained until manually deleted |
| **Format** | Optimised for speed | Portable across Flink versions |

### Event Time vs Processing Time

Flink has first-class support for event-time processing via watermarks, building on the theory from [[Stream Processing Theory]]:

```python
# Assign watermarks with bounded out-of-orderness
watermark_strategy = WatermarkStrategy \
    .for_bounded_out_of_orderness(Duration.of_seconds(10)) \
    .with_timestamp_assigner(
        lambda event, timestamp: event.event_timestamp
    )

stream = env.from_source(source, watermark_strategy, "Source")
```

#### Watermark Propagation

```
Source 1 (watermark: 10:05) ──┐
                               ├──► Operator (watermark = min(10:05, 10:03) = 10:03)
Source 2 (watermark: 10:03) ──┘
```

An operator's watermark is the **minimum** across all input watermarks. This ensures correctness but means a slow source holds back the entire pipeline.

#### Late Data Handling

```python
stream.key_by(lambda e: e.key) \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .allowed_lateness(Time.minutes(2)) \
    .side_output_late_data(late_output_tag) \
    .sum("value")
```

- **Allowed lateness**: window results are updated if late events arrive within the allowed period
- **Side output**: events that arrive after the allowed lateness are routed to a side output for separate handling (logging, dead-letter sink)

### Table API / Flink SQL

Flink provides a relational API on top of the DataStream API, enabling streaming SQL:

```sql
-- Create a Kafka-backed streaming table
CREATE TABLE page_views (
    user_id STRING,
    page_url STRING,
    view_time TIMESTAMP(3),
    WATERMARK FOR view_time AS view_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'page_views',
    'properties.bootstrap.servers' = 'broker:9092',
    'format' = 'json'
);

-- Streaming aggregation with tumbling window
SELECT
    user_id,
    TUMBLE_START(view_time, INTERVAL '1' HOUR) AS window_start,
    COUNT(*) AS view_count
FROM page_views
GROUP BY
    user_id,
    TUMBLE(view_time, INTERVAL '1' HOUR);
```

#### Temporal Joins

Join a stream against a versioned table (e.g., slowly changing dimension):

```sql
-- Join orders with the exchange rate valid at order time
SELECT
    o.order_id,
    o.amount * r.rate AS converted_amount
FROM orders AS o
JOIN currency_rates FOR SYSTEM_TIME AS OF o.order_time AS r
    ON o.currency = r.currency;
```

Temporal joins avoid the need for windowed joins when one side is a versioned reference table — the lookup uses the version valid at the event's timestamp.

### Flink CDC (Change Data Capture)

Flink CDC connectors read database change logs directly, without requiring a separate Kafka cluster:

```sql
-- Read CDC events directly from MySQL
CREATE TABLE products (
    id INT,
    name STRING,
    price DECIMAL(10, 2),
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'mysql-host',
    'port' = '3306',
    'username' = 'flink',
    'password' = '***',
    'database-name' = 'shop',
    'table-name' = 'products'
);
```

- Built on **Debezium** under the hood — reads MySQL binlog, PostgreSQL WAL, etc.
- Supports initial snapshot + continuous streaming in a single job
- Can feed into Flink's Table API for real-time materialised views
- Alternative to the Kafka Connect + Debezium pipeline for simpler architectures

### Deployment Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Standalone** | Fixed cluster of JMs/TMs managed manually | Development, small deployments |
| **YARN** | Flink requests containers from Hadoop YARN | Existing Hadoop infrastructure |
| **Kubernetes (native)** | Flink talks to K8s API to create TaskManager pods | Cloud-native, elastic scaling |
| **Kubernetes (operator)** | Flink Kubernetes Operator manages lifecycle declaratively | Production K8s — preferred approach |

#### Application Mode vs Session Mode

| Aspect | Session Mode | Application Mode |
|--------|-------------|-----------------|
| **Cluster lifecycle** | Long-lived, shared cluster | One cluster per job |
| **Isolation** | Jobs share resources | Full resource isolation |
| **JAR upload** | Client uploads to session cluster | JAR downloaded inside the cluster |
| **Use case** | Development, ad-hoc queries | Production workloads |

### Flink vs Spark Structured Streaming

| Dimension | Apache Flink | Spark Structured Streaming |
|-----------|-------------|---------------------------|
| **Processing model** | True record-at-a-time streaming | Micro-batch (default) or continuous (experimental) |
| **Latency** | Milliseconds | Seconds (micro-batch), ~100ms (continuous, limited ops) |
| **State management** | Built-in, managed (RocksDB, heap) | State store API with HDFS-backed checkpoints |
| **Exactly-once** | Native via Chandy-Lamport checkpoints | Via micro-batch transactional writes |
| **Event time** | First-class watermark support, rich late data handling | Watermark support, but less flexible |
| **SQL support** | Flink SQL (streaming-native) | Spark SQL (batch-first, extended for streaming) |
| **Ecosystem** | Growing; strong in streaming-first orgs | Massive; dominant in batch + ML workflows |
| **Backpressure** | Credit-based flow control (built-in) | Micro-batch naturally limits intake |
| **Deployment** | Standalone, YARN, K8s | YARN, K8s, Databricks, EMR |
| **Best for** | Low-latency streaming, complex event processing | Unified batch + streaming, organisations already using Spark |

For [[PySpark Streaming]] specifics including `readStream`/`writeStream` patterns, see the PySpark vault section.

---

## Part 2: AWS Kinesis

### Kinesis Data Streams

A managed, real-time data streaming service. The core building block of Kinesis.

```
Producers ──► Shard 1 ──► Consumers
              Shard 2 ──► Consumers
              Shard 3 ──► Consumers
```

#### Shards and Partition Keys

| Concept | Description |
|---------|-------------|
| **Shard** | Unit of capacity. Each shard provides 1 MB/s write, 2 MB/s read (shared), 1000 records/s write |
| **Partition key** | Determines shard placement via MD5 hash. Same key always routes to same shard |
| **Sequence number** | Assigned by Kinesis on ingestion — monotonically increasing within a shard |
| **Retention** | Default 24 hours, configurable up to 365 days |

#### Capacity Planning

```
Required shards = MAX(
    incoming_write_MB_per_sec / 1,
    incoming_records_per_sec / 1000,
    outgoing_read_MB_per_sec / 2    # per consumer (shared mode)
)
```

**Example**: 5 MB/s write, 8000 records/s, 3 consumers at 2 MB/s each:
- Write throughput: 5 / 1 = 5 shards
- Record rate: 8000 / 1000 = 8 shards
- Read throughput: (3 x 2) / 2 = 3 shards (shared), or 3 shards with enhanced fan-out
- **Result**: 8 shards (governed by record rate)

#### On-Demand vs Provisioned Mode

| Aspect | Provisioned | On-Demand |
|--------|------------|-----------|
| **Capacity** | Manual shard count | Auto-scales up to 200 MB/s write |
| **Cost model** | Per shard-hour | Per GB ingested + retrieved |
| **Scaling** | Manual split/merge or auto-scaling via CloudWatch | Automatic |
| **Best for** | Predictable, steady workloads | Variable/unpredictable traffic |

### Kinesis Data Firehose

Fully managed delivery service — no consumers to write or manage:

```
Sources ──► Firehose ──► Transformation (optional Lambda) ──► Destination
                              │
                              ├── S3 (Parquet, ORC, JSON, CSV)
                              ├── Redshift (via S3 COPY)
                              ├── OpenSearch
                              ├── Splunk
                              ├── HTTP endpoint
                              └── Third-party (Datadog, New Relic, etc.)
```

| Feature | Detail |
|---------|--------|
| **Buffering** | Time-based (60-900s) or size-based (1-128 MB), whichever is met first |
| **Compression** | GZIP, Snappy, Zip for S3 delivery |
| **Format conversion** | JSON to Parquet/ORC via AWS Glue schema |
| **Transformation** | Inline Lambda for record-level transforms |
| **Error handling** | Failed records written to a separate S3 error prefix |
| **Delivery guarantee** | At-least-once |

Firehose is the simplest path from streaming data to [[AWS Data Services for Data Engineering|S3/Redshift/OpenSearch]] with no infrastructure to manage.

### Kinesis Data Analytics

Run SQL or Apache Flink applications on streaming data without managing infrastructure:

#### SQL-Based (Legacy)

```sql
-- Tumbling window aggregation
CREATE OR REPLACE STREAM "OUTPUT_STREAM" (
    ticker VARCHAR(6), min_price DOUBLE, max_price DOUBLE
);
CREATE OR REPLACE PUMP "STREAM_PUMP" AS
INSERT INTO "OUTPUT_STREAM"
SELECT STREAM
    ticker, MIN(price), MAX(price)
FROM "SOURCE_SQL_STREAM_001"
GROUP BY ticker, STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '1' MINUTE);
```

**Note**: SQL-based Kinesis Data Analytics is considered legacy. AWS recommends the Apache Flink runtime for new applications.

#### Apache Flink Runtime (Managed Service for Apache Flink)

- Runs standard Flink applications (Java, Scala, Python) on managed infrastructure
- Automatic scaling, checkpointing, and fault tolerance
- Reads from Kinesis Data Streams, MSK (Kafka), or S3
- Full access to Flink's DataStream API, Table API, and Flink SQL
- Integrates with AWS Glue Data Catalog for schema management

### Enhanced Fan-Out

Standard consumers share the 2 MB/s per-shard read limit. Enhanced fan-out provides **dedicated throughput**:

```
                    ┌──► Consumer A (2 MB/s dedicated, push via HTTP/2)
Shard 1 (2 MB/s) ──┤
                    ├──► Consumer B (2 MB/s dedicated, push via HTTP/2)
                    └──► Consumer C (2 MB/s dedicated, push via HTTP/2)

Total read: 6 MB/s from a single shard (vs 2 MB/s shared)
```

| Aspect | Shared (Standard) | Enhanced Fan-Out |
|--------|-------------------|------------------|
| **Throughput** | 2 MB/s shared across all consumers | 2 MB/s per consumer per shard |
| **Delivery** | Pull (GetRecords polling) | Push (SubscribeToShard, HTTP/2) |
| **Latency** | ~200ms (polling interval) | ~70ms (push) |
| **Max consumers** | 5 per shard (practical limit) | Up to 20 registered consumers |
| **Cost** | Included in shard cost | Additional per consumer-shard-hour + per GB |

Use enhanced fan-out when multiple applications read from the same stream and need independent, low-latency throughput.

### Producer Patterns

| Producer | Description | Use Case |
|----------|-------------|----------|
| **AWS SDK (PutRecord/PutRecords)** | Direct API call; simple, synchronous | Low-volume, simple applications |
| **Kinesis Producer Library (KPL)** | High-performance C++ library with Java wrapper. Batches, compresses, retries automatically | High-throughput production workloads |
| **Kinesis Agent** | Pre-built Java application that monitors files and sends to Kinesis | Log file ingestion from EC2 instances |
| **AWS IoT Core** | Rules engine routes messages to Kinesis | IoT device telemetry |

**KPL features**: micro-batching (aggregation), collection (multiple records per API call), retry with backoff, CloudWatch metrics. Note: KPL introduces slight latency (~100ms) due to buffering — configurable via `RecordMaxBufferedTime`.

### Consumer Patterns

| Consumer | Description | Use Case |
|----------|-------------|----------|
| **Kinesis Client Library (KCL)** | Managed consumer with shard leasing, checkpointing, load balancing across workers | Standard multi-instance consumer |
| **AWS Lambda** | Event source mapping triggers Lambda per batch of records | Serverless processing, low-to-moderate throughput |
| **Kinesis Data Analytics (Flink)** | Managed Flink application for complex processing | Streaming analytics, windowed aggregations |
| **Kinesis Data Firehose** | Managed delivery — no code required | Direct delivery to S3, Redshift, OpenSearch |
| **AWS SDK (GetRecords)** | Low-level polling API | Custom consumers, testing |

**KCL details**:
- One KCL worker per shard (max) — distributes shards across application instances
- Checkpoint tracking via DynamoDB lease table
- Handles shard splits and merges transparently
- KCL 2.x supports enhanced fan-out

### Cost Model

| Component | Pricing Dimension | Approximate Cost (US East) |
|-----------|------------------|---------------------------|
| **Shard hour** (provisioned) | Per shard per hour | ~$0.015/shard/hour |
| **On-demand ingestion** | Per GB ingested | ~$0.08/GB |
| **PUT payload units** | Per 1M units (25KB each) | ~$0.014/1M units |
| **Extended retention** (>24h) | Per shard-hour | ~$0.020/shard/hour (up to 7 days) |
| **Long-term retention** (>7d) | Per GB-month stored | ~$0.023/GB-month |
| **Enhanced fan-out** | Per consumer-shard-hour + per GB retrieved | ~$0.015/consumer-shard/hour + $0.013/GB |

**Cost optimisation tips**:
- Use on-demand mode for unpredictable traffic to avoid over-provisioning
- Aggregate small records with KPL to maximise 25KB PUT payload units
- Limit enhanced fan-out to consumers that genuinely need dedicated throughput
- Use Firehose for delivery to S3 — no shard costs, only pay per GB ingested

### Kinesis vs Kafka Comparison

| Dimension | AWS Kinesis Data Streams | Apache Kafka (Self-Managed / MSK) |
|-----------|------------------------|-----------------------------------|
| **Management** | Fully managed (no brokers) | Self-managed or semi-managed (MSK) |
| **Scaling unit** | Shard (split/merge or on-demand) | Partition (add partitions, add brokers) |
| **Throughput per unit** | 1 MB/s in, 2 MB/s out per shard | ~10 MB/s per partition (hardware-dependent) |
| **Max retention** | 365 days | Unlimited (configurable) |
| **Replay** | Yes (within retention) | Yes (within retention) |
| **Ordering** | Per shard | Per partition |
| **Exactly-once** | At-least-once (dedup in consumer) | Native exactly-once (transactional API) |
| **Multi-consumer** | Shared (2 MB/s) or enhanced fan-out | Independent consumer groups (no shared limit) |
| **Ecosystem** | AWS-native (Lambda, Firehose, Analytics) | Kafka Connect, Schema Registry, ksqlDB, broad community |
| **Cost model** | Per shard-hour + PUT units | Infrastructure (EC2/EBS) or MSK broker-hours |
| **Vendor lock-in** | AWS only | Cloud-agnostic (self-managed) or MSK (AWS) |
| **Best for** | AWS-native architectures, serverless integration | High-throughput, multi-cloud, complex event processing |

---

## Part 3: Comprehensive Comparison Table

| Dimension | Apache Kafka | Apache Flink | AWS Kinesis | Spark Structured Streaming |
|-----------|-------------|-------------|-------------|---------------------------|
| **Primary role** | Message broker / event log | Stream processing engine | Managed streaming platform | Stream processing engine (batch-first) |
| **Processing model** | Pub/sub with consumer groups | True record-at-a-time | Pub/sub with consumers | Micro-batch (default) |
| **Latency** | Low (ms for produce/consume) | Very low (ms) | Low (~70ms fan-out, ~200ms shared) | Higher (seconds, micro-batch interval) |
| **State management** | Kafka Streams has built-in state | Built-in, managed (RocksDB/heap) | None (stateless delivery) | State store with HDFS checkpoints |
| **Exactly-once** | Transactional API (Kafka 0.11+) | Chandy-Lamport checkpointing | At-least-once (consumer dedup) | Micro-batch transactional writes |
| **Windowing** | Kafka Streams: tumbling, sliding, session | Full: tumbling, sliding, session, global + custom triggers | Kinesis Analytics (Flink): full windowing | Tumbling, sliding, session (via `groupBy`) |
| **SQL support** | ksqlDB (separate component) | Flink SQL (native) | Kinesis Analytics SQL (legacy) | Spark SQL (mature) |
| **Scaling** | Add partitions + brokers | Adjust parallelism + TaskManagers | Add shards or use on-demand | Add executors |
| **Deployment** | Self-managed, Confluent Cloud, MSK | Standalone, YARN, K8s, managed (AWS, Confluent) | AWS managed (no infra) | YARN, K8s, Databricks, EMR |
| **Ecosystem** | Connect, Schema Registry, ksqlDB | CDC, Table API, ML (Flink ML) | Firehose, Lambda, Glue | MLlib, Delta Lake, broad Spark ecosystem |
| **Retention** | Configurable (unlimited) | N/A (processing engine, not storage) | Up to 365 days | N/A (processing engine, not storage) |
| **Managed options** | Confluent Cloud, Amazon MSK | Amazon Managed Flink, Ververica | Native AWS service | Databricks, Amazon EMR |
| **Best for** | Durable event log, decoupling producers/consumers | Low-latency stateful stream processing | AWS-native streaming with serverless consumers | Unified batch + streaming on Spark |

---

## When to Use What

| Scenario | Recommended |
|----------|-------------|
| Need a durable, replayable event log | [[Apache Kafka Fundamentals\|Apache Kafka]] or Kinesis Data Streams |
| Low-latency stateful stream processing | Apache Flink |
| Simple streaming ETL to S3/Redshift on AWS | Kinesis Data Firehose |
| Unified batch and streaming with existing Spark | [[PySpark Streaming\|Spark Structured Streaming]] |
| Serverless event processing on AWS | Kinesis + Lambda |
| Complex event processing with event-time semantics | Apache Flink |
| CDC pipeline from databases | Flink CDC or [[Apache Kafka Fundamentals\|Kafka]] + Debezium |
| Organisation already on AWS, minimal ops overhead | Kinesis ecosystem + Managed Flink |

---

## Related Notes

- [[Stream Processing Theory]] — foundational concepts: watermarks, windowing, exactly-once semantics, the Beam model
- [[Apache Kafka Fundamentals]] — Kafka architecture, producers, consumers, Connect, Schema Registry
- [[PySpark Streaming]] — Spark Structured Streaming with PySpark
- [[AWS Data Services for Data Engineering]] — broader AWS data platform context
- [[Event-Driven Architecture]] — patterns for event-driven systems
