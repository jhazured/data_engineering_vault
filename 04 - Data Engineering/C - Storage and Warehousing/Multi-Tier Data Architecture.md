# Multi-Tier Data Architecture

## Overview

A five-tier database architecture separates data by purpose, access pattern, and lifecycle. Each tier is a separate Snowflake database with its own RBAC grants.

```
T0_CONTROL ─── Audit, security, job control, monitoring
T1_TRANSIENT_STAGING ─── Raw landing zone (Fivetran, data loaders)
T2_PERSISTENT_STAGING ─── dbt staging models (cleaned, typed, incremental)
T3_INTEGRATION ─── dbt dimensions + facts (business logic applied)
T4_PRESENTATION ─── Consumption views (dashboards, BI tools)
```

## Data Flow

```
External Sources (APIs, databases, files)
         │
         ▼
T1_TRANSIENT_STAGING.LOGISTICS    ← Fivetran / sample data loader
         │                           Raw data, source units, no transforms
         ▼
T2_PERSISTENT_STAGING.LOGISTICS   ← dbt incremental staging models
         │                           Type casting, null handling, _ingested_at
         ▼
T3_INTEGRATION.LOGISTICS
  ├── dimensions/                 ← SCD2 dimensions from snapshots
  │     TBL_DIM_CUSTOMER            Unit conversions, business logic
  │     TBL_DIM_VEHICLE             Segment mapping, derived flags
  │     TBL_DIM_DATE
  └── facts/                      ← Fact tables joining dimensions
        TBL_FACT_SHIPMENTS           Profit calculations, KPIs
        TBL_FACT_VEHICLE_TELEMETRY   Derived events, health scores
         │
         ▼
T4_PRESENTATION.LOGISTICS        ← Views only (no storage cost)
  VW_CONSOLIDATED_DASHBOARD         Queried via OUT_ warehouse
  VW_DIM_CUSTOMER_CURRENT
  VW_SUSTAINABILITY_METRICS

T0_CONTROL
  ├── AUDIT/     ← Cost monitoring, query analysis, data freshness views
  ├── SECURITY/  ← Data classification catalogue, access history
  └── JOB_CONTROL/ ← Pipeline run log (dbt on-run-end hook)
```

## Tier Responsibilities

### T0 — Control

| Schema | Purpose |
|--------|---------|
| `AUDIT` | Cost monitoring, warehouse usage, query performance, data skew analysis |
| `SECURITY` | Data classification catalogue, access history auditing |
| `JOB_CONTROL` | Pipeline run log (one row per dbt model per run) |

**Materialization:** Tables and views. Not managed by dbt models except audit/security views.

### T1 — Transient Staging

- **Raw landing zone** — data arrives here from Fivetran, APIs, or data loaders
- **Source units preserved** — imperial measurements, original formats
- **Standard audit columns:** `created_at`, `updated_at`, `is_deleted`, `_loaded_at`
- **NOT managed by dbt** — dbt declares T1 as `sources` only

### T2 — Persistent Staging

- **1:1 with source tables** — one staging model per source
- **Clean pass-through** — type casting, TRIM, COALESCE, null handling
- **NO business logic** — segment mapping, unit conversions, derived fields belong in T3
- **Incremental** with `_ingested_at` watermark pattern
- **SCD2 snapshots** also live here

### T3 — Integration

- **Business logic applied here** — customer type mapping, unit conversions, derived KPIs
- **Dimensions:** SCD Type 2 from snapshots, with surrogate keys
- **Facts:** Join dimensions via point-in-time SCD2 ranges
- **Incremental merge** with `merge_update_columns` to protect immutable fields

### T4 — Presentation

- **Views only** — no storage cost, always fresh
- **Pre-filtered** — e.g., `vw_dim_customer_current` filters `is_current = true`
- **Dashboard-ready** — rolling averages, YoY comparisons, trend classification
- **Queried via `OUT_` warehouses** for cost isolation from ETL

## Environment Separation

Each tier exists per environment with a prefix:

| Environment | T1 | T2 | T3 | T4 |
|-------------|----|----|----|----|
| Dev | `DEV_T1_TRANSIENT_STAGING` | `DEV_T2_PERSISTENT_STAGING` | `DEV_T3_INTEGRATION` | `DEV_T4_PRESENTATION` |
| Test | `TEST_T1_...` | `TEST_T2_...` | `TEST_T3_...` | `TEST_T4_...` |
| Prod | `PROD_T1_...` | `PROD_T2_...` | `PROD_T3_...` | `PROD_T4_...` |

dbt routes models to the correct database via:
```yaml
+database: "{{ env_var('SF_ENV', 'DEV') | upper }}_T2_PERSISTENT_STAGING"
```

## Warehouse Isolation

| Prefix | Purpose | Roles | Size |
|--------|---------|-------|------|
| `IN_` | Data ingestion (Fivetran) | Service accounts | X-Small |
| `TRN_` | Transformation (dbt) | ENGINEER, CHANGE_CONTROL | X-Small |
| `OUT_` | Consumption (BI, dashboards) | ANALYST, CONSUMER | X-Small+ |

All warehouses use `AUTO_SUSPEND = 60` (dev) or `120` (prod) seconds, `AUTO_RESUME = TRUE`, `INITIALLY_SUSPENDED = TRUE`.

## Load Priority Execution

Models run in dependency order using tags:

```bash
dbt snapshot                           # Capture SCD2 state
dbt run --select tag:load_priority_1   # T2 staging
dbt run --select tag:load_priority_2a  # T3 dimensions
dbt run --select tag:load_priority_2b  # T3 facts (depend on dimensions)
dbt run --select tag:load_priority_3   # T4 presentation views
```

## Why This Architecture?

| Benefit | How |
|---------|-----|
| **Cost isolation** | Separate warehouses for ingestion/transform/consumption |
| **Access control** | Per-tier database grants = easy RBAC |
| **Lineage clarity** | Data flows strictly T1→T2→T3→T4 |
| **Schema evolution** | `on_schema_change='sync_all_columns'` in T2 absorbs source drift |
| **Audit trail** | T0 pipeline log + access history for compliance |
| **Environment parity** | Same tier structure across dev/test/prod |
