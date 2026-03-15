tags: #hadoop #mapreduce #hdfs #yarn #distributed-systems #big-data #batch-processing

# Hadoop & MapReduce Fundamentals

Core concepts from "Hadoop: The Definitive Guide" (Tom White, 4th Ed.) and the original MapReduce paper (Dean & Ghemawat, OSDI 2004). Covers HDFS architecture, the MapReduce programming model, YARN resource management, and the broader Hadoop ecosystem.

---

## The MapReduce Programming Model

### Origins: The Google MapReduce Paper (2004)

MapReduce was designed at Google to address the complexity of distributing computations across thousands of commodity machines. The key insight: express computations as two user-defined functions — **Map** and **Reduce** — and let the framework handle parallelisation, fault tolerance, data distribution, and load balancing.

**Formal types:**

```
map    (k1, v1)        -> list(k2, v2)
reduce (k2, list(v2))  -> list(v2)
```

**Canonical example — word count:**

```
map(String key, String value):
    for each word w in value:
        EmitIntermediate(w, "1")

reduce(String key, Iterator values):
    int result = 0
    for each v in values:
        result += ParseInt(v)
    Emit(AsString(result))
```

### Execution Flow (Six Steps)

1. **Split** — input files are divided into M splits (typically 16-64 MB each)
2. **Assign** — a master assigns map and reduce tasks to idle workers
3. **Map** — workers read splits, parse key/value pairs, pass to user Map function; intermediate pairs buffered in memory
4. **Partition & spill** — buffered pairs are partitioned into R regions (one per reducer) using `hash(key) mod R`, sorted by key, and written to local disc
5. **Shuffle & sort** — reduce workers fetch their partitions from map workers via RPC, then merge-sort by key
6. **Reduce** — for each unique key, the reduce function is invoked; output is written to the final output file

The master coordinates everything: tracking task state (idle, in-progress, completed), propagating intermediate file locations from mappers to reducers, and detecting worker failures via periodic heartbeats.

### Fault Tolerance

| Failure type | Recovery mechanism |
|---|---|
| Worker failure | Master re-executes completed map tasks (output is on local disc) and in-progress tasks on other workers. Completed reduce tasks do not need re-execution (output is in GFS/HDFS). |
| Master failure | Checkpoints of master state; abort and retry if master dies (single point of failure in original design). |
| Stragglers | Near completion, the master launches **backup executions** of remaining in-progress tasks. Whichever copy finishes first is used. |

### Key Optimisations

- **Data locality** — schedule map tasks on or near machines holding input data replicas; when running on a significant fraction of the cluster, most input is read locally
- **Combiner function** — a local pre-aggregation step on the map side that reduces data transferred to reducers (must be commutative and associative, e.g. `max`, `sum`, but not `mean`)
- **Compression** — compressing map output reduces disc I/O and network transfer; LZO, LZ4, or Snappy are fast choices

---

## HDFS — Hadoop Distributed Filesystem

HDFS is designed for **very large files** (hundreds of MB to petabytes), **streaming data access** (write-once, read-many), and **commodity hardware** (expects node failures). See also [[Distributed Systems Fundamentals]].

### Design Trade-offs

HDFS is **not** suited for:

- **Low-latency access** — optimised for throughput, not millisecond reads (use HBase instead)
- **Lots of small files** — each file/directory/block consumes ~150 bytes of NameNode memory
- **Multiple writers or random writes** — append-only, single-writer model

### Block Storage

- Default block size: **128 MB** (large to minimise seek overhead relative to transfer time)
- Files smaller than one block do not consume a full block of underlying storage
- Blocks are the unit of replication: each block is replicated to **3** separate machines by default
- Block abstraction enables files larger than any single disc and simplifies storage management

### NameNode and DataNode Architecture

```
                    +-----------+
   Client -------> | NameNode  |  (master)
                    +-----------+
                   /      |      \
          +--------+  +--------+  +--------+
          |DataNode|  |DataNode|  |DataNode|   (workers)
          +--------+  +--------+  +--------+
```

**NameNode (master):**
- Maintains the filesystem tree, metadata, and file-to-block mapping
- Stores namespace image and edit log persistently on local disc
- Block locations are **not** persisted — reconstructed from DataNode reports at startup
- Single point of failure in basic configuration

