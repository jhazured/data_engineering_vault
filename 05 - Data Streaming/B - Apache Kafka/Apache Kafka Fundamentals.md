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
