# Apache Kafka Fundamentals

A distributed event streaming platform for high-throughput, fault-tolerant, real-time data pipelines and event-driven architectures.

## Core Concepts

### Topics, Partitions & Offsets

```
Topic: "shipments"
├── Partition 0: [offset 0] [offset 1] [offset 2] [offset 3] ...
├── Partition 1: [offset 0] [offset 1] [offset 2] ...
└── Partition 2: [offset 0] [offset 1] [offset 2] [offset 3] [offset 4] ...
```

- **Topic**: Named feed of messages (like a table in a database)
- **Partition**: Ordered, immutable sequence of messages within a topic — unit of parallelism
- **Offset**: Sequential ID within a partition — uniquely identifies each message
- **Key**: Optional — determines which partition a message goes to (same key → same partition → ordering guarantee)

### Messages

```
Message = {
    key: "CUST_001",              # Partition routing (optional)
    value: '{"event": "order"}',   # Payload (bytes)
    timestamp: 1710000000000,      # Event time or ingestion time
    headers: {"source": "api"}     # Metadata (optional)
}
```

## Producers

### Key Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `acks` | `1` | Durability guarantee (0=fire-and-forget, 1=leader-ack, all=full ISR ack) |
| `retries` | `2147483647` | Retry count on transient failures |
| `batch.size` | `16384` | Bytes to batch before sending |
| `linger.ms` | `0` | Wait time to fill batch (higher = more batching, more latency) |
| `compression.type` | `none` | `snappy` (fast), `gzip` (better ratio), `lz4`, `zstd` |
| `enable.idempotence` | `true` (Kafka 3.0+) | Prevents duplicate messages on retry |

### Partitioning Strategy

```
Key present     → hash(key) % num_partitions  → deterministic partition
Key absent      → round-robin across partitions
Custom          → implement Partitioner interface
```

**Same key always goes to the same partition** — this guarantees ordering per key (e.g., all events for `CUST_001` are ordered).

### Delivery Guarantees (Producer Side)

| `acks` | Guarantee | Trade-off |
|--------|-----------|-----------|
| `0` | Fire-and-forget | Fastest, may lose messages |
| `1` | Leader acknowledged | Good balance — message survives leader crash only if replicated in time |
| `all` | All ISR replicas acknowledged | Strongest — no data loss if ISR > 1 |

**For reliable delivery:** `acks=all` + `enable.idempotence=true` + `min.insync.replicas=2`

## Consumers

### Consumer Groups

```
Topic: "shipments" (3 partitions)

Consumer Group A:
  Consumer 1 → Partition 0, Partition 1
  Consumer 2 → Partition 2

Consumer Group B (independent):
  Consumer 3 → Partition 0, Partition 1, Partition 2
```

- Each partition is consumed by **exactly one consumer** within a group
- Multiple groups read the same topic independently (pub/sub pattern)
- Adding consumers to a group triggers **rebalancing** — partitions are redistributed
- Max useful consumers = number of partitions (extras sit idle)

### Offset Management

```
Partition 0: [0] [1] [2] [3] [4] [5] [6] [7]
                          ↑              ↑
                   committed offset   current position
                   (last processed)   (being read)
```

- **Auto-commit** (`enable.auto.commit=true`): Offsets committed periodically — risk of reprocessing after crash
- **Manual commit** (`commitSync()` / `commitAsync()`): Application controls when offset is committed — enables exactly-once processing

### Rebalancing

Triggered when:
- Consumer joins or leaves the group
- New partitions added to the topic
- Consumer fails heartbeat check

**During rebalancing, consumption pauses.** Minimize by:
- Using `static.group.instance.id` for stable assignments
- Keeping processing time < `max.poll.interval.ms`
- Using cooperative rebalancing (`partition.assignment.strategy=cooperative-sticky`)

## Broker Architecture

### Replication

