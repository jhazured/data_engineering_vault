#gcp #dataflow #apache-beam #streaming #batch #data-engineering

# GCP Dataflow & Apache Beam

Apache Beam is a unified programming model for batch and streaming data processing. Google Cloud Dataflow is a fully managed runner for Beam pipelines. This note covers the Beam model, Python SDK patterns, Dataflow-specific features, and comparisons with Apache Spark.

See also: [[GCP Data Services for Data Engineering]], [[Apache Kafka Fundamentals]], [[Data Engineering Lifecycle]]

---

## 1 — The Apache Beam Model

Beam's core philosophy is **write once, run anywhere** — the same pipeline code runs on Dataflow, Spark, Flink, or other runners without modification.

### Core Abstractions

| Concept | Description |
|---------|-------------|
| **Pipeline** | The top-level container representing the entire data processing job. Encapsulates all transforms and I/O |
| **PCollection** | An immutable, distributed dataset. Can be bounded (batch) or unbounded (streaming). Elements are unordered within a PCollection |
| **PTransform** | A data processing operation. Takes one or more PCollections as input and produces one or more PCollections as output |
| **DoFn** | A user-defined function applied element-by-element within a `ParDo` transform. The fundamental unit of custom logic |
| **Runner** | The execution engine. Translates the pipeline graph into concrete operations on a backend (Dataflow, Spark, Flink, Direct) |
| **Pipeline Options** | Configuration for the pipeline — runner type, project, temp location, parallelism settings |

### Pipeline Structure

```
Pipeline
  └── Read (PTransform) → PCollection
        └── Filter (PTransform) → PCollection
              └── GroupByKey (PTransform) → PCollection
                    └── Format (PTransform) → PCollection
                          └── Write (PTransform) → Output
```

Every Beam pipeline follows this pattern: **Read → Transform → Write**. Transforms are composed into a directed acyclic graph (DAG) that the runner optimises and executes.

---

## 2 — Python SDK Fundamentals

### Minimal Pipeline

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions([
    "--runner=DataflowRunner",
    "--project=my-gcp-project",
    "--region=europe-west2",
    "--temp_location=gs://my-bucket/temp/",
    "--staging_location=gs://my-bucket/staging/",
])

with beam.Pipeline(options=options) as p:
    (
        p
        | "ReadFromGCS" >> beam.io.ReadFromText("gs://my-bucket/input/*.csv")
        | "ParseCSV" >> beam.Map(lambda line: line.split(","))
        | "FilterValid" >> beam.Filter(lambda row: len(row) >= 5)
        | "FormatOutput" >> beam.Map(lambda row: ",".join(row))
        | "WriteToGCS" >> beam.io.WriteToText("gs://my-bucket/output/result")
    )
```

### DoFn Lifecycle

A `DoFn` has lifecycle methods that the runner calls at specific points:

```python
class EnrichEventDoFn(beam.DoFn):
    def setup(self):
        """Called once per worker instance — initialise heavy resources."""
        self.db_client = create_db_connection()

    def start_bundle(self):
        """Called before processing a bundle of elements."""
        self.buffer = []

    def process(self, element, timestamp=beam.DoFn.TimestampParam):
        """Called for each element. Yields zero or more output elements."""
        enriched = self.db_client.lookup(element["user_id"])
        yield {
            **element,
            "user_name": enriched.get("name"),
            "processed_at": timestamp.to_utc_datetime().isoformat(),
        }

    def finish_bundle(self):
        """Called after processing a bundle — flush buffers."""
        if self.buffer:
            yield from self._flush_buffer()

    def teardown(self):
        """Called once when the worker is shutting down — cleanup resources."""
        self.db_client.close()
