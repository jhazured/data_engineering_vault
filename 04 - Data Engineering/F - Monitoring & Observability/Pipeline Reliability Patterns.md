---
tags:
  - data-engineering
  - reliability
  - monitoring
  - observability
  - pipelines
  - fault-tolerance
sources:
  - "Rebuilding Reliable Data Pipelines Through Modern Tools (Malaska, 2019)"
  - "Designing Data-Intensive Applications (Kleppmann, 2017)"
created: 2026-03-15
---

# Pipeline Reliability Patterns

This note consolidates reliability patterns for data pipelines, covering DAG anatomy, bottleneck identification, failure modes, retry strategies, isolation, metrics collection, SLOs, and monitoring culture. Cross-references: [[Common Data Engineering Patterns]], [[Pipeline Observability & Monitoring]], [[Apache Airflow & Orchestration Patterns]].

---

## 1. DAG Anatomy

A Directed Acyclic Graph (DAG) is the foundational abstraction for all data processing. Every pipeline -- whether a single Spark job or a multi-system workflow -- can be expressed as a DAG of nodes and edges with no cycles.

### Single-Job DAGs

A single-job DAG represents one SQL statement or one Spark job. Each node is a unique operation, and edges represent data transmission through memory or network.

Key stages in a single-job DAG:

- **Inputs** -- reading from sources with a defined level of parallelism (e.g., number of files, partitions, or stream offsets)
- **Map-only processes** -- record-level transformations that require no data exchange between nodes (map, flatMap, forEach)
- **Shuffle** -- redistributing data across partitions by a key, involving serialisation, network transfer, and sorting/grouping
- **Repartitioning** -- changing the number of partitions without the overhead of sorting or grouping
- **Outputs** -- writing results to storage

Advanced single-job techniques include:

- **Broadcast joins** -- sending a small dataset to every executor to avoid shuffling the larger dataset
- **Remote cache lookups** -- fetching join data from a low-latency cache instead of shuffling
- **Checkpointing** -- persisting intermediate state so that long-running jobs (30+ minutes) can recover without restarting from scratch

### Pipeline DAGs

A pipeline DAG composes multiple jobs, where each node is either a **processing node** or a **storage node**:

- Processing nodes always read from and write to storage nodes; there are no direct processing-to-processing edges
- Pipelines always begin and end with storage nodes
- **Storage reuse** through data marts allows multiple downstream processing nodes to share the output of expensive or consistency-critical operations (e.g., a pre-joined, pre-sorted dataset)

### Dependency Types

Within a pipeline DAG, dependencies fall into several categories:

- **Sequential** -- Job B cannot start until Job A completes (e.g., enrichment depends on ingestion)
- **Fan-out** -- One job's output feeds multiple downstream consumers
- **Fan-in** -- A job requires outputs from several upstream jobs before it can proceed
- **Conditional** -- A job fires only if certain data-quality or threshold checks pass

Dependency management tools such as [[Apache Airflow & Orchestration Patterns|Apache Airflow]] provide DAG composition, dependency triggering, notifications, and audit trails for critical paths.

---

## 2. Bottleneck Identification

Bottlenecks silently degrade performance and can bring otherwise correct pipelines to a halt. Identifying the *type* of bottleneck is the first step to resolving it.

### Data Skew

Skew occurs when key values are unevenly distributed, causing some partitions to process far more data than others. The classic indicator is a small number of tasks running much longer than the rest.

Mitigation strategies:

- **Salting keys** -- appending a random suffix to hot keys to spread load, then aggregating a second time
- **Double reduce** -- first reduce at a finer granularity (e.g., postcode), then roll up to the coarser level (e.g., region)
- **Filtering hot keys** -- process the top-N skewed keys separately with higher parallelism

### Resource Contention

In shared environments, jobs compete for CPU, memory, disk I/O, and network bandwidth. Symptoms include high queue wait times and locked-but-unused resources.

- Monitor CPU utilisation per executor: low CPU with slow progress suggests round-trip or I/O blocking
- Track locked-but-unused resources to identify over-provisioned or poorly parallelised jobs
- Use container or process isolation (see Section 5) to limit the blast radius of resource-hungry jobs

### I/O Bottlenecks

- **Input bottlenecks** -- sources that do not support parallel reads (e.g., single-stream JDBC/ODBC) constrain throughput regardless of available compute
- **Output bottlenecks** -- writing to systems with limited ingest capacity or poor partitioning
- **Serialisation costs** -- converting between storage formats (e.g., Parquet to JSON) is CPU-intensive; keeping data in columnar formats reduces overhead

