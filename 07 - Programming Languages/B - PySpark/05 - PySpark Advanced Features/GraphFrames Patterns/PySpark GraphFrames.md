
#pyspark #graphframes #graph-algorithms #pagerank #connected-components #data-lineage #entity-resolution

## Overview

GraphFrames is a graph processing library for Apache Spark built on DataFrames, providing both an expressive API for graph queries and a suite of standard graph algorithms. Unlike the older GraphX (RDD-based), GraphFrames integrate seamlessly with Spark SQL and the DataFrame API, making them accessible from PySpark. See also [[PySpark Core Concepts]], [[PySpark Advanced Features]], and [[PySpark Performance Optimization]].

---

## Setup And Installation

```python
# GraphFrames is a separate package — install via Maven coordinates
# spark-submit --packages graphframes:graphframes:0.8.3-spark3.5-s_2.12

# Or in a notebook / programmatic session
spark = (
    SparkSession.builder
    .appName("GraphFrames")
    .config("spark.jars.packages", "graphframes:graphframes:0.8.3-spark3.5-s_2.12")
    .getOrCreate()
)

from graphframes import GraphFrame
```

---

## GraphFrame Creation

A GraphFrame requires two DataFrames: **vertices** (nodes) and **edges** (relationships).

```python
from pyspark.sql import SparkSession
from graphframes import GraphFrame

# Vertices must have an "id" column
vertices = spark.createDataFrame([
    ("alice", "Alice", "Engineering"),
    ("bob", "Bob", "Marketing"),
    ("charlie", "Charlie", "Engineering"),
    ("diana", "Diana", "Finance"),
    ("eve", "Eve", "Marketing"),
], ["id", "name", "department"])

# Edges must have "src" and "dst" columns
edges = spark.createDataFrame([
    ("alice", "bob", "collaborates"),
    ("bob", "charlie", "manages"),
    ("charlie", "diana", "reports_to"),
    ("diana", "eve", "mentors"),
    ("eve", "alice", "collaborates"),
    ("alice", "charlie", "collaborates"),
], ["src", "dst", "relationship"])

g = GraphFrame(vertices, edges)

# Basic properties
print(f"Vertices: {g.vertices.count()}")
print(f"Edges: {g.edges.count()}")
print(f"In-degrees:")
g.inDegrees.show()
print(f"Out-degrees:")
g.outDegrees.show()
```

### Building From Existing Data

```python
# From a table of relationships (e.g. transactions, communications)
transactions = spark.read.parquet("/mnt/data/transactions/")

edges = (
    transactions
    .select(
        F.col("sender_id").alias("src"),
        F.col("receiver_id").alias("dst"),
        F.col("amount"),
        F.col("transaction_date"),
    )
)

# Derive vertices from the edge endpoints
all_ids = (
    edges.select(F.col("src").alias("id"))
    .union(edges.select(F.col("dst").alias("id")))
    .distinct()
)

# Enrich vertices with attributes
customers = spark.read.parquet("/mnt/data/customers/")
vertices = all_ids.join(customers, all_ids["id"] == customers["customer_id"], "left")

g = GraphFrame(vertices, edges)
```

---

## Motif Finding

Motif finding uses a domain-specific language to express structural patterns in the graph.

```python
# Find triangles: A -> B -> C -> A
triangles = g.find("(a)-[e1]->(b); (b)-[e2]->(c); (c)-[e3]->(a)")
triangles.select("a.name", "b.name", "c.name").show()

# Find paths of length 2
paths = g.find("(a)-[e1]->(b); (b)-[e2]->(c)")
paths.filter("a.id != c.id").show()

# Find mutual collaborators
mutual = g.find("(a)-[e1]->(b); (b)-[e2]->(a)")
mutual.filter("e1.relationship = 'collaborates'").show()

# Named edges allow filtering on edge properties
high_value_paths = g.find("(a)-[e1]->(b); (b)-[e2]->(c)")
high_value_paths.filter("e1.amount > 10000 AND e2.amount > 10000").show()

# Negation — find vertices with no outgoing edges
no_outgoing = g.find("(a)-[]->(b)")
all_vertices = g.vertices.select("id")
isolated = all_vertices.subtract(no_outgoing.select(F.col("a.id").alias("id")))
```

---

## Graph Algorithms

### PageRank

Identify the most important nodes based on the link structure.

```python
# Run PageRank with a reset probability of 0.15
pr = g.pageRank(resetProbability=0.15, maxIter=20)

# Results are attached to the vertices DataFrame
pr.vertices.select("id", "name", "pagerank").orderBy("pagerank", ascending=False).show(10)

# Personalised PageRank — importance relative to a specific node
ppr = g.pageRank(resetProbability=0.15, maxIter=20, sourceId="alice")
ppr.vertices.orderBy("pagerank", ascending=False).show(10)
```

