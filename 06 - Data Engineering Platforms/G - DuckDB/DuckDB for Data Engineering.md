Tags: #duckdb #olap #data-engineering #sql #analytics #embedded-database

---

# DuckDB for Data Engineering

## What Is DuckDB?

DuckDB is an **embedded, in-process OLAP database** designed for analytical workloads. It runs inside the host process (Python, R, Java, Node.js, or a standalone CLI) with **zero external dependencies** — there is no server to install, configure, or maintain.

Key characteristics:

- **Columnar storage** — data is stored and processed column-by-column, which is far more efficient for analytical queries that touch a subset of columns across many rows.
- **In-process execution** — the database engine runs in the same process as the application, eliminating network round-trips entirely.
- **Zero dependencies** — a single binary or library with no external requirements. Installation is as simple as `pip install duckdb`.
- **ACID compliant** — full transactional support with write-ahead logging.
- **Open source** — MIT licensed, with an active community and regular releases.

Think of DuckDB as "SQLite for analytics." Where SQLite excels at transactional (OLTP) workloads, DuckDB is purpose-built for analytical (OLAP) queries over large datasets.

---

## Architecture

### Vectorised Execution Engine

DuckDB processes data in **vectors** (batches of ~2048 values) rather than row-by-row (Volcano model) or full-column-at-a-time. This vectorised approach:

- Maximises CPU cache utilisation by keeping working data in L1/L2 cache.
- Reduces per-tuple interpretation overhead.
- Enables SIMD (Single Instruction, Multiple Data) optimisations.

### Parallel Query Processing

DuckDB automatically parallelises query execution across available CPU cores. The query planner partitions work into morsel-driven tasks that run concurrently, with no user configuration required. This means a `GROUP BY` aggregation on a 10 GB Parquet file will saturate all available cores out of the box.

### Memory Management

DuckDB can operate within a configurable memory limit and will spill to disk when that limit is exceeded. This makes it viable on laptops and CI runners with constrained resources:

```sql
SET memory_limit = '2GB';
SET temp_directory = '/tmp/duckdb_spill';
```

---

## Key Use Cases in Data Engineering

| Use Case | Why DuckDB Fits |
|---|---|
| **Local development** | Run analytical queries on production-sized samples without a cloud warehouse connection. |
| **Testing & CI pipelines** | Spin up a disposable database in milliseconds — no infrastructure needed. |
| **Ad-hoc analysis** | Query Parquet, CSV, and JSON files directly from the command line or a notebook. |
| **Data validation** | Run quality checks (row counts, null rates, schema conformance) as part of a build pipeline. |
| **ELT prototyping** | Develop and iterate on transformation logic locally before deploying to a warehouse. |
| **File format conversion** | Convert between CSV, Parquet, and JSON with a single `COPY` statement. |

---

## Reading External Files Directly

One of DuckDB's most powerful features is its ability to **query external files in place** without loading them into a table first.

### Parquet Files

```sql
-- Query a local Parquet file
SELECT customer_id, order_total
FROM read_parquet('orders/2025/*.parquet')
WHERE order_total > 100.00;

-- Hive-partitioned dataset
SELECT *
FROM read_parquet('s3://my-bucket/events/**/*.parquet', hive_partitioning = true);
```

### Remote Files (S3, HTTP)

DuckDB has built-in support for reading from S3, GCS, Azure Blob Storage, and plain HTTP URLs. CSV and JSON are also supported via `read_csv()` and `read_json_auto()`.

```sql
-- Install and load the httpfs extension (bundled by default)
INSTALL httpfs;
LOAD httpfs;

-- Configure S3 credentials
SET s3_region = 'eu-west-1';
SET s3_access_key_id = 'AKIA...';
SET s3_secret_access_key = '...';

-- Query Parquet directly from S3
SELECT
    event_type,
    COUNT(*) AS event_count,
    AVG(duration_ms) AS avg_duration
FROM read_parquet('s3://data-lake-prod/events/year=2025/month=03/*.parquet')
GROUP BY event_type
ORDER BY event_count DESC;
```