### Network Saturation

When data must travel over the wire -- between regions, between cloud and on-premises, or during shuffles -- the network can become the bottleneck.

Remediation approaches:

1. **Compression** -- LZO (~2x), GZIP (~8x), BZIP (~9.5x); columnar formats like Parquet can yield an additional 30% reduction
2. **Send less data** -- pull only changed records (CDC), nest repeated fields, increase batch intervals
3. **Move the compute** -- co-locate processing with the data in the same region or availability zone
4. **Increase parallelism** -- more threads can saturate the available bandwidth more effectively

### Driver Bottlenecks

In frameworks like Spark, work accidentally placed on the driver (via `collect`, `parallelize`, or `take`) creates a single-threaded bottleneck. If the driver CPU is high while executor CPUs are low, a code review is warranted.

---

## 3. Failure Modes

Kleppmann draws a critical distinction: "A fault is one component deviating from its spec, whereas a failure is when the system as a whole stops providing the required service." The goal is to design fault-tolerant systems that prevent faults from escalating into failures.

### Transient vs Permanent Failures

| Characteristic | Transient | Permanent |
|---|---|---|
| **Examples** | Network blip, GC pause, temporary resource exhaustion, node restart | Schema mismatch, null in non-nullable field, corrupt data, constraint violation |
| **Retry useful?** | Yes | No -- retrying wastes resources and delays diagnosis |
| **Detection** | Timeout expiry, intermittent error codes | Deterministic exception on same input |
| **Response** | Automatic retry with backoff | Route to dead-letter queue, alert, require human intervention |

Spark, for instance, retries failed tasks a configurable number of times. For input data errors, the job fails on the same record every attempt, burning hours of wasted compute. Cancelling retries early for deterministic failures is essential.

### Partial Failures

In distributed systems, some nodes may succeed while others fail. Kleppmann notes that unlike supercomputers (which checkpoint and restart entirely), cloud-native systems must handle partial failure gracefully:

- MapReduce tolerates individual task failures by retrying at the task level; output from failed tasks is discarded
- Streaming engines like Flink use periodic checkpoints; on failure, operators roll back to the last checkpoint
- Spark's RDD lineage allows recomputation of lost partitions from their parent data

### Cascading Failures

A failure in one pipeline job can propagate downstream through dependent jobs. Without dependency management:

- Downstream jobs may start on incomplete or missing data
- Resource contention from retrying upstream jobs starves downstream jobs
- A single slow or failing job can trigger a backlog across the entire pipeline

Mitigation requires explicit dependency graphs, failure notifications to downstream consumers, and the ability to pause or skip dependent stages. See [[Apache Airflow & Orchestration Patterns]] for orchestration approaches.

---

## 4. Retry Strategies

### Exponential Backoff

Kleppmann warns that naive retries on overload errors create a feedback loop that worsens the problem. Best practices:

- Start with a short delay (e.g., 100ms) and double it on each subsequent retry
- Add jitter (randomised delay) to prevent thundering-herd effects when many clients retry simultaneously
- Cap the maximum backoff interval to bound worst-case latency
- Distinguish transient errors (retry) from permanent errors (do not retry)

### Circuit Breaker

When a downstream system is consistently failing, continuing to send requests wastes resources and delays recovery. The circuit breaker pattern:

1. **Closed** -- requests flow normally; failures are counted
2. **Open** -- after a threshold of failures, all requests are short-circuited with an immediate error
3. **Half-open** -- after a cooldown period, a limited number of probe requests are allowed through to test recovery

This prevents cascading failures and gives the failing system time to recover.

### Dead-Letter Queues

For records that cannot be processed (malformed data, schema mismatches, unexpected nulls), routing to a dead-letter queue (DLQ) preserves the failing records for later inspection without blocking the rest of the pipeline.

Key data-quality checks to run before processing:

- Null checking on required fields
- Type validation (strings where numbers are expected)
- Schema conformance checks
- Range validation (negative values, division by zero, date format mismatches)

### Idempotent Operations

Kleppmann dedicates significant attention to idempotence as a reliability mechanism. An idempotent operation produces the same result whether executed once or multiple times. This is critical for safe retries:

- **Naturally idempotent** -- setting a key-value pair to a fixed value
- **Made idempotent** -- including the source offset or transaction ID with each write, so duplicate writes are detected and skipped

