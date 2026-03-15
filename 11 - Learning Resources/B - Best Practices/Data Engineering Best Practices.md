# Data Engineering Best Practices

> Concise, actionable rules for production-grade data engineering. Quick reference, not tutorials.

---

## 1. Pipeline Design

- **Idempotency** — same input must always produce same output. Use `MERGE`/`INSERT OVERWRITE` keyed on natural or surrogate keys. Stamp records with `_loaded_at` and `_batch_id`.
- **Incremental over full loads** — design for incremental from day one. Track high-water marks in a state table. Full refresh only on schema changes or corruption.
- **Fail fast** — validate inputs at the top of every step (schema, row counts, nulls). Raise specific exceptions with context: table, batch ID, timestamp.
- **Separate orchestration from transformation** — orchestrator handles scheduling/retries. Transformation lives in dbt/SQL/Python, never in DAG definitions.
- **Version control everything** — SQL, dbt, Terraform, DAGs, Docker in Git. Tag releases. No manual production changes.

---

## 2. Data Quality

- **Validate at boundaries** — check on ingestion (source to raw) and promotion (staging to production). Assert row counts, null rates, uniqueness. Quarantine bad records.
- **Test early and often** — dbt tests on every model: `unique`, `not_null`, `accepted_values`, `relationships`. Custom tests for business rules. Run in CI before merge.
- **Data contracts** — define schemas explicitly (JSON Schema, protobuf, Avro). Contract changes require versioning and consumer notification. Break the build on violations.
- **Monitor freshness/volume/schema drift** — alert on missed SLA windows. Track row counts per load. Detect column changes automatically.
- **Quality gates** — staging must pass all tests before production. No manual overrides without an audit trail.

---

## 3. SQL

**CTEs over subqueries.**
- Readable, testable, reusable. Name descriptively: `filtered_orders`, not `cte1`.
- One logical step per CTE — chain them for complex transformations.

**Avoid SELECT \*.**
- Explicit column lists prevent breakage on schema changes and enable column-level lineage.
- Exception: `SELECT * FROM {{ ref('model') }}` in dbt staging models mirroring source exactly.

**Qualify window functions.**
- Always include `PARTITION BY` and `ORDER BY` — implicit ordering is database-dependent.
- Use `QUALIFY` (Snowflake/BigQuery) to filter window results without subquery wrapping.

**Use MERGE for upserts.**
```sql
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.value = s.value, t.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT (id, value, updated_at) VALUES (s.id, s.value, CURRENT_TIMESTAMP());
```

**Test with dbt.**
- `dbt test` after every model change. `dbt build` in CI for models + tests in dependency order.

---

## 4. Python

**Type hints everywhere.**
- All function signatures, return types, and class attributes.
- Use `Optional`, `Union`, `list[str]`, `dict[str, Any]` — enforce with `mypy` in CI.

**Structured logging, not print.**
```python
import structlog
logger = structlog.get_logger()
logger.info("step_complete", table="orders", rows=1542, duration_s=3.7)
```
- JSON-structured logs are searchable, filterable, alertable.
- Include `correlation_id`, `batch_id`, `table_name` in every log entry.

**Factory pattern for pluggable sources.**
```python
def get_source(source_type: str) -> BaseSource:
    sources = {"s3": S3Source, "api": APISource, "db": DatabaseSource}
    return sources[source_type]()
```
- New sources = one class + one registry entry. No `if/elif` chains.

**Retry with backoff for external calls.**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=30))
def call_api(url: str) -> dict: ...
```
- All HTTP, database, and cloud SDK calls need retry logic.

**Requirements per environment.**
- Pinned `requirements.txt` for production, `requirements-dev.txt` for extras (pytest, mypy, ruff).
- Use `pip-compile` or `poetry.lock` for deterministic installs.

---

## 5. Snowflake

**Right-size warehouses.**
- Start with XS. Scale up only when query times breach SLAs.
- Separate warehouses per workload: `LOADING_WH`, `TRANSFORM_WH`, `ANALYTICS_WH`.
- Multi-cluster warehouses for concurrency, not larger single sizes.

**Use resource monitors.**
- Credit quotas per warehouse — alert at 75%, suspend at 100%.
- Monitor daily. A runaway query can burn your monthly budget in hours.

**Cluster on high-cardinality filter columns.**
- Tables with 100M+ rows, clustered on `WHERE`/`JOIN` columns.
- Typical candidates: `event_date`, `customer_id`, `region`.
- Check clustering depth with `SYSTEM$CLUSTERING_INFORMATION`.

**RBAC with functional roles.**
- Create roles by function: `LOADER`, `TRANSFORMER`, `ANALYST`, `ADMIN`.
- Grant roles to roles, then roles to users — never privileges directly to users.

**Tag sensitive data.**
- `ALTER TABLE SET TAG pii = 'email'`. Apply masking policies tied to tags.
- Classify columns during ingestion, not as an afterthought.

---

## 6. dbt

**Incremental models for large tables.**
```sql
{{ config(materialized='incremental', unique_key='id', incremental_strategy='merge') }}
SELECT * FROM {{ source('raw', 'events') }}
{% if is_incremental() %}
  WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

