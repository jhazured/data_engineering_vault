---
tags:
  - snowflake
  - streaming
  - cdc
  - real-time
  - data-engineering
created: 2026-03-15
---

# Snowflake Streams & Real-Time Tasks

Snowflake Streams provide native change data capture (CDC) on tables, enabling event-driven processing without external tooling. Combined with Snowflake Tasks, they form a lightweight real-time pipeline layer within the warehouse itself.

---

## 1. What Is a Snowflake Stream?

A stream is a metadata object that records DML changes (inserts, updates, deletes) made to a source table since the stream's current offset. It does not copy data -- it tracks change metadata against the table's micro-partitions.

```sql
CREATE OR REPLACE STREAM str_raw_shipments_stream
  ON TABLE LOGISTICS_DW_DEV.ANALYTICS_RAW.SHIPMENTS
  COMMENT = 'CDC stream for real-time shipment ingestion';
```

When you query a stream, it returns rows that have changed since the last time the stream was consumed. Each row carries three metadata columns:

| Column | Type | Meaning |
|--------|------|---------|
| `METADATA$ACTION` | VARCHAR | `INSERT` or `DELETE` |
| `METADATA$ISUPDATE` | BOOLEAN | `TRUE` if the row is part of an update (appears as DELETE + INSERT pair) |
| `METADATA$ROW_ID` | VARCHAR | Unique identifier for the changed row |

An UPDATE produces two rows: a DELETE of the old value and an INSERT of the new value, both with `METADATA$ISUPDATE = TRUE`.

---

## 2. Stream Types

### Standard Stream (Default)

Tracks all DML: inserts, updates, and deletes. Uses both the current and historical micro-partition state.

```sql
CREATE STREAM str_fact_shipments ON TABLE TBL_FACT_SHIPMENTS;
-- Equivalent to: APPEND_ONLY = FALSE (default)
```

Best for: dimension tables, fact tables with updates/deletes, SCD processing.

### Append-Only Stream

Tracks only INSERT operations. Lower overhead because it does not need to track row-level changes.

```sql
CREATE STREAM str_telemetry_append
  ON TABLE TBL_FACT_VEHICLE_TELEMETRY
  APPEND_ONLY = TRUE;
```

Best for: event/log tables, telemetry data, immutable fact tables where rows are never updated. Telemetry and IoT data are natural fits -- they arrive as new rows only.

### Insert-Only Stream (On External Tables)

For external tables (S3, GCS, Azure Blob), only `INSERT_ONLY = TRUE` is supported -- external tables do not track updates or deletes.

---

## 3. Stream Offset and Consumption

A stream maintains an **offset** -- a pointer to the table's change history. The offset advances **only when the stream is consumed inside a DML transaction that commits successfully**.

```
Table timeline:   [t0] ──insert──> [t1] ──update──> [t2] ──insert──> [t3]
Stream offset:     ^                                  ^
                   last consumed                      current position
                   (after successful DML)             (pending changes)
```

Key behaviours:

- **Reading a stream does not advance the offset** -- only a committed DML that reads from the stream advances it
- **Multiple reads before consumption** return the same change set
- If the consuming transaction rolls back, the offset remains unchanged
- A stream becomes stale if the retention period of the source table is exceeded without consumption (default 14 days for Snowflake standard tables)

### Consumption Pattern: MERGE Into Target

The most common pattern -- merge changed rows into a target table, which atomically advances the stream offset.

```sql
MERGE INTO dim_customers AS tgt
USING str_stg_customers_stream AS src
  ON tgt.customer_id = src.customer_id
WHEN MATCHED AND src.METADATA$ACTION = 'INSERT' AND src.METADATA$ISUPDATE = TRUE
  THEN UPDATE SET tgt.customer_name = src.customer_name,
                  tgt.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND src.METADATA$ACTION = 'INSERT'
  THEN INSERT (customer_id, customer_name, created_at)
       VALUES (src.customer_id, src.customer_name, CURRENT_TIMESTAMP());
```