Idempotence, combined with deterministic processing and log-based message replay, enables effectively-once semantics without the overhead of distributed transactions.

---

## 5. Container and Process Isolation

Isolation prevents one job from starving or crashing another. Malaska identifies three levels, each trading cost efficiency for safety.

### Node Isolation

- Each job runs on dedicated nodes; maximum isolation, minimum resource efficiency
- Cloud spin-up times (10-20 minutes for EMR) add significant overhead for short jobs
- Rarely justified on-premises due to hardware cost

### Container Isolation

- Jobs share physical nodes but are allocated dedicated CPU and memory within containers (e.g., YARN, Kubernetes, Mesos)
- Advantages: faster startup (seconds), better utilisation, consistent environments, centralised monitoring
- Limitations: containers cannot isolate disk I/O or network bandwidth -- noisy-neighbour problems persist at these layers

Key scheduling configurations for container environments:

- **Resource thresholds** -- cap the CPU/memory a single job or user can claim
- **Priority weighting** -- ensure critical jobs receive a larger share of contested resources
- **Preemptive killing** -- reclaim resources from low-priority long-running jobs for high-priority arrivals
- **Kill triggers** -- terminate jobs exceeding a maximum runtime to prevent resource hoarding
- **Autoscaling** -- scale the underlying node pool based on queue depth and predicted load

### Process Isolation

- Multiple jobs share the same container/process (e.g., Spark Thrift Server, Presto)
- Startup overhead measured in milliseconds -- ideal for high-concurrency analytical workloads and many small jobs
- Weakest isolation: a misbehaving job can affect co-tenants

### Resource Budgets

Regardless of isolation level, set explicit resource budgets:

- Memory limits per job (with headroom -- Java applications routinely exceed their allocation)
- Concurrency limits to prevent resource fragmentation
- Queue length and wait-time monitoring to detect capacity shortfalls early
- Never let resource failures pass unaddressed -- sporadic out-of-memory errors indicate a latent misconfiguration

---

## 6. Metrics to Collect

Malaska organises pipeline metrics into several tiers. Collecting them progressively -- not all at once -- is key to sustainable adoption.

### Throughput Metrics

- **Record counts** in and out of each job
- **Bytes processed** per unit time
- **Job execution duration** (start time, end time, elapsed)
- **Queue wait time** -- how long a job waited before resources were available

### Latency Metrics

- **End-to-end pipeline latency** -- from source event time to availability in the destination
- **Per-stage latency** -- time spent in each DAG stage (input, shuffle, output)
- **Round-trip latency** to external systems (caches, databases, schema registries)

### Error Rate Metrics

- **Job failure count** broken down by cause: resource failure, coding error, data-quality issue
- **Resource failure frequency** -- how often compute or storage nodes fail, time to recover
- **SLA misses** -- count of occasions where pipeline output was late or incomplete

### Data Completeness Metrics

- **Record counts** compared against source expectations
- **Column cardinality levels** -- detecting unexpected drops in uniqueness
- **Column top-N values** -- spotting distribution shifts that may indicate upstream issues
- **File sizes** -- anomalous file sizes (too large or too small) signal ingestion problems

### Resource Utilisation Metrics

- **CPU utilisation** per node, container, and job
- **Locked resources** vs **locked-but-unused resources** -- the gap reveals over-provisioning
- **Degrees of skew** -- ratio of slowest to fastest partition within a shuffle stage
- **Shuffle keys used** -- repeated shuffles on the same key indicate design inefficiency

### Cost Metrics

- **Processing cost** -- per-job or proportional share in shared environments
- **Storage cost** -- including replication across data marts
- **Transmission cost** -- cross-region and cross-cloud data movement (often the hidden budget killer)
- **Cost per dataset** -- total cost to produce, store, and serve a dataset, weighed against its business value

### Operational Metrics

- **Effort to deploy** -- number of manual steps per release (each step is a source of error)
- **Issue ticket count and coverage** -- percentage of real incidents captured by tickets
- **Run-book coverage** -- percentage of incidents with documented resolution procedures
- **Time to identify** -- elapsed time from incident start to awareness
- **Time to resolve** -- elapsed time from ticket creation to confirmed resolution

---

## 7. Pipeline SLOs

Service-level objectives (SLOs) translate business requirements into measurable reliability targets. Without explicit SLOs, "good enough" is undefined and arguments about reliability become subjective.

### Latency Targets

