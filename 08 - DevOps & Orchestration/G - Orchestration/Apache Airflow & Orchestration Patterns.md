# Apache Airflow & Orchestration Patterns

## Why Orchestration Matters

Modern data platforms comprise dozens of interdependent tools -- [[dbt Advanced Patterns & Cost Optimisation|dbt]] for transformation, [[Snowflake SQL Pipeline Patterns|Snowflake]] for warehousing, [[PySpark Production Engineering|Spark]] for heavy compute, and [[GitHub Actions for Python Projects|CI/CD]] for deployment. Without a dedicated orchestrator, coordinating these systems devolves into fragile cron jobs, ad-hoc scripts, and manual restarts that no one understands at 3 AM.

An orchestrator provides:

- **Dependency management** -- express relationships between tasks so upstream failures prevent downstream corruption.
- **Scheduling** -- time-based and event-based triggering, not just `crontab -e` on a bastion host.
- **Retry and backoff** -- automatic recovery from transient failures (API rate limits, credential rotation, cluster cold starts).
- **Monitoring and alerting** -- visibility into run status, duration trends, SLA misses, and Slack/PagerDuty integration.
- **Auditability** -- a historical record of every run, its inputs, its outputs, and who triggered it.

Orchestration is the glue layer -- it does not move data itself but ensures the right tool fires at the right time, in the right order, with the right configuration.

---

## Airflow Architecture

Apache Airflow (originally built at Airbnb, 2014) is the de-facto open-source orchestrator. It models workflows as Directed Acyclic Graphs (DAGs) defined in Python.

### Core Components

| Component | Role |
|---|---|
| **Scheduler** | Parses DAG files, determines which tasks are ready to run, submits them to the executor. Runs continuously. |
| **Webserver** | Flask-based UI for monitoring DAGs, viewing logs, triggering manual runs, managing connections. |
| **Workers** | Processes that actually execute task code. May run on the same host or on remote nodes (Celery/K8s). |
| **Metadata DB** | PostgreSQL (or MySQL) backend storing DAG state, task instance status, connection credentials, XCom data. |
| **Triggerer** | (Airflow 2.x) Async event loop for deferrable operators -- frees up worker slots while waiting on external events. |

### Executor Types

| Executor | When to Use |
|---|---|
| **SequentialExecutor** | Local development only. Single task at a time. |
| **LocalExecutor** | Small-to-medium workloads on a single machine. One process per task. |
| **CeleryExecutor** | Distributed workers via Redis/RabbitMQ. Good for on-prem or VM-based setups. |
| **KubernetesExecutor** | Each task runs in its own K8s pod. Full isolation, dynamic scaling, ideal for heterogeneous workloads. See [[Kubernetes for Data Workloads]]. |
| **CeleryKubernetesExecutor** | Hybrid: fast tasks on Celery, heavy tasks on K8s pods. |

### DAG Bag

The DAG Bag is the collection of all DAG files that the scheduler scans. Files live in a configured `dags_folder` and are re-parsed on a configurable interval (`dag_dir_list_interval`). Keep DAG parse time under 30 seconds; avoid heavy imports at module level.

---

## DAG Design

### Python DAG Definition

A DAG file is a standard Python module. Airflow imports it and looks for `DAG` objects (or the `@dag` decorator in TaskFlow API).

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    "owner": "data-engineering",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "email_on_failure": True,
    "email": ["data-alerts@company.com"],
}

with DAG(
    dag_id="daily_warehouse_refresh",
    default_args=default_args,
    schedule="0 6 * * *",        # 06:00 UTC daily
    start_date=datetime(2025, 1, 1),
    catchup=False,
    tags=["warehouse", "production"],
) as dag:

    extract = BashOperator(
        task_id="extract_source_data",
        bash_command="python /opt/etl/extract.py --date {{ ds }}",
    )

    transform = PythonOperator(
        task_id="transform_data",
        python_callable=run_transform,
        op_kwargs={"execution_date": "{{ ds }}"},
    )

    load = BashOperator(
        task_id="load_to_warehouse",
        bash_command="python /opt/etl/load.py --date {{ ds }}",
    )

    extract >> transform >> load