```
Topic "shipments", Partition 0, replication.factor=3:
  Broker 1: Leader    ← producers write here, consumers read here
  Broker 2: Follower  (ISR)  ← replicates from leader
  Broker 3: Follower  (ISR)  ← replicates from leader
```

- **Leader**: Handles all reads and writes for a partition
- **Followers**: Replicate from leader, join the ISR (In-Sync Replica set)
- **ISR**: Followers that are caught up — eligible to become leader on failure
- **`min.insync.replicas`**: Minimum ISR count required for `acks=all` writes to succeed

### Leader Election

When a leader broker fails:
1. Controller detects failure via ZooKeeper/KRaft
2. An ISR follower is elected as new leader
3. Producers and consumers redirect to new leader
4. **No data loss** if `acks=all` + `min.insync.replicas >= 2`

## Kafka Connect

Pre-built connectors for integrating Kafka with external systems:

```
Source Connectors (into Kafka):
  Database CDC → Kafka topic (Debezium)
  File system  → Kafka topic
  S3           → Kafka topic

Sink Connectors (out of Kafka):
  Kafka topic → Snowflake
  Kafka topic → S3/GCS
  Kafka topic → Elasticsearch
```

### Schema Registry

Centralized schema management for producers/consumers:
- Enforces schema compatibility (backward, forward, full)
- Supports Avro, Protobuf, JSON Schema
- Prevents breaking changes from reaching consumers

## Data Pipeline Patterns

### Event Streaming Pipeline

```
Source DB → CDC (Debezium) → Kafka → Stream Processing → Kafka → Sink (Snowflake)
```

### Log Aggregation

```
App Server 1 ─┐
App Server 2 ──┼─→ Kafka Topic "logs" → Elasticsearch / S3
App Server 3 ─┘
```

### Event Sourcing with Kafka

Kafka as the immutable event log — the source of truth:

```
Events (immutable):
  OrderCreated → OrderShipped → OrderDelivered

Materialized Views (derived, rebuildable):
  Current order status table
  Customer order count aggregate
```

## Key Sizing Decisions

| Decision | Guidance |
|----------|----------|
| **Partitions per topic** | Start with `max(throughput_MB/s, consumer_count)`. More partitions = more parallelism but more overhead |
| **Replication factor** | 3 for production (tolerates 1 broker failure with `min.insync.replicas=2`) |
| **Retention** | `retention.ms` (time) or `retention.bytes` (size) — balance storage vs replay capability |
| **Segment size** | `segment.bytes` — smaller = faster cleanup, larger = fewer files |
| **Compression** | `snappy` for speed, `zstd` for best ratio |

## Kafka vs Alternatives

| Feature | Kafka | RabbitMQ | AWS Kinesis | Pulsar |
|---------|-------|----------|-------------|--------|
| Model | Log-based | Queue-based | Log-based | Log-based |
| Ordering | Per-partition | Per-queue | Per-shard | Per-partition |
| Replay | Yes (retention) | No (consumed = gone) | Yes (24h-365d) | Yes |
| Throughput | Very high | Moderate | High | Very high |
| Managed | Confluent Cloud, MSK | CloudAMQP | Native AWS | StreamNative |

## Schema Registry Architecture

The [[Schema Registry]] is a separate service (not part of core Apache Kafka) that provides centralised schema management. The most widely used implementation is Confluent Schema Registry.

### How It Works

```
Producer                         Schema Registry                    Consumer
   │                                  │                                │
   ├── 1. Register schema ──────────► │                                │
   │◄── 2. Return schema ID ──────── │                                │
   │                                  │                                │
   ├── 3. Serialize: [schema_id | payload bytes] ──► Kafka topic       │
   │                                  │                                │
   │                                  │    4. Consumer reads record ──►│
   │                                  │◄── 5. Fetch schema by ID ──── │
   │                                  │──── 6. Return schema ────────►│
   │                                  │              Deserialize ──── │
```

