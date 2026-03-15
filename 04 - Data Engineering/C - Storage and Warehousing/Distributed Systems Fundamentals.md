# Distributed Systems Fundamentals

Core concepts from "Designing Data-Intensive Applications" — the foundation for understanding data platforms, replication, partitioning, and consistency.

## Storage Engines

### B-Trees vs LSM-Trees

| Aspect | B-Trees | LSM-Trees (SSTables) |
|--------|---------|---------------------|
| Used by | PostgreSQL, MySQL, Oracle | Cassandra, RocksDB, LevelDB |
| Write pattern | In-place update | Append-only (write to memtable → flush to disk) |
| Read performance | Fast (single lookup) | Slower (check multiple levels) |
| Write performance | Slower (random I/O) | Fast (sequential writes) |
| Space amplification | Lower | Higher (compaction needed) |
| Best for | Read-heavy, transactional | Write-heavy, time-series, logs |

### Column-Oriented Storage

Used by analytical databases (Snowflake, BigQuery, Redshift, Parquet files):

```
Row-oriented:    [id, name, revenue] [id, name, revenue] ...
Column-oriented: [id, id, id, ...] [name, name, name, ...] [revenue, revenue, revenue, ...]
```

**Benefits:** Compression (similar values together), vectorized processing, only read needed columns.

## Replication

### Single-Leader Replication

```
Writes → Leader → Follower 1
                → Follower 2
Reads  ← Leader or Followers
```

- **Synchronous:** Leader waits for follower ACK → strong consistency, slower writes
- **Asynchronous:** Leader doesn't wait → faster writes, possible stale reads
- **Semi-synchronous:** One follower synchronous, rest async (common compromise)

### Multi-Leader Replication

Multiple nodes accept writes (e.g., multi-region deployments):
- Each leader replicates to all others
- **Conflict resolution required** — last-write-wins, merge, custom logic
- Use case: multi-datacenter, collaborative editing

### Leaderless Replication

All nodes accept reads and writes (e.g., Cassandra, DynamoDB):
- **Quorum reads/writes:** Read from R nodes, write to W nodes, where R + W > N
- **R=2, W=2, N=3** — tolerates 1 node failure for both reads and writes
- **Hinted handoff** — temporarily store writes for unavailable nodes

## Partitioning (Sharding)

### Key Range Partitioning

```
Partition 1: A-F    Partition 2: G-M    Partition 3: N-Z
```

- Good for range queries
- Risk of **hot spots** if keys are sequential (e.g., timestamps)

### Hash Partitioning

```
Partition = hash(key) % num_partitions
```

- Even distribution
- No efficient range queries
- Used by: Kafka (topic partitions), Cassandra, DynamoDB

### Rebalancing Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| Fixed partitions | Pre-create many partitions, assign to nodes | Simple but requires upfront planning |
| Dynamic splitting | Split when partition gets too large | More complex, better balance |
| Consistent hashing | Hash ring with virtual nodes | Minimal data movement on rebalance |

## Transactions

### ACID Properties

| Property | Meaning |
|----------|---------|
| **Atomicity** | All-or-nothing — partial failures are rolled back |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data survives crashes |

### Isolation Levels

| Level | Prevents | Allows | Performance |
|-------|----------|--------|-------------|
| **Read Uncommitted** | Nothing | Dirty reads, non-repeatable reads, phantoms | Fastest |
| **Read Committed** | Dirty reads | Non-repeatable reads, phantoms | Fast |
| **Snapshot Isolation** | Dirty reads, non-repeatable reads | Write skew, phantoms | Moderate |
| **Serializable** | Everything | Nothing | Slowest |

**Snowflake uses snapshot isolation** (MVCC) — each query sees a consistent snapshot.

## Consistency Models

| Model | Guarantee | Example |
|-------|-----------|---------|
| **Linearizability** | Reads always return most recent write | Single-leader sync replication |
| **Causal consistency** | Causally related operations are ordered | "If I wrote it, I can read it" |
| **Eventual consistency** | All replicas converge eventually | Cassandra, DynamoDB |

**CAP Theorem:** In a network partition, you must choose between **Consistency** (all nodes see same data) and **Availability** (all requests get a response). You can't have both during a partition.

## Data Encoding Formats

| Format | Schema | Human-Readable | Size | Schema Evolution |
|--------|--------|----------------|------|------------------|
| **JSON** | Optional (JSON Schema) | Yes | Large | Flexible but fragile |
| **Avro** | Required (embedded) | No | Small | Excellent (reader/writer schemas) |
| **Protobuf** | Required (.proto files) | No | Small | Good (field numbers) |
| **Thrift** | Required (.thrift files) | No | Small | Good |
| **Parquet** | Embedded | No | Very small (columnar) | Good (column addition) |

**For data pipelines:** Avro (row-based, great schema evolution) for streaming; Parquet (columnar, great compression) for analytics/warehousing.

## Consensus

Algorithms for distributed agreement:

| Algorithm | Used By | Approach |
|-----------|---------|----------|
| **Raft** | etcd, CockroachDB | Leader election + log replication |
| **Paxos** | Google Spanner | Multi-phase voting |
| **ZAB** | ZooKeeper | Leader-based atomic broadcast |
| **KRaft** | Kafka (replacing ZooKeeper) | Raft-based metadata management |

## Key Design Trade-offs

| Decision | Option A | Option B |
|----------|----------|----------|
| Consistency vs Availability | Strong consistency (CP) | High availability (AP) |
| Latency vs Durability | Async replication (fast) | Sync replication (durable) |
| Normalization vs Denormalization | Normalized (less redundancy) | Denormalized (faster reads) |
| Batch vs Streaming | High throughput, high latency | Low latency, more complexity |
| Partitioning key | Range (good for scans) | Hash (even distribution) |