### Consumption Pattern: INSERT Into Downstream Table

Simpler alternative when the target is an append-only table (e.g. alerts, audit logs).

```sql
INSERT INTO monitoring.real_time_alerts (alert_type, severity, message, related_id)
SELECT 'SHIPMENT_DELAY', 'HIGH',
       'Shipment ' || shipment_id || ' is behind schedule',
       shipment_id
FROM str_fact_shipments_stream
WHERE actual_duration_minutes > planned_duration_minutes * 1.2;
```

The INSERT commits, advancing the stream offset. Unmatched rows (those that did not satisfy the WHERE clause) are also consumed -- the offset advances past all pending changes, not only the filtered subset.

---

## 4. Conditional Execution with SYSTEM$STREAM_HAS_DATA()

`SYSTEM$STREAM_HAS_DATA()` returns a BOOLEAN indicating whether a stream has unconsumed change records. It is primarily used in the `WHEN` clause of a Snowflake Task to avoid unnecessary warehouse spin-up.

```sql
CREATE OR REPLACE TASK tsk_real_time_shipment_alerts
  WAREHOUSE = COMPUTE_WH_XS
  SCHEDULE = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('STR_fact_shipments_stream')
AS
  INSERT INTO monitoring.real_time_alerts ...
  FROM STR_fact_shipments_stream
  WHERE ...;
```

Without the `WHEN` clause, the task would start the warehouse every minute regardless of whether there is new data -- wasting credits. With it, the warehouse only starts when there are changes to process.

The function is lightweight: it checks stream metadata without scanning the underlying table.

---

## 5. Task Scheduling with Streams

A Snowflake Task is a scheduled SQL statement. Combined with streams, tasks become event-driven micro-batch processors.

### Task Anatomy

```sql
CREATE OR REPLACE TASK tsk_vehicle_monitoring
  WAREHOUSE = COMPUTE_WH_XS          -- compute resource
  SCHEDULE = '1 MINUTE'               -- check frequency
  WHEN SYSTEM$STREAM_HAS_DATA('str_fact_vehicle_telemetry_stream')
  COMMENT = 'Process vehicle telemetry changes for alerting'
AS
  <SQL statement consuming the stream>;
```

| Property | Purpose |
|----------|---------|
| `WAREHOUSE` | Assigned compute; auto-starts and auto-suspends |
| `SCHEDULE` | How often to evaluate the `WHEN` condition (CRON or interval) |
| `WHEN` | Boolean guard -- task body executes only if this returns TRUE |
| `COMMENT` | Documentation (good practice for operational clarity) |

Tasks are created in a **suspended** state. Activate with `ALTER TASK ... RESUME`.

### Task DAGs (Dependent Tasks)

Tasks can form directed acyclic graphs using the `AFTER` clause:

```sql
CREATE TASK tsk_child_task
  WAREHOUSE = COMPUTE_WH_XS
  AFTER tsk_parent_task
AS
  <SQL>;
```

Only the root task has a `SCHEDULE`; child tasks run after their parent completes successfully. This is useful for multi-step stream processing (e.g. ingest -> transform -> alert).

For batch task scheduling with stored procedures, see [[Snowflake SQL Pipeline Patterns]].

---

## 6. Real-Time Alert Patterns

Drawn from a logistics analytics platform, these patterns demonstrate stream-triggered alerting across operational domains.

### Shipment Delay Alerts

```sql
-- Task body: fires when shipment fact stream has data
INSERT INTO monitoring.real_time_alerts
SELECT
    CASE
        WHEN actual_duration_minutes > planned_duration_minutes * 1.5
            THEN 'SHIPMENT_CRITICAL_DELAY'
        WHEN actual_duration_minutes > planned_duration_minutes * 1.2
            THEN 'SHIPMENT_DELAY'
    END AS alert_type,
    CASE
        WHEN actual_duration_minutes > planned_duration_minutes * 1.5 THEN 'CRITICAL'
        ELSE 'HIGH'
    END AS severity,
    'Shipment ' || shipment_id || ' is ' ||
        ROUND((actual_duration_minutes - planned_duration_minutes) / 60.0, 1) ||
        ' hours behind schedule' AS message,
    CURRENT_TIMESTAMP() AS alert_timestamp,
    shipment_id AS related_id,
    'SHIPMENT' AS entity_type
FROM str_fact_shipments_stream
WHERE actual_duration_minutes > planned_duration_minutes * 1.2
  AND METADATA$ACTION IN ('INSERT', 'UPDATE');
```

