# Data Validation & Quality Frameworks

## Validation Landscape

Comparison of validation frameworks used across the data engineering stack:

| Framework | Scope | Validation Style | Best For | Version (project) |
|-----------|-------|-----------------|----------|-------------------|
| **Great Expectations** | DataFrame / batch data | Declarative expectations | Post-extraction data profiling, pipeline checkpoints | `0.18.8` |
| **Pydantic** | Python objects / config | Type-enforced models | Pipeline config parsing, API payloads, settings | `2.5.2` |
| **Cerberus** | Dict / document validation | Schema rules dict | Lightweight config validation, nested docs | `1.3.5` |
| **pandera** | DataFrame schemas | Pythonic column contracts | Pandas/PySpark schema enforcement inline | not in project |
| **dbt tests** | SQL warehouse tables | YAML declarations + SQL | Warehouse-layer data quality | see [[dbt Testing & Data Quality]] |

Key distinction: Great Expectations and pandera validate **data in motion** (DataFrames mid-pipeline), while dbt tests validate **data at rest** (tables already loaded into the warehouse).

---

## Great Expectations

### Data Context and Setup

The Data Context is the entry point — it manages expectation suites, data sources, and checkpoints:

```python
import great_expectations as gx

context = gx.get_context()

# Register a pandas datasource
datasource = context.sources.add_pandas("pandas_source")
data_asset = datasource.add_dataframe_asset(name="customers_extract")
```

### Core Expectations

Expectations map directly to the validation rules in `ValidationUtils`:

```python
# Null checks — mirrors ValidationUtils.validate_data_quality 'not_null_columns'
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_not_be_null("email")

# Range validation — mirrors 'value_ranges' rules
validator.expect_column_values_to_be_between(
    "order_amount", min_value=0, max_value=1_000_000
)

# Type expectations
validator.expect_column_values_to_be_of_type("created_at", "datetime64[ns]")

# Uniqueness — mirrors ValidationUtils.check_duplicates
validator.expect_column_values_to_be_unique("transaction_id")

# Set membership
validator.expect_column_values_to_be_in_set(
    "status", ["ACTIVE", "INACTIVE", "PENDING"]
)

# Regex patterns
validator.expect_column_values_to_match_regex(
    "email", r"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$"
)

# Row count bounds (volume anomaly detection)
validator.expect_table_row_count_to_be_between(min_value=1000, max_value=500_000)
```

### Checkpoints

Checkpoints bundle a validator + actions (notify, store results, update Data Docs):

```python
checkpoint = context.add_or_update_checkpoint(
    name="post_extraction_checkpoint",
    validations=[{
        "batch_request": batch_request,
        "expectation_suite_name": "customers_suite",
    }],
    action_list=[
        {"name": "store_validation_result", "action": {"class_name": "StoreValidationResultAction"}},
        {"name": "update_data_docs", "action": {"class_name": "UpdateDataDocsAction"}},
    ],
)

result = checkpoint.run()
if not result.success:
    raise RuntimeError("Data quality checkpoint failed")
```

### Data Docs

Great Expectations auto-generates an HTML quality report site. Host it on GCS for team visibility:

```python
context.build_data_docs()
# Output: great_expectations/uncommitted/data_docs/local_site/index.html
```

---

## Pydantic for Pipeline Config Validation

Pydantic v2 (`2.5.2`) enforces types at parse time — ideal for pipeline configuration objects.

### Model Definitions