- Schemas are stored in the registry, not in each message — only a compact **schema ID** (typically 4 bytes) is embedded in the record
- Serialisers and deserialisers handle registration and lookup transparently — application code does not interact with the registry directly
- The registry caches schemas locally after the first fetch, so network overhead is minimal

### Supported Formats

| Format | Strengths | Typical Use |
|--------|-----------|-------------|
| **Avro** | Compact binary encoding, strong schema evolution, backward/forward compatibility built in | Most common for Kafka; recommended default |
| **Protobuf** | Language-neutral IDL, efficient binary format, widely used in gRPC ecosystems | Cross-platform services, gRPC integration |
| **JSON Schema** | Human-readable payloads, easy debugging, broad tooling support | Lightweight integrations, REST-adjacent systems |

### Schema Compatibility Modes

| Mode | Rule | Use Case |
|------|------|----------|
| **BACKWARD** | New schema can read data written with the previous schema | Consumers upgraded before producers (default) |
| **FORWARD** | Previous schema can read data written with the new schema | Producers upgraded before consumers |
| **FULL** | Both backward and forward compatible | Independent upgrades in any order |
| **NONE** | No compatibility check | Development/testing only |

**Safe evolution patterns**: adding optional fields with defaults (backward compatible), removing optional fields (forward compatible). Renaming or changing field types breaks compatibility.

### Integration with Kafka Connect

Kafka Connect uses **converters** to serialise/deserialise between Connect data objects and Kafka records. Key settings:

```properties
key.converter=io.confluent.connect.avro.AvroConverter
value.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter.schema.registry.url=http://schema-registry:8081
```

This allows connectors to produce and consume schema-aware records without custom serialisation logic.

## Delivery Guarantee Comparison

| Guarantee | Producer Config | Consumer Behaviour | Trade-off | When to Use |
|-----------|----------------|--------------------|-----------|-------------|
| **At-most-once** | `acks=0`, no retries | Commit offset before processing | No duplicates, but messages may be lost | Metrics, logging where some loss is acceptable |
| **At-least-once** | `acks=all`, retries enabled | Commit offset after processing | No data loss, but duplicates possible on retry | Default for most pipelines; combine with idempotent sinks |
| **Exactly-once** | `acks=all` + `enable.idempotence=true` + transactional API | Transactional consumer (`isolation.level=read_committed`) | Strongest guarantee, moderate overhead | Financial transactions, deduplication-sensitive workloads |

### End-to-End Exactly-Once Strategies

Kafka alone provides at-least-once. Achieving exactly-once end-to-end requires one of:

1. **Idempotent writes to the sink** — write with a unique key (topic + partition + offset); duplicates overwrite with the same value
2. **Transactional offset + data commits** — store offsets and processed results in the same external transaction (e.g., database), then use `consumer.seek()` on restart to resume from the stored offset
3. **Kafka Transactions (Kafka 0.11+)** — the transactional producer atomically writes to output topics and commits consumer offsets in a single transaction (used by [[Kafka Streams]])

## Producer Reliability Configuration

### The Reliable Producer Recipe

```properties
acks=all                              # Wait for all ISR replicas
enable.idempotence=true               # Deduplicate retries at the broker (sequence numbers)
max.in.flight.requests.per.connection=5  # Max with idempotence (broker reorders if needed)
retries=2147483647                    # Effectively infinite — let the producer keep trying
delivery.timeout.ms=120000            # Upper bound on total delivery time (retries included)
min.insync.replicas=2                 # Broker-side: require at least 2 ISR for acks=all
```

### How Idempotent Producers Work

When `enable.idempotence=true`, the broker assigns each producer a **Producer ID (PID)** and tracks a **sequence number** per partition. If a retry arrives with a sequence number the broker has already seen, the duplicate is silently discarded. This prevents duplicates caused by network-level retries without any application-level deduplication.

