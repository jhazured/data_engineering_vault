# ETL Pipeline Templates & Patterns

This note documents pipeline configuration patterns drawn from the `gcp_datamigration` framework -- a Python-based ETL system that uses YAML/Jinja2 configs to define extract-transform-load jobs declaratively. Cross-reference with [[Data Ingestion Patterns]], [[Core dbt Fundamentals]], and [[SCD Type 2 Patterns]] for the broader data engineering picture.

---

## 1 -- ETL vs ELT Decision Framework

The choice between ETL (extract-transform-load) and ELT (extract-load-transform) shapes every downstream decision: where compute runs, how much latency you tolerate, and what skills the team needs.

### Trade-offs at a glance

| Dimension | ETL (Spark / Python) | ELT (dbt / SQL) | Hybrid |
|---|---|---|---|
| **Compute location** | External cluster (Dataproc, EMR, local) | Inside the warehouse (BigQuery, Snowflake) | Split across both |
| **Latency** | Higher -- data moves twice | Lower -- transform in-place | Varies by stage |
| **Complexity** | Python/Scala code, unit tests, CI/CD | SQL models, ref-based DAGs | Both skill sets required |
| **Cost model** | Cluster hours + egress | Query slots / credits | Blended |
| **Schema evolution** | Manual migration scripts | `dbt run` handles incremental | Depends on layer |
| **Best fit** | Complex transformations, ML features, binary/image data | Aggregations, joins, window functions, reporting | Raw ingest + warehouse transforms |

### When to pick each

- **Spark ETL** -- unstructured or semi-structured files, heavy data science feature engineering, cross-system joins that cannot happen inside one warehouse.
- **dbt ELT** -- analytics engineering on already-landed data; the warehouse is the single source of truth and SQL is sufficient. See [[Core dbt Fundamentals]].
- **Hybrid** -- the `gcp_datamigration` framework exemplifies this: Python handles extraction and lightweight pandas transforms, then loads into BigQuery where further SQL modelling can occur.

---

## 2 -- YAML-based Pipeline Configuration

The framework separates *what* a pipeline does (YAML config) from *how* it does it (Python framework code). The `ConfigManager` class in `framework/config.py` enforces three required top-level sections:

```python
required_sections = ['job_name', 'source', 'destination']
```

A minimal config looks like:

```yaml
job_name: "sales_summary"

sources:
  orders:
    type: query
    connection: my_database
    query: "SELECT order_id, product_id, order_amount, order_date
            FROM orders WHERE status = 'COMPLETE'"

query:
  target_table: sales_summary
  sql: "SELECT p.product_id, COUNT(o.order_id) AS orders_count ...
        GROUP BY p.product_id"

mapping:
  product_id: prod_id
  orders_count: total_orders
```

### Key design principles

1. **Declarative over imperative** -- pipeline authors define sources, SQL, and column mappings without touching Python.
2. **Config validation at load time** -- `ConfigManager._validate_config()` rejects configs missing `job_name`, `source`, or `destination`, and requires both `source.type` and `destination.type`.
3. **Config caching** -- repeated loads of the same file + environment combo hit an in-memory cache (`config_cache`), avoiding redundant Jinja2 rendering.
4. **Separation of concerns** -- the `ETLProcessor` orchestrates three discrete phases (extract, transform, load) through dedicated classes (`DataExtractor`, `DataTransformer`, `DataLoader`), each receiving only its relevant config slice.

---

## 3 -- Jinja2 Templating for Pipelines

Config files use the `.yaml.j2` extension to signal Jinja2 rendering. The `ConfigManager` loads templates via `jinja2.Environment` with `FileSystemLoader` pointed at the template's parent directory, then renders with merged variables:

```python
env_vars = self._load_environment_variables()
template_vars = {**env_vars, **(variables or {})}
rendered_yaml = template.render(**template_vars)
config = yaml.safe_load(rendered_yaml)
```

### Variable injection

Variables flow from two sources merged together:

1. **Environment variables** -- `PROJECT_ID`, `REGION`, `BQ_LOCATION`, `GCS_TEMP_BUCKET`, plus anything in `env/{environment}.env`.
2. **Caller-supplied variables** -- passed as the `variables` dict to `load_config()`.

The product inventory template shows dynamic job naming:

```yaml
job_name: "product_inventory_report_{{ report_date }}"
```

### Conditional blocks

Jinja2 `{% if %}` blocks let a single template serve multiple use cases. From `product_inventory_report.yaml.j2`:

