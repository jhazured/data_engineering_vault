#spark #architecture #fundamentals #distributed-computing #data-engineering #pyspark

## Overview

This note covers the foundational theory of Apache Spark as a distributed computing engine: its internal architecture, execution model, query optimisation pipeline, and deployment modes. It complements [[PySpark Core Concepts]], which focuses on the PySpark API surface (SparkSession, DataFrames, transformations, and actions). The theory here underpins the practical patterns found across the vault's PySpark notes.

> **Source**: _Spark: The Definitive Guide_ by Bill Chambers and Matei Zaharia (O'Reilly, 2018).

---

## Spark as a Unified Computing Engine

Spark is a **unified analytics engine** for large-scale data processing. Two design principles distinguish it from earlier big-data frameworks:

1. **Unified platform** -- a single engine spans batch processing, streaming ([[PySpark Streaming]]), machine learning, and graph analytics, rather than requiring separate systems for each workload.
2. **Computing engine, not storage** -- Spark intentionally decouples compute from storage. It reads from and writes to external systems (HDFS, S3, ADLS, Delta Lake, JDBC sources) but does not store data itself. This lets organisations adopt Spark without migrating away from existing storage layers.

Spark's Structured APIs (DataFrames, Datasets, Spark SQL) sit atop a lower-level RDD abstraction. All high-level code ultimately compiles down to RDD operations that execute across the cluster.

---

## Cluster Architecture

A running Spark Application consists of three types of process, each with a distinct role.

### The Driver Process

The **driver** is the controller of a Spark Application. It runs the user's `main()` function and is responsible for:

- Maintaining all state and metadata for the application
- Responding to user input (interactive or programmatic)
- Analysing, distributing, and scheduling work across executors
- Interfacing with the cluster manager to request physical resources

The driver holds the **SparkContext** (accessible via `SparkSession`), which represents the connection to the cluster and provides access to low-level APIs such as RDDs, accumulators, and broadcast variables.

### Executor Processes

**Executors** are worker processes launched on cluster nodes. Each executor has exactly two responsibilities:

1. Execute the tasks assigned to it by the driver
2. Report task status (success, failure, metrics) back to the driver

Each Spark Application receives its own set of executor processes -- executors are never shared across applications. This isolation prevents interference between concurrent applications on the same cluster.

### The Cluster Manager

The **cluster manager** maintains the pool of physical machines and allocates resources to Spark Applications on demand. It has its own "master" and "worker" abstractions, which map to physical machines (as opposed to Spark's driver and executors, which are JVM processes).

Spark supports several cluster managers:

| Cluster Manager | Key Characteristics |
|---|---|
| **Standalone** | Built into Spark; simplest to set up; suitable for single-team clusters |
| **Apache YARN** | Hadoop ecosystem standard; mature multi-tenant resource management; dominant in on-premises deployments |
| **Apache Mesos** | General-purpose cluster manager; fine-grained resource sharing; less commonly used today |
| **Kubernetes** | Container-native orchestration; growing adoption for cloud-native Spark deployments (see [[Kubernetes for Data Workloads]]) |

> The book (2018) lists three managers. Kubernetes support was added as GA in Spark 3.1 (2021) and is now the preferred option for cloud-native deployments.

---

## Execution Modes

The **execution mode** determines where the driver process runs relative to the cluster.

### Cluster Mode

The cluster manager launches the driver on a worker node inside the cluster, alongside the executors. The submitting client exits after submission. This is the **recommended mode for production** because it minimises network latency between driver and executors.

### Client Mode

The driver remains on the machine that submitted the application (the "gateway" or "edge" node). Only the executors run on cluster worker nodes. Useful for interactive development (notebooks, spark-shell) where the user needs direct access to driver output.

### Local Mode

The entire application -- driver and executors -- runs as threads on a single machine. No cluster is involved. Suitable for development, testing, and learning, but never appropriate for production workloads.

```bash
# Cluster mode submission
spark-submit \
  --class com.example.MyApp \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 10 \
  --executor-memory 8g \
  --executor-cores 4 \
  my-app.jar

# Client mode (interactive / debugging)
spark-submit \
  --master yarn \
  --deploy-mode client \
  my_script.py
```

---

## The Catalyst Optimiser

Catalyst is Spark's extensible query optimisation framework. It sits at the heart of the Structured APIs and is responsible for transforming user code into an efficient physical execution plan. Understanding Catalyst explains why DataFrames and Spark SQL outperform hand-written RDD code in most cases.

### Optimisation Pipeline

The pipeline proceeds through four phases:

#### 1. Analysis (Unresolved to Resolved Logical Plan)

User code (DataFrame operations or SQL) is parsed into an **unresolved logical plan** -- a tree of abstract transformations where table and column references have not yet been validated. The **analyser** resolves these references against the **catalogue** (Hive Metastore, Unity Catalog, or in-memory catalogue). If a referenced table or column does not exist, the plan is rejected at this stage.

#### 2. Logical Optimisation

The resolved logical plan passes through the **Catalyst Optimiser**, a rule-based engine that applies transformations to produce an **optimised logical plan**. Key optimisation rules include:

- **Predicate pushdown** -- filters are moved as close to the data source as possible, reducing the volume of data read
- **Column pruning** -- only the columns actually referenced downstream are retained
- **Constant folding** -- constant expressions are pre-computed at plan time
- **Join reordering** -- joins are reordered based on table statistics to minimise intermediate result sizes
- **Boolean simplification** -- redundant boolean expressions are simplified

Catalyst is **extensible**: libraries (e.g. Delta Lake) can register their own optimisation rules.

#### 3. Physical Planning

The optimised logical plan is converted into one or more **physical plans** -- concrete execution strategies specifying which algorithms to use (e.g. SortMergeJoin vs. BroadcastHashJoin). Spark evaluates candidate physical plans using a **cost model** informed by data statistics (e.g. table sizes, partition counts) and selects the cheapest.

#### 4. Code Generation

The selected physical plan is compiled into optimised Java bytecode via **whole-stage code generation**. Rather than interpreting each operator individually, Spark fuses multiple operators within a stage into a single function, dramatically reducing virtual function call overhead and improving CPU cache utilisation.

```
User Code (DataFrame / SQL)
        |
        v
  Unresolved Logical Plan
        |  [Analysis -- resolve against catalogue]
        v
  Resolved Logical Plan
        |  [Logical optimisation rules]
        v
  Optimised Logical Plan
        |  [Physical planning + cost model]
        v
  Physical Plan(s) --> Best Physical Plan
        |  [Whole-stage code generation]
        v
  RDD Execution on Cluster
```

### Why This Matters for PySpark

Because Catalyst operates on the logical plan rather than on user code, **DataFrame and SQL operations in Python achieve the same performance as Scala**. The optimiser sees the same abstract plan regardless of language. This is not the case for RDD operations or UDFs, which bypass Catalyst (see [[PySpark Performance Optimization]]).

---

## Tungsten Execution Engine

Tungsten is Spark's low-level execution backend, focused on maximising hardware efficiency. While Catalyst handles logical optimisation, Tungsten handles physical execution performance.

### Key Capabilities

- **Binary in-memory format** -- Spark stores data in its own compact binary representation rather than as JVM objects. This avoids Java object overhead, reduces garbage collection pressure, and enables more data to fit in memory.
- **Off-heap memory management** -- data can be stored outside the JVM heap, giving Spark direct control over memory layout and avoiding GC pauses.
- **Cache-aware computation** -- algorithms are designed to exploit CPU L1/L2/L3 cache locality, processing data in tight loops over contiguous memory.
- **Whole-stage code generation** -- as described above, Tungsten fuses operators into single functions, producing code similar to hand-written, loop-based programs.

The practical implication is that Structured API operations (DataFrames, Datasets, SQL) benefit from Tungsten automatically, whereas raw RDD operations do not. This is a core reason the Structured APIs are recommended for virtually all workloads.

---

## Jobs, Stages, and Tasks

Understanding Spark's execution hierarchy is essential for performance tuning and debugging via the Spark UI.

### Hierarchy

```
Spark Application
  └── Job (one per action)
        └── Stage (bounded by shuffle operations)
              └── Task (one per partition per stage)
```

### Jobs

An **action** (e.g. `collect()`, `count()`, `write()`) triggers a **job**. Each job corresponds to a complete computation from source data to result. In general, one action produces one job.

### Stages

A job is divided into **stages** at shuffle boundaries. Within a stage, Spark pipelines as many narrow transformations as possible without moving data across the network. A new stage begins whenever a **shuffle** (wide transformation) is required -- e.g. `groupBy()`, `join()`, `repartition()`, `orderBy()`.

The number of tasks in a stage equals the number of input partitions for that stage. After a shuffle, the number of output partitions is controlled by `spark.sql.shuffle.partitions` (default: 200).

### Tasks

A **task** is the smallest unit of work: a single stage applied to a single partition, running on a single executor core. Tasks within a stage execute in parallel across the cluster. The degree of parallelism is therefore determined by the number of partitions.

### Practical Example

```python
df1 = spark.range(2, 10_000_000, 2)   # 8 partitions (default)
df2 = spark.range(2, 10_000_000, 4)   # 8 partitions (default)

step1 = df1.repartition(5)            # Shuffle -> new stage (5 partitions)
step2 = df2.repartition(6)            # Shuffle -> new stage (6 partitions)
step3 = step1.selectExpr("id * 5 as id")
step4 = step3.join(step2, ["id"])     # Shuffle -> new stage (200 partitions)
result = step4.selectExpr("sum(id)")

result.collect()  # Action triggers execution
```

This produces six stages: two for the initial ranges (8 tasks each), two for the repartitions (5 and 6 tasks), one for the join (200 tasks from default shuffle partitions), and one for the final aggregation (1 task).

---

## RDD Fundamentals

While the Structured APIs are recommended for most work, understanding RDDs is important because they remain the foundation upon which everything else is built.

### What is an RDD?

A **Resilient Distributed Dataset** is an immutable, partitioned collection of records that can be operated on in parallel. "Resilient" refers to fault tolerance: if a partition is lost due to node failure, Spark can recompute it from its **lineage** -- the chain of transformations that produced it.

### Key Properties

1. **Immutability** -- transformations produce new RDDs; existing RDDs are never modified
2. **Partitioning** -- data is split into partitions distributed across the cluster
3. **Lineage** -- each RDD records the transformations applied to its parent, enabling fault recovery without data replication
4. **Lazy evaluation** -- transformations are recorded but not executed until an action is called
5. **Type flexibility** -- RDDs can hold arbitrary objects (no schema requirement)

### When RDDs Are Still Appropriate

- Working with unstructured data where no schema naturally applies
- Requiring fine-grained control over physical data placement (custom partitioners)
- Operations that cannot be expressed through SQL or DataFrame APIs
- Interacting with legacy Spark code

### When to Avoid RDDs

RDDs bypass the Catalyst optimiser and Tungsten execution engine. Operations on RDDs:

- Do not benefit from predicate pushdown, column pruning, or join reordering
- Store data as serialised JVM objects (higher memory overhead, more GC pressure)
- Require manual optimisation of shuffle operations and data layout

For structured and semi-structured data, DataFrames will almost always outperform equivalent RDD code.

---

## Structured APIs: DataFrames, Datasets, and Spark SQL

The Structured APIs provide a higher-level abstraction over RDDs. They share a common optimisation pipeline (Catalyst + Tungsten) and differ primarily in type safety and language availability.

### DataFrames

A DataFrame is a distributed collection of data organised into named columns -- conceptually equivalent to a table in a relational database. Internally, a DataFrame is a `Dataset[Row]`, where `Row` is Spark's internal binary representation.

In PySpark, **DataFrames are the primary abstraction**. There is no Dataset API in Python because Python is dynamically typed, making compile-time type safety impossible.

### Datasets (Scala/Java Only)

Datasets add compile-time type safety to the DataFrame API. A `Dataset[T]` associates each record with a specific JVM class, enabling the compiler to catch type errors before runtime. Not available in PySpark.

### Spark SQL

Spark SQL allows users to express queries as SQL strings. These queries pass through the same Catalyst pipeline as DataFrame operations, so there is **no performance difference** between equivalent DataFrame and SQL code. SQL is particularly useful for:

- Analysts more comfortable with SQL syntax
- Complex analytical queries with CTEs, window functions, and subqueries
- Interoperability with BI tools via JDBC/ODBC

### Row: The Internal Format

The `Row` type is Spark's internal representation, stored in Tungsten's optimised binary format. When you operate on DataFrames in any language, Spark works with `Row` objects in this compact format rather than language-native objects, which is why Python DataFrame operations achieve near-Scala performance.

---

## Application Life Cycle

A Spark Application follows a well-defined life cycle, whether submitted via `spark-submit`, a notebook, or an orchestration tool.

### 1. Client Request

The user submits an application (JAR, Python script) to the cluster manager, requesting resources for the driver process.

### 2. Launch

The driver starts and initialises a `SparkSession`. It communicates with the cluster manager to request executor processes. The cluster manager launches executors on worker nodes and reports their locations back to the driver.

### 3. Execution

The driver translates user code into jobs, stages, and tasks. Tasks are scheduled onto executors, which execute them and report results back. During execution:

- The driver maintains the DAG of operations
- Executors read data from sources, perform transformations, and write shuffle data
- The driver coordinates shuffle exchanges between stages

### 4. Completion

When the application finishes (or fails), the driver exits and the cluster manager reclaims executor resources.

---

## Broadcast Variables and Accumulators

These are Spark's two types of **shared variable**, used for specific communication patterns that the standard task model does not cover.

### Broadcast Variables

A broadcast variable distributes a read-only value to every worker node once, rather than serialising it with every task. This is critical for performance when a large lookup table or ML model needs to be available on every executor -- without broadcasting, the value would be shipped once per task, wasting network bandwidth.

### Accumulators

Accumulators are write-only variables that executors can add to. Only the driver can read the accumulated value. Common uses include counting malformed records or tracking custom metrics during job execution.

See [[PySpark Advanced Features]] for implementation details.

---

## Spark Configuration Hierarchy

Spark configuration values can be set at multiple levels. When the same property is set at multiple levels, the following precedence applies (highest to lowest):

1. **SparkConf** set in application code (`.config()` on SparkSession builder)
2. **Command-line flags** passed to `spark-submit` (e.g. `--executor-memory`)
3. **spark-defaults.conf** on the cluster
4. **Default values** built into Spark

Key configuration categories:

| Category | Examples |
|---|---|
| **Execution** | `spark.sql.shuffle.partitions`, `spark.default.parallelism` |
| **Memory** | `spark.executor.memory`, `spark.driver.memory`, `spark.memory.fraction` |
| **Serialisation** | `spark.serializer` (KryoSerializer recommended for network-intensive workloads) |
| **Shuffle** | `spark.sql.adaptive.enabled`, `spark.sql.adaptive.coalescePartitions.enabled` |

See [[PySpark Performance Optimization]] for tuning guidance.

---

## Key Takeaways

1. Spark is a **computing engine**, not a storage system -- it reads from and writes to external sources
2. The **driver** orchestrates; **executors** execute; the **cluster manager** allocates resources
3. The **Catalyst optimiser** transforms logical plans into efficient physical plans, making DataFrame and SQL code performant regardless of language
4. **Tungsten** maximises hardware efficiency through binary formats, off-heap memory, and code generation
5. All high-level operations compile down to **RDD transformations**, but using RDDs directly forgoes Catalyst and Tungsten benefits
6. **Cluster mode** is preferred for production; **client mode** for interactive work; **local mode** for development only
7. Understanding the **job > stage > task** hierarchy is essential for debugging and performance tuning

---

## Related Notes

- [[PySpark Core Concepts]] -- SparkSession, DataFrames, transformations vs. actions, narrow vs. wide
- [[PySpark Performance Optimization]] -- tuning shuffle partitions, memory, joins, and serialisation
- [[PySpark Streaming]] -- Structured Streaming built on the same Catalyst engine
- [[PySpark Advanced Features]] -- broadcast variables, accumulators, UDFs
- [[PySpark Production Engineering]] -- deployment patterns and operational concerns
- [[Kubernetes for Data Workloads]] -- container-native Spark deployment
- [[Databricks Platform]] -- managed Spark with additional optimisations (Photon, Delta)

---

## Tags

#spark #architecture #catalyst #tungsten #distributed-computing #cluster-management #fundamentals #pyspark

---

_Last Updated: 2026-03-15_
