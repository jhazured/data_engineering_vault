tags: #troubleshooting #runbook #snowflake #pyspark #dbt #airflow #kafka #data-engineering

# Data Engineering Troubleshooting Runbooks

Structured runbooks for diagnosing and resolving common data engineering production issues. Each runbook follows a consistent format: symptoms, diagnosis steps, resolution, and prevention.

---

## Runbook 1: Snowflake Slow Queries

### Symptoms

- Dashboard refresh times exceeding SLA thresholds
- dbt model run durations increasing over time without data volume changes
- Users reporting query timeouts or long wait times
- Warehouse credit consumption spiking without corresponding workload increase

### Diagnosis Steps

**Step 1: Check the Query Profile.**

Open the query in the Snowflake UI and inspect the Query Profile. Look for these indicators:

| Indicator | Meaning |
|-----------|---------|
| **Bytes spilled to local storage** | Sort/join/aggregate exceeds memory -- data written to SSD |
| **Bytes spilled to remote storage** | Severe memory pressure -- data written to S3 (very slow) |
| **Low partition pruning ratio** | Scanning far more micro-partitions than necessary |
| **Exploding joins** | Output rows vastly exceed input rows (bad join keys) |
| **Queuing time > 0** | Warehouse concurrency saturated |

**Step 2: Check clustering efficiency.**

```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('schema.table_name', '(date_column, id_column)');
```

If `average_depth` > 5, the table is poorly clustered for query patterns. If `average_overlap` is high, micro-partitions contain overlapping value ranges.

**Step 3: Check warehouse sizing and utilisation.**

```sql
SELECT
    warehouse_name,
    AVG(avg_running),
    AVG(avg_queued_load),
    AVG(avg_blocked)
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY AVG(avg_queued_load) DESC;
```

High `avg_queued_load` indicates concurrency bottlenecks. High `avg_running` with low `avg_queued_load` indicates the warehouse is appropriately sized.

**Step 4: Identify resource-intensive queries.**