```yaml
query:
  "SELECT product_id, product_name, category, stock_quantity
  FROM products
  WHERE stock_quantity <= {{ min_stock_threshold | default(50) }}
  {% if category_filter is defined and category_filter %}
  AND category = '{{ category_filter }}'
  {% endif %}"
```

When `category_filter` is not supplied the clause is omitted entirely, producing a broader query.

### Default values

The `| default()` filter prevents failures when optional variables are absent:

```yaml
WHEN stock_quantity <= {{ critical_stock_level | default(10) }} THEN 'Critical'
WHEN stock_quantity <= {{ low_stock_level | default(30) }} THEN 'Low'
```

### Environment-specific overrides

The `ConfigManager` loads `env/{environment}.env` files automatically. Passing `environment='prod'` loads `env/prod.env`, overriding defaults for `PROJECT_ID`, bucket names, and region without touching the template. This keeps the same `.yaml.j2` file deployable across dev, staging, and production.

---

## 4 -- Source Configuration Patterns

Sources are declared under a top-level `sources:` key. Each source has a name, type, connection, and query.

### Table source (full table read)

```yaml
sources:
  customers:
    type: table
    connection: my_database
    query: "SELECT customer_id, customer_name, active FROM customers"
```

### Query source (filtered/joined read)

```yaml
sources:
  orders:
    type: query
    connection: my_database
    query: "SELECT order_id, customer_id, order_date
            FROM orders WHERE status = 'COMPLETE'"
```

The distinction between `type: table` and `type: query` signals to the `DataExtractor` whether to do a full-table scan or execute arbitrary SQL. Both ultimately pass SQL to the connection, but the type hint lets the framework optimise (e.g., table reads may use bulk export APIs).

### Multi-source pipelines

Both `customer_order_frequency` and `sales_summary` define two sources that are later joined in the `query.sql` block:

```yaml
sources:
  orders:
    type: query
    connection: my_database
    query: "SELECT order_id, product_id, order_amount, order_date
            FROM orders WHERE status = 'COMPLETE'"
  products:
    type: query
    connection: my_database
    query: "SELECT product_id, product_name, category FROM products"
```

The `query.sql` block references these as if they were tables, enabling the framework to register each source as a temporary view before running the join SQL.

For other source types (GCS files, APIs, Cloud SQL), the same block structure applies with type-specific keys such as `format`, `bucket`, `endpoint`, etc. See [[Data Ingestion Patterns]] for ingestion-specific configurations.

---

## 5 -- Transformation Chains

The `ETLProcessor` passes the `transformations` list from config to `DataTransformer.apply_transformations()`. If no transformations are configured, the raw extracted data flows straight to load:

```python
transformations = self.config.get('transformations', [])
if not transformations:
    self.logger.info("No transformations configured, returning original data")
    return data
```

### Column mapping (rename)

All three project templates use a `mapping:` section for column renaming:

```yaml
mapping:
  product_id: prod_id
  product_name: prod_name
  category: prod_category
  orders_count: total_orders
  total_revenue: revenue_total
  avg_order_value: average_order_value
```

### SQL-level transformations

Rather than configuring filter/aggregate steps as separate YAML entries, the templates push transformation logic into the `query.sql` block. This is a pragmatic choice: SQL is already the lingua franca for tabular transforms, and embedding it in the config keeps the pipeline self-contained.

From `customer_order_frequency.yaml.j2`:

```yaml
query:
  target_table: customer_order_frequency
  sql: "SELECT
      c.customer_id,
      COUNT(o.order_id) AS total_orders,
      COUNT(DISTINCT DATE_TRUNC('month', o.order_date)) AS active_months,
      ROUND(COUNT(o.order_id) * 1.0 /
        NULLIF(COUNT(DISTINCT DATE_TRUNC('month', o.order_date)), 0), 2)
        AS avg_orders_per_month
    FROM customers c
      LEFT JOIN orders o ON c.customer_id = o.customer_id
    WHERE c.active = TRUE
    GROUP BY c.customer_id"
```

### Declarative transformation YAML (general pattern)

For frameworks that support it, transformation chains can be expressed as ordered steps:

```yaml
transformations:
  - type: filter
    condition: "active = TRUE"
  - type: aggregate
    group_by: [customer_id]
    measures:
      - column: order_id
        function: count
        alias: total_orders
  - type: derive
    column: avg_monthly_orders
    expression: "total_orders / months_active"
  - type: rename
    mapping:
      customer_id: cust_id
```

The `gcp_datamigration` framework supports this via `DataTransformer`, though the project templates prefer inline SQL for readability.

