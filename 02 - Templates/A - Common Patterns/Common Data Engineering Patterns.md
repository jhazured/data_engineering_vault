Tags: #data-engineering #design-patterns #pipelines #architecture #reference

---

# Common Data Engineering Patterns

This note catalogues reusable design patterns that appear repeatedly across data engineering projects. Each pattern includes a brief description, guidance on when to apply it, illustrative pseudo-code where helpful, and cross-references to vault notes that implement or extend the pattern.

See also: [[Data Ingestion Patterns]], [[ETL Pipeline Templates & Patterns]], [[Data Validation & Quality Frameworks]]

---

## 1. Idempotent Pipelines

**Description:** An idempotent pipeline produces the same result regardless of how many times it is executed with the same input. This property is essential for reliable retry logic, backfill operations, and recovery from partial failures.

**Common Strategies:**

- **Delete-reload** — Drop and fully replace the target partition or table before writing. Simple but can be expensive for large datasets.
- **Upsert (MERGE)** — Insert new rows and update existing ones based on a natural or surrogate key.
- **Immutable partitions** — Write to date- or batch-keyed partitions that are never modified in place; re-running overwrites the entire partition atomically.

**When to Use:**

- Any pipeline that may be retried automatically or manually.
- Backfill and reprocessing workflows.
- Event-driven architectures where duplicate delivery is possible.

**Pseudo-code — Delete-Reload:**

```sql
-- Delete-reload for a daily partition
BEGIN TRANSACTION;

DELETE FROM analytics.page_views
WHERE event_date = '{{ ds }}';

INSERT INTO analytics.page_views
SELECT * FROM staging.page_views
WHERE event_date = '{{ ds }}';

COMMIT;
```

**Pseudo-code — Upsert:**

```sql
MERGE INTO warehouse.customers AS target
USING staging.customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET target.email = source.email,
               target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (customer_id, email, created_at, updated_at)
    VALUES (source.customer_id, source.email, source.created_at, source.updated_at);
```

See also: [[Incremental Loading Strategies]], [[ETL Pipeline Templates & Patterns]]

---

## 2. Staging Pattern

**Description:** Data is first landed in a raw staging area without transformation. It is then validated and cleansed before being promoted to a final, trusted layer. This separation provides an audit trail and isolates failures to the appropriate layer.

**Layers:**

1. **Raw / Landing** — Byte-for-byte copy of the source data. No schema enforcement.
2. **Validated / Cleansed** — Schema applied, data-quality checks run, malformed records routed to a dead letter queue.
3. **Curated / Final** — Business logic applied, data conformed to the target model.

**When to Use:**

- Ingesting from unreliable or schema-variable sources.
- Regulatory environments requiring raw data retention.
- Pipelines where debugging requires access to original payloads.

**Pseudo-code:**

```python
def run_pipeline(batch_date: str) -> None:
    # Step 1 — Land raw data
    raw_df = extract_from_source(batch_date)
    write_to_raw_layer(raw_df, partition=batch_date)

    # Step 2 — Validate
    valid_df, rejected_df = validate(raw_df, schema=SOURCE_SCHEMA)
    write_to_dead_letter(rejected_df, partition=batch_date)

    # Step 3 — Promote
    curated_df = apply_business_logic(valid_df)
    write_to_curated_layer(curated_df, partition=batch_date)
```

See also: [[Data Validation & Quality Frameworks]], [[Data Ingestion Patterns]]

---

## 3. Dead Letter Queue (DLQ)

**Description:** Records that fail parsing, validation, or transformation are routed to a separate store — the dead letter queue — rather than halting the entire pipeline. Engineers can inspect, correct, and replay failed records independently.

**When to Use:**

- Streaming pipelines where halting is unacceptable.
- Batch pipelines processing heterogeneous or user-submitted data.
- Any system where partial success is preferable to total failure.

**Implementation Approaches:**

