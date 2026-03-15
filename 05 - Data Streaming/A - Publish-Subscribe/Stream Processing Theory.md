# Stream Processing Theory

Core concepts from "Streaming Systems" (Akidau et al.) and "Big Data" (Marz & Warren) — the theoretical foundation for real-time data processing.

## Event Time vs Processing Time

The most fundamental distinction in streaming:

```
Event Time:      When the event actually occurred (embedded in the data)
Processing Time: When the system processes the event (wall clock)

         Event Time
         ├─── Event A (10:00:01) ─── Processed at 10:00:05
         ├─── Event B (10:00:02) ─── Processed at 10:00:15  ← late!
         └─── Event C (10:00:03) ─── Processed at 10:00:04
```

**Processing time ≠ event time** because of network delays, buffering, retries, and out-of-order delivery. A system that uses processing time will produce incorrect results when events arrive late.

## Watermarks

Watermarks track progress in event time — "I believe I have seen all events up to time T."

```
Event Stream: ──A(10:00)──C(10:03)──B(10:02)──D(10:05)──E(10:04)──
Watermark:    ──10:00─────10:02─────10:02─────10:04─────10:04──────
```

### Perfect vs Heuristic Watermarks

| Type | Guarantee | Use Case |
|------|-----------|----------|
| **Perfect** | No late data (all events arrive before watermark advances) | Bounded sources (files, batch) |
| **Heuristic** | Best guess — some events may be late | Unbounded sources (Kafka, IoT) |

**Heuristic watermarks:** Track the oldest unprocessed event time, advance conservatively. Late events either trigger recomputation or are dropped.

## Windowing

Group events into finite chunks for aggregation:

### Window Types

```
Tumbling (Fixed):     |---5min---|---5min---|---5min---|
                      Non-overlapping, no gaps

Sliding (Hopping):    |---5min---|
                         |---5min---|
                            |---5min---|
                      Overlapping, configurable slide interval

Session:              |--gap--|  |---events---|  |--gap--|  |--events--|
                      Dynamic, based on activity gaps

Global:               |─────────────── all events ──────────────────|
                      Single window (entire stream)
```

### Window Configuration

| Parameter | Tumbling | Sliding | Session |
|-----------|----------|---------|---------|
| Window size | Fixed (e.g., 5 min) | Fixed (e.g., 5 min) | Dynamic |
| Slide interval | = window size | < window size (e.g., 1 min) | N/A |
| Gap duration | N/A | N/A | Fixed (e.g., 30 min inactivity) |

## Triggers

Determine **when** window results are emitted:

| Trigger | Fires When | Use Case |
|---------|-----------|----------|
| **Event-time** (watermark) | Watermark passes end of window | Default — emit when "complete" |
| **Processing-time** | Wall clock interval | Periodic partial results |
| **Count** | N elements in window | Every 100 events |
| **Composite** | Combination of above | Early results + final on watermark |

### Early and Late Firings

```
Window: 10:00-10:05

Early firing (processing time, every 1 min):
  10:01 → partial result (3 events)
  10:02 → partial result (7 events)

Watermark firing:
  10:05 → "complete" result (15 events)

Late firing (late event arrives):
  10:07 → updated result (16 events)
```

## Accumulation Modes

What happens when a window fires multiple times:

| Mode | Behavior | Output |
|------|----------|--------|
| **Discarding** | Each firing contains only new data since last firing | Incremental deltas |
| **Accumulating** | Each firing contains the full window result | Running totals |
| **Accumulating & Retracting** | Emits retraction of previous + new result | Correct downstream aggregation |

## Exactly-Once Processing

Three delivery guarantee levels:

| Guarantee | Meaning | Mechanism |
|-----------|---------|-----------|
| **At-most-once** | May lose events | Fire and forget |
| **At-least-once** | May duplicate events | Retry on failure |
| **Exactly-once** | Each event processed exactly once | Idempotent writes + checkpointing |

**Exactly-once in practice:**
- Kafka: Transactional producers + idempotent consumers
- Spark Structured Streaming: Checkpointing + write-ahead logs
- Flink: Distributed snapshots (Chandy-Lamport algorithm)

## State Management

Streaming operations that remember state across events:

| Operation | State Needed | Example |
|-----------|-------------|---------|
| Windowed aggregation | Running counts/sums per window | "Revenue per 5-min window" |
| Joins | Buffer events from both streams | "Enrich orders with customer data" |
| Deduplication | Seen event IDs | "Skip already-processed events" |
| Pattern detection | Event sequences | "Alert on 3 failed logins in 1 min" |

**State backends:** In-memory (fast, lost on failure), RocksDB (persistent, survives restarts), external store (Redis, database).

## Lambda vs Kappa Architecture

### Lambda Architecture

```
                    ┌─── Batch Layer (MapReduce/Spark) ──→ Batch View ──┐
All Data ───┤                                                           ├──→ Query
                    └─── Speed Layer (Storm/Kafka Streams) → Real-time View ─┘
```