```sql
SELECT
    query_id,
    query_text,
    total_elapsed_time / 1000 AS elapsed_seconds,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage,
    partitions_scanned,
    partitions_total,
    ROUND(partitions_scanned / NULLIF(partitions_total, 0) * 100, 1) AS scan_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
    AND total_elapsed_time > 60000
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

### Resolution

| Problem | Resolution |
|---------|------------|
| **Spilling to local/remote** | Scale up the warehouse (XS to S, S to M). If a single query, refactor to reduce data volume with earlier filters or pre-aggregation |
| **Poor partition pruning** | Add or adjust clustering keys on the columns used in WHERE/JOIN clauses. Enable automatic clustering |
| **Exploding joins** | Check join keys for duplicates. Add deduplication (ROW_NUMBER) upstream of the join |
| **Concurrency queuing** | Enable multi-cluster warehouses (MIN_CLUSTER_COUNT=1, MAX_CLUSTER_COUNT=3) or separate workloads to dedicated warehouses |
| **Cartesian products** | Fix missing or incorrect JOIN conditions. Review query logic |
| **Full table scans** | Add WHERE clauses on clustered columns. Avoid functions on filter columns (use range predicates instead) |

### Prevention

- Set `STATEMENT_TIMEOUT_IN_SECONDS` on all warehouses to prevent runaway queries
- Monitor query durations via scheduled queries against `QUERY_HISTORY` with alerting on regression
- Review Query Profile for any dbt model exceeding 5 minutes during CI
- Establish clustering keys during table creation, not as a remediation step
- Use resource monitors to detect unexpected credit consumption early

---

## Runbook 2: PySpark OOM Errors

### Symptoms

- `java.lang.OutOfMemoryError: Java heap space` on executors or driver
- `Container killed by YARN for exceeding memory limits`
- `SparkException: Job aborted due to stage failure` with OOM in the stack trace
- Tasks marked as FAILED with `ExecutorLostFailure` (executor exited with non-zero exit code)
- Spark UI showing tasks with excessive shuffle read/write or spill

### Diagnosis Steps

**Step 1: Identify whether the OOM is on the driver or executor.**

- **Driver OOM**: Occurs during `collect()`, `toPandas()`, `broadcast()`, or when the driver collects too much data. Stack trace references `org.apache.spark.SparkContext` or `org.apache.spark.sql.Dataset.collect`.
- **Executor OOM**: Occurs during shuffle, sort, join, or aggregation. Stack trace references `org.apache.spark.executor.Executor`.

**Step 2: Check Spark UI for stage details.**

- Open the Spark UI and navigate to the failed stage
- Check **Shuffle Read/Write** -- excessive shuffle indicates data skew or too-wide shuffles
- Check **Spill (Memory)** and **Spill (Disk)** -- high spill indicates insufficient executor memory for the operation
- Check **Task Duration** distribution -- if a few tasks take orders of magnitude longer than others, data skew is likely

**Step 3: Check partition sizes.**

```python
df.rdd.mapPartitions(lambda it: [sum(1 for _ in it)]).collect()
# Or more efficiently:
df.groupBy(spark_partition_id()).count().orderBy("count", ascending=False).show()
```

If the largest partition is 10x+ the median, data skew is the problem.

**Step 4: Review broadcast join thresholds.**

```python
spark.conf.get("spark.sql.autoBroadcastJoinThreshold")  # Default: 10MB
```

If a table slightly exceeds this threshold, it triggers a sort-merge join with full shuffle instead of a broadcast join.

### Resolution

| Problem | Resolution |
|---------|------------|
| **Driver OOM from collect()** | Never `collect()` large datasets. Use `take(n)`, `show(n)`, or write to storage. For `toPandas()`, use `pandas_api()` or Arrow-based conversion with limits |
| **Executor OOM from shuffle** | Increase `spark.executor.memory` (e.g. 4g to 8g). Increase `spark.sql.shuffle.partitions` (default 200; try 400-1000 for large datasets). Reduce shuffle by filtering early |
| **Data skew** | Salt the skewed key: append a random suffix, join on salted key, aggregate, then remove salt. Or use `broadcast()` hint if one side is small. Databricks: enable adaptive query execution (AQE) with `spark.sql.adaptive.enabled=true` |
| **Broadcast threshold too low** | Increase `spark.sql.autoBroadcastJoinThreshold` to 50-100MB if one join side fits in memory. Force with `F.broadcast(small_df)` |
| **Too few partitions** | Repartition: `df.repartition(200, "key_column")`. For writes: `df.coalesce(target_file_count)` |
| **Too many partitions** | Reduce `spark.sql.shuffle.partitions`. Use `coalesce()` before writes to avoid small files |

**Memory configuration reference:**

```python
spark.conf.set("spark.executor.memory", "8g")
spark.conf.set("spark.executor.memoryOverhead", "2g")  # Off-heap (20-25% of executor memory)
spark.conf.set("spark.driver.memory", "4g")
spark.conf.set("spark.sql.shuffle.partitions", "400")
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

### Prevention

- Enable Adaptive Query Execution (AQE) by default -- it handles skew and partition coalescing automatically
- Never use `collect()` or `toPandas()` without row limits in production code
- Set `spark.executor.memoryOverhead` to at least 20% of `spark.executor.memory`
- Monitor Spark UI metrics in CI for long-running jobs; alert on spill increases
- Use `df.cache()` sparingly -- cached DataFrames consume executor memory and can trigger OOM on subsequent operations
- Profile data skew before writing joins: check key cardinality and distribution

---

## Runbook 3: dbt Compilation and Runtime Errors

### Symptoms

- `dbt run` or `dbt build` fails during compilation or execution
- Error messages referencing circular dependencies, missing refs, schema mismatches, or incremental logic failures
- Source freshness checks failing with stale data warnings
- CI pipeline failing on pull request builds

### Diagnosis Steps and Resolutions

#### 3a: Dependency Cycles

**Symptom**: `Compilation Error: Found a cycle: model_a -> model_b -> model_a`

**Diagnosis**: Run `dbt ls --output json | jq '.depends_on'` to inspect the dependency graph. Cycles often arise from two models referencing each other or from circular macro calls.

**Resolution**:
- Break the cycle by introducing an intermediate model
- If both models need the same data, extract the shared logic into a third model they both reference
- Use `dbt docs generate` and inspect the DAG visually to identify the cycle