```

### Key Operators

| Operator | Purpose |
|---|---|
| `BashOperator` | Run shell commands. Used in CI/CD-triggered DAGs (see logistics-analytics-platform `automation.yml` calling `master_orchestrator.py`). |
| `PythonOperator` | Run a Python callable. Keep functions importable and testable. |
| `SnowflakeOperator` | Execute SQL against [[Snowflake SQL Pipeline Patterns|Snowflake]]. Requires `snowflake` connection. |
| `DbtCloudOperator` | Trigger a dbt Cloud job and poll for completion. See [[dbt Advanced Patterns & Cost Optimisation]]. |
| `SparkSubmitOperator` | Submit a Spark job to a cluster. See [[PySpark Production Engineering]]. |
| `KubernetesPodOperator` | Spin up an arbitrary container as a task. See [[Kubernetes for Data Workloads]]. |
| `BigQueryInsertJobOperator` | Run BigQuery SQL on [[GCP Data Services for Data Engineering|GCP]]. |

### Task Dependencies

```python
# Linear chain
extract >> transform >> load

# Fan-out
extract >> [validate_schema, validate_row_counts, validate_nulls]

# Fan-in
[validate_schema, validate_row_counts, validate_nulls] >> load

# Mixed
extract >> transform >> [branch_a, branch_b]
branch_a >> merge
branch_b >> merge
merge >> notify
```

### TaskGroups

Group related tasks visually and logically without creating a SubDAG (SubDAGs are deprecated).

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup("data_quality_checks") as quality_group:
    check_nulls = PythonOperator(task_id="check_nulls", ...)
    check_dupes = PythonOperator(task_id="check_dupes", ...)
    check_ranges = PythonOperator(task_id="check_ranges", ...)

extract >> quality_group >> load
```

### Dynamic Task Mapping (Airflow 2.3+)

Generate tasks at runtime based on data -- replaces the need for loops in DAG definition.

```python
@task
def get_source_tables():
    return ["orders", "customers", "products", "shipments"]

@task
def process_table(table_name: str):
    # Extract and load a single table
    run_pipeline(table_name)

tables = get_source_tables()
process_table.expand(table_name=tables)
```

---

## Connections & Hooks

### Airflow Connections

Connections store credentials for external systems. Managed via the UI, CLI, or environment variables (`AIRFLOW_CONN_{ID}`).

| Connection Type | Example conn_id | Used For |
|---|---|---|
| Snowflake | `snowflake_prod` | [[Snowflake SQL Pipeline Patterns|Snowflake]] queries, dbt runs |
| AWS | `aws_default` | S3 reads/writes, Glue jobs, Redshift |
| Google Cloud | `google_cloud_default` | BigQuery, GCS, [[GCP Data Services for Data Engineering|Dataflow]] |
| Databricks | `databricks_default` | Spark job submission, Delta Lake operations |
| SSH | `sftp_vendor` | Legacy file drops from vendors |

### Hooks

Hooks are the Python interface to connections. Operators use hooks internally; you can also use them in `PythonOperator` callables for custom integrations.

```python
from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook

def run_custom_query(**context):
    hook = SnowflakeHook(snowflake_conn_id="snowflake_prod")
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute("CALL my_stored_procedure(%(date)s)", {"date": context["ds"]})
```

### Secrets Backends

For production deployments, store credentials in a secrets manager rather than the Airflow metadata DB.

| Backend | Configuration |
|---|---|
| **AWS SSM Parameter Store** | `SecretsManagerBackend` with `connections_prefix=/airflow/connections` |
| **GCP Secret Manager** | `CloudSecretManagerBackend` -- integrates with [[GCP Data Services for Data Engineering|Cloud Composer]] |
| **HashiCorp Vault** | `VaultBackend` with AppRole or Kubernetes auth |
| **AWS Secrets Manager** | `SecretsManagerBackend` with `connections_prefix` |

Configure in `airflow.cfg`:

```ini
[secrets]
backend = airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend
backend_kwargs = {"connections_prefix": "airflow-connections", "variables_prefix": "airflow-variables", "gcp_project_id": "my-project"}
```

---

## Scheduling Patterns

### Cron Expressions

Airflow uses standard cron syntax or preset strings.

```python
schedule="0 6 * * *"          # Daily at 06:00 UTC
schedule="0 */2 * * *"        # Every 2 hours
schedule="0 6 * * 1"          # Every Monday at 06:00
schedule="@daily"             # Alias for 0 0 * * *
schedule="@hourly"            # Alias for 0 * * * *
```

Real-world example from the logistics platform -- the `dbt_ci_cd.yml` workflow uses `cron: '0 2 * * *'` for nightly data quality checks, and the `automation.yml` uses `cron: '0 */6 * * *'` for health checks every six hours.