- A dedicated Kafka topic (e.g. `orders.dlq`) for streaming workloads.
- A partitioned table or object-storage prefix for batch workloads.
- Include original payload, error message, timestamp, and source metadata.

**Pseudo-code:**

```python
for record in incoming_batch:
    try:
        validated = schema.validate(record)
        output_buffer.append(validated)
    except ValidationError as e:
        dead_letter_queue.send(
            payload=record,
            error=str(e),
            source_topic=record.metadata.topic,
            failed_at=datetime.utcnow(),
        )
```

See also: [[Apache Kafka Fundamentals]], [[Data Validation & Quality Frameworks]]

---

## 4. Circuit Breaker

**Description:** Borrowed from electrical engineering and popularised in microservice design, the circuit breaker pattern monitors consecutive failures and "trips" (opens the circuit) when a threshold is reached. This prevents a failing pipeline from overwhelming a downstream system or accumulating costly errors.

**States:**

| State       | Behaviour                                              |
| ----------- | ------------------------------------------------------ |
| **Closed**  | Normal operation; failures are counted.                |
| **Open**    | All requests are immediately rejected or skipped.      |
| **Half-Open** | A limited number of probe requests are allowed through to test recovery. |

**When to Use:**

- Pipelines calling external APIs with rate limits or availability issues.
- Cross-system data loads where the target database may become unresponsive.
- Streaming consumers where repeated deserialization failures indicate a poisoned topic.

**Pseudo-code:**

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout_s: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.reset_timeout_s = reset_timeout_s
        self.state = "CLOSED"
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if self._timeout_elapsed():
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Circuit is open — skipping call")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

See also: [[ETL Pipeline Templates & Patterns]]

---

## 5. Backpressure

**Description:** Backpressure is a flow-control mechanism that slows or pauses a fast producer when the consumer cannot keep up. Without backpressure, unbounded buffering leads to out-of-memory errors, data loss, or cascading failures.

**Techniques:**

- **Bounded queues** — Producer blocks or drops when the queue is full.
- **Rate limiting** — Throttle the producer to a maximum throughput.
- **Consumer-driven pull** — Consumer requests batches at its own pace (e.g. Kafka consumer `poll()`).
- **Reactive streams** — Demand signalling as in Project Reactor or Akka Streams.

**When to Use:**

- Streaming architectures where source throughput is variable.
- Pipelines feeding rate-limited APIs or databases with write throttling.
- Multi-stage pipelines where stages have different processing speeds.

**Pseudo-code — Bounded Queue:**

```python
import queue

buffer = queue.Queue(maxsize=1000)

# Producer — blocks when buffer is full
def produce(records):
    for record in records:
        buffer.put(record, block=True, timeout=30)

# Consumer — processes at its own pace
def consume():
    while True:
        batch = []
        while len(batch) < 100 and not buffer.empty():
            batch.append(buffer.get(timeout=5))
        if batch:
            write_to_sink(batch)
```

See also: [[Apache Kafka Fundamentals]], [[Data Ingestion Patterns]]

---

## 6. Fan-Out / Fan-In

**Description:** A single input is distributed (fanned out) to multiple parallel workers for processing, and the results are then aggregated (fanned in) into a unified output. This pattern maximises throughput for embarrassingly parallel workloads.

**When to Use:**

- Processing large files by splitting into chunks.
- Running the same transformation against multiple partitions or tenants.
- Parallel API calls followed by result consolidation.

**Pseudo-code:**

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def process_chunk(chunk_path: str) -> dict:
    df = read_parquet(chunk_path)
    summary = compute_aggregates(df)
    return summary

# Fan-out
chunk_paths = split_file(source_path, chunk_size_mb=128)

results = []
with ProcessPoolExecutor(max_workers=8) as executor:
    futures = {executor.submit(process_chunk, p): p for p in chunk_paths}

    # Fan-in
    for future in as_completed(futures):
        results.append(future.result())