**Constraints when idempotence is enabled:**
- `acks` must be `all`
- `max.in.flight.requests.per.connection` must be 5 or fewer
- `retries` must be greater than 0

### Retry Behaviour and Ordering

| Scenario | `max.in.flight` | Risk | Mitigation |
|----------|-----------------|------|------------|
| Retries with multiple in-flight batches | > 1 | Out-of-order writes if batch N fails but batch N+1 succeeds | Enable idempotence (handles reordering) or set `max.in.flight=1` |
| Retries disabled | N/A | Data loss on transient errors | Always enable retries in reliable systems |
| Idempotence enabled | <= 5 | None — broker deduplicates and reorders | Recommended default |

### Error Handling Strategy

- **Retriable errors** (e.g., `LEADER_NOT_AVAILABLE`, network timeouts): let the producer retry automatically
- **Non-retriable errors** (e.g., `INVALID_CONFIG`, serialisation failures): handle in the error callback — log, send to a dead-letter topic, or alert
- **Timeout exhaustion** (`delivery.timeout.ms` exceeded): the producer gives up — route the failed record to a fallback path

## Consumer Reliability Patterns

### Offset Commit Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **Auto-commit** | `enable.auto.commit=true`, committed every `auto.commit.interval.ms` (default 5s) | Simple but risks reprocessing if consumer crashes between commits |
| **Sync commit per batch** | `commitSync()` at end of poll loop | Blocks until broker confirms; strongest guarantee, higher latency |
| **Async commit with sync fallback** | `commitAsync()` in the loop, `commitSync()` in `finally` block on shutdown | Good balance — async for throughput, sync for clean shutdown |
| **Commit per record** | `commitSync(offsets)` after each record | Minimises reprocessing window; highest overhead |
| **External offset storage** | Store offsets in the same database transaction as processed results; use `consumer.seek()` on startup | Enables true exactly-once when combined with transactional sinks |

### Rebalance Listeners

Implement `ConsumerRebalanceListener` to handle partition reassignment safely:

- **`onPartitionsRevoked()`** — called before partitions are taken away; commit current offsets and flush any in-progress work
- **`onPartitionsAssigned()`** — called after new partitions are assigned; restore state or seek to the correct offset (e.g., from an external store)

```
# Pattern: external offset storage with rebalance listener
onPartitionsRevoked  → commitDBTransaction()
onPartitionsAssigned → consumer.seek(partition, getOffsetFromDB(partition))
```

This pattern ensures that on rebalance, the new consumer owner starts from the exact offset stored in the database rather than the last Kafka-committed offset.

### Consumer Retry Patterns

When a record fails processing (e.g., downstream database unavailable):

1. **Buffer and retry** — commit the last successfully processed offset, buffer failed records, `pause()` the consumer to prevent new fetches, retry the buffer, then `resume()`
2. **Dead-letter topic** — write the failed record to a separate retry/DLQ topic, commit the offset, and continue; a dedicated consumer group handles retries from the DLQ

### Important Consumer Configuration for Reliability

| Parameter | Purpose | Guidance |
|-----------|---------|----------|
| `group.id` | Consumer group membership | Unique per logical application; same ID = shared consumption |
| `auto.offset.reset` | Behaviour when no committed offset exists | `earliest` = replay from start (safe, may reprocess); `latest` = skip to end (may miss messages) |
| `enable.auto.commit` | Automatic offset commits | Set `false` for exactly-once or fine-grained control |
| `max.poll.interval.ms` | Max time between poll calls before consumer is considered dead | Increase if processing is slow; exceeding triggers rebalance |
| `isolation.level` | Transactional read visibility | `read_committed` = only see committed transactional records |

## Backpressure Handling

Kafka's architecture provides **natural backpressure** — Kafka acts as a durable buffer between producers and consumers, decoupling their throughput. Producers do not need to slow down when consumers fall behind; data accumulates in Kafka until consumers catch up.

### Consumer Lag Monitoring

