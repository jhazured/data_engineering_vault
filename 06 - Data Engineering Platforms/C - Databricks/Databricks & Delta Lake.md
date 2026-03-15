# Databricks & Delta Lake

## Platform Overview

Databricks is a unified analytics platform built on Apache Spark, providing a lakehouse architecture that combines the best of data warehouses and data lakes.

**Core components:**
- **Workspace** -- collaborative environment for notebooks, repos, and dashboards
- **Clusters** -- managed Spark compute (All-Purpose for interactive dev, Job Clusters for pipelines, SQL Warehouses for BI)
- **Notebooks** -- multi-language cells (Python, SQL, Scala, R) with built-in visualization
- **Repos** -- Git integration for version-controlled notebooks and project files
- **Unity Catalog** -- centralized governance with three-level namespace (`catalog.schema.table`)
- **Photon Engine** -- C++ vectorized query engine for accelerated SQL and DataFrame workloads

## Delta Lake Fundamentals

Open-source storage layer bringing ACID transactions to data lakes, built on Parquet files with a transaction log.

| Feature | What It Does |
|---------|-------------|
| **ACID Transactions** | Atomic writes -- no partial or corrupt data |
| **Time Travel** | Query historical versions: `SELECT * FROM t VERSION AS OF 5` |
| **Schema Evolution** | Add columns without rewriting: `.option("mergeSchema", "true")` |
| **Schema Enforcement** | Rejects writes that do not match the table schema |
| **OPTIMIZE/Z-Order** | Compact files and co-locate data: `OPTIMIZE t ZORDER BY (col)` |
| **VACUUM** | Remove old data files: `VACUUM t RETAIN 168 HOURS` |
| **ANALYZE** | Collect statistics for the query optimizer |
| **Change Data Feed** | Track row-level changes for CDC downstream |

```sql
-- Time travel
SELECT * FROM shipments VERSION AS OF 42;
SELECT * FROM shipments TIMESTAMP AS OF '2024-01-15 10:00:00';
RESTORE TABLE shipments TO VERSION AS OF 42;

-- Upsert via MERGE
MERGE INTO silver.shipments AS target
USING bronze.shipments_raw AS source
ON target.shipment_id = source.shipment_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

## Medallion Architecture

The project implements a three-layer architecture orchestrated by `MedallionPipeline`, chaining `BronzeLayerProcessor` -> `SilverLayerProcessor` -> `GoldLayerProcessor`. See [[Delta Lake Operations & Patterns]].

| Layer | Purpose | Key Metadata Columns |
|-------|---------|---------------------|
| **Bronze** | Raw ingestion, append-only | `_bronze_ingestion_timestamp`, `_bronze_source`, `_bronze_batch_id` |
| **Silver** | Cleaned, deduplicated, standardized | `_silver_processed_timestamp`, `_silver_source_table`, `_silver_quality_score` |
| **Gold** | Aggregated KPIs, ML features, reports | `time_group`, aggregation-level summaries |

### Bronze Layer -- Raw Ingestion

`BronzeLayerProcessor` validates raw data and stamps metadata:

```python
df["_bronze_ingestion_timestamp"] = get_utc_now()
df["_bronze_source"] = source
df["_bronze_batch_id"] = f"batch_{get_utc_now().strftime('%Y%m%d_%H%M%S')}"
df["_bronze_record_count"] = len(df)
```

Quality metrics calculated at ingestion: total records, null counts, duplicate count, completeness percentage, memory usage. Persisted as Parquet.

### Silver Layer -- Cleaning & Standardization

Chains four sub-processors, assessing quality before and after to measure improvement:

1. **DataCleaningProcessor** -- dedup, null handling, text standardization, phone/email validation, outlier capping
2. **DataStandardizationProcessor** -- country/state codes to ISO, status to single-char, date/currency normalization
3. **DataQualityProcessor** -- five-dimension quality scoring
4. **DataEnrichmentProcessor** -- geographic regions, temporal features, customer segmentation

### Gold Layer -- Business Metrics & Aggregations

`GoldLayerProcessor` orchestrates:

1. **BusinessMetricsProcessor** -- revenue, customer count, conversion, retention, churn, AOV, CLV
2. **AggregationProcessor** -- time-based grouping (daily/weekly/monthly/quarterly/yearly)
3. **MLFeatureProcessor** -- numeric transforms, one-hot encoding, cyclical temporal features, interaction features
4. **ReportingProcessor** -- executive summary, customer analytics, financial report, operational metrics

## Data Quality Framework

See [[Data Validation & Quality Frameworks]] for the broader pattern.

### Five-Dimension Scoring

```python
class DataQualityMetrics:
    completeness: float   # Percentage of non-null values
    accuracy: float       # Format validation (email regex, phone patterns)
    consistency: float    # Date format consistency, numeric type alignment
    validity: float       # Business rule checks (age 0-150, valid status codes)
    uniqueness: float     # Duplicate detection on business keys (id, email)
    overall_score: float  # Average of the five dimensions
    quality_level: DataQualityLevel