**DataNodes (workers):**
- Store and retrieve blocks on demand
- Report block lists to the NameNode periodically via heartbeats

### NameNode Resilience

1. **Metadata replication** — NameNode writes persistent state to multiple filesystems (local disc + remote NFS) synchronously and atomically
2. **Secondary NameNode** — periodically merges namespace image with edit log (checkpoint); not a hot standby
3. **High Availability (Hadoop 2+)** — active-standby pair of NameNodes sharing an edit log via a Quorum Journal Manager (QJM); DataNodes send block reports to both; failover managed by ZooKeeper

### Data Flow — Reading a File

1. Client calls `open()` on `DistributedFileSystem`
2. NameNode returns DataNode addresses for the first blocks, sorted by proximity
3. Client reads from the closest DataNode for each block; transparently fails over to another replica on error
4. Blocks are read sequentially; NameNode is contacted for additional block batches as needed

### Data Flow — Writing a File

1. Client calls `create()` — NameNode creates file entry (no blocks yet)
2. `DFSOutputStream` splits data into packets; `DataStreamer` asks NameNode to allocate blocks
3. Packets are written to a **replication pipeline** (e.g. 3 DataNodes in sequence)
4. Each DataNode acknowledges; packets removed from ack queue only after all replicas confirm
5. On DataNode failure: pipeline is closed, failed node removed, remaining data written to surviving nodes; NameNode arranges async re-replication

### Replica Placement Strategy

- **1st replica** — same node as the client (or random node if client is external)
- **2nd replica** — different rack (off-rack), random node
- **3rd replica** — same rack as 2nd, different node
- Balances reliability (two racks), write bandwidth (single switch traversal), and read performance (two racks to choose from)

---

## YARN — Yet Another Resource Negotiator

YARN (Hadoop 2+) decouples resource management from the MapReduce programming model, enabling multiple distributed computing frameworks (MapReduce, Spark, Tez, Flink) to share a single cluster. See also [[PySpark Core Concepts]] for Spark's integration with YARN.

### Core Daemons

| Component | Role |
|---|---|
| **ResourceManager** | One per cluster. Allocates resources, schedules applications. |
| **NodeManager** | One per node. Launches and monitors containers. |
| **ApplicationMaster** | One per application. Negotiates resources with RM, coordinates tasks with NMs. |
| **Container** | A constrained set of resources (memory, CPU) for an application process. May be a Unix process or Linux cgroup. |

### Application Lifecycle

1. Client contacts ResourceManager to launch an ApplicationMaster
2. ResourceManager allocates a container on a NodeManager for the AM
3. AM requests additional containers from the ResourceManager (with locality constraints)
4. AM launches tasks in containers via NodeManagers
5. Tasks report progress to the AM; AM reports to the ResourceManager

### YARN vs MapReduce 1

| MapReduce 1 | YARN |
|---|---|
| JobTracker (scheduling + monitoring) | ResourceManager + ApplicationMaster (separated concerns) |
| TaskTracker | NodeManager |
| Fixed-size slots (map slots, reduce slots) | Fine-grained containers (any task can use any resource) |
| ~4,000 node scalability ceiling | Designed for 10,000+ nodes |
| MapReduce only | Multi-tenant: MR, Spark, Tez, Flink, etc. |

### Scheduler Options

- **FIFO Scheduler** — simple queue; first submitted, first served; not suitable for shared clusters
- **Capacity Scheduler** — dedicated queues per organisation with configurable capacity; supports queue elasticity (borrowing idle capacity from other queues)
- **Fair Scheduler** — dynamically balances resources between all running jobs; no reserved capacity needed; each job gets its fair share

**Delay scheduling** — both Capacity and Fair Schedulers support waiting briefly for a data-local container rather than immediately assigning a remote one, improving data locality.

---

## MapReduce Job Execution on YARN (In Detail)

### Shuffle and Sort — The Heart of MapReduce

The shuffle transfers map outputs to reducers and is the most network-intensive phase.

**Map side:**

