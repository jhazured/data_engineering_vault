# Event-Driven Architecture

An architectural style where system components communicate by producing and consuming events — enabling loose coupling, scalability, and real-time responsiveness.

## Core Concepts

### Events vs Commands vs Queries

| Type | Direction | Intent | Example |
|------|-----------|--------|---------|
| **Event** | Broadcast | "This happened" (past tense) | `OrderShipped` |
| **Command** | Targeted | "Do this" (imperative) | `ShipOrder` |
| **Query** | Request/response | "Tell me" | `GetOrderStatus` |

Events are **facts** — immutable records of something that happened. They don't prescribe what to do; consumers decide how to react.

## Event Sourcing

Instead of storing current state, store the sequence of events that led to it:

```
Traditional (state-based):
  orders table: { id: 1, status: "shipped", total: 150.00 }

Event Sourced:
  events: [
    { type: "OrderCreated", orderId: 1, total: 150.00 },
    { type: "PaymentReceived", orderId: 1, amount: 150.00 },
    { type: "OrderShipped", orderId: 1, trackingId: "TRK123" },
  ]
```

**Benefits:**
- Complete audit trail — every state change is recorded
- Temporal queries — "what was the order status at 3pm yesterday?"
- Replay — rebuild state from events (fix bugs, build new views)
- No data loss — events are append-only, never deleted or updated

**Trade-offs:**
- More complex to query current state (need materialized views)
- Event schema evolution must be managed carefully
- Storage grows over time (snapshotting helps)

### Materialized Views from Events

Derive current-state views by replaying events:

```
Event Stream                    Materialized View
─────────────                   ─────────────────
OrderCreated(id=1)        →     orders: { 1: { status: "created" } }
PaymentReceived(id=1)     →     orders: { 1: { status: "paid" } }
OrderShipped(id=1)        →     orders: { 1: { status: "shipped" } }
```

Multiple views from the same event stream:
- **Order status view** — current state per order
- **Revenue dashboard** — aggregated totals
- **Shipping analytics** — delivery metrics

## CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries):

```
                    ┌──────────────┐
  Commands ────────▶│  Write Model  │──── Events ────▶ Event Store
                    └──────────────┘                      │
                                                          ▼
                    ┌──────────────┐              ┌──────────────┐
  Queries ◀────────│  Read Model   │◀──── Build ──│  Projections  │
                    └──────────────┘              └──────────────┘
```

**Why separate?**
- Write model optimized for consistency and validation
- Read model optimized for query patterns (denormalized, cached)
- Scale reads and writes independently
- Different storage for each (relational for writes, search index for reads)

## Event-Driven Patterns

### Choreography vs Orchestration

**Choreography** — services react to events independently:
```
OrderService publishes OrderCreated
  → PaymentService reacts: processes payment, publishes PaymentReceived
  → InventoryService reacts: reserves stock, publishes StockReserved
  → ShippingService reacts: schedules pickup
```

**Orchestration** — a central coordinator directs the flow:
```
OrderSaga orchestrates:
  1. Tell PaymentService: process payment
  2. Wait for response
  3. Tell InventoryService: reserve stock
  4. Wait for response
  5. Tell ShippingService: schedule pickup
```

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| Coupling | Loose | Tighter (coordinator knows all steps) |
| Visibility | Hard to trace end-to-end | Central view of the process |
| Failure handling | Each service handles its own | Coordinator manages compensation |
| Best for | Simple flows | Complex multi-step transactions |

### Saga Pattern

Manage distributed transactions without two-phase commit:

```
Forward actions:         Compensating actions:
1. Create Order    ←→    Cancel Order
2. Reserve Stock   ←→    Release Stock
3. Charge Payment  ←→    Refund Payment
4. Ship Order      ←→    Cancel Shipment
```

If step 3 fails, execute compensating actions for steps 2 and 1 in reverse order.

## Kafka as Event Backbone

Kafka naturally supports event-driven architecture:

- **Topics as event channels** — each domain publishes to its topic
- **Consumer groups as services** — each service reads independently
- **Log retention as event store** — replay events for new consumers
- **Compacted topics as state** — keep only latest value per key

### Kafka Streams

Stream processing library embedded in your application (no separate cluster):

```
Events in (topic) → Transform/Aggregate → Events out (topic)
```

Use cases:
- Real-time aggregations (counts, sums, averages)
- Joins between event streams
- Stateful transformations with local state stores

### KSQL / ksqlDB

SQL interface over Kafka streams:

```sql
-- Create a stream from a topic
CREATE STREAM shipments (
    shipment_id VARCHAR KEY,
    status VARCHAR,
    weight DOUBLE
) WITH (kafka_topic='shipments', value_format='JSON');

-- Continuous query — materializes a table
CREATE TABLE shipment_counts AS
    SELECT status, COUNT(*) AS cnt
    FROM shipments
    GROUP BY status
    EMIT CHANGES;
```

## Schema Evolution

Events evolve over time. Manage compatibility:

| Strategy | Rule | Example |
|----------|------|---------|
| **Backward compatible** | New schema can read old data | Add optional field |
| **Forward compatible** | Old schema can read new data | Remove optional field |
| **Full compatible** | Both directions work | Add optional field with default |

**Tools:** Confluent Schema Registry (Avro, Protobuf), AWS Glue Schema Registry

## Eventual Consistency

Event-driven systems are **eventually consistent** — there's a delay between an event being published and all consumers processing it.

**Handling strategies:**
- **Idempotent consumers** — processing the same event twice produces the same result
- **Deduplication** — track processed event IDs
- **Read-your-writes** — route queries to the same partition that was just written
- **UI optimistic updates** — show expected state immediately, reconcile later

## When to Use Event-Driven Architecture

**Good fit:**
- Multiple consumers need the same data (fan-out)
- Real-time reactions to business events
- Audit trail / compliance requirements
- Decoupling between teams/services
- Rebuilding views from historical events

**Poor fit:**
- Simple request/response CRUD
- Strong consistency required immediately
- Small team, simple domain (overhead not justified)
- Low-latency synchronous operations