```

### Common PTransforms

| Transform | Purpose | Example |
|-----------|---------|---------|
| `beam.Map(fn)` | 1-to-1 element transformation | `beam.Map(lambda x: x.upper())` |
| `beam.FlatMap(fn)` | 1-to-many (returns iterable) | `beam.FlatMap(lambda x: x.split())` |
| `beam.Filter(fn)` | Keep elements where fn returns True | `beam.Filter(lambda x: x > 0)` |
| `beam.ParDo(DoFn)` | General-purpose parallel processing | `beam.ParDo(EnrichEventDoFn())` |
| `beam.GroupByKey()` | Group values by key from (K, V) pairs | Groups all values for each key |
| `beam.CoGroupByKey()` | Join multiple PCollections by key | Relational join equivalent |
| `beam.CombinePerKey(fn)` | Aggregate values per key | `beam.CombinePerKey(sum)` |
| `beam.CombineGlobally(fn)` | Aggregate all elements | `beam.CombineGlobally(beam.combiners.CountCombineFn())` |
| `beam.Flatten()` | Merge multiple PCollections | Union of PCollections |
| `beam.Partition(fn, n)` | Split into n PCollections | Route elements to different outputs |
| `beam.Reshuffle()` | Force checkpoint / prevent fusion | Useful at streaming boundaries |

### Multiple Outputs

```python
class RouteEventsDoFn(beam.DoFn):
    VALID = "valid"
    INVALID = "invalid"

    def process(self, element):
        if element.get("event_id") and element.get("timestamp"):
            yield beam.pvalue.TaggedOutput(self.VALID, element)
        else:
            yield beam.pvalue.TaggedOutput(self.INVALID, element)


results = (
    events
    | "Route" >> beam.ParDo(RouteEventsDoFn())
        .with_outputs(RouteEventsDoFn.VALID, RouteEventsDoFn.INVALID)
)

valid_events = results[RouteEventsDoFn.VALID]
invalid_events = results[RouteEventsDoFn.INVALID]
```

---

## 3 — Windowing

Windowing divides unbounded PCollections into finite chunks for aggregation. Every element is assigned to one or more windows based on its event-time timestamp.

### Window Types

| Window Type | Description | Use Case |
|------------|-------------|----------|
| **Fixed** | Non-overlapping, uniform-duration windows | Hourly/daily aggregations |
| **Sliding** | Overlapping windows with a period and duration | Moving averages, rolling sums |
| **Session** | Dynamic windows based on activity gaps | User session analysis |
| **Global** | Single window containing all elements (default) | Batch processing |

```python
from apache_beam import window

# Fixed windows: 1-hour windows
events | beam.WindowInto(window.FixedWindows(3600))

# Sliding windows: 30-minute windows every 10 minutes
events | beam.WindowInto(window.SlidingWindows(1800, 600))

# Session windows: gap of 10 minutes between events
events | beam.WindowInto(window.Sessions(600))
```

### Event Time vs Processing Time

```
Event Time:      when the event actually occurred (embedded in the data)
Processing Time: when the event is processed by the pipeline

Late data:       events that arrive after their window has "closed"
Watermark:       the system's estimate of how far behind event time
                 processing has progressed — "I believe all events
                 before time T have arrived"
```

Beam uses **watermarks** to determine when a window is complete. The watermark advances as data arrives, and triggers fire when the watermark passes the end of a window.

---

## 4 — Triggers

Triggers control when results for a window are emitted. They decouple the window (which events belong together) from when results materialise.

### Trigger Types

| Trigger | Behaviour | Use Case |
|---------|-----------|----------|
| **AfterWatermark** | Fires when watermark passes end of window | Default — emit when window is "complete" |
| **AfterProcessingTime** | Fires after a processing-time delay | Periodic early results |
| **AfterCount** | Fires after N elements | Micro-batching |
| **Repeatedly** | Wraps another trigger, re-fires each time it triggers | Continuous updates |
| **AfterEach** | Fires triggers in sequence | Complex multi-phase emit |

### Combining Triggers with Late Data

```python
from apache_beam import trigger