**Test every model.**
- At minimum: `unique` and `not_null` on primary keys.
- `accepted_values` on status/category columns. `relationships` for every FK.

**Use tags for execution order.**
- `tags: ['hourly']`, `tags: ['daily']`, `tags: ['weekly']`.
- Run with `dbt run --select tag:hourly`. Use `+` for upstream: `dbt run --select +tag:daily`.

**Macros for repeated logic.**
- Date spines, surrogate keys, SCD Type 2 — write once, reuse everywhere.
- Keep in `macros/` with clear names: `generate_surrogate_key.sql`.

**Document in schema.yml.**
- Every model, every column. No exceptions for production models.
- `description:` fields power `dbt docs generate`. Include business context.

---

## 7. DevOps

**CI/CD for all SQL/dbt/Terraform.**
- PR triggers: lint, compile, run tests in CI environment.
- Merge to main deploys to staging. Manual promotion to production.
- No direct pushes to main — enforce branch protection.

**Docker for reproducible environments.**
- One Dockerfile per project. Pin base image versions.
- Multi-stage builds: dependencies in stage 1, slim runtime in stage 2.
- `docker-compose.yml` for local development with all dependencies.

**Infrastructure as code.**
- Snowflake roles, warehouses, databases — all in Terraform.
- Drift detection: plan regularly, alert on manual changes.

**Secret management.**
- AWS Secrets Manager, Azure Key Vault, or HashiCorp Vault.
- Never in code, configs, or committed env files. Rotate on schedule.

**Blue-green deployments.**
- Deploy new version alongside old. Validate, then switch traffic.
- For dbt: build to staging schema, test, swap with production.
- Rollback plan must execute in under 5 minutes.

---

## 8. Security

**Least privilege.**
- Grant minimum permissions required. Start with nothing, add as needed.
- Service accounts get only what their pipeline touches — no admin rights.

**Encrypt at rest and in transit.**
- TLS for all connections — no exceptions.
- Enable encryption on S3 buckets, databases, and warehouses. Keys in KMS.

**Audit all access.**
- Enable query logging and access history (Snowflake `ACCESS_HISTORY`).
- Log who accessed what data, when, and from where. Retain 1-7 years.

**Classify data.**
- PII, financial, operational, public — tag every dataset.
- Classification drives retention, access, encryption, and audit requirements.

**Row-level security for multi-tenant.**
- Secure views or Snowflake row access policies.
- Filter by tenant ID in a centralized policy — not scattered `WHERE` clauses.

---

## 9. Cost Management

**Monitor cloud spend daily.**
- Dashboards for Snowflake credits, S3 storage, compute costs.
- Alert on anomalies: 2x daily average triggers investigation. Assign cost centers.

**Auto-suspend warehouses.**
- `AUTO_SUSPEND = 60` for interactive, immediate for batch loading.
- Never leave warehouses running with no active queries.

**Lifecycle policies for storage.**
- S3 Infrequent Access after 30 days, Glacier after 90.
- Set `DATA_RETENTION_TIME_IN_DAYS` by actual recovery need. Delete staging/temp automatically.

**Reserved capacity for predictable workloads.**
- Capacity contracts for steady state (30-40% savings). On-demand for burst only.

**Query cost limits.**
- `STATEMENT_TIMEOUT_IN_SECONDS` to prevent runaway queries.
- `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` to avoid long queue waits.

---

## 10. Documentation

**README every project.**
- What it does, how to set up, how to run, who owns it. Keep it updated.

**Architecture decision records (ADRs).**
- Context, decision, consequences. One page per decision in `docs/adr/`.
- Document the *why* — future you will thank present you.

**Data dictionaries.**
- Every table and column in business terms. Types, valid ranges, null policy, source system.
- Generate from dbt `schema.yml` as single source of truth.

**Runbooks for operational procedures.**
- Step-by-step for common incidents: pipeline failure, data backfill, access requests.
- Include commands, expected outputs, escalation contacts. Test quarterly.

---

## 11. Per-Domain Best Practices: SQL

### Query Patterns

**Use CTEs for readability and composability.**
- Structure queries as a chain of named CTEs, each performing one logical step.
- Final `SELECT` assembles the result from upstream CTEs.
- Prefer `WITH` over nested subqueries -- they are easier to test, debug, and refactor.