### Data-Aware Scheduling (Datasets) -- Airflow 2.4+

Trigger DAGs when upstream DAGs update a dataset, replacing time-based polling with event-driven orchestration.

```python
from airflow.datasets import Dataset

orders_dataset = Dataset("snowflake://prod/raw.orders")

# Producer DAG
with DAG("ingest_orders", schedule="@hourly") as producer:
    load_task = PythonOperator(
        task_id="load_orders",
        python_callable=load_orders,
        outlets=[orders_dataset],       # Marks this dataset as updated
    )

# Consumer DAG -- runs whenever orders_dataset is updated
with DAG("transform_orders", schedule=[orders_dataset]) as consumer:
    transform_task = PythonOperator(...)
```

### Timetables (Airflow 2.2+)

Custom scheduling logic when cron is not expressive enough (e.g., "run on business days only" or "run on the last Friday of each month").

### Catchup vs Backfill

- **`catchup=True`** (default): Airflow schedules runs for every missed interval between `start_date` and now. Use for idempotent historical processing.
- **`catchup=False`**: Only schedule the most recent interval. Use for forward-looking pipelines where reprocessing old data is not needed.
- **Backfill CLI**: `airflow dags backfill -s 2025-01-01 -e 2025-03-01 daily_warehouse_refresh` -- manually trigger historical runs.

### SLA Monitoring

Set `sla` on individual tasks to receive alerts when execution exceeds expected duration.

```python
transform = PythonOperator(
    task_id="transform",
    python_callable=run_transform,
    sla=timedelta(hours=2),       # Alert if task takes > 2 hours
)
```

---

## Common DAG Patterns

### ETL Pipeline (Extract >> Transform >> Load)

The classic pattern. Each phase is a separate task or TaskGroup.

```python
extract >> transform >> load >> validate >> notify
```

### dbt DAG (Seed >> Run >> Test)

Orchestrate dbt from Airflow to centralize scheduling and monitoring. See [[Core dbt Fundamentals]] and [[dbt Tag & Execution Strategy]].

```python
dbt_seed = BashOperator(task_id="dbt_seed", bash_command="dbt seed --project-dir /opt/dbt")
dbt_run  = BashOperator(task_id="dbt_run",  bash_command="dbt run --project-dir /opt/dbt --select tag:daily")
dbt_test = BashOperator(task_id="dbt_test", bash_command="dbt test --project-dir /opt/dbt --select tag:critical")

dbt_seed >> dbt_run >> dbt_test
```

### Fan-Out / Fan-In

Process multiple sources in parallel, then merge results.

```python
sources = ["orders", "customers", "products"]
extract_tasks = [PythonOperator(task_id=f"extract_{s}", ...) for s in sources]
merge = PythonOperator(task_id="merge_all", ...)

start >> extract_tasks >> merge >> load
```

### Conditional Branching

Use `BranchPythonOperator` to choose execution paths at runtime.

```python
from airflow.operators.python import BranchPythonOperator

def choose_branch(**context):
    if context["ds_nodash"] in holidays:
        return "skip_processing"
    return "run_full_pipeline"

branch = BranchPythonOperator(task_id="branch", python_callable=choose_branch)
branch >> [run_full_pipeline, skip_processing]
```

### Trigger Rules

Control when a task is eligible to run.

| Rule | Behaviour |
|---|---|
| `all_success` | (default) All parents succeeded. |
| `all_failed` | All parents failed -- used for error-handling tasks. |
| `one_success` | At least one parent succeeded -- used after branching. |
| `none_failed` | No parent failed (skipped is OK) -- common after `BranchPythonOperator`. |
| `all_done` | All parents finished regardless of status -- used for cleanup. |

---

## Cloud Composer / MWAA

### Google Cloud Composer

Cloud Composer is managed Airflow on [[GCP Data Services for Data Engineering|GCP]]. DAG files are deployed to a GCS bucket that the Composer environment watches.

```bash
# Deploy DAGs to Composer (from CI/CD)
gsutil -m rsync -r -d ./dags gs://composer-bucket-region/dags/

# Trigger a DAG from CI/CD
gcloud composer environments run my-composer-env \
    --location us-central1 \
    dags trigger -- daily_warehouse_refresh \
    --conf '{"source": "ci_cd", "commit": "abc123"}'
```

