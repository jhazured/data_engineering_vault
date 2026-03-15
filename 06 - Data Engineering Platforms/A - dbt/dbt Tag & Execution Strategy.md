# dbt Tag & Execution Strategy

## Why Tags Matter

Tags control **execution order**, **selective runs**, **cost tracking**, and **pipeline logging**. In a multi-tier architecture, models must run in dependency order to avoid stale references.

## Tag Hierarchy

### Load Priority Tags (Execution Order)

| Tag | Layer | Purpose | Run Order |
|-----|-------|---------|-----------|
| `load_priority_1` | T2 Staging | Raw → clean staging models | 1st |
| `load_priority_2a` | T3 Dimensions | Staging → dimension tables | 2nd |
| `load_priority_2b` | T3 Facts | Staging + dimensions → fact tables | 3rd |
| `load_priority_3` | T4 Presentation | Marts → consumption views | 4th |

### Layer Tags (Classification)

| Tag | Purpose |
|-----|---------|
| `t0` | Control/audit/security models |
| `staging` | T2 staging models |
| `dimensions` | T3 dimension tables |
| `facts` | T3 fact tables |
| `presentation` | T4 consumption views |
| `snapshots` | SCD Type 2 snapshot models |

### Technical Tags

| Tag | Purpose |
|-----|---------|
| `incremental` | Models using incremental materialization |
| `scd_type_2` | Snapshot-backed slowly changing dimensions |
| `derived` | Models derived from other models (not directly from sources) |
| `high_volume` | Large/frequent data (telemetry, IoT) — monitor costs separately |
| `dev_only` | Excluded from staging/production runs |

### Business Context Tags

| Tag | Purpose |
|-----|---------|
| `customers` | Customer-related models |
| `shipments` | Shipment/delivery models |
| `vehicles` | Fleet/vehicle models |
| `data_quality` | Models used in quality monitoring |

## Execution Order

### Full DAG Run

```bash
# Phase 1: Snapshots (capture SCD2 state before transforms)
dbt snapshot

# Phase 2: Staging (T2)
dbt run --select tag:load_priority_1

# Phase 3: Dimensions (T3 — must complete before facts)
dbt run --select tag:load_priority_2a

# Phase 4: Facts (T3 — depend on dimensions)
dbt run --select tag:load_priority_2b

# Phase 5: Presentation (T4)
dbt run --select tag:load_priority_3

# Phase 6: Tests
dbt test --select tag:critical
```

### Selective Runs

```bash
# Only staging models
dbt run --select tag:staging

# Only customer-related models across all layers
dbt run --select tag:customers

# Everything except dev-only models (for staging/prod)
dbt run --exclude tag:dev_only

# Nightly data quality checks
dbt test --select tag:data_quality
```

## Configuring Tags

### In dbt_project.yml (folder-level)

```yaml
models:
  my_project:
    t2_persistent_staging:
      logistics:
        +tags: ["staging", "load_priority_1", "incremental"]

    t3_integration:
      logistics:
        dimensions:
          +tags: ["dimensions", "load_priority_2a", "incremental"]
        facts:
          +tags: ["facts", "load_priority_2b", "incremental"]

    t4_presentation:
      logistics:
        +tags: ["presentation", "business", "load_priority_3"]
```

### In model config blocks (model-level)

```sql
{{ config(
    materialized='incremental',
    tags=['staging', 'telemetry', 'incremental', 'high_volume']
) }}
```

### Snapshots

```yaml
snapshots:
  my_project:
    +tags: ["snapshots", "scd_type_2"]
```

## Tags in Pipeline Logging

The `on-run-end` hook uses tags to derive which layer a model belongs to:

```sql
{% set layer_tags = ['t0', 'staging', 'dimensions', 'facts', 'presentation'] %}
{% set ns = namespace(layer='unknown') %}
{% for tag in node.tags %}
  {% if tag in layer_tags and ns.layer == 'unknown' %}
    {% set ns.layer = tag %}
  {% endif %}
{% endfor %}
```

This writes to the pipeline run log, enabling per-layer execution time analysis.

## CI/CD Integration

Tags map directly to CI/CD pipeline stages:

| CI Stage | dbt Command | Tags Used |
|----------|------------|-----------|
| Build | `dbt parse` | N/A (syntax only) |
| Test (MR) | `dbt test` | All |
| Deploy Staging | `dbt run --exclude tag:dev_only` | Excludes dev_only |
| Deploy Prod | `dbt snapshot && dbt run --exclude tag:dev_only` | Excludes dev_only |
| Nightly QA | `dbt test --select tag:data_quality` | data_quality only |