- **Batch layer:** Reprocesses all data periodically — complete and accurate
- **Speed layer:** Processes only recent data — fast but approximate
- **Serving layer:** Merges batch + speed views for queries
- **Problem:** Two codebases for the same logic (batch + streaming)

### Kappa Architecture

```
All Data ──→ Stream Processing (Kafka Streams/Flink) ──→ Serving Layer ──→ Query
             (replay from Kafka when needed)
```

- **Single codebase** — all processing is streaming
- **Replay:** Reprocess historical data by replaying Kafka topics
- **Simpler** to maintain, but requires Kafka-like durable log
- **Trade-off:** Replay can be slow for very large datasets

## The Four Questions of Streaming

From the Beam model (Akidau):

1. **What** results are being computed? → Transformations (map, aggregate, join)
2. **Where** in event time? → Windowing (tumbling, sliding, session)
3. **When** in processing time are results materialized? → Triggers + watermarks
4. **How** do refinements relate? → Accumulation (discarding, accumulating, retracting)

Every streaming pipeline answers these four questions, explicitly or implicitly.

## Practical Examples

Hands-on [[Apache Kafka]] code using the `confluent-kafka` Python client. These patterns apply to most publish-subscribe systems with minor adaptation.

### Kafka Producer

```python
from confluent_kafka import Producer
import json
import socket

conf = {
    "bootstrap.servers": "localhost:9092",
    "client.id": socket.gethostname(),
    "acks": "all",                # Wait for all replicas
    "retries": 5,
    "retry.backoff.ms": 300,
    "enable.idempotence": True,   # Exactly-once producer semantics
}

producer = Producer(conf)

def delivery_callback(err, msg):
    """Called once per message to indicate delivery result."""
    if err is not None:
        print(f"Delivery failed for {msg.key()}: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}")

def publish_event(topic: str, key: str, payload: dict) -> None:
    producer.produce(
        topic=topic,
        key=key.encode("utf-8"),
        value=json.dumps(payload).encode("utf-8"),
        callback=delivery_callback,
    )
    producer.poll(0)  # Trigger delivery callbacks

# Flush remaining messages on shutdown
producer.flush(timeout=10)
```

### Kafka Consumer with Offset Management

```python
from confluent_kafka import Consumer, KafkaException
import json

conf = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-processing-group",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,  # Manual offset management
}

consumer = Consumer(conf)
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            raise KafkaException(msg.error())

        payload = json.loads(msg.value().decode("utf-8"))
        process_order(payload)  # Your business logic

        # Commit only after successful processing (at-least-once)
        consumer.commit(message=msg, asynchronous=False)
except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

### Simple Stream Processor

A lightweight pattern that reads from one topic, transforms, and writes to another — the core of [[Lambda vs Kappa Architecture|Kappa architecture]].

```python
from confluent_kafka import Consumer, Producer
import json

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "enrichment-processor",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,
})
producer = Producer({"bootstrap.servers": "localhost:9092", "acks": "all"})

consumer.subscribe(["raw-events"])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None or msg.error():
        continue

    event = json.loads(msg.value().decode("utf-8"))

    # --- Transform / enrich ---
    enriched = {
        **event,
        "processed_at": datetime.utcnow().isoformat(),
        "region": lookup_region(event.get("ip")),
    }

    producer.produce(
        topic="enriched-events",
        key=msg.key(),
        value=json.dumps(enriched).encode("utf-8"),
    )
    producer.poll(0)
    consumer.commit(message=msg, asynchronous=False)
```

### Error Handling Patterns

Robust consumers need strategies for poison pills (unparsable messages) and transient failures.

```python
from confluent_kafka import Consumer, Producer
import json, logging

dead_letter_producer = Producer({"bootstrap.servers": "localhost:9092"})

def handle_message(msg) -> None:
    """Process a single message with dead-letter queue fallback."""
    try:
        payload = json.loads(msg.value().decode("utf-8"))
        process(payload)
    except json.JSONDecodeError:
        # Poison pill — send to dead-letter topic, do not retry
        logging.error(f"Unparsable message at offset {msg.offset()}")
        dead_letter_producer.produce(
            topic="orders.dead-letter",
            key=msg.key(),
            value=msg.value(),
            headers={"error": "json_decode_error"},
        )
    except TransientError:
        # Retriable — raise so the consumer can retry
        raise
    except Exception as exc:
        # Unexpected — log and dead-letter to avoid blocking the consumer
        logging.exception(f"Unexpected error: {exc}")
        dead_letter_producer.produce(
            topic="orders.dead-letter",
            key=msg.key(),
            value=msg.value(),
            headers={"error": str(exc)},
        )
```

| Pattern | When to Use | Mechanism |
|---------|-------------|-----------|
| **Dead-letter queue** | Unparsable or permanently failed messages | Route to a separate topic for investigation |
| **Exponential back-off** | Transient downstream failures (DB timeout) | Retry with increasing delays |
| **Circuit breaker** | Downstream service outage | Stop calling after N failures, recheck periodically |
| **Idempotent writes** | At-least-once delivery causing duplicates | Use a unique event ID as a deduplication key |