Environment configuration includes PyPI packages, environment variables, and Airflow config overrides -- all managed via Terraform. See [[Terraform for Data Infrastructure]].

### AWS Managed Workflows for Apache Airflow (MWAA)

MWAA is the AWS equivalent. DAGs and plugins are stored in S3. Supports Airflow 2.x with automatic scaling of workers.

```bash
# Deploy DAGs to MWAA
aws s3 sync ./dags s3://mwaa-bucket/dags/ --delete

# Trigger via CLI (using MWAA CLI token)
aws mwaa create-cli-token --name my-mwaa-env
# Then POST to the CLI endpoint
```

### CI/CD Integration

Both managed services integrate with [[GitHub Actions for Python Projects|GitHub Actions]] and [[Jenkins Pipeline Patterns for Data Engineering|Jenkins]].

The logistics-analytics-platform `dbt_ci_cd.yml` shows the pattern: push to `main` triggers deployment which could include syncing DAG files to the managed Airflow bucket. The `automation.yml` goes further -- running parallel orchestration jobs on a 6-hour schedule with `workflow_dispatch` for manual triggers.

---

## Dagster as Alternative

Dagster takes an asset-first approach -- you define the data artifacts you want to exist, and Dagster figures out the execution plan.

### Core Concepts

| Concept | Airflow Equivalent |
|---|---|
| **Software-defined asset** | No direct equivalent (closest is a task + dataset). |
| **Op** | Operator / task. |
| **Job** | DAG. |
| **Graph** | Task dependencies. |
| **IO Manager** | No equivalent -- manages reading/writing assets to storage (S3, Snowflake, etc.). |
| **Resource** | Connection / hook. |

### Software-Defined Assets

```python
from dagster import asset

@asset
def raw_orders():
    """Extracts orders from source API."""
    return extract_from_api("orders")

@asset
def clean_orders(raw_orders):
    """Cleans and validates raw orders."""
    return clean(raw_orders)

@asset
def orders_summary(clean_orders):
    """Aggregates orders by region."""
    return aggregate(clean_orders)
```

Dependencies are inferred from function signatures. Dagster builds the execution graph automatically.

### When to Choose Dagster

- Greenfield project with no existing Airflow investment.
- Team values asset lineage and data catalogue features.
- Want integrated development experience (Dagster UI, asset materialisation history).
- Dagster Cloud provides serverless execution.

---

## Prefect as Alternative

Prefect positions itself as "Airflow done right" -- simpler deployment, native Python, and a modern execution model.

### Core Concepts

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=60)
def extract_data(source: str):
    return run_extraction(source)

@task
def transform_data(raw_data):
    return apply_transforms(raw_data)

@flow(name="daily-etl")
def daily_etl():
    raw = extract_data("orders")
    transformed = transform_data(raw)
    load_to_warehouse(transformed)
```

### Key Differences from Airflow

| Feature | Airflow | Prefect |
|---|---|---|
| DAG definition | Separate from execution | Code IS the workflow |
| Scheduling | Built into scheduler | Prefect Cloud or self-hosted server |
| Dynamic workflows | Dynamic task mapping (2.3+) | Native Python loops and conditionals |
| Deployment | DAG files in folder | `prefect deploy` with `prefect.yaml` |
| Concurrency | Executor-level | Task-level concurrency limits, work pools |

### When to Choose Prefect

- Small-to-medium team wanting fast time-to-value.
- Python-heavy workloads where natural Python flow control is preferred.
- Want managed infrastructure without operating Airflow/Composer.
- Prefect Cloud offers a generous free tier for evaluation.

---

## Testing DAGs

### Unit Testing Operators

```python
import pytest
from airflow.models import DagBag

def test_dag_loads_without_errors():
    """Validate that all DAGs parse without import errors."""
    dag_bag = DagBag(dag_folder="/opt/airflow/dags", include_examples=False)
    assert len(dag_bag.import_errors) == 0, f"DAG import errors: {dag_bag.import_errors}"

def test_dag_has_expected_tasks():
    dag_bag = DagBag(dag_folder="/opt/airflow/dags", include_examples=False)
    dag = dag_bag.get_dag("daily_warehouse_refresh")
    task_ids = [t.task_id for t in dag.tasks]
    assert "extract_source_data" in task_ids
    assert "transform_data" in task_ids
    assert "load_to_warehouse" in task_ids