(
    events
    | beam.WindowInto(
        window.FixedWindows(3600),
        trigger=trigger.AfterWatermark(
            early=trigger.AfterProcessingTime(60),     # speculative results every 60s
            late=trigger.AfterCount(1)                  # re-fire for each late element
        ),
        accumulation_mode=trigger.AccumulationMode.ACCUMULATING,
        allowed_lateness=7200  # accept late data up to 2 hours
    )
)
```

### Accumulation Modes

| Mode | Behaviour | Trade-off |
|------|-----------|-----------|
| **ACCUMULATING** | Each pane includes all elements seen so far | Downstream sees growing totals; must handle duplicates |
| **DISCARDING** | Each pane includes only new elements since last fire | Downstream sees deltas; simpler but loses context |

---

## 5 — Side Inputs

Side inputs provide additional data to a `DoFn` beyond the main PCollection — typically a lookup table or configuration.

```python
# Create a side input from a PCollection
currency_rates = (
    p
    | "ReadRates" >> beam.io.ReadFromText("gs://bucket/rates.csv")
    | "ParseRates" >> beam.Map(parse_rate)  # returns (currency_code, rate)
)

# Use as a dictionary side input
enriched = (
    transactions
    | "ConvertCurrency" >> beam.ParDo(
        ConvertCurrencyDoFn(),
        rates=beam.pvalue.AsDict(currency_rates)
    )
)


class ConvertCurrencyDoFn(beam.DoFn):
    def process(self, element, rates):
        rate = rates.get(element["currency"], 1.0)
        yield {**element, "amount_gbp": element["amount"] * rate}
```

### Side Input Patterns

| Pattern | Method | When to Use |
|---------|--------|-------------|
| **AsDict** | `beam.pvalue.AsDict(pcoll)` | Key-value lookups from (K, V) pairs |
| **AsList** | `beam.pvalue.AsList(pcoll)` | Small collections needed as a list |
| **AsSingleton** | `beam.pvalue.AsSingleton(pcoll)` | Single-value configuration |
| **AsIter** | `beam.pvalue.AsIter(pcoll)` | Iterate without materialising full list |

**Caveat:** side inputs are broadcast to all workers. Keep them small (ideally < 1 GB). For larger lookups, use a `DoFn` with `setup()` to load data from an external store.

---

## 6 — Streaming vs Batch

Beam unifies batch and streaming under the same API. The key differences are operational:

| Aspect | Batch | Streaming |
|--------|-------|-----------|
| **PCollection** | Bounded | Unbounded |
| **Windowing** | Global window (default) | Required (fixed, sliding, session) |
| **Triggers** | After all data processed | Configurable (watermark, count, time) |
| **Watermark** | Advances to +infinity at end | Continuously estimated from data |
| **Late data** | Not applicable | Managed via allowed lateness |
| **Checkpointing** | Not needed | Automatic (Dataflow manages) |
| **Sources** | Files, databases, bounded reads | Pub/Sub, Kafka, unbounded reads |
| **Cost model** | Job-based (start/finish) | Always-on (per-second billing) |

### Streaming Pipeline Example

```python
with beam.Pipeline(options=options) as p:
    (
        p
        | "ReadPubSub" >> beam.io.ReadFromPubSub(
            topic="projects/my-project/topics/events",
            timestamp_attribute="event_timestamp"
        )
        | "ParseJSON" >> beam.Map(json.loads)
        | "AddTimestamp" >> beam.Map(
            lambda x: beam.window.TimestampedValue(
                x, x["event_timestamp"]
            )
        )
        | "Window" >> beam.WindowInto(
            window.FixedWindows(300),
            trigger=trigger.AfterWatermark(
                early=trigger.AfterProcessingTime(60)
            ),
            accumulation_mode=trigger.AccumulationMode.DISCARDING,
            allowed_lateness=3600
        )
        | "CountPerUser" >> beam.combiners.Count.PerKey()
        | "FormatBQ" >> beam.Map(format_bq_row)
        | "WriteBQ" >> beam.io.WriteToBigQuery(
            "project:dataset.user_event_counts",
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
            create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED
        )
    )