### Joining Across File Formats

```sql
-- Join a CSV dimension table with a Parquet fact table
SELECT
    c.customer_name,
    c.region,
    SUM(o.order_total) AS total_spend
FROM read_parquet('orders/*.parquet') AS o
JOIN read_csv('reference/customers.csv') AS c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_name, c.region
ORDER BY total_spend DESC
LIMIT 20;
```

---

## Python Integration

DuckDB integrates seamlessly with the Python data ecosystem. The `duckdb` module provides both a relational API and direct SQL execution.

### Zero-Copy Integration with pandas and Polars

DuckDB can query pandas DataFrames and [[pandas & Polars for Data Engineering|Polars]] LazyFrames directly — no data copying required:

```python
import pandas as pd
import duckdb

# Create a pandas DataFrame
orders = pd.read_csv('orders.csv')

# Query it with SQL — DuckDB reads the DataFrame in place
result = duckdb.sql("""
    SELECT product_category, COUNT(*) AS order_count
    FROM orders
    WHERE order_date >= '2025-01-01'
    GROUP BY product_category
""").fetchdf()
```

```python
import polars as pl
import duckdb

# Polars LazyFrame
lf = pl.scan_parquet('events/*.parquet')

# DuckDB can query Polars DataFrames directly
result = duckdb.sql("""
    SELECT event_type, COUNT(*) AS cnt
    FROM lf
    GROUP BY event_type
""").pl()  # Returns a Polars DataFrame
```

### Apache Arrow Integration

DuckDB supports zero-copy reads from and writes to Apache Arrow tables, making it an excellent query engine within Arrow-based pipelines:

```python
import duckdb

# Return results as an Arrow table
arrow_table = duckdb.sql("""
    SELECT * FROM read_parquet('large_dataset.parquet')
    WHERE country = 'GB'
""").fetch_arrow_table()
```

---

## SQL Dialect Highlights

DuckDB's SQL dialect extends standard SQL with several productivity features that reduce boilerplate in analytical queries. See also [[SQL Query Optimization]].

### SELECT * EXCLUDE and REPLACE

```sql
-- Select all columns except sensitive ones
SELECT * EXCLUDE (email, phone_number)
FROM customers;

-- Replace a column expression in-line
SELECT * REPLACE (ROUND(revenue, 2) AS revenue)
FROM monthly_summary;
```

### COLUMNS() Expression

```sql
-- Apply an aggregation to all numeric columns matching a pattern
SELECT MIN(COLUMNS('.*_amount')), MAX(COLUMNS('.*_amount'))
FROM transactions;
```

### PIVOT and UNPIVOT

```sql
-- Pivot rows to columns
PIVOT monthly_sales
ON month
USING SUM(revenue)
GROUP BY product;

-- Unpivot columns to rows
UNPIVOT quarterly_report
ON q1, q2, q3, q4
INTO NAME quarter VALUE revenue;
```

### List, Struct Types and Friendly SQL

DuckDB supports **list** and **struct** types for nested data (`LIST(col)`, `{'key': val}`), plus several "friendly SQL" conveniences:

- `GROUP BY ALL` / `ORDER BY ALL` — automatically groups or orders by all non-aggregated columns.
- `FROM table SELECT ...` — `FROM`-first syntax.
- `UNION BY NAME` — unions tables by column name rather than position.

---

## DuckDB vs Spark vs Snowflake vs SQLite

Choosing the right tool depends on scale, concurrency, and deployment context. See also [[PySpark Core Concepts]].