**Window functions for ranking, running totals, and deduplication.**
```sql
-- Deduplicate using ROW_NUMBER
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) AS rn
    FROM raw_customers
)
SELECT * FROM ranked WHERE rn = 1;
```

**QUALIFY for concise window filtering (Snowflake, BigQuery, Databricks).**
```sql
SELECT *
FROM raw_events
QUALIFY ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY received_at DESC) = 1;
```

**Parameterise date ranges.**
- Never hard-code dates. Use variables, macros, or Jinja templates.
- Pattern: `WHERE event_date BETWEEN {{ start_date }} AND {{ end_date }}`.

**Use MERGE for idempotent loads.**
- MERGE handles insert/update/delete in a single atomic statement.
- Always include `updated_at` timestamps in the MATCHED clause.

### Anti-Patterns

- **SELECT DISTINCT as a fix for duplicates** -- indicates a join or source issue. Fix the root cause rather than masking it.
- **Correlated subqueries in SELECT** -- replace with JOINs or window functions. Correlated subqueries execute once per row.
- **OR in JOIN conditions** -- creates cross joins internally. Refactor into UNION ALL of separate joins.
- **Implicit type conversions** -- always CAST explicitly. Implicit conversions prevent predicate pushdown and clustering pruning.
- **Functions on indexed/clustered columns in WHERE** -- `WHERE YEAR(event_date) = 2026` cannot use clustering. Write `WHERE event_date >= '2026-01-01' AND event_date < '2027-01-01'`.
- **Cartesian joins without explicit CROSS JOIN** -- accidental cartesian products from missing ON clauses. Always specify join conditions.
- **Overusing UNION when UNION ALL suffices** -- `UNION` forces a sort/distinct operation. Use `UNION ALL` unless deduplication is genuinely needed.

---

## 12. Per-Domain Best Practices: Python

### Project Structure

```
project_root/
    src/
        project_name/
            __init__.py
            extract/          # Source-specific extractors
            transform/        # Pure transformation functions
            load/             # Target-specific loaders
            utils/            # Shared helpers (logging, config, retry)
    tests/
        unit/                 # Fast, no external dependencies
        integration/          # Requires database/API access
        fixtures/             # Shared test data
    pyproject.toml            # Project metadata, dependencies, tool config
    Dockerfile
    Makefile                  # Common commands: test, lint, format, build
```

### Testing Strategy

- **Unit tests** -- test every transformation function in isolation. Use `pytest` with parametrised fixtures for edge cases (nulls, empty DataFrames, type mismatches).
- **Integration tests** -- test against real connections in CI. Use test containers or ephemeral cloud resources. Clean up after each test.
- **Property-based testing** -- use `hypothesis` for data transformation functions. Generate random inputs and assert invariants (row count preservation, column types, non-negativity).

```python
import pytest
import pandas as pd
from project_name.transform import deduplicate_customers

def test_deduplicate_keeps_latest():
    df = pd.DataFrame({
        "customer_id": [1, 1, 2],
        "name": ["Alice", "Alice Updated", "Bob"],
        "updated_at": ["2026-01-01", "2026-02-01", "2026-01-15"],
    })
    result = deduplicate_customers(df)
    assert len(result) == 2
    assert result.loc[result["customer_id"] == 1, "name"].iloc[0] == "Alice Updated"
```

### Packaging

- Use `pyproject.toml` (PEP 621) as the single source of truth for metadata, dependencies, and tool configuration.
- Pin dependencies with `pip-compile` or `poetry.lock` for reproducible builds.
- Publish internal packages to a private PyPI (AWS CodeArtifact, GCP Artifact Registry, Azure Artifacts).
- Prefer `src/` layout to prevent accidental imports from the project root.

### Code Quality Tooling

| Tool | Purpose |
|------|---------|
| `ruff` | Linting and formatting (replaces flake8, isort, black) |
| `mypy` | Static type checking |
| `pytest` | Test runner with fixtures and parametrisation |
| `pre-commit` | Git hooks for automated checks before commit |

---

## 13. Per-Domain Best Practices: dbt

### Model Organisation

Follow the staging/intermediate/marts layering convention:

```
models/
    staging/           # 1:1 with source tables, light renaming/casting
        stg_orders.sql
        stg_customers.sql
    intermediate/      # Business logic joins and transformations
        int_order_items_enriched.sql
    marts/             # Final business entities for consumers
        fct_orders.sql
        dim_customers.sql
```

- **Staging** -- one model per source table. `SELECT` from `{{ source() }}`, rename columns to consistent conventions, cast types, filter soft deletes. Materialise as views.
- **Intermediate** -- join staging models, apply business rules. Materialise as ephemeral or views.
- **Marts** -- consumer-facing tables. Materialise as tables or incremental.

### Naming Conventions

