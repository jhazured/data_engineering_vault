# SQL Query Optimization

Techniques for writing efficient SQL and understanding how the query engine executes your queries.

## Execution Plans

The query optimizer converts SQL into a physical execution plan. Read it to understand why a query is slow.

### Reading an Execution Plan

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.customer_name, SUM(s.revenue)
FROM customers c
JOIN shipments s ON c.customer_id = s.customer_id
WHERE s.shipment_date >= '2024-01-01'
GROUP BY c.customer_name;

-- Snowflake
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
-- Or use Query Profile in Snowsight UI
```

### Key Plan Indicators

| Indicator | Good Sign | Bad Sign |
|-----------|-----------|----------|
| **Scan type** | Index scan, partition pruning | Full table scan on large table |
| **Join type** | Hash join (equi-join), merge join | Nested loop on large tables |
| **Rows estimated vs actual** | Close match | Orders of magnitude off (stale stats) |
| **Bytes scanned** | Small relative to table | Scanning entire table |
| **Spilling to disk** | None | Memory spill (needs larger warehouse or better query) |

## Join Optimization

### Join Order Matters

The optimizer usually picks the best order, but with complex multi-way joins, it may not:

```sql
-- Let the optimizer work — use explicit JOIN syntax
SELECT ...
FROM fact_shipments f
JOIN dim_customer c ON f.customer_id = c.customer_id    -- Large to small
JOIN dim_date d ON f.date_key = d.date_key              -- FK to PK
JOIN dim_vehicle v ON f.vehicle_id = v.vehicle_id
WHERE d.year = 2024
```

**Principle:** Filter early, join later. Place the most selective filters in WHERE or as early JOIN conditions.

### Join Types and Performance

| Join Type | When Used | Cost |
|-----------|-----------|------|
| **Hash join** | Equi-joins, one side fits in memory | Fast for most analytics |
| **Merge join** | Both sides sorted on join key | Fast if pre-sorted |
| **Nested loop** | Small outer table, indexed inner | Slow on large tables |
| **Broadcast join** | Small dimension joined to large fact | Copies small table to all nodes |

### Anti-Patterns

```sql
-- BAD: Implicit cross join (Cartesian product)
SELECT * FROM orders, customers WHERE orders.cust_id = customers.id;

-- GOOD: Explicit JOIN
SELECT * FROM orders JOIN customers ON orders.cust_id = customers.id;

-- BAD: Function on join column prevents index use
WHERE UPPER(c.customer_name) = UPPER(s.name)

-- GOOD: Normalise data upstream, join on clean columns
WHERE c.customer_name = s.name
```

## Predicate Pushdown

Move filters as close to the data scan as possible:

```sql
-- GOOD: Filter in WHERE (pushed down to scan)
SELECT c.customer_name, SUM(s.revenue)
FROM shipments s
JOIN customers c ON s.customer_id = c.customer_id
WHERE s.shipment_date >= '2024-01-01'
GROUP BY c.customer_name;

-- BAD: Filter in HAVING after aggregation (all rows scanned first)
SELECT c.customer_name, SUM(s.revenue) AS total
FROM shipments s
JOIN customers c ON s.customer_id = c.customer_id
GROUP BY c.customer_name
HAVING SUM(s.revenue) > 0;  -- Use WHERE when possible instead
```

## SELECT Only What You Need

```sql
-- BAD: SELECT * scans all columns (columnar storage reads every column)
SELECT * FROM fact_shipments WHERE date_key = 20240115;

-- GOOD: Only needed columns (columnar storage skips unneeded columns)
SELECT shipment_id, customer_id, revenue
FROM fact_shipments WHERE date_key = 20240115;
```

In columnar databases (Snowflake, BigQuery, Redshift), this dramatically reduces bytes scanned.

## Partition Pruning

Queries that filter on the partition/cluster key scan far less data:

```sql
-- Snowflake: clustering key is shipment_date
-- GOOD: Filter on cluster key — pruning eliminates most micro-partitions
SELECT * FROM fact_shipments WHERE shipment_date = '2024-01-15';

-- BAD: No filter on cluster key — full scan
SELECT * FROM fact_shipments WHERE customer_id = 'CUST_001';
```

### Snowflake Clustering

```sql
-- Set clustering key
ALTER TABLE fact_shipments CLUSTER BY (shipment_date, customer_id);

-- Check clustering quality
SELECT SYSTEM$CLUSTERING_INFORMATION('fact_shipments');
```

## Subquery vs JOIN vs CTE

| Approach | When to Use | Performance |
|----------|------------|-------------|
| **JOIN** | Standard lookups | Optimizer handles well |
| **CTE** | Readability, reuse within query | Usually materialised once |
| **Correlated subquery** | Row-by-row comparison | Slow — avoid on large tables |
| **EXISTS** | Check existence (not values) | Often faster than IN for large sets |

```sql
-- SLOW: Correlated subquery (executes per row)
SELECT * FROM customers c
WHERE (SELECT COUNT(*) FROM shipments s WHERE s.customer_id = c.customer_id) > 10;

-- FAST: JOIN with aggregation
SELECT c.*, ship_counts.cnt
FROM customers c
JOIN (SELECT customer_id, COUNT(*) AS cnt FROM shipments GROUP BY 1) ship_counts
  ON c.customer_id = ship_counts.customer_id
WHERE ship_counts.cnt > 10;
```

## UNION vs UNION ALL

```sql
-- UNION: Removes duplicates (requires sort/distinct — expensive)
SELECT customer_id FROM orders_2023
UNION
SELECT customer_id FROM orders_2024;

-- UNION ALL: Keeps all rows (no dedup — much faster)
SELECT customer_id FROM orders_2023
UNION ALL
SELECT customer_id FROM orders_2024;
```

**Always use UNION ALL unless you specifically need deduplication.**

## Aggregate Optimization

```sql
-- COUNT(*) vs COUNT(column)
COUNT(*)           -- Counts all rows (fast — uses metadata if available)
COUNT(column)      -- Counts non-NULL values (must read the column)
COUNT(DISTINCT x)  -- Expensive — requires sort or hash

-- Approximate count for large datasets (Snowflake)
APPROX_COUNT_DISTINCT(customer_id)  -- HyperLogLog, ~2% error, much faster
```

## Common Snowflake-Specific Optimisations

| Technique | Command | When |
|-----------|---------|------|
| **Clustering** | `ALTER TABLE ... CLUSTER BY (col)` | Large tables queried by specific columns |
| **Search Optimization** | `ALTER TABLE ... ADD SEARCH OPTIMIZATION` | Point lookups on large tables |
| **Result caching** | Automatic (24h) | Identical queries within cache window |
| **Materialized views** | `CREATE MATERIALIZED VIEW` | Frequent aggregation queries |
| **Warehouse sizing** | Scale up for complex queries | Spilling to disk, slow scans |

## Query Optimization Checklist

1. Check execution plan — any full table scans?
2. Filter on partition/cluster key?
3. SELECT only needed columns?
4. JOINs on indexed/clustered columns?
5. Functions on join/filter columns preventing pruning?
6. UNION ALL instead of UNION?
7. EXISTS instead of IN for large subqueries?
8. Appropriate warehouse size for the workload?