1. Map output written to a **circular memory buffer** (100 MB default)
2. When buffer reaches 80% threshold, a background thread spills to disc
3. Before spilling: data is **partitioned** by reducer, **sorted** by key within each partition, and optionally run through the **combiner**
4. Multiple spill files are merged into a single sorted, partitioned output file
5. Output served to reducers over HTTP

**Reduce side:**

1. **Copy phase** — reducer fetches its partition from each completed mapper (5 parallel copy threads by default)
2. **Sort/merge phase** — copied segments are merged (in-memory or on disc) maintaining sort order; uses a merge factor of 10 by default
3. **Reduce phase** — reduce function invoked for each key group; output written directly to HDFS (first replica on the local DataNode)

### Performance Tuning Principles

- Minimise spills on the map side (increase `mapreduce.task.io.sort.mb`)
- Use combiners to reduce shuffle volume
- Compress map output (`mapreduce.map.output.compress = true`)
- On the reduce side, keep intermediate data in memory where possible
- Monitor the `SPILLED_RECORDS` counter to identify excessive spilling

### Speculative Execution

If a task is running significantly slower than peers (a "straggler"), Hadoop launches a backup copy. Whichever finishes first is used. This mirrors the backup tasks mechanism from the original Google paper.

---

## File Formats in the Hadoop Ecosystem

Choosing the right file format has significant implications for storage efficiency, query performance, and compatibility. See also [[Open Table Formats - Iceberg Hudi & Delta]].

### SequenceFile

- Hadoop's native binary container format for key-value pairs
- Supports **record-level** and **block-level** compression (block is recommended)
- Splittable — sync markers allow MapReduce to find record boundaries
- Good for small files (pack them into a single SequenceFile) and intermediate data

### Avro

- Language-neutral data serialisation system created by Doug Cutting
- Schema defined in **JSON**; always present at read and write time (self-describing files)
- Compact binary encoding (no field identifiers needed since schema is present)
- Supports **schema evolution** — reader and writer schemas need not be identical (add/remove optional fields)
- Splittable container format (Avro datafiles) with compression support
- Supported by MapReduce, Pig, Hive, Spark, and most Hadoop ecosystem tools

### Parquet

- **Columnar** storage format, efficient for analytical queries (skip unneeded columns)
- Based on Google's **Dremel** paper for nested data encoding
- Smaller file sizes via column-specific encoding (delta, run-length, dictionary)
- Splittable with compression support
- Widely supported: MapReduce, Pig, Hive, Spark, Presto, Impala
- Stores schema in file metadata (self-describing)

### ORC (Optimised Row Columnar)

- Columnar format from the Hive project
- Similar benefits to Parquet: predicate pushdown, column pruning, compression
- Optimised for Hive workloads; strong integration with Hive ACID transactions

### Format Selection Guide

| Use case | Recommended format |
|---|---|
| Schema evolution, cross-language interop | Avro |
| Analytical queries, column pruning | Parquet or ORC |
| Small file consolidation, intermediate data | SequenceFile |
| Hive-centric pipelines with ACID | ORC |
| General-purpose modern pipelines | Parquet (most broadly supported) |

---

## Hadoop Ecosystem Overview

### Apache Hive — SQL on Hadoop

- Data warehousing framework providing **HiveQL** (SQL dialect) over HDFS data
- Created at Facebook for analysts with SQL skills to query large datasets
- Translates queries into MapReduce, Tez, or Spark jobs
- Organises data into tables with schemas stored in the **metastore** (typically backed by a relational database)
- Supports `LOAD DATA` (file copy into warehouse directory, no transformation) and standard DDL/DML
- Best for batch analytics, not low-latency queries (though Hive on Tez/LLAP improves latency)

### Apache Pig — Data Flow Language

- High-level data flow language (**Pig Latin**) for expressing data transformations
- Compiles to MapReduce, Tez, or Spark jobs
- Designed for ETL pipelines and ad hoc data exploration
- Supports user-defined functions (UDFs) and schema propagation
- Algebraic functions (e.g. `MAX`, `SUM`) can leverage the combiner for efficiency

### Apache HBase — Real-Time Random Access