```python
from pydantic import BaseModel, Field, field_validator
from typing import Dict, Any, Optional, List, Literal

class SourceConfig(BaseModel):
    """Mirrors the source_config dict used by DataExtractor"""
    type: Literal["bigquery", "cloudsql", "gcs", "local_file", "api"]
    query: Optional[str] = None
    bucket: Optional[str] = None
    file_pattern: Optional[str] = None
    file_path: Optional[str] = None
    url: Optional[str] = None
    csv_options: Dict[str, Any] = Field(default_factory=dict)

    @field_validator("query")
    @classmethod
    def query_required_for_sql_sources(cls, v, info):
        if info.data.get("type") in ("bigquery", "cloudsql") and not v:
            raise ValueError("query is required for bigquery/cloudsql sources")
        return v

class DestinationConfig(BaseModel):
    type: Literal["bigquery", "cloudsql", "gcs", "local_file", "console"]
    dataset: Optional[str] = None
    table: Optional[str] = None
    write_disposition: str = "WRITE_APPEND"
    partition_field: Optional[str] = None
    clustering_fields: List[str] = Field(default_factory=list)

class TransformationConfig(BaseModel):
    type: str
    conditions: List[Dict[str, Any]] = Field(default_factory=list)
    columns: List[str] = Field(default_factory=list)
    rules: Dict[str, Any] = Field(default_factory=dict)

class PipelineConfig(BaseModel):
    """Top-level pipeline configuration"""
    source: SourceConfig
    destination: DestinationConfig
    transformations: List[TransformationConfig] = Field(default_factory=list)
    job_name: str
    schedule: Optional[str] = None
```

### Settings Management

```python
from pydantic_settings import BaseSettings

class PipelineSettings(BaseSettings):
    project_id: str
    gcs_bucket: str
    google_application_credentials: str
    log_level: str = "INFO"
    max_retry_attempts: int = 3

    model_config = {"env_prefix": "PIPELINE_"}
```

Parse with `PipelineSettings()` — values come from environment variables prefixed with `PIPELINE_`.

---

## Schema Validation Patterns

### Column Presence Validation

From `ValidationUtils.validate_dataframe_schema` in `framework/utils.py`:

```python
# Project pattern: fail-fast on missing columns
ValidationUtils.validate_dataframe_schema(df, expected_columns=['customer_id', 'email', 'created_at'])
# Raises ValueError("Missing required columns: {'email'}") if column absent
```

### Type Checking at Transform Boundaries

From `DataTransformer._convert_types` in `framework/transform.py`:

```python
# Type conversion with coercion — non-castable values become NaN/NaT
transformations = [
    {
        "type": "convert_types",
        "conversions": {
            "order_amount": "float",
            "customer_id": "int",
            "created_at": "datetime",
            "is_active": "boolean"
        }
    }
]
```

Supported target types: `int` (nullable Int64), `float`, `string`, `datetime`, `boolean`.

### Config Validation at Extraction Boundary

From `BaseExtractor._validate_config` in `framework/extract.py`:

```python
# Each extractor declares its required fields and validates on init
class BigQueryExtractor(BaseExtractor):
    def __init__(self, config):
        super().__init__(config)
        self._validate_config(['query'])  # Fails immediately if missing

class GCSExtractor(BaseExtractor):
    def __init__(self, config):
        super().__init__(config)
        self._validate_config(['bucket', 'file_pattern'])
```

This pattern repeats in `framework/load.py` — every loader validates its own config fields before any data flows.

---

## Data Quality Rules

### Null Checks

```python
# ValidationUtils pattern from framework/utils.py
rules = {
    "not_null_columns": ["customer_id", "email", "order_date"]
}
ValidationUtils.validate_data_quality(df, rules)
# Raises ValueError("Found 12 null values in column 'email'")
```

### Range Validation

```python
rules = {
    "value_ranges": {
        "order_amount": (0, 1_000_000),
        "discount_pct": (0.0, 1.0),
        "quantity": (1, 10_000)
    }
}
ValidationUtils.validate_data_quality(df, rules)
```

### Referential Integrity

Check that foreign keys exist in parent table (complement to dbt `relationships` test):

```python
def validate_referential_integrity(child_df, parent_df, child_key, parent_key):
    """Validate FK references resolve to parent table"""
    orphan_keys = set(child_df[child_key]) - set(parent_df[parent_key])
    if orphan_keys:
        raise ValueError(
            f"Referential integrity violation: {len(orphan_keys)} orphan keys in '{child_key}'"
        )
```

### Freshness Checks

