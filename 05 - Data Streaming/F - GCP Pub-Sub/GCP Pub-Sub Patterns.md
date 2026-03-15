**Tags:** #gcp #pubsub #streaming #messaging #data-engineering

# GCP Pub/Sub Patterns

Google Cloud Pub/Sub is a fully managed, serverless messaging service for event-driven architectures. It decouples producers from consumers, scales automatically, and integrates tightly with the GCP data ecosystem.

## Architecture Overview

```
Publisher ──→ Topic ──→ Subscription A ──→ Subscriber (pull)
                   ├──→ Subscription B ──→ Subscriber (push endpoint)
                   └──→ Subscription C ──→ BigQuery table (direct write)
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | Named channel to which publishers send messages |
| **Subscription** | Named resource representing a message stream from a topic |
| **Message** | Data (up to 10 MB) plus optional attributes (key-value metadata) |
| **Acknowledgement** | Subscriber confirms successful processing; unacked messages are redelivered |
| **Retention** | Messages retained for up to 31 days (configurable) |

### Push vs Pull Delivery

| Aspect | Pull | Push |
|--------|------|------|
| **Mechanism** | Subscriber polls for messages | Pub/Sub sends HTTP POST to an endpoint |
| **Scaling** | Subscriber controls throughput | Pub/Sub controls send rate |
| **Authentication** | Service account on subscriber | OAuth token / OIDC on push endpoint |
| **Best for** | Batch consumers, variable load | Cloud Run, Cloud Functions, webhooks |
| **Back-pressure** | Built-in (subscriber controls pull rate) | Must return error codes to slow delivery |

## Ordering Keys

By default, Pub/Sub does not guarantee message order. To enforce ordering within a logical partition, use **ordering keys**.

```python
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "orders")

# All messages with the same ordering_key are delivered in order
future = publisher.publish(
    topic_path,
    data=b'{"order_id": "123", "status": "shipped"}',
    ordering_key="customer-456",
)
print(f"Published message ID: {future.result()}")
```

Key constraints:
- Ordering is per-subscription, per-ordering-key
- If a message with an ordering key fails to be acknowledged, subsequent messages with that key are held until the failed message is acked or nacked
- Ordering keys must be enabled on the subscription (`enable_message_ordering=True`)

## Dead-Letter Topics

Messages that cannot be processed after repeated delivery attempts are routed to a dead-letter topic for investigation.

```
Topic ──→ Subscription ──→ Subscriber (fails repeatedly)
                │
                └──→ Dead-letter topic ──→ DLQ subscription ──→ Alerting / manual review
```

Configuration:
- `max_delivery_attempts` — number of redelivery attempts before forwarding (default: 5)
- The dead-letter topic must exist and the Pub/Sub service account must have publish permission

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "orders-sub")

# Configure dead-letter policy on the subscription
dead_letter_policy = pubsub_v1.types.DeadLetterPolicy(
    dead_letter_topic="projects/my-project/topics/orders-dlq",
    max_delivery_attempts=10,
)

subscriber.update_subscription(
    request={
        "subscription": {
            "name": subscription_path,
            "dead_letter_policy": dead_letter_policy,
        },
        "update_mask": {"paths": ["dead_letter_policy"]},
    }
)
```

## Exactly-Once Processing

Pub/Sub offers **exactly-once delivery** at the subscription level (GA since 2022). When enabled, the service deduplicates redeliveries so that each message is delivered to the subscriber exactly once within the acknowledgement deadline.

Requirements:
- Subscription must have `enable_exactly_once_delivery=True`
- Subscribers must use the streaming pull client (not legacy pull)
- Acknowledgements are verified — the client receives an `AcknowledgeConfirmation` indicating success or failure

> [!note]
> Exactly-once delivery does not guarantee exactly-once processing end-to-end. Your subscriber's side effects (e.g., database writes) must still be idempotent or use transactional semantics. See [[Stream Processing Theory#Exactly-Once Processing]] for the broader theory.

## BigQuery Subscriptions

A BigQuery subscription writes messages directly to a [[BigQuery]] table without a separate subscriber application.

```
Topic ──→ BigQuery Subscription ──→ BigQuery Table
```

| Feature | Detail |
|---------|--------|
| **Schema** | Topic schema (Avro / Protocol Buffers) is mapped to BigQuery columns |
| **Metadata columns** | `subscription_name`, `message_id`, `publish_time`, `attributes` added automatically |
| **Write mode** | Append-only; use BigQuery DML or scheduled queries for deduplication |
| **Use cases** | Log analytics, event archiving, audit trails |

This eliminates the need for a [[Dataflow]] pipeline in simple ingest scenarios.

## Dataflow Integration