- Define maximum acceptable end-to-end latency from data production to consumption
- Account for the critical path through the pipeline DAG -- the longest chain of dependent jobs determines the minimum possible latency
- Use historical execution times to set realistic targets, and track percentiles (p50, p95, p99) rather than averages

### Freshness Targets

- Freshness measures how recent the data is at the point of consumption
- Batch pipelines: freshness is bounded by the scheduling interval plus execution time
- Streaming pipelines: freshness is bounded by processing delay plus any windowing or checkpointing intervals
- Distinguish between **event-time freshness** (time since the event occurred) and **processing-time freshness** (time since the data was ingested)

### Completeness Targets

- Define the acceptable percentage of source records that must appear in the destination (e.g., 99.9% within the SLA window)
- Monitor record counts at each stage to detect data loss early
- Account for intentional filtering (e.g., dead-letter routing) when calculating completeness

### Defining and Communicating SLOs

- Tie SLOs to downstream consumer needs: a dashboard refreshing every hour has different requirements from a fraud-detection system operating in near-real-time
- Publish SLOs alongside the pipeline's dependency graph so that upstream and downstream teams share a common understanding
- Track SLO adherence over time using error budgets -- if the pipeline has consumed its error budget, prioritise reliability work over feature development

---

## 8. Monitoring Culture

Collecting metrics is necessary but not sufficient. Malaska emphasises that culture determines whether metrics translate into action.

### What to Collect

Start with the metrics that address your most painful failure modes (see Section 6 for the full taxonomy). Prioritise:

1. **Job execution events** -- who ran what, when, with which version of the code
2. **Job execution information** -- durations, inputs, outputs, data lineage
3. **Job meta information** -- SLA requirements, run-book references, downstream dependencies, fallback plans
4. **Data-about-the-data** -- record counts, cardinality, distribution statistics
5. **Job optimisation signals** -- CPU utilisation, skew, shuffle keys, locked resources
6. **Cost signals** -- processing, storage, and transmission costs per dataset

### How to React to Signals

- **Make collection easy and transparent** -- if developers must manually instrument every job, coverage will always have gaps. Aim for automatic, framework-level collection wherever possible
- **Make collection a deployment requirement** -- no job reaches production without baseline metric emission
- **Compare run times against thresholds** -- flag jobs whose duration deviates significantly from historical norms. Feed these signals into anomaly-detection models over time
- **Label operational events** -- classify alerts as noise, contextual, or high-value. Over time, these labels train models to surface genuine incidents and suppress false alarms
- **Link events to resolutions** -- build a knowledge base connecting alert patterns to proven remediation steps, enabling faster response and eventual automation
- **Demand data in post-incident reviews** -- every incident review should produce charts, root-cause analysis, and updated alert configurations. Track how long it takes to gather this information; shrinking that time is itself a reliability improvement
- **Track coverage gaps** -- after every incident, ask whether the metrics in place were sufficient to detect it. If not, add the missing signal before the next occurrence

### Building the Feedback Loop

Kleppmann's insight that "deliberately inducing faults ensures the fault-tolerance machinery is continually exercised" (cf. Netflix Chaos Monkey) applies equally to monitoring. Periodically verify that:

- Alerts fire when expected (inject synthetic failures)
- Run-books are current and actionable
- Escalation paths are understood by on-call engineers
- SLO dashboards reflect the metrics that matter to downstream consumers

The ultimate goal is a self-improving system: incidents generate data, data generates insights, insights generate preventive measures, and preventive measures reduce future incidents.

---

## Key Takeaways

1. Model every pipeline as a DAG of DAGs -- single-job DAGs nested within pipeline DAGs -- to reason about optimisation at every level
2. Classify bottlenecks (skew, I/O, network, driver) before reaching for more hardware; the fix depends on the cause
3. Distinguish transient from permanent failures and respond differently: retry with backoff for the former, dead-letter and alert for the latter
4. Idempotent operations plus deterministic replay enable effectively-once semantics without heavyweight distributed transactions
5. Choose the right isolation level (node, container, process) based on the trade-off between safety and resource efficiency
6. Collect metrics progressively, starting with what hurts most, and make collection automatic rather than opt-in
7. Define explicit SLOs for latency, freshness, and completeness -- then track adherence with error budgets
8. Build a monitoring culture that demands data in incident reviews, labels operational events, and continuously closes coverage gaps

---

## Related Notes

- [[Common Data Engineering Patterns]]
- [[Pipeline Observability & Monitoring]]
- [[Apache Airflow & Orchestration Patterns]]