```

---

## 7 — Dataflow Runner Specifics

### Flex Templates

Flex Templates package a pipeline as a Docker container, allowing custom dependencies and runtime environments:

```dockerfile
# Dockerfile
FROM gcr.io/dataflow-templates-base/python311-template-launcher-base:latest

ENV FLEX_TEMPLATE_PYTHON_PY_FILE="/template/pipeline.py"
ENV FLEX_TEMPLATE_PYTHON_REQUIREMENTS_FILE="/template/requirements.txt"

COPY pipeline.py /template/
COPY requirements.txt /template/

RUN pip install --no-cache-dir -r /template/requirements.txt
```

```bash
# Build and register the template
gcloud dataflow flex-template build \
    gs://my-bucket/templates/my-pipeline.json \
    --image-gcr-path "eu.gcr.io/my-project/my-pipeline:latest" \
    --sdk-language PYTHON \
    --flex-template-base-image PYTHON3 \
    --py-path pipeline.py \
    --metadata-file metadata.json

# Launch from template
gcloud dataflow flex-template run "my-job-$(date +%Y%m%d)" \
    --template-file-gcs-location gs://my-bucket/templates/my-pipeline.json \
    --region europe-west2 \
    --parameters input_topic=projects/my-project/topics/events \
    --parameters output_table=my-project:dataset.output
```

### Classic vs Flex Templates

| Aspect | Classic Templates | Flex Templates |
|--------|------------------|----------------|
| **Packaging** | Pre-staged on GCS | Docker container |
| **Dependencies** | Limited to pre-installed | Any Python/Java dependency |
| **Customisation** | Parameters only | Full Docker control |
| **Launch time** | ~30 seconds | ~2-5 minutes (container build) |
| **Use case** | Simple, standard pipelines | Complex dependencies, custom images |

### Autoscaling

Dataflow automatically scales workers based on backlog and CPU utilisation:

```python
# Pipeline options for autoscaling
options = PipelineOptions([
    "--autoscaling_algorithm=THROUGHPUT_BASED",
    "--max_num_workers=50",
    "--num_workers=5",          # initial workers
    "--worker_machine_type=n2-standard-4",
    "--disk_size_gb=100",
    "--worker_disk_type=compute.googleapis.com/projects//zones//diskTypes/pd-ssd",
])
```

| Parameter | Purpose |
|-----------|---------|
| `--autoscaling_algorithm` | `THROUGHPUT_BASED` (default) or `NONE` to disable |
| `--max_num_workers` | Upper bound on worker count |
| `--num_workers` | Initial worker count |
| `--worker_machine_type` | VM type (affects cost and per-worker capacity) |
| `--number_of_worker_harness_threads` | Threads per worker (Python SDK) |

**Streaming autoscaling** adjusts workers based on backlog (unprocessed messages). **Batch autoscaling** adjusts based on remaining work and elapsed time.

---

## 8 — Monitoring & Observability

### Dataflow Monitoring UI

The Dataflow console provides:

- **Job graph** — visual DAG of pipeline stages with element counts and processing rates
- **Worker metrics** — CPU, memory, disk utilisation per worker
- **Watermark lag** — how far behind event time the pipeline is (critical for streaming)
- **System lag** — time between data arrival and processing completion
- **Error logs** — stack traces from failed elements, linked to specific stages

### Key Metrics

| Metric | What It Indicates | Alert Threshold |
|--------|------------------|-----------------|
| **System lag** | Processing delay | > 5 minutes for near-real-time |
| **Data freshness** | Time since latest output | Depends on SLA |
| **Backlog bytes** | Unprocessed Pub/Sub data | Growing trend |
| **Worker CPU** | Compute utilisation | Sustained > 80% (scale up) |
| **Element count** | Throughput per stage | Drop below baseline |
| **Error count** | Failed elements | Any non-zero |

### Custom Metrics

```python
from apache_beam.metrics import Metrics