| Layer | Prefix | Example |
|-------|--------|---------|
| Staging | `stg_` | `stg_stripe__payments` |
| Intermediate | `int_` | `int_payments_enriched` |
| Facts | `fct_` | `fct_orders` |
| Dimensions | `dim_` | `dim_customers` |
| Snapshots | `snap_` | `snap_orders` |
| Macros | Verb-first | `generate_surrogate_key` |

Use double underscores to separate source system from entity: `stg_shopify__orders`.

### Testing Strategy

- **Primary keys** -- `unique` + `not_null` on every model's primary key. No exceptions.
- **Foreign keys** -- `relationships` test for every foreign key reference.
- **Business rules** -- custom schema tests or `dbt_expectations` package for range checks, regex patterns, recency.
- **Source freshness** -- `loaded_at_field` in source definitions with `warn_after` and `error_after` thresholds.
- **CI testing** -- `dbt build` in CI on every pull request. Use `--select state:modified+` to test only changed models and their downstream dependants.

### Materialisation Selection

| Materialisation | When to Use |
|----------------|-------------|
| **View** | Staging models, lightweight transformations, always-fresh requirements |
| **Table** | Mart models < 100M rows, infrequently queried intermediate results |
| **Incremental** | Large fact tables (> 100M rows), event streams, append-heavy data |
| **Ephemeral** | Intermediate CTEs that should not be materialised, used only by one downstream model |
| **Snapshot** | SCD Type 2 tracking of source data changes |

Avoid incremental models until the table justifies it -- incremental adds complexity (merge keys, late-arriving data handling, full-refresh requirements).

---

## 14. Per-Domain Best Practices: Snowflake

### Warehouse Sizing and Management

- **Start small** -- XS for most workloads. Scale up only when query profile shows queuing or spilling.
- **Workload isolation** -- separate warehouses for loading (`LOADING_WH`), transformation (`TRANSFORM_WH`), analytics (`ANALYTICS_WH`), and dbt CI (`CI_WH`).
- **Multi-cluster for concurrency** -- when queries queue due to concurrency, use multi-cluster warehouses (scaling policy: standard for predictable, economy for variable).
- **Auto-suspend aggressively** -- `AUTO_SUSPEND = 60` for interactive warehouses, `AUTO_SUSPEND = 0` (immediate) for batch workloads that run and stop.
- **Query timeout** -- set `STATEMENT_TIMEOUT_IN_SECONDS = 3600` to prevent runaway queries consuming credits for hours.

### Clustering Keys

- **When to cluster** -- tables with > 500M rows where queries consistently filter on the same columns.
- **Column selection** -- choose columns that appear in `WHERE` and `JOIN` clauses. Prefer low-to-medium cardinality (dates, categories) over high cardinality (UUIDs).
- **Compound keys** -- cluster on up to 3-4 columns. Order from lowest to highest cardinality.
- **Monitor** -- `SELECT SYSTEM$CLUSTERING_INFORMATION('schema.table', '(cluster_col)')`. Target average clustering depth < 2.
- **Automatic Clustering** -- enable `ALTER TABLE SET CLUSTER BY (col1, col2)` and let Snowflake manage reclustering automatically.

### Query Profiling

Use the Query Profile in the Snowflake UI for performance diagnosis:

| Indicator | Problem | Resolution |
|-----------|---------|------------|
| **Bytes spilled to local storage** | Insufficient memory for sort/join | Scale up warehouse or reduce data volume with filters |
| **Bytes spilled to remote storage** | Severe memory pressure | Scale up warehouse; add clustering to reduce scan |
| **Partition pruning ratio low** | Poor clustering or missing filters | Add clustering keys; add date range filters |
| **Exploding joins** | Join produces more rows than inputs | Check join keys for duplicates; add deduplication upstream |
| **Full table scan** | No predicates or clustering | Add WHERE clauses on clustered columns |

### Cost Control

- **Resource monitors** -- create per-warehouse and account-level monitors. Alert at 75%, suspend at 100% of monthly budget.
- **Query tagging** -- `ALTER SESSION SET QUERY_TAG = 'dbt_run_daily'`. Enables cost attribution by pipeline, team, or project.
- **Zero-copy clones** -- use `CREATE TABLE clone_table CLONE source_table` for dev/test environments instead of full copies.
- **Transient tables** -- use `CREATE TRANSIENT TABLE` for staging tables that do not need Fail-safe storage (saves 7 days of storage costs).
- **Time Travel minimisation** -- set `DATA_RETENTION_TIME_IN_DAYS = 1` for staging/transient data (default is 1 for standard, up to 90 for enterprise).
- **Storage monitoring** -- query `SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS` monthly to identify bloated tables with excessive Time Travel or Fail-safe storage.