Ensure data is not stale before processing — the Python equivalent of [[dbt Testing & Data Quality#Source Freshness]]:

```python
from datetime import datetime, timedelta

def check_freshness(df, timestamp_col, max_age_hours=24):
    """Fail if newest record is older than threshold"""
    latest = pd.to_datetime(df[timestamp_col]).max()
    age = datetime.utcnow() - latest
    if age > timedelta(hours=max_age_hours):
        raise ValueError(f"Data is stale: latest record is {age.total_seconds()/3600:.1f}h old")
```

### Volume Anomaly Detection

Flag unexpected row count deviations between pipeline runs:

```python
def check_volume_anomaly(current_count, historical_avg, tolerance=0.5):
    """Alert if row count deviates beyond tolerance from historical average"""
    lower_bound = historical_avg * (1 - tolerance)
    upper_bound = historical_avg * (1 + tolerance)
    if not (lower_bound <= current_count <= upper_bound):
        raise ValueError(
            f"Volume anomaly: {current_count} rows (expected {lower_bound:.0f}-{upper_bound:.0f})"
        )
```

---

## Duplicate Detection

### Exact Match Dedup

From `ValidationUtils.check_duplicates` and `DataTransformer._drop_duplicates`:

```python
# Detection — count duplicates on composite key
dup_count = ValidationUtils.check_duplicates(df, columns=["customer_id", "order_date"])
# Returns: 42

# Removal — keep first occurrence
deduped = DataTransformer._drop_duplicates(df, config={
    "columns": ["customer_id", "order_date"],
    "keep": "first"   # Options: 'first', 'last', False (drop all)
})
```

### Composite Key Validation

Verify that a set of columns forms a valid composite key (zero duplicates expected):

```python
def validate_composite_key(df, key_columns):
    """Assert that key_columns form a unique composite key"""
    dup_count = df.duplicated(subset=key_columns).sum()
    if dup_count > 0:
        dupes = df[df.duplicated(subset=key_columns, keep=False)]
        raise ValueError(
            f"Composite key {key_columns} has {dup_count} duplicate rows. "
            f"Sample:\n{dupes.head(5)}"
        )
```

### Fuzzy Matching

For entity resolution and near-duplicate detection (not in the project, but a common extension):

```python
from thefuzz import fuzz

def find_fuzzy_duplicates(df, column, threshold=85):
    """Identify near-duplicate values using Levenshtein similarity"""
    values = df[column].dropna().unique()
    pairs = []
    for i, v1 in enumerate(values):
        for v2 in values[i+1:]:
            score = fuzz.ratio(str(v1), str(v2))
            if score >= threshold:
                pairs.append((v1, v2, score))
    return pairs
```

---

## Pipeline Integration

### Where to Validate

The `gcp_datamigration` framework validates at four boundaries:

```
Extract ──> [1] ──> Transform ──> [2] ──> [3] ──> Load ──> [4]
```

| Checkpoint | What to Validate | Strategy | Project Pattern |
|-----------|-----------------|----------|-----------------|
| **[1] Post-extract** | Schema presence, row count > 0, freshness | Fail-fast | `BaseExtractor._validate_config`, schema check |
| **[2] Pre-transform** | Column types, null counts, value distributions | Fail-fast | `ValidationUtils.validate_dataframe_schema` |
| **[3] Post-transform** | Business rules, dedup verification, range checks | Configurable | `DataTransformer._validate_data_quality` |
| **[4] Pre-load** | Final schema match to destination, row count reconciliation | Fail-fast | `BaseLoader._validate_config` |

### Fail-Fast vs Log-and-Continue

```python
class ValidationStrategy:
    FAIL_FAST = "fail_fast"        # Raise exception, halt pipeline
    LOG_AND_CONTINUE = "log_warn"  # Log warning, continue processing

def validate_with_strategy(df, rules, strategy=ValidationStrategy.FAIL_FAST):
    """Apply validation rules with configurable failure handling"""
    issues = []

    for col in rules.get("not_null_columns", []):
        null_count = df[col].isnull().sum()
        if null_count > 0:
            msg = f"Found {null_count} nulls in '{col}'"
            if strategy == ValidationStrategy.FAIL_FAST:
                raise ValueError(msg)
            issues.append(msg)

    if issues:
        logging.warning(f"Validation issues (non-blocking): {issues}")

    return df, issues
```

Use **fail-fast** for primary keys, foreign keys, and critical business columns. Use **log-and-continue** for soft quality checks where partial data is acceptable (e.g., optional fields, non-critical metrics).

---

## Metrics and Reporting

### Quality Score Tracking

From `MetricsCollector` in `framework/utils.py` — structured JSON metrics at each pipeline stage:

```python
metrics = MetricsCollector()
metrics.record_job_start("customer_pipeline")
metrics.record_extraction_complete(records_count=15420)
metrics.record_transformation_complete(input_records=15420, output_records=15105)
metrics.record_load_complete({
    "success": True, "rows_loaded": 15105, "destination": "project.dataset.customers"
})
metrics.record_job_complete("customer_pipeline", duration=42.3, records_processed=15105)
```

Each call emits structured JSON to the logger, which integrates with [[Google Cloud Platform|GCP Cloud Logging]] and can feed dashboards.

### Quality Score Computation

```python
def compute_quality_score(df, rules):
    """Compute a 0-100 quality score across multiple dimensions"""
    checks = []

    # Completeness: % of non-null values in required columns
    for col in rules.get("not_null_columns", []):
        completeness = 1 - (df[col].isnull().sum() / len(df))
        checks.append(completeness)

    # Validity: % of values within expected ranges
    for col, (lo, hi) in rules.get("value_ranges", {}).items():
        in_range = ((df[col] >= lo) & (df[col] <= hi)).mean()
        checks.append(in_range)

    # Uniqueness: % of unique values in key columns
    for col in rules.get("unique_columns", []):
        uniqueness = 1 - (df[col].duplicated().sum() / len(df))
        checks.append(uniqueness)

    return round((sum(checks) / len(checks)) * 100, 2) if checks else 100.0
```

### Alerting Thresholds

Define tiered alerting based on quality score trends:

| Score Range | Severity | Action |
|------------|----------|--------|
| 95-100 | OK | No action |
| 85-94 | Warning | Slack notification, log to monitoring |
| 70-84 | Critical | PagerDuty alert, pipeline paused for review |
| < 70 | Failure | Pipeline halted, incident created |

```python
def evaluate_quality_alert(score, job_name):
    """Evaluate quality score against alerting thresholds"""
    if score >= 95:
        return "ok"
    elif score >= 85:
        logging.warning(f"[{job_name}] Quality score {score} — below warning threshold")
        # send_slack_notification(...)
        return "warning"
    elif score >= 70:
        logging.error(f"[{job_name}] Quality score {score} — critical threshold breached")
        # send_pagerduty_alert(...)
        return "critical"
    else:
        raise RuntimeError(f"[{job_name}] Quality score {score} — pipeline halted")
```

---

## Data Profiling

Data profiling is the systematic analysis of a dataset's structure, content, and quality before and during pipeline execution. It answers the question: "What does the data actually look like?" rather than "Does it pass predefined rules?"

### What to Profile

| Dimension | What It Measures | Example Metrics |
|-----------|-----------------|-----------------|
| **Completeness** | Presence of values | Null rate per column, row fill rate |
| **Uniqueness** | Distinct value density | Cardinality, duplicate rate, unique percentage |
| **Distribution** | Shape of values | Mean, median, standard deviation, histogram, skewness |
| **Outliers** | Extreme or anomalous values | Values beyond 3 standard deviations, IQR fence violations |
| **Type consistency** | Data type adherence | Mixed types in string columns, date format variations |
| **Pattern conformity** | Format adherence | Email regex match rate, phone number format compliance |
| **Referential integrity** | Cross-table consistency | Orphan foreign keys, missing parent records |

### Profiling Tools

| Tool | Approach | Output | Best For |
|------|----------|--------|----------|
| **ydata-profiling** (formerly pandas-profiling) | DataFrame analysis, HTML report | Interactive HTML with correlations, histograms | Exploratory profiling, ad-hoc analysis |
| **whylogs** | Lightweight statistical logging | Mergeable profile objects (`.bin` files) | Production pipelines, drift detection |
| **Great Expectations Profiler** | Auto-generates expectation suites from data | Expectation suite JSON | Bootstrap validation rules from data |
| **dbt profiling packages** (e.g., `dbt_profiler`) | SQL-based column stats | dbt models with profiling metrics | Warehouse-native profiling |

### Automated Profiling in CI

Integrate profiling into CI pipelines to catch schema drift, distribution shifts, and quality regressions before data reaches production.

**Pattern: whylogs profile comparison in CI**

```python
import whylogs as why
from whylogs.core.constraints.factories import (
    null_percentage_below,
    greater_than_number,
    column_is_probably_unique,
)
from whylogs.core.constraints import ConstraintsBuilder

def profile_and_validate(df, dataset_name: str):
    """Profile a DataFrame and validate against constraints."""
    # Generate a statistical profile
    profile = why.log(df).profile()
    profile_view = profile.view()

    # Define constraints (equivalent to expectations)
    builder = ConstraintsBuilder(dataset_profile_view=profile_view)
    builder.add_constraint(null_percentage_below(column_name="customer_id", number=0.01))
    builder.add_constraint(greater_than_number(column_name="order_amount", number=0.0))
    builder.add_constraint(column_is_probably_unique(column_name="transaction_id"))
    constraints = builder.build()

    report = constraints.generate_constraints_report()
    if report.failures:
        raise ValueError(
            f"Profiling constraints failed for {dataset_name}: "
            f"{[f.name for f in report.failures]}"
        )
    return profile_view
```

**Pattern: ydata-profiling HTML report generation**

```python
from ydata_profiling import ProfileReport

def generate_profile_report(df, title: str, output_path: str):
    """Generate an HTML profiling report for review."""
    profile = ProfileReport(
        df,
        title=title,
        explorative=True,
        correlations={"cramers": {"calculate": True}},
        missing_diagrams={"bar": True, "matrix": True},
    )
    profile.to_file(output_path)
    return output_path
```

**Pattern: drift detection between profiling runs**

```python
import whylogs as why

def detect_drift(reference_profile_path: str, current_df):
    """Compare current data against a reference profile for drift."""
    ref_profile = why.read(reference_profile_path).view()
    current_profile = why.log(current_df).profile().view()

    # Compare column-level statistics
    ref_summary = ref_profile.to_pandas()
    curr_summary = current_profile.to_pandas()

    drift_columns = []
    for col in ref_summary.index:
        if col in curr_summary.index:
            ref_mean = ref_summary.loc[col].get("distribution/mean", 0)
            curr_mean = curr_summary.loc[col].get("distribution/mean", 0)
            if ref_mean and abs(curr_mean - ref_mean) / abs(ref_mean) > 0.25:
                drift_columns.append(col)

    if drift_columns:
        raise ValueError(f"Distribution drift detected in columns: {drift_columns}")
```

### CI Integration Checklist

- [ ] Generate profiles on every staging load (before promotion to production)
- [ ] Store reference profiles as versioned artefacts (e.g., in GCS or S3)
- [ ] Compare current run profiles against reference to detect drift
- [ ] Fail the pipeline if null rates, cardinality, or distributions exceed thresholds
- [ ] Publish HTML profiling reports to a shared location for analyst review
- [ ] Use whylogs for lightweight production profiling; ydata-profiling for deep exploratory analysis

---

## See Also

- [[dbt Testing & Data Quality]] — warehouse-layer testing with dbt generic/singular tests
- [[Python Testing with pytest]] — unit and integration testing patterns for pipeline code
- [[Apache Airflow]] — orchestrating validation checkpoints within DAGs