### Connected Components

Find clusters of nodes that are reachable from one another.

```python
# Requires a checkpoint directory for iterative computation
spark.sparkContext.setCheckpointDir("/tmp/graphframes_checkpoint")

cc = g.connectedComponents()
cc.select("id", "name", "component").show()

# Count and size of each component
cc.groupBy("component").count().orderBy("count", ascending=False).show()
```

### Strongly Connected Components

Directed variant — every node in a component can reach every other node.

```python
scc = g.stronglyConnectedComponents(maxIter=10)
scc.select("id", "component").show()
```

### Shortest Paths

Find shortest paths from all vertices to a set of landmark vertices.

```python
# Compute shortest path distances to landmark vertices
sp = g.shortestPaths(landmarks=["alice", "diana"])
sp.select("id", "distances").show(truncate=False)

# Output: distances is a map, e.g. {"alice": 0, "diana": 2}
```

### Triangle Counting

Count the number of triangles passing through each vertex — useful for clustering coefficient analysis.

```python
tc = g.triangleCount()
tc.select("id", "count").orderBy("count", ascending=False).show()
```

### Label Propagation

Community detection algorithm that propagates labels through the graph.

```python
lp = g.labelPropagation(maxIter=5)
lp.select("id", "name", "label").show()

# Group by community label
lp.groupBy("label").agg(
    F.count("*").alias("community_size"),
    F.collect_list("name").alias("members")
).show(truncate=False)
```

---

## Subgraph Extraction

### Filter-Based Subgraphs

```python
# Filter edges by relationship type
collab_graph = GraphFrame(
    g.vertices,
    g.edges.filter(F.col("relationship") == "collaborates")
)

# Filter vertices by attribute and keep only relevant edges
eng_vertices = g.vertices.filter(F.col("department") == "Engineering")
eng_edges = g.edges.join(eng_vertices.select("id"), g.edges["src"] == eng_vertices["id"])
eng_edges = eng_edges.join(
    eng_vertices.select(F.col("id").alias("id2")),
    eng_edges["dst"] == F.col("id2")
).select("src", "dst", "relationship")

eng_graph = GraphFrame(eng_vertices, eng_edges)
```

### K-Hop Neighbourhood

```python
def get_k_hop_neighbourhood(graph, start_id, k=2):
    """Extract the subgraph within k hops of a starting vertex."""
    visited = {start_id}
    frontier = {start_id}

    for _ in range(k):
        # Find neighbours of the current frontier
        frontier_df = spark.createDataFrame([(v,) for v in frontier], ["id"])
        neighbours = (
            graph.edges
            .join(frontier_df, graph.edges["src"] == frontier_df["id"])
            .select("dst")
            .distinct()
            .collect()
        )
        new_ids = {row["dst"] for row in neighbours} - visited
        visited.update(new_ids)
        frontier = new_ids

        if not frontier:
            break

    # Build subgraph
    visited_df = spark.createDataFrame([(v,) for v in visited], ["vid"])
    sub_vertices = graph.vertices.join(visited_df, graph.vertices["id"] == visited_df["vid"])
    sub_edges = graph.edges.join(
        visited_df, graph.edges["src"] == visited_df["vid"]
    ).join(
        visited_df.select(F.col("vid").alias("vid2")),
        F.col("dst") == F.col("vid2")
    ).select("src", "dst", "relationship")

    return GraphFrame(sub_vertices.drop("vid"), sub_edges)
```

---

## Integration With Spark SQL

```python
# Register vertices and edges as temp views
g.vertices.createOrReplaceTempView("vertices")
g.edges.createOrReplaceTempView("edges")

# Use SQL for ad-hoc graph queries
spark.sql("""
    SELECT e.src, e.dst, v1.name AS src_name, v2.name AS dst_name
    FROM edges e
    JOIN vertices v1 ON e.src = v1.id
    JOIN vertices v2 ON e.dst = v2.id
    WHERE e.relationship = 'collaborates'
""").show()

# Combine algorithm results with SQL
pr = g.pageRank(resetProbability=0.15, maxIter=20)
pr.vertices.createOrReplaceTempView("pagerank_results")

spark.sql("""
    SELECT name, department, pagerank
    FROM pagerank_results
    WHERE department = 'Engineering'
    ORDER BY pagerank DESC
""").show()
```

---

## Use Cases In Data Engineering

### Data Lineage Graphs

