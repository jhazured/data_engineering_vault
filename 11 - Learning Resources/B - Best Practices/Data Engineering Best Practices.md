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