class ProcessEventDoFn(beam.DoFn):
    def __init__(self):
        self.events_processed = Metrics.counter("pipeline", "events_processed")
        self.events_dropped = Metrics.counter("pipeline", "events_dropped")
        self.processing_time = Metrics.distribution("pipeline", "processing_time_ms")

    def process(self, element):
        import time
        start = time.time()

        if self._is_valid(element):
            self.events_processed.inc()
            yield self._transform(element)
        else:
            self.events_dropped.inc()

        elapsed_ms = (time.time() - start) * 1000
        self.processing_time.update(elapsed_ms)
```

Custom metrics are visible in the Dataflow UI and can be exported to Cloud Monitoring for alerting.

---

## 9 — I/O Connectors

### Built-in Connectors

| Source/Sink | Read | Write |
|-------------|------|-------|
| **Google Cloud Storage** | `ReadFromText`, `ReadFromParquet`, `ReadFromAvro` | `WriteToText`, `WriteToParquet`, `WriteToAvro` |
| **BigQuery** | `ReadFromBigQuery` | `WriteToBigQuery` |
| **Pub/Sub** | `ReadFromPubSub` | `WriteToPubSub` |
| **Bigtable** | `ReadFromBigtable` | `WriteToBigtable` |
| **Kafka** | `ReadFromKafka` | `WriteToKafka` |
| **JDBC** | `ReadFromJdbc` | `WriteToJdbc` |
| **Avro** | `ReadFromAvro` | `WriteToAvro` |

### BigQuery Write Strategies

| Method | Description | Use Case |
|--------|-------------|----------|
| **FILE_LOADS** | Writes to GCS then bulk loads | Batch pipelines, large volumes |
| **STREAMING_INSERTS** | Real-time row-by-row inserts | Streaming with low latency needs |
| **STORAGE_WRITE_API** | Exactly-once via Storage Write API | Streaming with exactly-once guarantee |

```python
beam.io.WriteToBigQuery(
    "project:dataset.table",
    schema="event_id:STRING,timestamp:TIMESTAMP,value:FLOAT",
    write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
    create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
    method=beam.io.WriteToBigQuery.Method.STORAGE_WRITE_API,  # exactly-once
)
```

---

## 10 — Beam vs Spark Comparison

| Dimension | Apache Beam / Dataflow | Apache Spark |
|-----------|----------------------|--------------|
| **Programming model** | Pipeline + PCollection + PTransform | RDD / DataFrame / Dataset |
| **Unified batch/stream** | Native — same API, same code | Separate APIs (batch DataFrame vs Structured Streaming) |
| **Windowing** | First-class (fixed, sliding, session, custom) | Built-in but less flexible (tumbling, sliding, session) |
| **Trigger model** | Rich (watermark, processing time, count, composite) | Output modes (complete, append, update) |
| **Late data handling** | Allowed lateness + late triggers | Watermark-based with limited control |
| **Runners** | Dataflow, Spark, Flink, Direct, Samza | Spark (YARN, Kubernetes, standalone) |
| **Language support** | Java, Python, Go, Typescript | Scala, Java, Python, R, SQL |
| **Managed service** | Dataflow (GCP) | Databricks, EMR, Dataproc |
| **Autoscaling** | Automatic on Dataflow (throughput-based) | Dynamic allocation (executor-based) |
| **State management** | Per-key state, timers, bags, sets, maps | MapGroupsWithState, flatMapGroupsWithState |
| **SQL support** | Beam SQL (limited adoption) | Spark SQL (mature, widely used) |
| **Ecosystem** | Smaller, GCP-centric | Large (MLlib, GraphX, Delta Lake, extensive connectors) |
| **Shuffle** | Dataflow shuffle service (managed) | Sort-based shuffle, configurable |
| **Learning curve** | Steeper (watermarks, triggers, DoFn lifecycle) | Moderate (DataFrame API familiar to pandas users) |
| **Best for** | Streaming-first, GCP-native, event-time processing | Batch-heavy, multi-cloud, ML pipelines |

### When to Choose Beam/Dataflow

- Streaming-first architecture on GCP with Pub/Sub sources
- Complex event-time processing with late data and custom triggers
- Need for exactly-once streaming writes to BigQuery
- Pipeline portability across runners (though Dataflow is the primary target in practice)
- Serverless operation with no cluster management

### When to Choose Spark

- Batch-heavy workloads with occasional streaming
- Existing Spark expertise and ecosystem investment
- Multi-cloud or on-premises deployment
- ML pipelines (MLlib, feature engineering)
- Delta Lake or Iceberg table format integration

---

## 11 — Production Best Practices

### Pipeline Design

- **Avoid large side inputs** — broadcast cost grows with worker count. Use external lookups in `setup()` for data > 1 GB
- **Use Reshuffle strategically** — prevents fusion of stages that should checkpoint independently, but adds shuffle cost
- **Prefer CombineFn over GroupByKey** — combiners can partially aggregate before shuffling, reducing data movement
- **Set timestamps explicitly** — do not rely on Pub/Sub publish time if event time semantics matter
- **Test with DirectRunner** — fast local iteration before deploying to Dataflow

### Cost Management

| Strategy | Impact |
|----------|--------|
| Use `n2-standard-4` workers (not `n1-standard-1`) | Fewer workers, less overhead |
| Set `--max_num_workers` appropriately | Prevent runaway autoscaling |
| Use Flex Templates with prebuilt containers | Faster startup, less wasted compute |
| Batch pipelines: use `--experiments=shuffle_mode=service` | Managed shuffle reduces worker disk needs |
| Streaming: tune `--number_of_worker_harness_threads` | Match parallelism to workload |
| Use `FlexRS` (Flexible Resource Scheduling) for batch | Up to 40% cheaper using preemptible VMs |

### Error Handling

```python
class SafeProcessDoFn(beam.DoFn):
    DEAD_LETTER = "dead_letter"

    def process(self, element):
        try:
            result = transform(element)
            yield result
        except Exception as e:
            yield beam.pvalue.TaggedOutput(
                self.DEAD_LETTER,
                {"element": element, "error": str(e)}
            )