| Dimension | DuckDB | Spark | Snowflake | SQLite |
|---|---|---|---|---|
| **Workload type** | OLAP | OLAP / batch ETL | OLAP | OLTP |
| **Deployment** | Embedded / in-process | Distributed cluster | Managed cloud service | Embedded / in-process |
| **Scalability** | Single node (vertical) | Horizontal (multi-node) | Horizontal (elastic) | Single node |
| **Concurrency** | Single writer, multiple readers | High (cluster) | High (multi-cluster) | Single writer, multiple readers |
| **Setup complexity** | None | High (JVM, cluster manager) | Low (SaaS) | None |
| **Cost** | Free | Infrastructure + ops | Pay-per-query | Free |
| **File format support** | Parquet, CSV, JSON, Arrow | Parquet, ORC, CSV, Avro, Delta | Staged files, external tables | None (requires loading) |
| **Best for** | Local dev, CI, ad-hoc analysis | Large-scale distributed ETL | Production warehouse, BI | Application state, OLTP |

**When to use DuckDB:** Your data fits on a single machine (up to ~100-200 GB comfortably), you need fast iteration without infrastructure, or you want to run analytical queries in CI/CD.

**When to use Spark:** Your data exceeds what a single machine can handle, or you need a distributed execution engine for production-scale batch pipelines.

**When to use Snowflake:** You need a production data warehouse with role-based access control, concurrency for multiple analysts, and managed infrastructure.

---

## DuckDB in CI/CD Pipelines

DuckDB is an excellent choice for **data quality checks in CI/CD** because it requires no warehouse connection and starts in milliseconds.

### Example: Data Validation in a GitHub Actions Workflow

```yaml
# .github/workflows/data-quality.yml
name: Data Quality Checks
on: [push]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install DuckDB CLI
        run: |
          wget -q https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip
          unzip duckdb_cli-linux-amd64.zip

      - name: Run quality checks
        run: |
          ./duckdb -c "
            SELECT
              COUNT(*) AS row_count,
              COUNT(*) FILTER (WHERE customer_id IS NULL) AS null_customer_ids,
              COUNT(DISTINCT product_id) AS distinct_products
            FROM read_parquet('data/output/*.parquet');
          "
```

---

## DuckDB with dbt

The **dbt-duckdb** adapter lets you run [[Core dbt Fundamentals|dbt]] projects against DuckDB, which is particularly valuable for local development and testing. Instead of running transformations against a cloud warehouse during development, you can iterate locally with instant feedback.

### dbt Profile Configuration

```yaml
# ~/.dbt/profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: "target/dev.duckdb"
      threads: 4
      extensions:
        - httpfs
        - parquet
      settings:
        memory_limit: "4GB"

```

### Workflow

1. Develop models locally with `dbt run` against DuckDB.
2. Run `dbt test` — assertions execute in milliseconds.
3. When satisfied, switch the target profile to your production warehouse (Snowflake, BigQuery, etc.) and deploy.

This approach dramatically shortens the feedback loop and eliminates warehouse costs during development.

---

## MotherDuck — Cloud-Hosted DuckDB

**MotherDuck** is a managed cloud service that hosts DuckDB databases remotely while preserving the DuckDB SQL dialect and client experience. It enables:

- **Hybrid execution** — queries can run partially on the client and partially in the cloud.
- **Sharing** — databases can be shared across team members without exporting files.
- **Persistence** — cloud-hosted storage with automatic backups.

Connect with `duckdb.connect("md:my_database?motherduck_token=<token>")`. MotherDuck is worth evaluating when you need DuckDB's simplicity but want persistent, shareable cloud storage without managing infrastructure yourself.

---

## Limitations

DuckDB is not a general-purpose replacement for all database workloads. Key constraints to be aware of:

