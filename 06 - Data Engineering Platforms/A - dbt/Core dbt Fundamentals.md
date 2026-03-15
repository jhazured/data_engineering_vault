# Core dbt Fundamentals

## Introduction

dbt (Data Build Tool) handles the **T** in ELT — transforming data already loaded into your warehouse using SQL and software engineering best practices (version control, testing, documentation, modularity).

## dbt Project Structure

```
my_dbt_project/
├── dbt_project.yml     # Project config: name, profile, model settings, vars
├── packages.yml        # External package dependencies (dbt-utils, dbt-expectations)
├── profiles.yml        # Connection targets (dev/staging/prod) — NOT committed
├── models/             # SQL transformations
│   ├── staging/        # 1:1 with source tables, basic cleaning
│   ├── intermediate/   # Reusable business logic (optional layer)
│   └── marts/          # Final business-ready dimensions & facts
├── macros/             # Reusable Jinja SQL functions
├── snapshots/          # SCD Type 2 change tracking
├── seeds/              # CSV reference data loaded as tables
├── tests/              # Custom singular test SQL files
├── analyses/           # Ad-hoc queries (compiled but not materialized)
└── exposures.yml       # Downstream consumers (dashboards, apps)
```

## dbt_project.yml Configuration

The project file controls materializations, tags, database routing, and environment-specific variables.

### Environment-Specific Variables

```yaml
vars:
  # Global defaults
  start_date: '2023-01-01'
  data_freshness_hours: 6

  # Per-target overrides
  dev:
    materialized: 'view'            # Views in dev to save storage
    test_store_failures: false
    max_partition_days: 7
  prod:
    test_store_failures: false
    refresh_incremental: true
    max_partition_days: 365
```

Access in SQL with `{{ var('start_date') }}`.

### Model Configuration Hierarchy

```yaml
models:
  my_project:
    +materialized: view                    # Project default
    +persist_docs:
      relation: true
      columns: true

    staging:
      +materialized: incremental
      +schema: "STAGING"
      +tags: ["staging", "load_priority_1"]

    marts:
      dimensions:
        +materialized: incremental
        +tags: ["dimensions", "load_priority_2a"]
      facts:
        +materialized: incremental
        +tags: ["facts", "load_priority_2b"]
```

### Multi-Database Routing

For multi-tier architectures, route models to different databases per environment:

```yaml
models:
  my_project:
    staging:
      +database: "{{ env_var('SF_ENV', 'DEV') | upper }}_T2_PERSISTENT_STAGING"
    marts:
      +database: "{{ env_var('SF_ENV', 'DEV') | upper }}_T3_INTEGRATION"
    presentation:
      +database: "{{ env_var('SF_ENV', 'DEV') | upper }}_T4_PRESENTATION"
```

### generate_schema_name Override

By default dbt prepends `target.schema` to custom schema names. Override to use only the custom schema:

```sql
{% macro generate_schema_name(custom_schema_name, node) %}
    {% if custom_schema_name is none %}
        {{ target.schema }}
    {% else %}
        {{ custom_schema_name | trim }}
    {% endif %}
{% endmacro %}
```

## Models, Sources & Seeds

### Sources

Declare raw tables in `sources.yml` so dbt tracks lineage and freshness:

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

Reference in SQL: `{{ source('RAW', 'CUSTOMERS') }}`

### Seeds

CSV files loaded as tables — useful for reference/lookup data and classification tags:

```yaml
seeds:
  my_project:
    +database: "{{ env_var('SF_ENV', 'DEV') }}_T1_TRANSIENT_STAGING"
    +schema: "LOGISTICS"

    # Override for specific seeds
    data_classification_tags:
      +database: "{{ env_var('SF_ENV', 'DEV') }}_T0_CONTROL"
      +schema: "SECURITY"
```

Run with: `dbt seed --select data_classification_tags`

### Exposures

Declare downstream consumers for lineage visibility:

```yaml
exposures:
  - name: executive_dashboard
    type: dashboard
    maturity: high
    owner:
      name: Analytics Team
      email: team@example.com
    depends_on:
      - ref('vw_consolidated_dashboard')
```

## Materializations