final_output = merge_summaries(results)
write_output(final_output)
```

See also: [[ETL Pipeline Templates & Patterns]]

---

## 7. Watermark Pattern

**Description:** A watermark tracks how far a pipeline has progressed through its source data. On the next run, only records newer than the watermark are fetched. This enables efficient incremental loads without reprocessing the entire dataset.

**Watermark Storage:**

- A control table in the target database (`pipeline_name`, `last_watermark`, `updated_at`).
- A state file in object storage (JSON or Parquet).
- Orchestrator variables (e.g. Airflow XCom).

**When to Use:**

- Incremental extraction from OLTP databases.
- Streaming systems that need to resume from the last committed offset.
- Any pipeline where full reloads are too expensive.

**Pseudo-code:**

```python
def incremental_load(pipeline_name: str, source_table: str):
    last_watermark = get_watermark(pipeline_name)  # e.g. 2026-03-14T08:00:00Z

    new_records = query_source(
        f"SELECT * FROM {source_table} WHERE updated_at > '{last_watermark}'"
    )

    if new_records.is_empty():
        log.info("No new records — skipping.")
        return

    write_to_target(new_records)

    new_watermark = new_records["updated_at"].max()
    set_watermark(pipeline_name, new_watermark)
```

**Caveats:**

- Ensure the watermark column is indexed at the source.
- Account for clock skew and transaction visibility — consider subtracting a small overlap window.
- Combine with idempotent writes to handle the overlap safely.

See also: [[Incremental Loading Strategies]], [[Data Ingestion Patterns]]

---

## 8. Change Data Capture (CDC)

**Description:** CDC detects and propagates row-level changes (inserts, updates, deletes) from a source system to a target. It is the most efficient way to keep a replica or data warehouse in near-real-time sync with an operational database.

**Approaches:**

| Approach           | Mechanism                                      | Pros                              | Cons                                  |
| ------------------ | ---------------------------------------------- | --------------------------------- | ------------------------------------- |
| **Log-based**      | Read the database transaction log (WAL / binlog) | Low overhead, captures deletes    | Requires database-level access        |
| **Timestamp-based**| Query rows where `updated_at > last_watermark`  | Simple, no special permissions    | Misses deletes, clock-skew risk       |
| **Trigger-based**  | Database triggers write changes to an audit table | Captures all DML                  | Adds write overhead to the source     |

**When to Use:**

- Replicating OLTP data to an analytical warehouse.
- Feeding real-time event streams from databases.
- Maintaining materialised views across system boundaries.

**Log-based CDC with Debezium (Conceptual):**

```yaml
# Debezium connector configuration (simplified)
connector.class: io.debezium.connector.postgresql.PostgresConnector
database.hostname: source-db.internal
database.dbname: orders_db
table.include.list: public.orders,public.order_items
topic.prefix: cdc.orders
snapshot.mode: initial
```

Changes are emitted as structured events to Kafka topics, which downstream consumers process as inserts, updates, or deletes.

See also: [[Apache Kafka Fundamentals]], [[Incremental Loading Strategies]], [[Data Ingestion Patterns]]

---

## 9. Slowly Changing Dimensions (SCD)

**Description:** SCD patterns govern how dimension tables in a data warehouse handle changes to attribute values over time. The three most common types are:

### Type 1 — Overwrite

The current value is overwritten with the new value. No history is retained.

```sql
UPDATE dim_customer
SET city = 'Manchester'
WHERE customer_id = 42;
```

**Use when:** Historical accuracy of the attribute is not required.

### Type 2 — Full History

A new row is inserted for each change, with `effective_from`, `effective_to`, and `is_current` columns preserving the full history.

```sql
-- Close the current record
UPDATE dim_customer
SET effective_to = CURRENT_DATE - INTERVAL '1 day',
    is_current = FALSE
WHERE customer_id = 42 AND is_current = TRUE;