**Prevention**: Review the DAG on every pull request. Add a CI check that runs `dbt parse` -- it detects cycles at compilation time.

#### 3b: Schema Drift

**Symptom**: `Database Error: column "new_column" does not exist` or `Compilation Error: column "old_column" not found in source`

**Diagnosis**:
1. Check if the source schema has changed: `dbt source freshness` may pass even when schema drifts
2. Compare the current source schema against `schema.yml` definitions
3. Check if an upstream model changed its output columns

**Resolution**:
- Update `schema.yml` to reflect the new source schema
- If a source column was removed, update all downstream models that reference it
- For added columns: add to staging model if needed, or ignore if not required
- Run `dbt run --select +changed_model` to rebuild the downstream chain

**Prevention**:
- Use `dbt source freshness` combined with schema contract tests (`dbt-expectations` package)
- Enable dbt contracts (v1.5+) with `contract: {enforced: true}` on critical models
- Set up automated schema change detection in the ingestion layer (e.g. Auto Loader rescue mode)

#### 3c: Incremental Model Failures

**Symptom**: `Merge statement failed` or incremental model produces duplicates or misses records

**Diagnosis**:
1. Check the `unique_key` configuration -- is it truly unique in the source?
2. Check for late-arriving data that falls outside the incremental window
3. Run `dbt run --full-refresh --select model_name` to verify the model logic works on a full load
4. Check if the incremental predicate (`is_incremental()` filter) is correct

**Resolution**:
- If `unique_key` has duplicates, add deduplication logic upstream or use a composite key
- For late-arriving data, widen the lookback window: `WHERE event_date >= DATEADD('day', -3, (SELECT MAX(event_date) FROM {{ this }}))`
- If the schema has changed, run `--full-refresh` to rebuild the table
- For merge conflicts on Snowflake, ensure the merge key does not match multiple source rows

**Prevention**:
- Always test `unique` on the `unique_key` column(s)
- Schedule periodic full-refresh runs (e.g. weekly) to correct any drift
- Use `on_schema_change: 'sync_all_columns'` in incremental config to handle new columns automatically

#### 3d: Source Freshness Failures

**Symptom**: `Source freshness check failed: source.table loaded_at is older than threshold`

**Diagnosis**:
1. Check whether the upstream pipeline actually ran: inspect Airflow/orchestrator logs
2. Verify the `loaded_at_field` is correct and populated
3. Check if the source table has new data but the `loaded_at_field` was not updated

**Resolution**:
- If the upstream pipeline failed, fix and re-run the pipeline first
- If `loaded_at_field` is wrong, update `schema.yml` to reference the correct timestamp column
- If freshness is intermittently borderline, widen `warn_after` and `error_after` thresholds

**Prevention**:
- Include source freshness checks at the start of dbt DAG runs
- Configure `error_after` to block downstream models if sources are stale
- Monitor upstream pipeline SLAs independently of dbt

---

## Runbook 4: Airflow DAG Failures

### Symptoms

- Tasks stuck in `queued`, `scheduled`, or `up_for_retry` states
- DAG runs not starting on schedule
- Task logs showing `Connection refused`, `Pool full`, or `Timeout` errors
- Scheduler lag visible in Airflow UI (last heartbeat is stale)

### Diagnosis Steps and Resolutions

#### 4a: Task Retry Exhaustion

**Symptom**: Task reaches `max_retries` and moves to `failed` state. Downstream tasks are skipped.

**Diagnosis**:
1. Read the task log -- the root cause is in the last retry attempt
2. Common causes: API rate limits, database connection timeouts, transient cloud errors, authentication token expiry

**Resolution**:
- For transient errors: increase `retries` (default 0; set to 2-3) and `retry_delay` (use `timedelta(minutes=5)` with `retry_exponential_backoff=True`)
- For authentication errors: refresh tokens/credentials in the Airflow connection
- For resource limits: check if the external system is under load; add exponential backoff
- For persistent failures: fix the underlying issue (broken query, missing permissions, changed API endpoint)