---

## 6 -- Destination Configuration Patterns

The `ETLProcessor` delegates loading to `DataLoader`, initialised with the `destination` config block. The framework validates that `destination.type` is present.

### Write modes

Common write-mode patterns (supported by the framework's validation layer):

| Mode | Behaviour | Use case |
|---|---|---|
| `truncate` | Delete existing rows, then insert | Full-refresh dimension loads |
| `append` | Insert without deleting | Fact/event tables, audit logs |
| `empty` | Only write if target is empty | First-time seed loads |
| `merge` | Upsert on key columns | [[SCD Type 2 Patterns]], incremental loads |

### Target table naming

The product inventory template uses a date-stamped target table, creating snapshot isolation:

```yaml
query:
  target_table: product_inventory_report_{{ report_date }}
```

### Partitioning and clustering (BigQuery pattern)

```yaml
destination:
  type: bigquery
  project: "{{ PROJECT_ID }}"
  dataset: analytics
  table: sales_summary
  write_mode: truncate
  partitioning:
    field: order_date
    type: DAY
  clustering:
    fields: [category, product_id]
```

### Multi-destination loading

Some pipelines need to write the same transformed data to multiple sinks (e.g., BigQuery for analytics and GCS for archival). The pattern extends `destination` to a list:

```yaml
destinations:
  - type: bigquery
    dataset: analytics
    table: sales_summary
    write_mode: truncate
  - type: gcs
    bucket: "{{ GCS_TEMP_BUCKET }}"
    path: "archive/sales_summary/{{ report_date }}/"
    format: parquet
```

---

## 7 -- Parameterised SQL in Pipelines

Jinja2 variables turn static SQL into reusable, schedule-aware queries. The project templates demonstrate three parameterisation patterns.

### Date filters

```yaml
query: "SELECT ... FROM orders
        WHERE order_date >= '{{ start_date | default('2024-01-01') }}'"
```

### Threshold parameters

The inventory report uses tuneable stock-level thresholds:

```yaml
WHERE stock_quantity <= {{ min_stock_threshold | default(50) }}
...
CASE
  WHEN stock_quantity <= {{ critical_stock_level | default(10) }} THEN 'Critical'
  WHEN stock_quantity <= {{ low_stock_level | default(30) }} THEN 'Low'
  ELSE 'Normal'
END AS stock_status
```

This lets operations teams adjust alert sensitivity without modifying the template.

### Category filters (conditional injection)

```yaml
{% if category_filter is defined and category_filter %}
AND category = '{{ category_filter }}'
{% endif %}
```

The `is defined and category_filter` guard handles both missing variables and empty strings, keeping the generated SQL clean when the filter is not needed.

### Environment-level injection

The `ConfigManager` automatically provides `ENVIRONMENT`, `PROJECT_ID`, `REGION`, `BQ_LOCATION`, and `GCS_TEMP_BUCKET` as template variables, sourced from OS environment variables and `env/{environment}.env` files:

```python
env_vars = {
    'ENVIRONMENT': self.environment,
    'PROJECT_ID': os.getenv('PROJECT_ID'),
    'REGION': os.getenv('REGION', 'us-central1'),
    'BQ_LOCATION': os.getenv('BQ_LOCATION', 'US'),
    'GCS_TEMP_BUCKET': os.getenv('GCS_TEMP_BUCKET'),
}
```

---

## 8 -- Common Pipeline Archetypes

Drawing from the project templates and general ETL practice, these are the recurring pipeline shapes.

### Dimension load (full refresh)

Refreshes a dimension table entirely on each run. Suits slowly changing reference data.

```yaml
job_name: "dim_customer_load"
sources:
  customers:
    type: table
    connection: warehouse_db
    query: "SELECT customer_id, customer_name, segment, region FROM customers"
query:
  target_table: dim_customer
  sql: "SELECT * FROM customers"
mapping:
  customer_id: cust_key
  customer_name: cust_name
destination:
  type: bigquery
  write_mode: truncate
```

See [[SCD Type 2 Patterns]] for handling historical dimension changes.

### Fact aggregation

The `sales_summary` template is the canonical example -- join fact and dimension sources, aggregate, and load:

```yaml
job_name: "sales_summary"
sources:
  orders:
    type: query
    connection: my_database
    query: "SELECT order_id, product_id, order_amount, order_date
            FROM orders WHERE status = 'COMPLETE'"
  products:
    type: query
    connection: my_database
    query: "SELECT product_id, product_name, category FROM products"
query:
  target_table: sales_summary
  sql: "SELECT p.product_id, p.product_name, p.category,
      COUNT(o.order_id) AS orders_count,
      SUM(o.order_amount) AS total_revenue,
      AVG(o.order_amount) AS avg_order_value
    FROM products p
      LEFT JOIN orders o ON p.product_id = o.product_id
    WHERE o.order_date >= '2024-01-01'
    GROUP BY p.product_id, p.product_name, p.category"
```

### Inventory / status snapshot

The `product_inventory_report` template captures point-in-time state with parameterised thresholds and a date-stamped target table:

```yaml
job_name: "product_inventory_report_{{ report_date }}"
query:
  target_table: product_inventory_report_{{ report_date }}
  sql: "SELECT product_id, product_name, category, stock_quantity,
      CASE
        WHEN stock_quantity <= {{ critical_stock_level | default(10) }} THEN 'Critical'
        WHEN stock_quantity <= {{ low_stock_level | default(30) }} THEN 'Low'
        ELSE 'Normal'
      END AS stock_status
    FROM products
    WHERE stock_quantity <= {{ min_stock_threshold | default(50) }}"
```

Each run produces an immutable snapshot table, enabling trend analysis across dates.

### CDC incremental (general pattern)

For change-data-capture pipelines, combine a watermark variable with merge write mode:

```yaml
job_name: "orders_incremental"
sources:
  orders:
    type: query
    connection: my_database
    query: "SELECT * FROM orders
            WHERE updated_at > '{{ last_watermark | default('1970-01-01') }}'"
query:
  target_table: orders_fact
  sql: "SELECT * FROM orders"
destination:
  type: bigquery
  write_mode: merge
  merge_keys: [order_id]
```

The `last_watermark` variable is supplied by the orchestrator (e.g., Airflow XCom) from the previous run's max `updated_at` value.

---

## 9 -- REST API Ingestion Patterns (Data Factory / Fabric)

When the source is a REST API rather than a database or file, ingestion requires pagination, rate-limit handling, and load-strategy decisions that don't apply to batch sources. These patterns are implemented as Data Factory (or Fabric) pipelines using Copy activities inside `Until` loops, driven by a control table.

See [[Microsoft Fabric & Azure Data Services#Control Table-Driven Pipeline Orchestration]] for the full metadata-driven pipeline architecture.

### Three Load Strategies

| Strategy | When to Use | T1 Behaviour | SCD2 Soft Deletes? |
|----------|-------------|-------------|:------------------:|
| **Snapshot** | Source returns complete dataset; small-to-medium tables (reference data, master data) | Truncate → full reload every run | Yes — absence = deletion |
| **Windowed** | Large tables with a date field; need recent history but not full reload | Truncate → load rolling N-day window (e.g. 90 days) | No — partial dataset |
| **Incremental** | Initial history load or CDC via `modifiedon` watermark | Append only — watermark advances each run | No — partial dataset |

**Key insight:** The same pipeline child template handles all three — only the URL construction and truncate behaviour differ, controlled by `load_type` in the control table.

### Offset-Based Pagination (OData-style)

For APIs that support `$top` / `$skip` / `$orderby` (OData convention), use an `Until` loop that terminates when the returned row count is less than the page size:

```
Until: v_row_count < p_page_size
│
├── [Copy Activity: copy_api_to_t1]
│   Source: GET /table/{entity}?$orderby=id&$top={page_size}&$skip={offset}
│   Sink:   Fabric Warehouse table (append)
│   Retry:  3 attempts, 30-second intervals
│
├── v_row_count   = copy_api_to_t1.output.rowsCopied
├── v_total_rows += v_row_count        (via temp variable swap)
└── v_offset     += p_page_size        (via temp variable swap)
```

**Why `$orderby` matters:** Without a stable sort order, the API may return overlapping or missing rows across pages. Always order by a unique, immutable column (e.g., `id`).

**Termination condition:** `v_row_count < p_page_size` catches both partial pages (fewer rows than requested) and empty responses (zero rows). For APIs that return exact multiples of page_size, the next iteration will return 0 rows and terminate.

**Variable self-assignment workaround:** Fabric pipelines cannot self-assign a variable (`v_offset = v_offset + page_size` is not allowed). Use a temp-variable swap:

```
v_offset_temp = v_offset + p_page_size    -- calculate into temp
v_offset      = v_offset_temp             -- swap back
```

### Snapshot Pattern (Full Reload)

```
API URL: /table/{entity}?$orderby={entity}.id&$top={page_size}&$skip={offset}
```

- **Before pagination:** `TRUNCATE TABLE [schema].[table]` — T1 is completely replaced each run
- **Page size:** Larger (e.g. 20,000) since the full dataset is read every time
- **After success:** T2 snapshot merge runs (includes soft deletes for absent records)
- **Schedule:** Typically 2x daily (e.g. 4:00 AM and 11:00 AM)

### Windowed Pattern (Rolling Date Range)

```
API URL: /table/{entity}?$filter=starttime>='{calculated_date}'&$orderby={entity}.id&$top={page_size}&$skip={offset}
```

- **Before pagination:** `TRUNCATE TABLE [schema].[table]` — old window data is replaced
- **Date calculation:** `today - rolling_days` from control table (e.g. 90 days for BAU, 3650 for initial load)
- **Page size:** Smaller (e.g. 10,000) since windowed data may be larger per page
- **After success:** T2 incremental merge runs (no soft deletes — windowed data is partial)
- **Use case:** Large transactional tables where full reload is too expensive but you need recent history refreshed

### Incremental Pattern (Watermark-Based)

```
API URL: /table/{entity}?$filter=modifiedon>'{last_watermark}'&$orderby={entity}.id&$top={page_size}&$skip={offset}
```

- **No truncate** — T1 is appended to (only new/changed records since last watermark)
- **Watermark source:** `last_watermark_ts` from control table, initialised to `1900-01-01` for first run
- **After success:** T2 incremental merge runs; watermark updated to current timestamp
- **Transition:** After the initial full history load completes, switch the table's `load_type` from `incremental` to `windowed` for BAU operations

### Per-Table API Query Overrides

Some APIs need table-specific query parameters (column filters, custom predicates). Store these in the control table's `api_query_suffix` column rather than branching pipeline logic:

```
-- Control table row:
api_query_suffix = '?$filter=starttime>=''2025-01-01'''

-- Pipeline URL construction:
/table/{src_table_name}{api_query_suffix}&$orderby=id&$top={page_size}&$skip={offset}

-- If api_query_suffix is NULL or empty:
/table/{src_table_name}?$orderby=id&$top={page_size}&$skip={offset}
```

The child pipeline handles the `?` vs `&` delimiter dynamically based on whether the suffix is present.

### Column Mapping via Control Table

The Copy activity's `translator` property accepts a JSON column mapping. Storing this in the control table (`column_mapping_json`) means no pipeline changes when the source schema evolves — just update the JSON:

```json
{
  "type": "TabularTranslator",
  "mappings": [
    {"source": {"path": "$['id']"},       "sink": {"name": "id"}},
    {"source": {"path": "$['name']"},     "sink": {"name": "name"}},
    {"source": {"path": "$['groupid']"},  "sink": {"name": "groupid"}}
  ]
}
```

This also serves as the source of truth for which columns are included in the hash for SCD2 change detection.

### Page Size Tuning

| Factor | Guidance |
|--------|----------|
| API rate limits | Larger pages = fewer calls. Match the API's maximum. |
| Response time | If pages take >30 seconds, reduce page size to avoid timeouts |
| Memory pressure | Fabric Copy activity stages through ADLS — large pages increase staging file size |
| Retry cost | On failure, only the current page is retried (3x with 30-second backoff) |

**Typical values:** 20,000 for snapshot (small reference tables), 10,000 for windowed (larger transactional data).

---

## Framework Orchestration Flow

The `ETLProcessor` in `framework/etl.py` ties everything together in a three-phase pipeline:

1. **Initialize** -- `ConfigManager` renders the Jinja2 template, parses YAML, validates required sections, and caches the result.
2. **Extract** -- `DataExtractor` reads each source defined in config.
3. **Transform** -- `DataTransformer` applies the transformation chain (or passes data through if none are configured).
4. **Load** -- `DataLoader` writes to the configured destination.

Metrics are collected at each phase boundary (`MetricsCollector`), and the processor supports a `dry_run` mode that extracts data but skips transform and load -- useful for validating source connectivity and row counts before a production run.

```python
processor = ETLProcessor(
    config_path='etls/sales/sales_summary.yaml.j2',
    environment='prod',
    dry_run=False
)
processor.execute()
```

---

## Related Notes

- [[Data Ingestion Patterns]] -- source-specific extraction strategies
- [[Core dbt Fundamentals]] -- ELT-side modelling with dbt
- [[SCD Type 2 Patterns]] -- handling historical dimension changes
- [[Data Quality & Testing Patterns]] -- validating pipeline outputs