**Consumer lag** is the difference between the latest produced offset and the last committed consumer offset. It is the single most important consumer health metric.

```
Partition 0:  [0] [1] [2] ... [98] [99] [100]
                                          ↑ latest produced
                              ↑ last committed by consumer
                              lag = 10 messages
```

**Monitoring approaches:**
- **Kafka consumer metrics**: `records-lag-max` — reported by the consumer itself, but unavailable if the consumer is offline
- **External lag monitoring** (preferred): tools like Burrow or custom scripts that compare broker partition offsets with consumer group committed offsets independently of the consumer process
- **Alerts**: set thresholds on sustained lag growth rather than absolute lag (which fluctuates naturally)

### Pause/Resume for Flow Control

When processing takes longer than the poll interval, use `consumer.pause()` and `consumer.resume()` to control fetching without triggering a rebalance:

```
1. poll() returns batch of records
2. Hand records to worker thread pool
3. consumer.pause(partitions)       # Stop fetching new data
4. Continue calling poll()           # Sends heartbeats, returns no records
5. Worker threads complete
6. consumer.resume(partitions)       # Resume fetching
```

This keeps the consumer alive (heartbeats continue) whilst preventing unbounded memory growth from unconsumed records.

### Scaling Consumers to Reduce Lag

- Add consumers to the group (up to the number of partitions) — partitions are redistributed automatically
- If lag persists at max consumers, increase the partition count on the topic to allow greater parallelism
- Tune `fetch.min.bytes` and `fetch.max.wait.ms` to optimise batch sizes for throughput vs latency

---

## Kafka Connect Framework

Kafka Connect is a scalable, fault-tolerant framework for streaming data between Kafka and external systems without writing custom producer/consumer code. It handles serialisation, offset management, fault tolerance, and parallelism out of the box.

### Architecture

```
                    Kafka Connect Cluster
┌──────────────────────────────────────────────────────┐
│  Worker 1                    Worker 2                 │
│  ┌─────────────────────┐     ┌─────────────────────┐ │
│  │ Connector Instance  │     │ Connector Instance  │ │
│  │  ├── Task 0         │     │  ├── Task 2         │ │
│  │  └── Task 1         │     │  └── Task 3         │ │
│  │                     │     │                     │ │
│  │ Converter (Avro)    │     │ Converter (Avro)    │ │
│  │ Transform (SMT)     │     │ Transform (SMT)     │ │
│  └─────────────────────┘     └─────────────────────┘ │
└──────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
    Kafka Brokers (data + offsets + config topics)
```

**Core components:**

| Component | Role |
|-----------|------|
| **Worker** | JVM process that executes connectors and tasks. Workers form a Connect cluster and distribute work |
| **Connector** | Plugin that defines how to interact with an external system. Manages task lifecycle and configuration |
| **Task** | The unit of parallelism. Each task handles a subset of the data (e.g., one table, one topic partition) |
| **Converter** | Serialises/deserialises between Connect's internal data format and Kafka record bytes (Avro, JSON, Protobuf) |
| **Transform (SMT)** | Single Message Transform — lightweight per-record transformations applied in the connector pipeline |

### Source vs Sink Connectors

```
Source Connector:                    Sink Connector:
External System → Kafka              Kafka → External System

┌──────────┐    ┌─────┐             ┌─────┐    ┌──────────┐
│ Database │──► │Kafka│             │Kafka│──► │    S3    │
│ (CDC)    │    │Topic│             │Topic│    │  Bucket  │
└──────────┘    └─────┘             └─────┘    └──────────┘

Source manages:                     Sink manages:
- Polling/CDC from source           - Consuming from Kafka
- Source offsets (position tracking) - Consumer offsets (automatic)
- Schema discovery                  - Writing to external system
- Partitioning across tasks         - Delivery guarantees
```

### Common Connectors