```

| Level | Score Range | Meaning |
|-------|-----------|---------|
| **EXCELLENT** | 95-100% | Production-ready |
| **GOOD** | 85-94% | Acceptable, minor gaps |
| **FAIR** | 70-84% | Needs attention |
| **POOR** | < 70% | Requires remediation |

### Cleaning Rules

```python
cleaning_rules = {
    "remove_duplicates": True,       # Dedup on business keys (id, email, phone)
    "handle_missing_values": True,   # Strings->"Unknown", numerics->median, bools->False
    "standardize_text": True,        # Trim whitespace, title case (skip id/email/phone)
    "normalize_phone_numbers": True, # Normalize to +1-XXX-XXX-XXXX
    "validate_emails": True,         # Regex validation, mark invalid as "Invalid"
    "clean_numeric_data": True,      # IQR outlier capping (1.5 * IQR bounds)
}
```

## Silver Layer Transformations

**Deduplication** -- removes duplicates on business keys, keeps first occurrence.

**Null handling:**

| Data Type | Fill Strategy |
|-----------|--------------|
| String/object | `"Unknown"` |
| Numeric | Column median |
| Boolean | `False` |

**Outlier detection (IQR)** -- outliers are capped, not removed:

```python
Q1 = df[column].quantile(0.25)
Q3 = df[column].quantile(0.75)
IQR = Q3 - Q1
df[column] = df[column].clip(lower=Q1 - 1.5*IQR, upper=Q3 + 1.5*IQR)
```

**Standardization mappings:**
- Country codes -- "USA"/"United States" -> "US", "UK"/"United Kingdom" -> "GB"
- Status values -- "active" -> "A", "inactive" -> "I", "pending" -> "P"
- Dates -> ISO `YYYY-MM-DD HH:MM:SS`; currency -> stripped symbols, float, 2 decimals

**Enrichment:**
- Geographic -- state-to-region (CA -> "West", NY -> "Northeast", TX -> "South", IL -> "Midwest")
- Temporal -- year/month/day/weekday extraction, `days_since_creation`
- Customer segmentation -- age group + activity status (e.g. "Young Active", "Mature Inactive")
- Derived fields -- email domain, age group binning, full name concatenation

## Gold Layer Metrics

### Business Metric Types

```python
class BusinessMetricType(Enum):
    REVENUE = "revenue"
    CUSTOMER_COUNT = "customer_count"
    CONVERSION_RATE = "conversion_rate"
    RETENTION_RATE = "retention_rate"
    CHURN_RATE = "churn_rate"
    AVERAGE_ORDER_VALUE = "average_order_value"
    CUSTOMER_LIFETIME_VALUE = "customer_lifetime_value"
```

**Aggregation levels:** DAILY, WEEKLY, MONTHLY, QUARTERLY, YEARLY -- uses `pd.Series.dt.to_period()` for time bucketing, then aggregates numeric (sum/mean/count/min/max) and categorical (count/nunique) columns.

**Report types:** executive summary, customer analytics, financial report, operational metrics.

## Databricks REST API

Connection manager pattern (`DatabricksConnection`) with `requests.Session` and Bearer token auth:

```python
@dataclass
class DatabricksConfig:
    host: str
    token: str
    cluster_id: Optional[str] = None
    catalog: str = "main"
    schema: str = "default"
