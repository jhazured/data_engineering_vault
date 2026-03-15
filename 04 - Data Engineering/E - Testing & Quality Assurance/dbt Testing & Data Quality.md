# dbt Testing & Data Quality

## Test Types

### Generic Tests (schema.yml)

Declarative tests applied in YAML — run with `dbt test`:

```yaml
models:
  - name: tbl_stg_customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: customer_type
        tests:
          - accepted_values:
              arguments:
                values: ['ENTERPRISE', 'STANDARD', 'BASIC']
      - name: vehicle_id
        tests:
          - relationships:
              to: ref('tbl_dim_vehicle')
              field: vehicle_id
```

| Test | What It Checks |
|------|---------------|
| `unique` | No duplicate values |
| `not_null` | No NULL values |
| `accepted_values` | Column values are in an allowed set |
| `relationships` | Referential integrity (FK exists in target) |

### Singular Tests (custom SQL)

Custom business rule tests in `tests/business_rules/`:

```sql
-- tests/business_rules/test_route_efficiency_bounds.sql
-- Fails if any efficiency score is outside 0-100

SELECT
  shipment_id,
  route_efficiency_score
FROM {{ ref('tbl_fact_shipments') }}
WHERE route_efficiency_score < 0
   OR route_efficiency_score > 100
```

If this query returns any rows, the test **fails**.

### Seasonal Demand Pattern Test

Detect unusual monthly demand spikes that may indicate data issues:

```sql
-- Fails if any month's shipment count deviates >50% from the 2-year average

WITH monthly_counts AS (
  SELECT
    DATE_TRUNC('month', shipment_date) AS month,
    COUNT(*) AS shipment_count
  FROM {{ ref('tbl_fact_shipments') }}
  WHERE shipment_date >= DATEADD(year, -2, CURRENT_DATE())
  GROUP BY 1
),
stats AS (
  SELECT AVG(shipment_count) AS avg_count FROM monthly_counts
)

SELECT month, shipment_count, avg_count
FROM monthly_counts
CROSS JOIN stats
WHERE shipment_count > avg_count * 1.5
   OR shipment_count < avg_count * 0.5
```

## store_failures

Persist test failures to a table for analysis:

```yaml
tests:
  my_project:
    +store_failures: "{{ var('test_store_failures', false) }}"
    +severity: warn    # warn (pipeline continues) vs error (pipeline stops)
```

Enable per environment:
```yaml
vars:
  dev:
    test_store_failures: false    # Dev: fast feedback
  staging:
    test_store_failures: true     # Staging: persist for analysis
```

## sqlfluff Linting

SQL linter with dbt awareness — catches style issues before they reach production:

```bash
# Lint all models using dbt templater
cd dbt && sqlfluff lint models/ \
  --config .sqlfluff \
  --templater dbt \
  --ignore templating \
  --ignore parsing
```

**`--ignore templating --ignore parsing`**: Suppress Jinja-related false positives while still catching SQL style issues.

### .sqlfluff Configuration

```ini
[sqlfluff]
templater = dbt
dialect = snowflake

[sqlfluff:rules:aliasing.table]
aliasing = explicit

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = upper
```

## Source Freshness

Monitor when sources were last updated:

```yaml
sources:
  - name: RAW
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _loaded_at
    tables:
      - name: CUSTOMERS
      - name: SHIPMENTS
```

```bash
dbt source freshness    # Check all sources
```

## Test Execution Patterns

```bash
# All tests
dbt test

# Only critical tests (production)
dbt test --select tag:critical

# Only data quality tests (nightly schedule)
dbt test --select tag:data_quality

# Tests for a specific model
dbt test --select tbl_fact_shipments

# Store failures for analysis
dbt test --store-failures
```

## dbt-expectations Package

Extended test library for data quality (installed via `packages.yml`):

```yaml
packages:
  - package: calogica/dbt_expectations
    version: "0.10.10"
```

Provides tests like:
- `expect_column_values_to_be_between`
- `expect_column_values_to_match_regex`
- `expect_table_row_count_to_be_between`
- `expect_column_pair_values_A_to_be_greater_than_B`

## Great Expectations Integration

For CI/CD pipelines, run Great Expectations alongside dbt:

```yaml
# .gitlab-ci.yml
dbt:data-quality-monitor:
  script:
    - pip install pandas great-expectations
    - dbt test --project-dir dbt --target prod --select tag:data_quality
    - python scripts/generate_quality_report.py
  artifacts:
    paths: [reports/quality_report.html]
    expire_in: 30 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

## Quality Report Generation

Python script that queries dbt test results and generates an HTML report:

```python
# scripts/generate_quality_report.py
import json
from pathlib import Path

run_results = json.loads(Path('dbt/target/run_results.json').read_text())

failed_tests = [
    r for r in run_results['results']
    if r['status'] in ('fail', 'error')
]

# Generate HTML summary with pass/fail counts, failed test details
```

## Test Severity Levels

| Severity | Behaviour | When to Use |
|----------|-----------|-------------|
| `error` | Pipeline fails | Critical data integrity (unique keys, not null on PKs) |
| `warn` | Pipeline continues, warning logged | Data quality monitoring, non-blocking checks |

```yaml
columns:
  - name: customer_id
    tests:
      - unique:
          severity: error      # Must pass
      - not_null:
          severity: error      # Must pass
  - name: satisfaction_score
    tests:
      - not_null:
          severity: warn       # Nice to have
```