| Connector | Type | Description | Common Use |
|-----------|------|-------------|------------|
| **Debezium** | Source | CDC from databases (Postgres, MySQL, SQL Server, Oracle, MongoDB) | Real-time change capture for event-driven architectures |
| **JDBC Source** | Source | Poll-based reads from any JDBC database | Incremental loads from OLTP databases |
| **JDBC Sink** | Sink | Write to any JDBC database | Stream processing results to operational stores |
| **S3 Sink** | Sink | Write to AWS S3 in Parquet, Avro, JSON, or CSV | Data lake ingestion from Kafka |
| **GCS Sink** | Sink | Write to Google Cloud Storage | GCP data lake ingestion |
| **Elasticsearch Sink** | Sink | Index documents in Elasticsearch/OpenSearch | Real-time search indexing |
| **BigQuery Sink** | Sink | Stream records to BigQuery | GCP analytics pipeline |
| **Snowflake Sink** | Sink | Load data into Snowflake via internal stage | Warehouse ingestion |
| **HTTP Source/Sink** | Both | REST API integration | Webhook ingestion, API-based sinks |
| **FileStream** | Both | Read/write local files | Development and testing only |

### Distributed vs Standalone Mode

| Aspect | Distributed Mode | Standalone Mode |
|--------|-----------------|-----------------|
| **Workers** | Multiple (cluster) | Single process |
| **Fault tolerance** | Tasks redistributed on worker failure | No failover |
| **Configuration** | REST API (dynamic) | Properties file (static) |
| **Offset storage** | Kafka topics (`connect-offsets`, `connect-configs`, `connect-status`) | Local file |
| **Scaling** | Add/remove workers dynamically | Cannot scale |
| **Use case** | Production | Development, edge deployments |

### Connector Configuration Examples

```json
{
  "name": "postgres-cdc-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres.internal",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${secrets:postgres/password}",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    "schema.include.list": "public",
    "table.include.list": "public.orders,public.customers",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_pub",
    "topic.prefix": "cdc.orders",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
  }
}
```

```json
{
  "name": "s3-sink-parquet",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "topics": "cdc.orders.public.orders",
    "s3.bucket.name": "data-lake-raw",
    "s3.region": "eu-west-2",
    "s3.part.size": "5242880",
    "flush.size": "10000",
    "rotate.interval.ms": "3600000",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "partition.duration.ms": "3600000",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
    "locale": "en-GB",
    "timezone": "UTC",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081"
  }
}
```

### Single Message Transforms (SMTs)

SMTs apply lightweight per-record transformations within the connector pipeline — no need for a separate stream processing application.

| SMT | Purpose | Example |
|-----|---------|---------|
| **InsertField** | Add fields to records | Add ingestion timestamp, static metadata |
| **ReplaceField** | Rename, include, or exclude fields | Drop sensitive columns before sink |
| **MaskField** | Mask field values | Mask PII fields (e.g., email, phone) |
| **ExtractField** | Extract a field from a struct | Pull the `after` state from a Debezium envelope |
| **TimestampConverter** | Convert timestamp formats | Unix epoch to ISO string |
| **ValueToKey** | Set the record key from value fields | Use a business key for partitioning |
| **RegexRouter** | Rename topics via regex | Route `cdc.orders.public.orders` to `orders-raw` |
| **Filter** | Drop records matching a predicate | Filter test/internal records |
| **HeaderFrom** | Copy value fields to headers | Route metadata without modifying the payload |

```json
{
  "transforms": "extractState,addTimestamp,maskEmail",
  "transforms.extractState.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.addTimestamp.timestamp.field": "ingested_at",
  "transforms.maskEmail.type": "org.apache.kafka.connect.transforms.MaskField$Value",
  "transforms.maskEmail.fields": "email",
  "transforms.maskEmail.replacement": "***@***.***"
}
```

**Limitations of SMTs:** they operate on one record at a time. For joins, aggregations, or stateful processing, use [[Kafka Streams]] or a dedicated stream processor.