- **Single-node only** — there is no distributed execution. If your dataset exceeds what a single machine can handle (roughly 100-200 GB for comfortable interactive use, though larger is possible), you need a distributed engine like Spark or a cloud warehouse.
- **Not designed for concurrent multi-user workloads** — DuckDB supports a single writer at a time. It is not suitable as a backend for a multi-user application or a shared analytics server.
- **No built-in replication** — there is no native replication or high-availability mechanism. For production serving, a dedicated warehouse or database is more appropriate.
- **No row-level updates at scale** — while DuckDB supports `UPDATE` and `DELETE`, it is optimised for bulk reads and appends, not high-frequency transactional writes.
- **Extension ecosystem is growing but not exhaustive** — some connectors and integrations available in mature platforms may not yet exist for DuckDB.

---

## Related Notes

- [[PySpark Core Concepts]] — for distributed workloads that exceed single-node capacity.
- [[Core dbt Fundamentals]] — dbt transformation orchestration, pairs well with dbt-duckdb for local dev.
- [[pandas & Polars for Data Engineering]] — programmatic data manipulation alongside DuckDB SQL.
- [[SQL Query Optimization]] — query tuning principles applicable to DuckDB's SQL dialect.

---

## Recipe Cookbook

Practical query recipes for common data engineering tasks. Each recipe is self-contained and can be run in the DuckDB CLI or via Python.

### Read Remote Parquet and Join with Local CSV

```sql
INSTALL httpfs;
LOAD httpfs;

-- Join a remote Parquet fact table with a local CSV dimension
SELECT
    d.region_name,
    d.country,
    COUNT(*) AS order_count,
    SUM(f.amount) AS total_amount,
    AVG(f.amount) AS avg_amount
FROM read_parquet('s3://data-lake/orders/year=2025/**/*.parquet',
                  hive_partitioning = true) AS f
JOIN read_csv('reference_data/regions.csv',
              header = true,
              auto_detect = true) AS d
    ON f.region_id = d.region_id
GROUP BY d.region_name, d.country
ORDER BY total_amount DESC;
```

### Pivot Table from Long to Wide

```sql
-- Source: long-format table with (product, month, revenue)
-- Target: one row per product, one column per month
PIVOT (
    SELECT product, month_name, revenue
    FROM monthly_sales
)
ON month_name
USING SUM(revenue)
GROUP BY product
ORDER BY product;

-- Manual pivot with conditional aggregation (more control)
SELECT
    product,
    SUM(CASE WHEN month_name = 'January' THEN revenue END) AS january,
    SUM(CASE WHEN month_name = 'February' THEN revenue END) AS february,
    SUM(CASE WHEN month_name = 'March' THEN revenue END) AS march
FROM monthly_sales
GROUP BY product;
```

### Data Quality Checks

```sql
-- Comprehensive quality profile for a Parquet dataset
WITH source AS (
    SELECT * FROM read_parquet('output/customers.parquet')
)
SELECT
    COUNT(*) AS total_rows,

    -- Null counts per column
    COUNT(*) FILTER (WHERE customer_id IS NULL) AS null_customer_id,
    COUNT(*) FILTER (WHERE email IS NULL) AS null_email,
    COUNT(*) FILTER (WHERE created_at IS NULL) AS null_created_at,

    -- Duplicate detection
    COUNT(*) - COUNT(DISTINCT customer_id) AS duplicate_customer_ids,

    -- Distribution checks
    MIN(created_at) AS earliest_record,
    MAX(created_at) AS latest_record,
    APPROX_COUNT_DISTINCT(email) AS approx_unique_emails,

    -- Numeric distribution
    AVG(lifetime_value) AS avg_lifetime_value,
    MEDIAN(lifetime_value) AS median_lifetime_value,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY lifetime_value)
        AS p95_lifetime_value,
    STDDEV(lifetime_value) AS stddev_lifetime_value
FROM source;

-- Row-level duplicate check with detail
SELECT customer_id, COUNT(*) AS occurrences
FROM read_parquet('output/customers.parquet')
GROUP BY customer_id
HAVING COUNT(*) > 1
ORDER BY occurrences DESC
LIMIT 20;
```

### Export to Parquet with Partitioning