**Prevention**:
- Set `retries=2` and `retry_delay=timedelta(minutes=5)` as DAG-level defaults
- Use `on_failure_callback` to send alerts with task context (DAG ID, task ID, execution date, log URL)
- Test external connectivity in a dedicated health-check task at the start of critical DAGs

#### 4b: Dependency and Trigger Rule Issues

**Symptom**: Tasks skipped unexpectedly or running when they should not.

**Diagnosis**:
1. Check `trigger_rule` on the affected task (default is `all_success`)
2. Check upstream task states -- if any upstream failed or was skipped, `all_success` skips downstream
3. Check `depends_on_past` -- if True, the task waits for the previous DAG run's instance to succeed

**Resolution**:
- Use `trigger_rule='none_failed'` if the task should run when upstream tasks succeed or are skipped
- Use `trigger_rule='all_done'` for cleanup tasks that must always run
- Clear `depends_on_past` failures by manually marking the blocking instance as success
- For branching DAGs, ensure the branch not taken uses `trigger_rule='none_failed_min_one_success'`

**Prevention**:
- Document `trigger_rule` choices in task docstrings
- Avoid `depends_on_past=True` unless genuinely needed (it creates fragile dependency chains)
- Test branching logic with unit tests using `dag.test()` (Airflow 2.5+)

#### 4c: Pool Exhaustion

**Symptom**: Tasks stuck in `queued` state. Other tasks in the same pool are running at pool capacity.

**Diagnosis**:
1. Check pool usage: Admin > Pools in the Airflow UI
2. If `Running Slots` equals `Total Slots`, the pool is full
3. Identify which tasks are occupying slots and whether they are stuck or legitimately long-running

**Resolution**:
- Increase pool size if the external system can handle more concurrency
- Reduce `pool_slots` per task if individual tasks are over-allocated
- Kill stuck tasks that are holding slots without making progress
- Redistribute tasks across pools to balance load

**Prevention**:
- Set pool sizes based on the external system's concurrency limits (e.g. Snowflake warehouse max concurrent queries)
- Use `pool_slots=1` as default; increase only for tasks that genuinely need multiple slots
- Monitor pool utilisation with Airflow metrics (StatsD/Prometheus)

#### 4d: Scheduler Lag

**Symptom**: DAGs not starting on schedule. Scheduler heartbeat in the UI is stale (> 30 seconds). DAG run start times drifting from scheduled times.

**Diagnosis**:
1. Check scheduler logs for errors or warnings
2. Check `dag_dir_list_interval` and `min_file_process_interval` -- parsing too many DAG files can slow the scheduler
3. Check database latency -- slow metadata database queries stall the scheduler
4. Check if the number of active DAG runs exceeds `max_active_runs_per_dag`

**Resolution**:
- Reduce the number of DAG files by combining related DAGs or using dynamic DAG generation
- Increase scheduler resources (CPU/memory) or add a second scheduler (Airflow 2.0+ supports multiple schedulers)
- Optimise the metadata database: add indexes, upgrade instance size, enable connection pooling
- Set `max_active_runs_per_dag` appropriately (default 16; reduce to 1-3 for sequential DAGs)
- Clean up old DAG runs and task instances: `airflow db clean --clean-before-timestamp`

**Prevention**:
- Keep the number of DAG files under 200 per scheduler
- Use `.airflowignore` to exclude non-DAG Python files from parsing
- Monitor scheduler heartbeat interval and parse time with Airflow metrics
- Schedule database maintenance (vacuum/analyse) for the metadata database

---

## Runbook 5: Kafka Consumer Lag

### Symptoms

- Consumer group lag increasing over time (visible in Kafka monitoring tools, Burrow, or cloud console)
- Downstream systems reporting stale data
- Consumer logs showing `CommitFailedException` or rebalance warnings
- Processing latency between event production and consumption growing

### Diagnosis Steps

**Step 1: Measure current lag.**

```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
    --describe --group my-consumer-group
```

Check `LAG` column for each partition. If lag is growing on all partitions, the consumer cannot keep up with production rate. If lag is concentrated on specific partitions, data skew or stuck consumers are likely.

**Step 2: Check consumer throughput.**