results = events | beam.ParDo(SafeProcessDoFn()).with_outputs(
    SafeProcessDoFn.DEAD_LETTER, main="processed"
)

# Write dead-letter records for investigation
results[SafeProcessDoFn.DEAD_LETTER] | beam.io.WriteToText("gs://bucket/dlq/")
```

---

## Key Takeaways

1. **Beam unifies batch and streaming** — the same pipeline code handles both bounded and unbounded data, with windowing and triggers controlling how results materialise.
2. **Dataflow is the premier Beam runner** — managed autoscaling, shuffle service, exactly-once streaming, and deep GCP integration (Pub/Sub, BigQuery, GCS).
3. **Windowing and triggers are the hardest concepts** — master fixed windows first, then progress to session windows with late data handling.
4. **Side inputs for enrichment** — broadcast small lookup tables; use `setup()` for larger external stores.
5. **Flex Templates for production** — package dependencies in Docker, version control templates, launch programmatically.
6. **Monitor watermark lag and system lag** — these are the two most important streaming health indicators.
7. **Beam vs Spark is not either/or** — Beam excels at streaming-first on GCP; Spark excels at batch-heavy workloads with ML and broad ecosystem support.

---

*Related:* [[GCP Data Services for Data Engineering]] | [[Apache Kafka Fundamentals]] | [[Data Engineering Lifecycle]] | [[Data Ingestion Patterns]] | [[Monitoring & Alerting]]