```sql
-- Write partitioned Parquet output (Hive-style directory structure)
COPY (
    SELECT
        *,
        YEAR(event_date) AS year,
        MONTH(event_date) AS month
    FROM read_csv('raw_events.csv', header = true, auto_detect = true)
)
TO 'output/events'
(FORMAT PARQUET,
 PARTITION_BY (year, month),
 OVERWRITE_OR_IGNORE true,
 COMPRESSION 'zstd',
 ROW_GROUP_SIZE 100000);
```

The resulting directory structure is:

```
output/events/year=2025/month=1/data_0.parquet
output/events/year=2025/month=2/data_0.parquet
...
```

### Compare Two Datasets (EXCEPT)

```sql
-- Find rows in the new dataset that are not in the baseline
-- Useful for regression testing after refactoring a transformation
SELECT 'in_new_only' AS diff_type, *
FROM read_parquet('output/v2/customers.parquet')
EXCEPT
SELECT 'in_new_only', *
FROM read_parquet('output/v1/customers.parquet')

UNION ALL

SELECT 'in_baseline_only' AS diff_type, *
FROM read_parquet('output/v1/customers.parquet')
EXCEPT
SELECT 'in_baseline_only', *
FROM read_parquet('output/v2/customers.parquet');

-- Summary count of differences
SELECT
    (SELECT COUNT(*) FROM (
        SELECT * FROM read_parquet('output/v2/customers.parquet')
        EXCEPT
        SELECT * FROM read_parquet('output/v1/customers.parquet')
    )) AS rows_added_or_changed,
    (SELECT COUNT(*) FROM (
        SELECT * FROM read_parquet('output/v1/customers.parquet')
        EXCEPT
        SELECT * FROM read_parquet('output/v2/customers.parquet')
    )) AS rows_removed_or_changed;
```

### JSON Unnesting

```sql
-- Unnest nested JSON arrays into relational rows
SELECT
    j.order_id,
    j.customer_name,
    unnested.item_name,
    unnested.quantity,
    unnested.unit_price,
    unnested.quantity * unnested.unit_price AS line_total
FROM read_json_auto('orders.json') AS j,
     LATERAL UNNEST(j.line_items) AS unnested(item_name, quantity, unit_price);

-- Deeply nested JSON with struct access
SELECT
    j->>'event_id' AS event_id,
    j->'metadata'->>'source' AS source,
    j->'metadata'->'tags' AS tags,
    json_array_length(j->'metadata'->'tags') AS tag_count
FROM read_json('events.ndjson', format = 'newline_delimited') AS t(j);
```

### Time-Series Gap Detection

```sql
-- Detect missing dates in a daily time series
WITH date_spine AS (
    SELECT UNNEST(generate_series(
        DATE '2025-01-01',
        DATE '2025-12-31',
        INTERVAL '1 day'
    )) AS expected_date
),
actual_dates AS (
    SELECT DISTINCT event_date
    FROM read_parquet('metrics/daily_metrics.parquet')
)
SELECT
    ds.expected_date AS missing_date,
    ds.expected_date::VARCHAR AS day_of_week
FROM date_spine ds
LEFT JOIN actual_dates ad
    ON ds.expected_date = ad.event_date
WHERE ad.event_date IS NULL
ORDER BY ds.expected_date;

-- Detect gaps larger than a threshold in irregular time series
WITH ordered AS (
    SELECT
        sensor_id,
        reading_time,
        LAG(reading_time) OVER (PARTITION BY sensor_id ORDER BY reading_time)
            AS prev_reading_time
    FROM read_parquet('iot/sensor_readings.parquet')
)
SELECT
    sensor_id,
    prev_reading_time AS gap_start,
    reading_time AS gap_end,
    AGE(reading_time, prev_reading_time) AS gap_duration
FROM ordered
WHERE AGE(reading_time, prev_reading_time) > INTERVAL '1 hour'
ORDER BY gap_duration DESC;