### Dead-Letter Queues

When a connector encounters a record it cannot process (deserialisation error, schema mismatch, sink write failure), a dead-letter queue (DLQ) captures the failed record instead of halting the connector.

```json
{
  "errors.tolerance": "all",
  "errors.deadletterqueue.topic.name": "dlq-s3-sink",
  "errors.deadletterqueue.topic.replication.factor": 3,
  "errors.deadletterqueue.context.headers.enable": true,
  "errors.log.enable": true,
  "errors.log.include.messages": true
}
```

| Setting | Purpose |
|---------|---------|
| `errors.tolerance=all` | Continue processing after errors (default `none` = fail immediately) |
| `errors.deadletterqueue.topic.name` | Topic to send failed records to |
| `errors.deadletterqueue.context.headers.enable` | Attach error context (exception, stack trace) as record headers |
| `errors.log.enable` | Log errors to the Connect worker log |

**DLQ processing pattern:** a separate consumer group reads the DLQ topic, inspects errors, and either fixes and replays records or alerts the operations team.

### Monitoring

**JMX Metrics (built-in):**

| Metric Group | Key Metrics |
|-------------|-------------|
| **Connector** | `connector-status`, `connector-type`, `connector-class` |
| **Task** | `source-record-poll-rate`, `sink-record-send-rate`, `offset-commit-success-percentage` |
| **Worker** | `connector-count`, `task-count`, `connector-startup-attempts-total` |
| **Error** | `total-errors-logged`, `total-records-skipped`, `deadletterqueue-produce-requests` |

**Prometheus integration:**

```yaml
# docker-compose excerpt for JMX exporter
KAFKA_CONNECT_OPTS: >-
  -javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx-exporter/connect.yml

# Prometheus scrape config
scrape_configs:
  - job_name: kafka-connect
    static_configs:
      - targets: ["connect-worker-1:9404", "connect-worker-2:9404"]
```

**Key alerts to configure:**

- Connector status changed to `FAILED` or `PAUSED`
- Task status changed to `FAILED`
- Consumer lag on Connect consumer group growing
- DLQ topic receiving records (non-zero produce rate)
- Offset commit failure rate above threshold

### REST API Operations

```bash
# List all connectors
GET /connectors

# Get connector status
GET /connectors/postgres-cdc-source/status

# Create or update a connector
PUT /connectors/postgres-cdc-source/config

# Pause / resume / restart
PUT /connectors/postgres-cdc-source/pause
PUT /connectors/postgres-cdc-source/resume
POST /connectors/postgres-cdc-source/restart

# Restart a specific failed task
POST /connectors/postgres-cdc-source/tasks/0/restart

# Delete a connector
DELETE /connectors/postgres-cdc-source
```

### Connect vs Custom Consumers

| Aspect | Kafka Connect | Custom Consumer Application |
|--------|--------------|---------------------------|
| **Development effort** | Configuration only (JSON) | Full application code |
| **Offset management** | Automatic (framework-managed) | Manual (commit strategies) |
| **Fault tolerance** | Built-in (task redistribution) | Must implement yourself |
| **Scaling** | Add workers + increase `tasks.max` | Add consumer instances |
| **Schema evolution** | Handled by converters + Schema Registry | Must implement yourself |
| **Transformations** | SMTs (limited to per-record) | Unlimited (joins, aggregations, state) |
| **Monitoring** | Built-in JMX metrics | Must instrument yourself |
| **Connector ecosystem** | 200+ pre-built connectors | Write from scratch |
| **Flexibility** | Limited to connector capabilities | Complete control |
| **When to use** | Standard integrations (DB→Kafka→S3) | Complex processing, custom protocols, non-standard systems |

**Rule of thumb:** use Connect for standard data integration (the plumbing). Use custom consumers or stream processors for business logic that requires stateful processing, joins, or complex transformations