| Type | When to Use | Storage | Rebuild |
|------|------------|---------|---------|
| **view** | Lightweight, always-fresh reads; presentation layer | None | Every query |
| **table** | Small static datasets; reference data | Full | Full rebuild |
| **incremental** | Large fact/dimension tables; append/merge patterns | Full | Only new/changed rows |
| **ephemeral** | Intermediate CTEs; avoid materializing helper queries | None (compiled inline) | N/A |

### Incremental Configuration

```sql
{{ config(
    materialized='incremental',
    unique_key='shipment_id',
    incremental_strategy='merge',
    merge_update_columns=['status', 'actual_delivery_date', 'updated_at'],
    on_schema_change='sync_all_columns',
    tags=['staging', 'incremental']
) }}

SELECT ...
FROM {{ source('RAW', 'SHIPMENTS') }}
{% if is_incremental() %}
  WHERE _loaded_at > (SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }})
{% endif %}
```

Key options:
- **`unique_key`**: Column(s) for merge deduplication
- **`merge_update_columns`**: Restrict which columns update on merge (protects immutable audit fields)
- **`on_schema_change`**: `sync_all_columns` auto-adds new source columns
- **`incremental_strategy`**: `merge` (Snowflake default), `append`, `delete+insert`

## Snapshots (SCD Type 2)

Track slowly changing dimensions with automatic versioning:

```sql
{% snapshot customers_snapshot %}
{{ config(
    target_database=env_var('SF_ENV', 'DEV') ~ '_T2_PERSISTENT_STAGING',
    target_schema='LOGISTICS',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
) }}
SELECT * FROM {{ source('RAW', 'CUSTOMERS') }}
{% endsnapshot %}
```

**Strategies:**
- **timestamp**: Uses `updated_at` column — simple and reliable when available
- **check**: Compares specific columns — use when no reliable timestamp exists

```sql
{{ config(
    strategy='check',
    check_cols=['traffic_level', 'congestion_delay_minutes', 'average_speed_kmh'],
) }}
```

**Added columns:** `dbt_scd_id` (surrogate key), `dbt_valid_from`, `dbt_valid_to`, `dbt_updated_at`

## on-run-end Hooks

Execute SQL after all models complete — useful for pipeline logging:

```yaml
# dbt_project.yml
on-run-end:
  - "{{ log_pipeline_run_results() }}"
```

The hook macro can iterate `results` to log each model's status, execution time, and row counts to an audit table.

## Packages

Declared in `packages.yml` and installed with `dbt deps`:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: "1.3.3"
  - package: calogica/dbt_expectations
    version: "0.10.10"
```

### dispatch Override

Use project macros before package defaults:

```yaml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ["my_project", "dbt_utils"]
```

## Environment & Profiles

`profiles.yml` defines connection targets (never committed to git):

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SF_ACCOUNT') }}"
      user: "{{ env_var('SF_USER') }}"
      password: "{{ env_var('SF_PASSWORD') }}"
      role: ENGINEER
      database: DEV_T2_PERSISTENT_STAGING
      warehouse: TRN_DEV_CENTRAL_WH
      schema: LOGISTICS
      threads: 4
    prod:
      type: snowflake
      role: CHANGE_CONTROL
      database: PROD_T2_PERSISTENT_STAGING
      warehouse: TRN_PROD_CENTRAL_WH
```

## Common Commands

```bash
dbt deps                                    # Install packages
dbt parse                                   # Validate syntax (no warehouse needed)
dbt run --select tag:load_priority_1        # Run by tag
dbt run --select staging.customers          # Run specific model
dbt test --select tag:critical              # Test subset
dbt snapshot                                # Run SCD2 snapshots
dbt docs generate && dbt docs serve         # Generate and view docs
dbt run --full-refresh --select my_model    # Rebuild incremental from scratch
```

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Project config** | `dbt_project.yml` controls materializations, tags, database routing per environment |
| **Sources** | Declare raw tables for lineage tracking and freshness checks |
| **Incremental** | Use `_loaded_at` watermark pattern with `merge_update_columns` to protect immutable fields |
| **Snapshots** | Timestamp strategy when `updated_at` is reliable; check strategy otherwise |
| **Schema override** | `generate_schema_name` macro controls how schemas are named across environments |
| **Hooks** | `on-run-end` for pipeline logging and post-run automation |