### Vehicle Maintenance Alerts

Telemetry streams suit the **append-only** type -- new readings arrive but are never updated.

```sql
-- Thresholds: engine temp > 95C, fuel < 10%, health score < 70
INSERT INTO monitoring.real_time_alerts
SELECT
    CASE
        WHEN engine_temp_c > 95         THEN 'ENGINE_OVERHEAT'
        WHEN fuel_level_percent < 5     THEN 'FUEL_CRITICAL'
        WHEN engine_health_score < 70   THEN 'MAINTENANCE_NEEDED'
        WHEN harsh_braking_events > 5   THEN 'HARSH_DRIVING'
    END AS alert_type,
    CASE
        WHEN engine_temp_c > 95 OR fuel_level_percent < 5 THEN 'CRITICAL'
        WHEN engine_health_score < 70 THEN 'HIGH'
        ELSE 'MEDIUM'
    END AS severity,
    <message construction> AS message,
    CURRENT_TIMESTAMP() AS alert_timestamp,
    vehicle_id AS related_id,
    'VEHICLE' AS entity_type
FROM str_fact_vehicle_telemetry_stream
WHERE engine_temp_c > 95
   OR fuel_level_percent < 10
   OR engine_health_score < 70
   OR harsh_braking_events > 5;
```

### Cost Monitoring Alerts

```sql
-- Fires when shipment costs exceed revenue thresholds
INSERT INTO monitoring.real_time_alerts
SELECT
    CASE
        WHEN (fuel_cost + delivery_cost) > revenue * 0.8 THEN 'COST_OVERRUN_CRITICAL'
        WHEN fuel_cost > planned_fuel_cost * 1.3          THEN 'FUEL_COST_SPIKE'
    END AS alert_type,
    ...
FROM str_fact_shipments_stream
WHERE (fuel_cost + delivery_cost) > revenue * 0.6
   OR fuel_cost > planned_fuel_cost * 1.2;
```

### Alert Target Table Structure

```sql
CREATE TABLE IF NOT EXISTS monitoring.real_time_alerts (
    alert_id       VARCHAR(36) DEFAULT UUID_STRING(),
    alert_type     VARCHAR(50) NOT NULL,
    severity       VARCHAR(20) NOT NULL,    -- CRITICAL, HIGH, MEDIUM
    message        TEXT NOT NULL,
    alert_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    related_id     VARCHAR(100),
    entity_type    VARCHAR(50),             -- SHIPMENT, VEHICLE, ROUTE
    created_at     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

---

## 7. Multi-Layer Stream Architecture

Streams can be layered across the data warehouse tiers:

```
Raw Layer Streams          Staging Layer Streams       Mart Layer Streams
(new source data)          (quality-checked changes)   (business-level CDC)
  str_raw_shipments   -->    str_stg_shipments    -->    str_fact_shipments
  str_raw_telematics  -->    str_stg_telemetry    -->    str_fact_telemetry
  str_raw_vehicles    -->    str_stg_vehicles     -->    str_dim_vehicles
```

Each tier's stream captures only the changes relevant to that layer. A task at the raw layer might clean and insert into staging; a task at the staging layer transforms and loads into marts; a task at the mart layer generates alerts.

---

## 8. Operational Considerations

### Staleness

A stream becomes **stale** if its offset falls behind the source table's data retention period. Once stale, it cannot be read -- you must recreate it. Monitor with:

```sql
SELECT stream_name, stale, stale_after
FROM information_schema.streams
WHERE stream_schema = 'MARTS';
```

### Verification Queries

```sql
-- Check all streams in a schema
SELECT stream_name, source_database, source_schema, source_table, created_on
FROM information_schema.streams WHERE stream_schema = 'MARTS';