For complex transformations, [[Apache Beam]] pipelines running on [[Dataflow]] are the standard Pub/Sub consumer.

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions([
    "--runner=DataflowRunner",
    "--project=my-project",
    "--region=europe-west2",
    "--streaming",
])

with beam.Pipeline(options=options) as p:
    (
        p
        | "Read" >> beam.io.ReadFromPubSub(
            subscription="projects/my-project/subscriptions/events-sub"
        )
        | "Parse" >> beam.Map(lambda msg: json.loads(msg))
        | "Window" >> beam.WindowInto(beam.window.FixedWindows(300))  # 5-min windows
        | "Aggregate" >> beam.CombinePerKey(sum)
        | "Write" >> beam.io.WriteToBigQuery(
            table="my-project:dataset.aggregated_events",
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
        )
    )
```

Common patterns:
- **Windowed aggregation** — tumbling or sliding windows over event time
- **Enrichment** — side inputs from BigQuery or Cloud Storage
- **Branching** — route messages to different sinks based on attributes
- **Dead-letter handling** — catch transform errors and write failures to a separate table

## Python Client Examples

### Publisher

```python
from google.cloud import pubsub_v1
from concurrent import futures
import json

publisher = pubsub_v1.PublisherClient(
    publisher_options=pubsub_v1.types.PublisherOptions(
        flow_control=pubsub_v1.types.PublishFlowControl(
            message_limit=1000,
            byte_limit=10 * 1024 * 1024,  # 10 MB
        ),
    ),
)
topic_path = publisher.topic_path("my-project", "events")

def publish_event(event: dict) -> str:
    data = json.dumps(event).encode("utf-8")
    future = publisher.publish(
        topic_path,
        data=data,
        event_type=event.get("type", "unknown"),  # Custom attribute
    )
    return future.result()  # Blocks until published
```

### Subscriber (Pull — Streaming)

```python
from google.cloud import pubsub_v1
from concurrent.futures import TimeoutError
import json

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "events-sub")

def callback(message: pubsub_v1.subscriber.message.Message) -> None:
    try:
        payload = json.loads(message.data.decode("utf-8"))
        process_event(payload)
        message.ack()
    except Exception as exc:
        print(f"Processing failed: {exc}")
        message.nack()  # Triggers redelivery

streaming_pull = subscriber.subscribe(subscription_path, callback=callback)
print(f"Listening on {subscription_path}...")

try:
    streaming_pull.result(timeout=None)  # Block indefinitely
except TimeoutError:
    streaming_pull.cancel()
    streaming_pull.result()
```

### Subscriber (Pull — Synchronous Batch)

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "events-sub")

response = subscriber.pull(
    request={"subscription": subscription_path, "max_messages": 100},
)

ack_ids = []
for msg in response.received_messages:
    payload = json.loads(msg.message.data.decode("utf-8"))
    process_event(payload)
    ack_ids.append(msg.ack_id)

if ack_ids:
    subscriber.acknowledge(
        request={"subscription": subscription_path, "ack_ids": ack_ids},
    )
```

## Pub/Sub vs Kafka Comparison

| Aspect | GCP Pub/Sub | Apache Kafka |
|--------|-------------|--------------|
| **Deployment** | Fully managed (serverless) | Self-managed or managed (Confluent, MSK) |
| **Scaling** | Automatic, no partitions to manage | Manual partition assignment |
| **Ordering** | Per ordering key (opt-in) | Per partition (guaranteed) |
| **Retention** | Up to 31 days | Configurable (unlimited with tiered storage) |
| **Replay** | Seek to timestamp or snapshot | Consumer offset reset to any position |
| **Exactly-once** | Subscription-level deduplication | Transactional producers + idempotent consumers |
| **Consumer groups** | Multiple subscriptions per topic | Consumer groups with partition assignment |
| **Throughput** | High (auto-scaled) | Very high (tuneable with partitions) |
| **Latency** | Typically < 100 ms | Typically < 10 ms |
| **Dead-letter** | Built-in (subscription config) | Manual (application-level routing) |
| **Ecosystem** | Dataflow, BigQuery, Cloud Functions | Kafka Streams, ksqlDB, Connect |
| **Cost model** | Per-message + data volume | Infrastructure cost (brokers, storage) |
| **Best for** | GCP-native pipelines, serverless event routing | High-throughput, low-latency, multi-cloud |

## Related Topics

- [[Stream Processing Theory]] — windowing, watermarks, exactly-once semantics
- [[Apache Kafka]] — deep dive into Kafka architecture and operations
- [[Dataflow]] — managed Apache Beam runner on GCP
- [[BigQuery]] — GCP's serverless data warehouse
- [[Event-Driven Architecture]] — broader patterns for event-based systems