-- Insert the new version
INSERT INTO dim_customer
    (customer_id, city, effective_from, effective_to, is_current)
VALUES
    (42, 'Manchester', CURRENT_DATE, '9999-12-31', TRUE);
```

**Use when:** You need to analyse metrics as they were at a point in time (e.g. revenue by customer region at time of sale).

### Type 3 — Previous Value Column

An additional column stores the prior value. Only one level of history is retained.

```sql
ALTER TABLE dim_customer ADD COLUMN previous_city VARCHAR(100);

UPDATE dim_customer
SET previous_city = city,
    city = 'Manchester'
WHERE customer_id = 42;
```

**Use when:** You need limited before/after comparison without full versioning.

See also: [[SCD Type 2 Patterns]], [[ETL Pipeline Templates & Patterns]]

---

## 10. Retry with Exponential Backoff

**Description:** When a transient failure occurs (network timeout, rate-limit response, temporary unavailability), the operation is retried after a progressively longer delay. Adding random jitter prevents multiple clients from retrying in lockstep (the "thundering herd" problem).

**When to Use:**

- API calls that may return 429 (Too Many Requests) or 503 (Service Unavailable).
- Database connections that occasionally fail under load.
- Any operation where the failure is likely temporary.

**Pseudo-code:**

```python
import random
import time

def retry_with_backoff(func, max_retries: int = 5, base_delay_s: float = 1.0):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise

            delay = base_delay_s * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.5)
            total_wait = delay + jitter

            log.warning(
                f"Attempt {attempt + 1} failed: {e}. "
                f"Retrying in {total_wait:.1f}s..."
            )
            time.sleep(total_wait)
```

**Key Considerations:**

- Set a maximum retry count to avoid infinite loops.
- Distinguish transient errors (retry) from permanent errors (fail immediately).
- Log every retry with context for observability.
- Consider a circuit breaker (see section 4) if failures persist beyond the retry budget.

See also: [[ETL Pipeline Templates & Patterns]], [[Data Ingestion Patterns]]

---

## 11. Schema-on-Read vs Schema-on-Write

**Description:** These two strategies represent opposite ends of the spectrum for when and how schema is enforced.

| Aspect                | Schema-on-Write                            | Schema-on-Read                              |
| --------------------- | ------------------------------------------ | ------------------------------------------- |
| **Schema enforcement**| At write time (before data is stored)      | At read time (when data is queried)         |
| **Storage format**    | Structured (relational tables, Avro, Parquet) | Semi-structured (JSON, CSV, raw logs)       |
| **Data quality**      | High — invalid data is rejected early      | Variable — quality depends on the reader    |
| **Flexibility**       | Low — schema changes require migration     | High — new fields are added without migration |
| **Query performance** | Generally faster (optimised storage)       | Generally slower (parsing at query time)    |
| **Typical tools**     | RDBMS, data warehouses, Delta Lake         | Data lakes, Hadoop, Athena over S3          |

**When to Choose Schema-on-Write:**

- Analytical workloads with well-understood, stable schemas.
- Regulatory or financial data where quality guarantees are mandatory.
- High-frequency queries where read performance is critical.

**When to Choose Schema-on-Read:**

- Exploratory or data-science workloads where the schema is evolving.
- Ingesting diverse source formats that cannot be normalised upfront.
- Landing zones in a data lake where raw preservation is the priority.

**Hybrid Approach (Recommended):**

Most modern architectures use both — schema-on-read in the raw/landing layer and schema-on-write in the curated/serving layer. This is the staging pattern (section 2) applied at an architectural level.

See also: [[Data Validation & Quality Frameworks]], [[Data Ingestion Patterns]]

---

## 12. Late-Arriving Data

**Description:** In many systems, records arrive after the processing window for their event time has closed. Without accommodation, these records are silently dropped or misattributed to the wrong period.

**Handling Strategies:**

- **Reprocessing windows** — Re-run the pipeline for the affected partition when late data is detected. Works well with idempotent pipelines (section 1).
- **Grace periods** — Keep partitions open for a configurable window (e.g. 72 hours) before finalising them. Common in streaming frameworks like Apache Flink.
- **Append-and-reconcile** — Append late records to a corrections table and periodically merge them into the main dataset.
- **Out-of-order tolerant aggregations** — Use event-time windowing with allowed lateness rather than processing-time windowing.

**When to Use:**

- Event-driven architectures where mobile or IoT devices submit data with delays.
- Cross-timezone batch pipelines where source-system extracts arrive at different times.
- Any pipeline where the `event_time` can differ significantly from the `ingestion_time`.

**Pseudo-code — Grace Period in Streaming:**

```python
# Apache Flink-style windowing (pseudo-code)
stream = (
    events
    .assign_timestamps_and_watermarks(
        max_out_of_orderness=timedelta(minutes=10)
    )
    .key_by("user_id")
    .window(TumblingEventTimeWindow.of(timedelta(hours=1)))
    .allowed_lateness(timedelta(hours=24))
    .aggregate(CountAggregator())
)
```

**Pseudo-code — Reprocessing Window:**

```python
def handle_late_data(record: dict) -> None:
    event_date = record["event_time"].date()
    target_partition = get_partition(event_date)

    if target_partition.is_finalised():
        mark_partition_for_reprocessing(event_date)
        write_to_late_arrivals_buffer(record)
    else:
        write_to_partition(record, event_date)