```python
# Model tables and transformations as a graph
lineage_vertices = spark.createDataFrame([
    ("raw_orders", "table", "bronze"),
    ("raw_customers", "table", "bronze"),
    ("clean_orders", "table", "silver"),
    ("clean_customers", "table", "silver"),
    ("order_summary", "table", "gold"),
    ("etl_clean_orders", "transformation", None),
    ("etl_join_summary", "transformation", None),
], ["id", "type", "layer"])

lineage_edges = spark.createDataFrame([
    ("raw_orders", "etl_clean_orders", "input"),
    ("etl_clean_orders", "clean_orders", "output"),
    ("raw_customers", "clean_customers", "input"),
    ("clean_orders", "etl_join_summary", "input"),
    ("clean_customers", "etl_join_summary", "input"),
    ("etl_join_summary", "order_summary", "output"),
], ["src", "dst", "relation"])

lineage = GraphFrame(lineage_vertices, lineage_edges)

# Find all upstream dependencies of gold table
upstream = lineage.shortestPaths(landmarks=["raw_orders", "raw_customers"])
upstream.filter(F.col("id") == "order_summary").select("distances").show(truncate=False)

# Impact analysis: what downstream tables are affected if raw_orders changes?
downstream = lineage.find("(a)-[]->(b); (b)-[]->(c)")
downstream.filter("a.id = 'raw_orders'").select("c.id").distinct().show()
```

### Entity Resolution

```python
# Model potential matches between records as edges
# Vertices = records, edges = similarity scores

match_edges = spark.createDataFrame([
    ("rec_001", "rec_042", 0.95),
    ("rec_001", "rec_103", 0.87),
    ("rec_042", "rec_103", 0.91),
    ("rec_200", "rec_201", 0.82),
], ["src", "dst", "similarity"])

records = spark.createDataFrame([
    ("rec_001", "John Smith", "john@example.com"),
    ("rec_042", "J. Smith", "jsmith@example.com"),
    ("rec_103", "Jonathan Smith", "john@example.com"),
    ("rec_200", "Jane Doe", "jane@example.com"),
    ("rec_201", "J. Doe", "jdoe@example.com"),
], ["id", "name", "email"])

# Filter edges above threshold
high_conf_edges = match_edges.filter(F.col("similarity") >= 0.85)
match_graph = GraphFrame(records, high_conf_edges)

# Connected components give entity clusters
spark.sparkContext.setCheckpointDir("/tmp/entity_resolution_checkpoint")
clusters = match_graph.connectedComponents()
clusters.select("id", "name", "component").orderBy("component").show()
```

### Dependency Mapping

```python
# Model service or job dependencies
services = spark.createDataFrame([
    ("api_gateway", "service", "critical"),
    ("auth_service", "service", "critical"),
    ("order_service", "service", "high"),
    ("payment_service", "service", "critical"),
    ("notification_service", "service", "medium"),
    ("etl_daily", "job", "high"),
], ["id", "type", "priority"])

dependencies = spark.createDataFrame([
    ("api_gateway", "auth_service", "sync"),
    ("api_gateway", "order_service", "sync"),
    ("order_service", "payment_service", "sync"),
    ("order_service", "notification_service", "async"),
    ("etl_daily", "order_service", "batch"),
], ["src", "dst", "call_type"])

dep_graph = GraphFrame(services, dependencies)

# Find critical path — services with highest PageRank
pr = dep_graph.pageRank(resetProbability=0.15, maxIter=20)
pr.vertices.orderBy("pagerank", ascending=False).show()

# Blast radius: if payment_service fails, what is affected?
blast = dep_graph.find("(a)-[]->(b)")
blast.filter("b.id = 'payment_service'").select("a.id", "a.priority").show()
```

### Network Analysis

```python
# Analyse communication or transaction networks for anomalies
# Identify hub nodes (high degree), bridge nodes, isolated clusters

degrees = g.degrees
high_degree = degrees.filter(F.col("degree") > 100)
high_degree.show()

# Combine with PageRank for influence scoring
pr = g.pageRank(resetProbability=0.15, maxIter=20)
influence = pr.vertices.join(degrees, "id").select(
    "id", "name", "pagerank", "degree"
).orderBy("pagerank", ascending=False)
influence.show(20)
```

---

## Performance Considerations

```text
1. Checkpoint directory — required for connected components and other iterative
   algorithms. Use a reliable filesystem (HDFS, S3, DBFS).

2. Graph size — GraphFrames operate on DataFrames, so standard Spark optimisations
   apply: partition pruning, broadcast joins for small vertex sets, caching.

3. Skewed graphs — power-law degree distributions cause data skew. Consider
   filtering out super-nodes or using salted joins.

4. Iteration count — algorithms like PageRank and label propagation converge;
   set maxIter high enough for convergence but not wastefully high.

5. Caching — cache the GraphFrame if running multiple algorithms:
   g.vertices.cache()
   g.edges.cache()
```

---

## Related Notes

- [[PySpark Core Concepts]] — DataFrame and SparkSession fundamentals
- [[PySpark Advanced Features]] — broader advanced feature coverage
- [[PySpark Performance Optimization]] — optimisation techniques for graph workloads
- [[PySpark Data Operations]] — joins and aggregations used in graph construction
