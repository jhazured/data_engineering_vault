# Data Ingestion Patterns

## Ingestion Methods

| Method | Best For | Tools |
|--------|----------|-------|
| **Managed ELT** | SaaS sources (CRM, ERP, databases) | Fivetran, Stitch, Airbyte |
| **Bulk Load** | Large file-based imports (CSV, Parquet) | Snowflake COPY INTO, `write_pandas` |
| **Streaming** | Real-time events, IoT telemetry | Kafka, Snowpipe, Kinesis |
| **API Ingestion** | External REST APIs (weather, traffic) | Python scripts, orchestrators |

## Landing Zone Conventions

All raw data lands in a transient staging tier with standard audit columns:

| Column | Purpose | Source |
|--------|---------|--------|
| `created_at` | When the record was created in the source system | Source system |
| `updated_at` | When the record was last modified in the source | Source system |
| `is_deleted` | Soft delete flag for logical deletions | Source system |
| `_loaded_at` | When the ingestion pipeline wrote this row | Ingestion tool |

The `_loaded_at` column is critical — downstream incremental models use it as the high-water mark:

```sql
WHERE _loaded_at > (SELECT MAX(_ingested_at) FROM {{ this }})
```

## Fivetran Integration

### How It Works

Fivetran manages the Extract and Load — it syncs source data into your warehouse landing zone on a schedule.

```
Source System → Fivetran → T1_TRANSIENT_STAGING.LOGISTICS.CUSTOMERS
                                (raw data, source units, _loaded_at added)
```

### Setup Pattern

1. Create a dedicated service account with key-pair auth:
```sql
CREATE USER FIVETRAN_SVC_USER
  LOGIN_NAME = 'FIVETRAN_SVC_USER'
  DEFAULT_ROLE = FIVETRAN_ROLE
  DEFAULT_WAREHOUSE = IN_PROD_CENTRAL_WH
  RSA_PUBLIC_KEY = '...';
```

2. Grant minimal access — only T1 write permissions:
```sql
GRANT USAGE ON DATABASE PROD_T1_TRANSIENT_STAGING TO ROLE FIVETRAN_ROLE;
GRANT ALL ON SCHEMA PROD_T1_TRANSIENT_STAGING.LOGISTICS TO ROLE FIVETRAN_ROLE;
GRANT USAGE ON WAREHOUSE IN_PROD_CENTRAL_WH TO ROLE FIVETRAN_ROLE;
```

3. Use a dedicated `IN_` (ingestion) warehouse to isolate costs.

### dbt Source Declaration

```yaml
sources:
  - name: RAW
    database: "{{ env_var('SF_ENV', 'DEV') }}_T1_TRANSIENT_STAGING"
    schema: LOGISTICS
    tables:
      - name: CUSTOMERS
        columns:
          - name: CUSTOMER_ID
            tests: [unique, not_null]
          - name: _LOADED_AT
            tests: [not_null]
```

## Python Bulk Loading

For development/testing or one-off loads, use the Snowflake Python connector:

```python
from snowflake.connector.pandas_tools import write_pandas

success, _, nrows, _ = write_pandas(
    conn,
    df,
    table_name='CUSTOMERS',
    database='DEV_T1_TRANSIENT_STAGING',
    schema='LOGISTICS',
    chunk_size=10000,
    auto_create_table=True,
)
```

### Sample Data Generation

Use Faker for realistic test data:

```python
from faker import Faker
import pandas as pd

fake = Faker('en_AU')
Faker.seed(42)

customers = pd.DataFrame({
    'customer_id': [f'CUST_{str(i).zfill(6)}' for i in range(1000)],
    'customer_name': [fake.company() for _ in range(1000)],
    'contact_email': [fake.email() for _ in range(1000)],
    'created_at': [fake.date_time_between(start_date='-3y') for _ in range(1000)],
    'updated_at': [fake.date_time_between(start_date='-1y') for _ in range(1000)],
    'is_deleted': False,
    '_loaded_at': datetime.now(),
})
```

## Unit Handling at Ingestion Boundaries

Sources often use different units than your analytics layer. The key decision: **where to convert?**

### Pattern: Preserve Source Units in T1/T2, Convert in T3

| Layer | Columns | Units |
|-------|---------|-------|
| T1 (raw) | `weight_lbs`, `distance_miles`, `fuel_efficiency_mpg` | Source (imperial) |
| T2 (staging) | `weight_lbs`, `distance_miles`, `fuel_efficiency_mpg` | Source (imperial) — pass-through |
| T3 (integration) | `weight_kg`, `distance_km`, `fuel_efficiency_l_100km` | Target (metric) |

**Why:** T2 is a clean pass-through — it should match the source exactly for audit/reconciliation. Business logic (including unit conversions) belongs in T3 where it's applied consistently.

## Soft Delete Handling

Sources often use logical deletes rather than physical deletes:

```sql
-- Staging model: normalise the flag
SELECT
  customer_id,
  COALESCE(is_deleted, FALSE) AS is_deleted,
  CURRENT_TIMESTAMP() AS _ingested_at
FROM {{ source('RAW', 'CUSTOMERS') }}
```

Downstream models can then filter: `WHERE NOT is_deleted`

## Source Freshness Monitoring

dbt can check when sources were last updated:

```yaml
sources:
  - name: RAW
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _loaded_at
    tables:
      - name: CUSTOMERS
```

Run with: `dbt source freshness`

## Staging → Final Table Pattern (Atomic Loading)

For pipelines that compute embeddings or heavy transforms during load, use a two-table pattern:

```python
try:
    # 1. Insert raw chunks into staging (no vectors/transforms)
    run_sql("INSERT INTO staging (...) VALUES (%s, ...)", chunks)

    # 2. Transform and move to final table
    run_sql("""
        INSERT INTO final_table
        SELECT *, AI_EMBED('model', chunk_text) AS vector
        FROM staging WHERE source_name = %s
    """, (source_name,))
finally:
    # 3. ALWAYS clean up staging — even on error
    run_sql("DELETE FROM staging WHERE source_name = %s", (source_name,))
```

**Why:** If embedding/transform fails mid-batch, staging can be cleaned up without corrupting the final table. The `try/finally` guarantees cleanup.

### Incremental vs Full Reload

```python
# Incremental: skip already-loaded files
existing = {row['SOURCE_NAME'] for row in run_sql("SELECT DISTINCT source_name FROM final")}
files_to_load = [f for f in all_files if f.name not in existing]

# Full reload: delete existing, then re-load (require --force flag for safety)
if mode == 'full_reload' and force:
    run_sql("DELETE FROM final WHERE source_name = %s", (name,))
```

### Batch Resilience

Failed files are logged but don't stop the pipeline:

```python
failed = []
for file in files:
    try:
        load(file)
    except Exception as e:
        logger.error(f"Failed: {file.name}: {e}")
        failed.append(file.name)
        continue  # Keep going
```

## Ingestion Anti-Patterns

| Anti-Pattern | Why It's Bad | Better Approach |
|-------------|-------------|-----------------|
| Transforming during ingestion | Couples extract/load with business logic | Load raw, transform in dbt |
| No `_loaded_at` column | Can't do incremental downstream | Always add a pipeline timestamp |
| Hardcoding credentials | Security risk | Use env vars or key-pair auth |
| Single warehouse for everything | No cost attribution | Separate IN/TRN/OUT warehouses |
| No soft delete flag | Can't track deletions | Include `is_deleted` from source |