```

See also: [[Incremental Loading Strategies]], [[Apache Kafka Fundamentals]], [[ETL Pipeline Templates & Patterns]]

---

## Quick Reference Matrix

| Pattern                        | Primary Benefit              | Complexity | Key Vault References                                          |
| ------------------------------ | ---------------------------- | ---------- | ------------------------------------------------------------- |
| Idempotent Pipelines           | Safe retries and backfills   | Low        | [[Incremental Loading Strategies]]                            |
| Staging Pattern                | Audit trail, failure isolation | Low      | [[Data Validation & Quality Frameworks]]                      |
| Dead Letter Queue              | Partial success over total failure | Medium | [[Apache Kafka Fundamentals]]                                |
| Circuit Breaker                | Protect downstream systems   | Medium     | [[ETL Pipeline Templates & Patterns]]                         |
| Backpressure                   | Flow control                 | Medium     | [[Apache Kafka Fundamentals]]                                 |
| Fan-Out / Fan-In               | Parallelism                  | Medium     | [[ETL Pipeline Templates & Patterns]]                         |
| Watermark                      | Efficient incremental loads  | Low        | [[Incremental Loading Strategies]]                            |
| CDC                            | Near-real-time replication   | High       | [[Apache Kafka Fundamentals]], [[Data Ingestion Patterns]]    |
| SCD                            | Dimensional history tracking | Medium     | [[SCD Type 2 Patterns]]                                       |
| Retry with Exponential Backoff | Transient failure resilience | Low        | [[Data Ingestion Patterns]]                                   |
| Schema-on-Read vs Write        | Flexibility vs quality       | Low        | [[Data Validation & Quality Frameworks]]                      |
| Late-Arriving Data             | Completeness guarantees      | High       | [[Incremental Loading Strategies]]                            |

---

## Further Reading

- [[Data Ingestion Patterns]] — Detailed ingestion architectures that build on these patterns.
- [[ETL Pipeline Templates & Patterns]] — Ready-to-use pipeline templates incorporating idempotency, staging, and retry logic.
- [[SCD Type 2 Patterns]] — Deep dive into Type 2 implementation variants.
- [[Data Validation & Quality Frameworks]] — Frameworks for the validation layer in the staging pattern.
- [[Apache Kafka Fundamentals]] — Streaming platform underpinning DLQ, backpressure, and CDC patterns.
- [[Incremental Loading Strategies]] — Watermark and late-arriving data strategies in practice.
