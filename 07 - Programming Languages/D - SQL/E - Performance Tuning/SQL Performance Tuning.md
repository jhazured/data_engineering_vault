# SQL Performance Tuning

Systematic approach to diagnosing and fixing slow SQL queries — focused on Snowflake and cloud data warehouses.

## Diagnosis Workflow

```
1. Identify slow query (monitoring views, Query Profile)
2. Check execution plan (bytes scanned, spill, partition pruning)
3. Identify bottleneck (scan, join, sort, aggregation, network)
4. Apply fix (index/cluster, rewrite, warehouse sizing, materialise)
5. Verify improvement (re-run, compare metrics)
```

## Snowflake Query Profile

Access via Snowsight > Query History > click query > Query Profile:

| Metric | What It Tells You |
|--------|-------------------|
| **Bytes scanned** | How much data was read — lower is better |
| **Partitions scanned vs total** | Pruning effectiveness (should be <10% for filtered queries) |
| **Spillage (local/remote)** | Query exceeded memory — needs larger warehouse or query rewrite |
| **Rows produced vs scanned** | Selectivity — high scan:produce ratio = poor filtering |
| **Queuing time** | Warehouse was busy — scale up or use dedicated warehouse |

## Indexing Strategies (Cloud DW)

Cloud warehouses don't have traditional B-tree indexes. Instead:

### Snowflake

| Technique | What It Does | When to Use |
|-----------|-------------|-------------|
| **Clustering** | Co-locates data in micro-partitions by column(s) | Large tables filtered by specific columns |
| **Search Optimization** | Builds search structures for point lookups | Equality filters, LIKE, IN, geography |
| **Materialized Views** | Pre-computed aggregations | Repeated expensive aggregations |
| **Result Cache** | Caches query results (24h) | Identical repeated queries |

```sql
-- Clustering
ALTER TABLE fact_shipments CLUSTER BY (shipment_date);

-- Verify clustering quality (0=perfectly clustered, higher=worse)
SELECT SYSTEM$CLUSTERING_INFORMATION('fact_shipments', '(shipment_date)');

-- Search Optimization
ALTER TABLE dim_customer ADD SEARCH OPTIMIZATION ON EQUALITY(customer_id);
```

### BigQuery

| Technique | Equivalent |
|-----------|-----------|
| Partitioning | `PARTITION BY DATE(shipment_date)` |
| Clustering | `CLUSTER BY customer_id, region` |
| Materialised views | `CREATE MATERIALIZED VIEW` |
| BI Engine | In-memory acceleration for dashboards |

## Warehouse Sizing (Snowflake)

| Symptom | Fix |
|---------|-----|
| Remote disk spillage | Scale UP (larger warehouse = more memory) |
| Queuing (many concurrent queries) | Scale OUT (multi-cluster warehouse) |
| Simple queries slow | Check if warehouse is suspended (auto-resume latency) |
| Large scans, no spillage | Optimise query (clustering, pruning, fewer columns) |

```sql
-- Scale up temporarily for a heavy job
ALTER WAREHOUSE TRN_PROD_CENTRAL_WH SET WAREHOUSE_SIZE = 'MEDIUM';
-- Run heavy query...
ALTER WAREHOUSE TRN_PROD_CENTRAL_WH SET WAREHOUSE_SIZE = 'X-SMALL';

-- Multi-cluster for concurrency
ALTER WAREHOUSE OUT_PROD_CENTRAL_WH SET
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3
    SCALING_POLICY = 'STANDARD';
```

## Query Rewrite Patterns

### Replace Correlated Subqueries

```sql
-- SLOW: Executes subquery per row
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM shipments s
    WHERE s.customer_id = c.customer_id
    AND s.revenue > 1000
);

-- FASTER: Semi-join
SELECT c.* FROM customers c
SEMI JOIN shipments s ON s.customer_id = c.customer_id AND s.revenue > 1000;

-- Or use window function
SELECT * FROM (
    SELECT c.*, MAX(s.revenue) OVER (PARTITION BY c.customer_id) AS max_rev
    FROM customers c LEFT JOIN shipments s ON c.customer_id = s.customer_id
) WHERE max_rev > 1000;
```

### Pre-aggregate Before Joining

```sql
-- SLOW: Join then aggregate (many rows processed)
SELECT c.customer_type, SUM(s.revenue)
FROM customers c JOIN shipments s ON c.customer_id = s.customer_id
GROUP BY c.customer_type;

-- FASTER: Aggregate first, join on smaller result
SELECT c.customer_type, agg.total_revenue
FROM customers c
JOIN (
    SELECT customer_id, SUM(revenue) AS total_revenue
    FROM shipments GROUP BY customer_id
) agg ON c.customer_id = agg.customer_id;
```

### Avoid Functions on Filter Columns

```sql
-- BAD: Function prevents partition pruning
WHERE DATE_TRUNC('month', shipment_date) = '2024-01-01'

-- GOOD: Range filter enables pruning
WHERE shipment_date >= '2024-01-01' AND shipment_date < '2024-02-01'

-- BAD: CAST prevents clustering benefit
WHERE CAST(customer_id AS VARCHAR) = '12345'

-- GOOD: Match the column type
WHERE customer_id = 12345
```

## Statistics and Cost Estimation

Cloud warehouses maintain automatic statistics, but they can be stale:

```sql
-- Snowflake: Check table statistics
SELECT * FROM DEV_T0_CONTROL.AUDIT.VW_TABLE_SIZE_ANALYSIS
WHERE table_name = 'FACT_SHIPMENTS';

-- BigQuery: Check table metadata
SELECT * FROM `project.dataset.INFORMATION_SCHEMA.TABLE_STORAGE`;
```

## Monitoring Slow Queries

```sql
-- Snowflake: Top queries by elapsed time (last 24h)
SELECT query_id, query_text, total_elapsed_time/1000 AS secs,
       bytes_scanned/1e9 AS gb_scanned, rows_produced
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
ORDER BY total_elapsed_time DESC
LIMIT 20;

-- Queries with high scan-to-produce ratio (poor selectivity)
SELECT query_id, rows_produced, bytes_scanned/1e6 AS mb_scanned,
       bytes_scanned / NULLIF(rows_produced, 0) AS bytes_per_row
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND rows_produced > 0
ORDER BY bytes_per_row DESC
LIMIT 20;
```

## Performance Tuning Checklist

| Check | Action |
|-------|--------|
| Bytes scanned too high? | Add clustering, filter on cluster key, reduce columns |
| Partition pruning low? | Ensure WHERE clause matches clustering/partition key |
| Spillage to disk? | Scale up warehouse or rewrite query to reduce data volume |
| High queuing time? | Use dedicated warehouse or enable multi-cluster |
| Expensive JOIN? | Pre-aggregate, ensure join on clustered columns |
| Full table scan? | Add WHERE filters, use Search Optimization |
| Repeated expensive query? | Create materialized view or cache in presentation layer |
| Function on filter column? | Rewrite as range filter |