- Distributed, column-family-oriented database built on HDFS
- Modelled after Google's **Bigtable** paper
- Provides low-latency random read/write access to very large datasets (billions of rows)
- Data model: tables with rows (sorted by row key), column families (declared at schema time), column qualifiers (added on the fly), and versioned cells
- **Regions** — tables are horizontally partitioned into regions distributed across RegionServers
- Write path: commit log (WAL on HDFS) then in-memory memstore; flushed to disc as HFiles; background compaction merges HFiles
- Depends on **ZooKeeper** for master election, region assignment, and cluster coordination
- Use when you need real-time access that HDFS/MapReduce batch processing cannot provide

### Apache ZooKeeper — Distributed Coordination

- Coordination service for distributed applications — handles partial failure gracefully
- Provides a stripped-down filesystem of **znodes** with ordering and notification primitives
- Building blocks for distributed locks, leader election, configuration management, group membership
- Runs as an ensemble (typically 3 or 5 nodes) with majority-quorum writes
- Used by HDFS HA (NameNode failover), HBase (master election, region assignment), YARN (ResourceManager HA), and Kafka
- Highly available and performant: 10,000+ write ops/second; read-dominant workloads are several times higher

### Apache Flume — Streaming Ingestion

- Designed for high-volume ingestion of event-based data (e.g. log files) into HDFS
- Architecture: **source** (produces events) -> **channel** (buffers; file or memory) -> **sink** (writes to HDFS, HBase, etc.)
- Agents can be chained in multi-tier topologies for aggregation and fan-out
- Configuration-driven (Java properties files); no custom code required for standard pipelines
- Supports at-least-once delivery guarantees with file channels
- See also [[Data Ingestion Patterns]] and [[Apache Kafka Fundamentals]] for streaming alternatives

### Apache Sqoop — Bulk Data Transfer

- Transfers data between **relational databases** (RDBMS) and **HDFS**
- Import: reads from RDBMS tables using JDBC; runs as a MapReduce job (parallel readers)
- Export: writes HDFS data back to RDBMS tables
- Supports Avro, SequenceFile, and delimited text output formats
- Can generate Java classes for imported records (ORM-style)
- Integrates with Hive: `--hive-import` flag creates Hive tables from imported data
- Useful for initial data loading and incremental imports (`--incremental` mode)

---

## Hadoop vs Modern Alternatives

While Hadoop remains foundational, modern data platforms have evolved significantly:

| Aspect | Hadoop/MapReduce | Modern alternatives |
|---|---|---|
| Processing model | Batch (disc-based shuffle) | Spark (in-memory), Flink (streaming-first) |
| SQL layer | Hive (batch latency) | Spark SQL, Trino/Presto, Snowflake |
| Storage | HDFS (on-premise clusters) | Cloud object storage (S3, GCS, ADLS) + [[Open Table Formats - Iceberg Hudi & Delta]] |
| Resource management | YARN | Kubernetes, cloud-native auto-scaling |
| Ease of use | Java-heavy, complex ops | Managed services, Python-first APIs |

**HDFS concepts remain relevant** — block storage, replication, data locality, and the NameNode/DataNode architecture inform the design of modern distributed storage. The MapReduce shuffle pattern underpins Spark's shuffle and every distributed SQL engine's join implementation.

---

## Key Takeaways

1. **MapReduce is a programming model, not just a framework** — the map/shuffle/reduce pattern appears in Spark, Flink, and every distributed query engine
2. **HDFS trades latency for throughput** — large blocks, streaming reads, append-only writes; not a general-purpose filesystem
3. **YARN decoupled resource management from compute** — enabling multi-framework clusters and paving the way for Kubernetes-based orchestration
4. **The combiner is an underappreciated optimisation** — local pre-aggregation can dramatically reduce shuffle volume, but only for associative, commutative operations
5. **File format choice matters enormously** — Parquet/ORC for analytics, Avro for schema evolution, SequenceFile for Hadoop-native workflows
6. **Fault tolerance through re-execution** — both the original Google paper and Hadoop use task re-execution (not checkpointing) as the primary recovery mechanism

---

## Sources

- Dean, J. & Ghemawat, S. (2004). *MapReduce: Simplified Data Processing on Large Clusters*. OSDI.
- White, T. (2015). *Hadoop: The Definitive Guide*, 4th Edition. O'Reilly Media.