- Monitor `records-consumed-rate` and `records-lag-max` JMX metrics
- Compare against producer throughput (`records-send-rate`)
- If consumer rate < producer rate, the consumer is falling behind

**Step 3: Check for rebalancing.**

Consumer logs showing `Revoking previously assigned partitions` or `JoinGroup` requests indicate frequent rebalancing. Each rebalance pauses consumption for all partitions in the group.

Common rebalance triggers:
- Consumer processing time exceeding `max.poll.interval.ms` (default 300 seconds)
- Consumer instances starting/stopping (deployment, auto-scaling)
- Network issues causing heartbeat timeouts (`session.timeout.ms`)

**Step 4: Check partition distribution.**

```bash
kafka-topics.sh --bootstrap-server broker:9092 \
    --describe --topic my-topic
```

Verify partition count and check for uneven message distribution across partitions.

### Resolution

| Problem | Resolution |
|---------|------------|
| **Consumer too slow (all partitions)** | Scale out: add more consumer instances (up to partition count). Increase `max.poll.records` if processing per record is fast. Batch processing where possible |
| **Rebalancing storms** | Increase `max.poll.interval.ms` to accommodate slow processing batches. Use `static.group.membership` (Kafka 2.3+) to reduce rebalance frequency during rolling deploys. Set `session.timeout.ms` = 30s and `heartbeat.interval.ms` = 10s |
| **Data skew (hot partitions)** | Re-key messages with a more evenly distributed key. Increase partition count (note: this resets consumer offsets for new partitions) |
| **Stuck consumer instance** | Restart the stuck consumer. If using Kubernetes, check liveness probes and resource limits. If the consumer OOMs, increase memory or reduce batch size |
| **Slow downstream processing** | Decouple consumption from processing: consume to an internal buffer (queue), process asynchronously. Use back-pressure mechanisms to throttle without losing messages |
| **Offset management issues** | If `enable.auto.commit=true` with slow processing, commits happen before processing completes -- a restart reprocesses from the last committed offset. Switch to manual commit after processing: `consumer.commitSync()` |

### Offset Management Best Practices

```python
# Manual commit pattern (Python confluent-kafka)
from confluent_kafka import Consumer

consumer = Consumer({
    'bootstrap.servers': 'broker:9092',
    'group.id': 'my-group',
    'enable.auto.commit': False,
    'auto.offset.reset': 'earliest',
    'max.poll.interval.ms': 600000,
})

consumer.subscribe(['my-topic'])

while True:
    messages = consumer.consume(num_messages=500, timeout=1.0)
    if messages:
        process_batch(messages)
        consumer.commit(asynchronous=False)
```

### Prevention

- Monitor consumer lag continuously with alerting thresholds (e.g. lag > 10,000 for > 5 minutes)
- Size consumer group to handle 2x expected peak throughput
- Use `static.group.membership` for Kubernetes deployments to avoid rebalancing on pod restarts
- Set `max.poll.interval.ms` based on worst-case processing time (not average)
- Implement dead-letter topics for messages that fail processing after retries -- do not block the consumer
- Test consumer resilience by simulating broker failures, network partitions, and consumer crashes in staging
- Partition count should exceed the maximum expected consumer count to allow scaling headroom

---

## General Troubleshooting Principles

1. **Read the error message carefully** -- most error messages contain the root cause. Do not guess; read.
2. **Check what changed** -- correlate the failure with recent deployments, configuration changes, or data volume spikes.
3. **Reproduce before fixing** -- understand the failure mode before applying a fix. Fixes without understanding often introduce new problems.
4. **Fix the root cause, not the symptom** -- increasing memory may stop an OOM, but the root cause might be a missing filter or a data skew issue.
5. **Document every incident** -- update the runbook with new symptoms and resolutions. Future engineers will thank you.
6. **Automate detection** -- if you diagnosed it manually once, add monitoring to detect it automatically next time.

---

**Related:** [[Data Engineering Best Practices]] | [[Snowflake Architecture & Key Concepts]] | [[PySpark Structured Streaming]] | [[Core dbt Fundamentals]] | [[Apache Airflow Core Concepts]] | [[Apache Kafka Deep Dive]]
