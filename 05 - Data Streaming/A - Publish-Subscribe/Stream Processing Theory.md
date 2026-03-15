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