```

| Category | Endpoints | Methods |
|----------|----------|---------|
| **Cluster lifecycle** | `/api/2.0/clusters/*` | `get_clusters`, `start_cluster`, `stop_cluster`, `restart_cluster` |
| **Job management** | `/api/2.0/jobs/*` | `get_jobs`, `run_job`, `get_job_run` |
| **SQL warehouse** | `/api/2.0/sql/statements` | `execute_sql` |
| **Workspace** | `/api/2.0/workspace/*` | `upload_file`, `list_workspace` |

Global singleton (`get_databricks_connection`) ensures one connection instance, initializing from app config when no explicit config is passed.

## Unity Catalog

Three-level namespace for centralized governance:

```
Metastore (org-level)
+-- Catalog ("prod", "dev")
|   +-- Schema ("bronze", "silver", "gold", "features", "models")
|   |   +-- Table / View / Function
```

Provisioned via Terraform (`databricks_catalog`, `databricks_schema` resources). Capabilities: fine-grained GRANT/REVOKE permissions (catalog/schema/table/column level), automatic data lineage tracking, audit logging, Delta Sharing protocol for cross-org data sharing.

## Auto Loader & Streaming

Incremental file ingestion with automatic schema inference:

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", "/checkpoints/schema")
    .option("cloudFiles.inferColumnTypes", "true")
    .load("/data/landing/")
)
(df.writeStream
    .format("delta")
    .option("checkpointLocation", "/checkpoints/bronze")
    .outputMode("append")
    .table("bronze.raw_data")
)
```

Key features: schema inference on first batch, automatic evolution on new columns, exactly-once via checkpointing, scales to millions of files.

### Delta Live Tables (DLT)

```python
@dlt.table(comment="Cleaned shipments")
@dlt.expect("valid_weight", "weight_kg > 0")              # Warn, keep row
@dlt.expect_or_drop("valid_status", "status IN ('PENDING','DELIVERED')")  # Drop row
@dlt.expect_or_fail("has_id", "shipment_id IS NOT NULL")   # Fail pipeline
def silver_shipments():
    return dlt.read("bronze_shipments").select(...)
```

## Infrastructure as Code

Uses [[Terraform for Data Infrastructure]] to provision the full platform.

**S3 data lake:** AES-256 server-side encryption, versioning enabled, lifecycle tiering (Standard -> Standard-IA at 30d -> Glacier at 90d -> Deep Archive at 365d).

**IAM:** Least-privilege role scoped to data lake bucket (`s3:GetObject`, `PutObject`, `DeleteObject`, `ListBucket`).

**Cluster config:**

```hcl
resource "databricks_cluster" "data_cluster" {
  spark_version           = "13.3.x-scala2.12"
  node_type_id            = "i3.xlarge"
  num_workers             = var.environment == "prod" ? 4 : 2
  autotermination_minutes = var.environment == "prod" ? 0 : 30
  data_security_mode      = "SINGLE_USER"
  spark_conf = {
    "spark.databricks.delta.merge.enableLowShuffle" = "true"
    "spark.sql.adaptive.enabled"                    = "true"
    "spark.sql.adaptive.coalescePartitions.enabled" = "true"
  }
}
```

**Workflow orchestration:** Bronze-to-silver and silver-to-gold `databricks_job` resources with hourly cron schedules and dependency chains.

## Security & Compliance

### Data Classification (4 Levels)

| Level | Examples | Encryption | Audit |
|-------|---------|-----------|-------|
| **Public** | Marketing, API docs | None | No |
| **Internal** | Operational metrics | Transit (TLS) | No |
| **Confidential** | Customer data, financials | At rest + transit | Yes |
| **Restricted** | PII, credentials, payments | At rest + transit | Yes |

**Encryption:** AES-256 at rest (S3 SSE), TLS 1.3 in transit, AWS KMS for key management.

**Access control:** Unity Catalog RBAC (GRANT/REVOKE), least-privilege IAM, `SINGLE_USER` cluster isolation, Databricks secret scopes.

**Audit:** Comprehensive logging for Confidential/Restricted data, real-time monitoring, alerting. Retention: Internal 3y, Confidential 7y, Restricted per regulation.

**Compliance frameworks:** GDPR (right to erasure, portability, breach notification), CCPA (consumer rights, opt-out, deletion), SOX (financial integrity, audit trails, change management).

## Performance

**Photon acceleration** -- C++ vectorized engine, enable via `runtime_engine = "PHOTON"`. Best for SQL-heavy workloads with large scans.

**Z-ordering** -- co-locates related data for predicate pushdown: `OPTIMIZE gold.metrics ZORDER BY (date, region)`.

**Partition pruning** -- Delta auto-skips irrelevant partitions. Combine with `ANALYZE TABLE` for fresh column statistics.

**Adaptive Query Execution (AQE)** -- dynamically adjusts shuffle partitions, converts sort-merge joins to broadcast joins, handles skew. Enabled via `spark.sql.adaptive.enabled = true`.

**Low-shuffle merge** -- `spark.databricks.delta.merge.enableLowShuffle = true` reduces shuffle during MERGE by leveraging file-level statistics.

## Validation Utilities

`DataValidator` (rule-based) and `SchemaValidator` (JSON-schema-style) in `src/utils/common/validation.py`. Built-in validators: `validate_email`, `validate_phone`, `validate_date`, `validate_positive_number`, `validate_not_empty`, `validate_json`. Schema validation supports type checking, min/max constraints, pattern matching, enum values, and nested objects. See [[Data Validation & Quality Frameworks]].

## Structured Logging

The project provides a `StructuredLogger` emitting JSON-formatted log entries and a `@log_performance` decorator for timing:

```python
@log_performance(get_logger(__name__))
def process_raw_data(self, data, source):
    # Automatically logs start time, completion, and duration
    ...
```

Log format: `{"timestamp": "...", "level": "INFO", "logger": "...", "message": "...", "module": "...", "function": "...", "line": 42}`. Supports console and file handlers with configurable log levels.

## Databricks vs Snowflake

| Aspect | Databricks | Snowflake |
|--------|-----------|-----------|
| Engine | Apache Spark | Proprietary MPP |
| Storage | Delta Lake (open Parquet) | Proprietary micro-partitions |
| Streaming | Native Structured Streaming | Snowpipe (micro-batch) |
| ML | MLflow, Feature Store | Cortex AI, Snowpark |
| Governance | Unity Catalog | Horizon |
| Open formats | Yes (Delta, Parquet, Iceberg) | Proprietary (Iceberg support) |

---

**Related:** [[Terraform for Data Infrastructure]] | [[Delta Lake Operations & Patterns]] | [[Data Validation & Quality Frameworks]] | Apache Spark Fundamentals | MLflow & Experiment Tracking