def test_dag_has_no_cycles():
    dag_bag = DagBag(dag_folder="/opt/airflow/dags", include_examples=False)
    for dag_id, dag in dag_bag.dags.items():
        assert dag.test_cycle() is False, f"Cycle detected in {dag_id}"
```

### CI/CD Pipeline for DAGs

The pattern visible in the logistics-analytics-platform workflows applies here too. See [[GitLab CI-CD for dbt]] for a comparable dbt-focused pipeline.

```
lint (sqlfluff + ruff) >> parse (dbt parse / DAG import) >> test (unit + integration) >> deploy (sync to GCS/S3)
```

In GitHub Actions:

```yaml
- name: Validate DAGs
  run: python -c "from airflow.models import DagBag; db=DagBag('.'); assert not db.import_errors"

- name: Run DAG unit tests
  run: pytest tests/dags/ -v

- name: Deploy to Composer
  if: github.ref == 'refs/heads/main'
  run: gsutil -m rsync -r -d ./dags gs://$COMPOSER_BUCKET/dags/
```

### Integration Tests

Test operators against real (or sandboxed) external systems.

```python
def test_snowflake_query_returns_data():
    hook = SnowflakeHook(snowflake_conn_id="snowflake_test")
    result = hook.get_first("SELECT COUNT(*) FROM raw.orders WHERE date = '2025-01-01'")
    assert result[0] > 0
```

---

## Best Practices

### Idempotent Tasks

Every task should produce the same result when run multiple times for the same logical date. Use `MERGE` or `DELETE + INSERT` rather than plain `INSERT`. See [[dbt Incremental Loading Patterns]] for the dbt perspective.

### No Side Effects in DAG Definition

DAG files are parsed repeatedly by the scheduler. Never put API calls, database queries, or heavy computation at the module level -- only inside operator callables.

```python
# BAD -- runs on every parse
data = requests.get("https://api.example.com/config").json()

# GOOD -- runs only when the task executes
@task
def fetch_config():
    return requests.get("https://api.example.com/config").json()
```

### XComs for Small Data Only

XComs (cross-communication) pass data between tasks via the metadata DB. Limit to metadata (file paths, row counts, status flags). Never pass DataFrames or large result sets -- write to object storage and pass the URI.

### External Triggers Over Polling

Use deferrable operators and the Triggerer to wait on external events (API completion, file arrival) without consuming worker slots. Prefer `ExternalTaskSensor(mode="reschedule")` over `mode="poke"`.

### Task-Level Retries with Exponential Backoff

```python
default_args = {
    "retries": 3,
    "retry_delay": timedelta(minutes=2),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(minutes=30),
}
```

### Observability

- Ship Airflow metrics to Prometheus/Grafana via StatsD. See [[Snowflake Cost Monitoring]] for analogous warehouse-level monitoring.
- Set SLAs on critical-path tasks.
- Use callbacks (`on_failure_callback`, `on_success_callback`) for Slack/PagerDuty alerts.
- Tag DAGs (`tags=["production", "warehouse"]`) for filtering in the UI.

### DAG Organisation

```
dags/
  ingestion/
    ingest_orders.py
    ingest_customers.py
  transformation/
    dbt_daily.py
    dbt_hourly.py
  reporting/
    weekly_kpis.py
  common/
    callbacks.py
    helpers.py
```

Keep one DAG per file. Use `common/` for shared utilities. Pin provider package versions in `requirements.txt`.

---

## See Also

- [[Jenkins Pipeline Patterns for Data Engineering]] -- CI/CD orchestration with Jenkins
- [[GitHub Actions for Python Projects]] -- GitHub-native CI/CD workflows
- [[GCP Data Services for Data Engineering]] -- Cloud Composer and GCP integration
- [[dbt Advanced Patterns & Cost Optimisation]] -- dbt patterns orchestrated by Airflow
- [[Core dbt Fundamentals]] -- dbt basics for DAG integration
- [[dbt Tag & Execution Strategy]] -- selective dbt execution from Airflow
- [[Kubernetes for Data Workloads]] -- KubernetesExecutor and KubernetesPodOperator
- [[Terraform for Data Infrastructure]] -- IaC for Composer/MWAA environments
- [[Docker & Container Patterns]] -- containerised task execution
- [[Snowflake SQL Pipeline Patterns]] -- Snowflake queries from Airflow operators
- [[dbt Testing & Data Quality]] -- testing patterns that integrate with orchestration
- [[Apache Airflow Deep Dive]] -- architecture internals, operators, XComs, production patterns