-- Check task state (should be 'started' after RESUME)
SELECT task_name, warehouse_name, schedule, state
FROM information_schema.tasks WHERE task_schema = 'MARTS';
```

### Deployment

Streams and tasks are typically deployed **after** the source tables exist (e.g. after dbt models). A deployment script runs the SQL files in order: create streams, create tasks, verify. See the `deploy_streams_and_tasks.sh` pattern for automated sequencing via SnowSQL.

### Cost Control

- Use `SYSTEM$STREAM_HAS_DATA()` in every stream-triggered task to avoid idle warehouse starts
- Size the warehouse appropriately (`XS` for alert-generation tasks; larger for heavy transformations)
- Set `AUTO_SUSPEND` on the warehouse to minimise idle time between task runs

---

## 9. Snowflake Streams vs Kafka CDC

Both systems capture changes, but they serve different architectural roles.

| Aspect | Snowflake Streams | [[Apache Kafka Fundamentals|Kafka]] CDC (Debezium) |
|--------|-------------------|-----------------------------------------------------|
| **Scope** | Changes within Snowflake tables only | Any source database (Postgres, MySQL, MongoDB, etc.) |
| **Architecture** | Native metadata layer -- no extra infrastructure | Requires Kafka cluster, Connect workers, Schema Registry |
| **Latency** | Micro-batch (1-minute minimum task schedule) | Near real-time (sub-second with log-based CDC) |
| **Consumer model** | Single consumer per stream (offset advances on DML commit) | Multiple independent consumer groups, replay capability |
| **Offset management** | Automatic -- advances on successful transaction commit | Manual or auto-commit; consumer controls position |
| **Replay** | Not possible after offset advances (recreate stream to reset) | Replay by resetting consumer group offset (within retention) |
| **Ordering** | Table-level ordering (single stream = single consumer) | Per-partition ordering; parallelism via partition count |
| **Schema evolution** | Follows source table DDL automatically | Managed via Schema Registry (Avro/Protobuf compatibility) |
| **Use case** | In-warehouse CDC, ELT micro-batch, Snowflake-native alerting | Cross-system event streaming, decoupled microservices, multi-target fan-out |
| **Cost model** | Warehouse credits only when task runs | Kafka cluster + Connect infrastructure (always-on) |

**When to choose Snowflake Streams**: the source and target are both in Snowflake, latency of 1+ minutes is acceptable, and you want zero additional infrastructure.

**When to choose Kafka CDC**: you need sub-second latency, multiple independent consumers, cross-system integration, or replay capability.

They can also be **complementary**: Kafka ingests CDC from source databases into Snowflake raw tables, and Snowflake Streams then propagate changes through the warehouse layers.

---

## 10. Stream Design Patterns Summary

| Pattern | Stream Type | Task Schedule | Example |
|---------|-------------|---------------|---------|
| Dimension sync | Standard | 5-15 min | MERGE new/changed customers into dim table |
| Fact alerting | Standard | 1 min | INSERT delay alerts from shipment fact changes |
| Telemetry monitoring | Append-only | 1 min | INSERT vehicle health alerts from telemetry inserts |
| Cost monitoring | Standard | 5 min | INSERT cost overrun alerts from shipment changes |
| Data quality gate | Standard | 5 min | Flag and quarantine rows failing quality checks |
| Multi-layer propagation | Standard | Chained (AFTER) | Raw -> staging -> mart via task DAG |

---

## Related

- [[Apache Kafka Fundamentals]] -- distributed event streaming and CDC with Debezium
- [[Snowflake SQL Pipeline Patterns]] -- batch task scheduling, stored procedures, star schema DDL
- [[Dimensional Modelling (Kimball)]] -- target schema patterns for stream-driven dimension updates
- [[Data Quality Frameworks]] -- quality checks that can be embedded in stream-consuming tasks
