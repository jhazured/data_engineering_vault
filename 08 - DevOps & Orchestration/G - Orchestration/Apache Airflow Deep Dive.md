tags: #airflow #orchestration #DAGs #pipelines #scheduling #data-engineering

# Apache Airflow Deep Dive

This note provides an in-depth treatment of Apache Airflow internals, drawing on practical patterns from *Data Engineering with Python* (Crickard, 2020) and production experience. For a broader orchestration overview including Dagster, Prefect, and comparisons, see [[Apache Airflow & Orchestration Patterns]].

---

## Airflow Architecture Internals

Apache Airflow was created at Airbnb as a workflow management platform. It comprises five core services that work together to schedule, execute, and monitor pipelines.

### Component Breakdown

| Component | Responsibility | Notes |
|---|---|---|
| **Web Server** | Flask-based UI for DAG monitoring, log viewing, manual triggering, and connection management | Default port 8080; configurable via `airflow webserver -p <port>` |
| **Scheduler** | Continuously parses DAG files, determines task readiness, submits work to the executor | Must be running for any scheduled execution; started with `airflow scheduler` |
| **Metadata Database** | Stores DAG state, task instance status, connection credentials, variables, XCom data | SQLite for development; PostgreSQL or MySQL for production |
| **Executor** | Determines how tasks are actually run (local process, Celery worker, K8s pod) | See [[Apache Airflow & Orchestration Patterns]] for executor comparison |
| **Triggerer** | (Airflow 2.x) Async event loop for deferrable operators -- frees worker slots while awaiting external events | Critical for sensor-heavy workloads |

### Installation and Initial Configuration

Airflow installs via pip with optional sub-packages that control which integrations are available:

```bash
# Set custom home directory (optional)
export AIRFLOW_HOME=/opt/airflow

# Install with provider extras
pip install 'apache-airflow[postgres,slack,celery]'

# Initialise the metadata database
airflow initdb

# Start the web server and scheduler
airflow webserver -p 8080
airflow scheduler
```

The `airflow.cfg` file controls all runtime behaviour. Key settings include:

- `dags_folder` -- the directory the scheduler scans for DAG files (default: `$AIRFLOW_HOME/dags`)
- `load_examples` -- set to `False` in production to remove sample DAGs
- `dag_dir_list_interval` -- how frequently (seconds) the scheduler re-scans the DAG directory

After editing `airflow.cfg`, restart the web server and run `airflow initdb` (or `airflow resetdb` for a full reset) to apply changes.

---

## DAG Structure and Definition

A DAG (Directed Acyclic Graph) is a Python module that defines tasks and their dependencies. The scheduler imports these modules and looks for `DAG` objects.

### Anatomy of a DAG File

Every DAG file follows a consistent structure:

```python
import datetime as dt
from datetime import timedelta
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator
import pandas as pd

# 1. Default arguments applied to all tasks
default_args = {
    'owner': 'data-engineering',
    'start_date': dt.datetime(2020, 3, 18),
    'retries': 1,
    'retry_delay': dt.timedelta(minutes=5),
}

# 2. DAG context manager
with DAG(
    'MyCSVDAG',
    default_args=default_args,
    schedule_interval=timedelta(minutes=5),
) as dag:

    # 3. Task definitions using operators
    print_starting = BashOperator(
        task_id='starting',
        bash_command='echo "Pipeline starting..."',
    )

    convert = PythonOperator(
        task_id='convertCSVtoJson',
        python_callable=csv_to_json,
    )

    # 4. Dependency chain
    print_starting >> convert
```

### Key Structural Rules

- **One DAG per file** is the recommended convention, though multiple DAGs per file are technically supported.
- **No side effects at module level** -- the scheduler parses DAG files repeatedly. API calls, database queries, or heavy computation at the top level will execute on every parse cycle, not just at task runtime.
- **`start_date` plus `schedule_interval`** determines the first actual run. A DAG with `start_date` of today and `schedule_interval` of daily will not run until tomorrow. This is a common source of confusion for newcomers.

### Default Arguments

The `default_args` dictionary is applied to every task in the DAG. Common parameters:

| Parameter | Purpose | Example |
|---|---|---|
| `owner` | Identifies the team or individual responsible | `'data-engineering'` |
| `start_date` | Earliest logical date for scheduling | `dt.datetime(2025, 1, 1)` |
| `retries` | Number of retry attempts on failure | `3` |
| `retry_delay` | Wait time between retries | `timedelta(minutes=5)` |
| `retry_exponential_backoff` | Progressively increase retry delay | `True` |
| `email_on_failure` | Send email notification on task failure | `True` |
| `email` | Recipient list for failure notifications | `['alerts@company.com']` |

---

## Operators In Depth

Operators define what a single task does. Airflow ships with many built-in operators and the provider ecosystem adds hundreds more.

### BashOperator

Executes shell commands. Useful for calling external scripts, file operations, and CI/CD integration.

```python
copy_file = BashOperator(
    task_id='copy_output',
    bash_command='cp /opt/data/output.csv /opt/data/archive/',
)
```

BashOperator supports Jinja templating for runtime values:

```python
extract = BashOperator(
    task_id='extract',
    bash_command='python /opt/etl/extract.py --date {{ ds }}',
)
```

Practical caution: if you use `mv` instead of `cp` and the pipeline re-runs, the source file will be missing. Always consider idempotency when choosing shell commands.

### PythonOperator

Calls a Python callable. The callable should be an importable, testable function -- not defined inline.

```python
def query_postgresql():
    import psycopg2 as db
    import pandas as pd
    conn = db.connect("dbname='warehouse' host='localhost' user='etl'")
    df = pd.read_sql("SELECT name, city FROM users", conn)
    df.to_csv('staged_data.csv')

get_data = PythonOperator(
    task_id='QueryPostgreSQL',
    python_callable=query_postgresql,
)
```

### Sensor Operators

Sensors wait for a condition to be met before allowing downstream tasks to proceed. They are essential for event-driven patterns.

| Sensor | Waits For |
|---|---|
| `FileSensor` | A file to appear at a specified path |
| `ExternalTaskSensor` | A task in another DAG to complete |
| `HttpSensor` | An HTTP endpoint to return a success status |
| `SqlSensor` | A SQL query to return a truthy result |
| `S3KeySensor` | An object to appear in an S3 bucket |

**Mode selection matters**: use `mode='reschedule'` rather than `mode='poke'` to release the worker slot between checks. With deferrable operators (Airflow 2.x), sensors can use the Triggerer component to avoid occupying any worker slot at all.

### Provider Operators

The Airflow provider ecosystem extends functionality to external systems:

| Operator | Provider Package | Purpose |
|---|---|---|
| `PostgresOperator` | `apache-airflow-providers-postgres` | Execute SQL against PostgreSQL |
| `SnowflakeOperator` | `apache-airflow-providers-snowflake` | Run queries on [[Snowflake SQL Pipeline Patterns|Snowflake]] |
| `SparkSubmitOperator` | `apache-airflow-providers-apache-spark` | Submit [[PySpark Production Engineering|Spark]] jobs |
| `KubernetesPodOperator` | `apache-airflow-providers-cncf-kubernetes` | Run containers in [[Kubernetes for Data Workloads|K8s]] |
| `DbtCloudRunJobOperator` | `apache-airflow-providers-dbt-cloud` | Trigger [[dbt Advanced Patterns & Cost Optimisation|dbt Cloud]] jobs |

---

## Task Dependencies and Flow Control

### Dependency Syntax

Airflow provides two equivalent approaches for setting task order:

```python
# Bit-shift operators (preferred, more readable)
extract >> transform >> load

# Method calls (equivalent)
extract.set_downstream(transform)
transform.set_downstream(load)
```

For fan-out and fan-in patterns:

```python
# Fan-out: one task feeds many
extract >> [validate_schema, validate_counts, validate_nulls]

# Fan-in: many tasks feed one
[validate_schema, validate_counts, validate_nulls] >> load
```

### BranchPythonOperator

Enables conditional execution paths at runtime:

```python
from airflow.operators.python import BranchPythonOperator

def choose_path(**context):
    if context['ds_nodash'] in holiday_dates:
        return 'skip_processing'
    return 'run_full_pipeline'

branch = BranchPythonOperator(
    task_id='branch',
    python_callable=choose_path,
)
```

After branching, downstream merge tasks should use `trigger_rule='none_failed'` to handle skipped branches correctly.

### Trigger Rules

| Rule | Behaviour |
|---|---|
| `all_success` | (Default) All upstream tasks succeeded |
| `all_failed` | All upstream tasks failed -- for error-handling paths |
| `one_success` | At least one upstream task succeeded |
| `none_failed` | No upstream task failed (skipped is acceptable) |
| `all_done` | All upstream tasks finished regardless of status -- for cleanup |

---

## Scheduling and Timing

### Schedule Interval Options

Airflow supports cron expressions, timedelta objects, and preset strings:

| Format | Expression | Meaning |
|---|---|---|
| Preset | `@once` | Run exactly once |
| Preset | `@hourly` | `0 * * * *` |
| Preset | `@daily` | `0 0 * * *` |
| Preset | `@weekly` | `0 0 * * 0` |
| Preset | `@monthly` | `0 0 1 * *` |
| Preset | `@yearly` | `0 0 1 1 *` |
| Cron | `0 6 * * *` | Daily at 06:00 UTC |
| Cron | `0 */2 * * *` | Every 2 hours |
| Timedelta | `timedelta(minutes=5)` | Every 5 minutes |

Cron format: `minute hour day_of_month month day_of_week`. The `@yearly` preset `0 0 1 1 *` means midnight on 1 January, any day of the week.

### Catchup and Backfill

- **`catchup=True`** (default): Airflow schedules runs for every missed interval between `start_date` and now. Use for idempotent historical reprocessing.
- **`catchup=False`**: Only the most recent interval is scheduled. Use for forward-looking pipelines.
- **CLI backfill**: `airflow dags backfill -s 2025-01-01 -e 2025-03-01 daily_refresh` triggers historical runs on demand.

### Data-Aware Scheduling (Airflow 2.4+)

Trigger DAGs based on dataset updates rather than time. See [[Apache Airflow & Orchestration Patterns]] for dataset examples.

---

## XComs (Cross-Communication)

XComs allow tasks to exchange small pieces of data via the metadata database.

### How XComs Work

A task pushes a value using `xcom_push`, and a downstream task retrieves it with `xcom_pull`. With the TaskFlow API, return values are automatically pushed as XComs.

```python
# Traditional approach
def extract(**context):
    row_count = run_extraction()
    context['ti'].xcom_push(key='row_count', value=row_count)

def validate(**context):
    count = context['ti'].xcom_pull(task_ids='extract', key='row_count')
    if count == 0:
        raise AirflowException("No rows extracted")
```

### XCom Best Practices

- **Small data only** -- XComs are stored in the metadata database. Limit to metadata: file paths, row counts, status flags, URIs.
- **Never pass DataFrames** or large result sets through XComs. Write to object storage (S3, GCS) and pass the URI instead.
- **Serialisation** -- XCom values must be JSON-serialisable by default. Custom XCom backends can extend this.

---

## Connections and Hooks

### Managing Connections

Connections store credentials for external systems. They can be managed via:

1. **Airflow UI** -- Admin > Connections
2. **CLI** -- `airflow connections add`
3. **Environment variables** -- `AIRFLOW_CONN_{CONN_ID}` format

```python
# Using a hook to access a connection
from airflow.providers.postgres.hooks.postgres import PostgresHook

def query_with_hook(**context):
    hook = PostgresHook(postgres_conn_id='warehouse_prod')
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM orders WHERE date = %s", (context['ds'],))
    return cursor.fetchone()[0]
```

### Secrets Backends

For production, store credentials in an external secrets manager rather than the Airflow metadata database. See [[Apache Airflow & Orchestration Patterns]] for backend configuration details.

---

## Error Handling and Validation

### Raising Exceptions in Tasks

When a task detects invalid data or a failed condition, raise `AirflowException` to mark the task as failed and trigger retry logic:

```python
from airflow.exceptions import AirflowException

def validate_data():
    from great_expectations import DataContext
    context = DataContext("/opt/pipeline/great_expectations")
    suite = context.get_expectation_suite("orders.validate")
    batch_kwargs = {
        "path": "/opt/pipeline/staged/orders.csv",
        "datasource": "files_datasource",
        "reader_method": "read_csv",
    }
    batch = context.get_batch(batch_kwargs, suite)
    results = context.run_validation_operator("action_list_operator", [batch])
    if not results["success"]:
        raise AirflowException("Validation Failed")
```

This integrates [[Data Quality & Testing Patterns|Great Expectations]] directly into the Airflow task graph, enabling automatic retries and failure notifications.

### Callbacks for Alerting

```python
def on_failure_alert(context):
    """Send Slack notification on task failure."""
    dag_id = context['dag'].dag_id
    task_id = context['task_instance'].task_id
    send_slack_message(f"FAILURE: {dag_id}.{task_id}")

default_args = {
    'on_failure_callback': on_failure_alert,
    'on_success_callback': None,
    'on_retry_callback': None,
}
```

### SLA Monitoring

Set `sla` on individual tasks to trigger alerts when execution exceeds expected duration:

```python
transform = PythonOperator(
    task_id='transform',
    python_callable=run_transform,
    sla=timedelta(hours=2),
)
```

---

## Production Pipeline Patterns

### Idempotency

A production pipeline must produce the same result regardless of how many times it runs for the same logical date. Strategies include:

- **Upsert operations** -- use `MERGE` or `INSERT ... ON CONFLICT UPDATE` in SQL; use the `upsert` index operation in Elasticsearch with a stable document ID.
- **Delete-and-reload** -- truncate the target partition, then insert fresh data. Simple but requires care with concurrent reads.
- **Immutable partitions** -- create a new partition or index (e.g., `orders_20250315`) on each run. Old partitions are never modified. This approach, advocated by functional data engineering, creates an immutable audit trail.

Without idempotency, accidental re-runs or retries after partial failures will produce duplicate records. See [[ETL Pipeline Templates & Patterns]] for implementation patterns.

### Atomicity

If a single operation in a transaction fails, all operations should fail. SQL databases provide this natively via transactions and rollback. NoSQL systems like Elasticsearch require application-level logic:

- Track every successfully indexed document
- If any failure occurs, delete all successfully indexed documents from that run
- Retry the entire batch

### Staging Pattern

Production pipelines benefit from staging data at both ends:

1. **Extract stage** -- write query results to a file before processing. If the pipeline fails midway, the staged file preserves the exact data snapshot without re-querying the source.
2. **Load stage** -- insert into a staging replica of the warehouse. Run validation suites against the replica. Only promote to production tables after validation passes.

Airflow naturally encourages staging because each task writes intermediate results to disk, and the next task reads from that file. This makes individual tasks independently retriable.

---

## Airflow Versus Apache NiFi

The source material uses both Airflow and NiFi extensively. They serve the same role -- pipeline orchestration -- but differ significantly in approach.

| Dimension | Apache Airflow | Apache NiFi |
|---|---|---|
| **Definition** | Python code (DAG files) | Visual drag-and-drop GUI with processors |
| **Origin** | Airbnb (2014) | National Security Agency (NSA) |
| **Accessibility** | Requires Python skills | Accessible to non-coders; configuration-driven |
| **Data handling** | Orchestrates external tools; does not move data itself | Moves data through flowfiles; processors transform in-stream |
| **Version control** | Standard Git on Python files | NiFi Registry (sub-project) with optional Git persistence |
| **Clustering** | Celery/K8s executors for distributed execution | Native clustering with automatic data distribution |
| **Backpressure** | Managed at executor level (pool slots, concurrency) | Built-in flowfile queue backpressure with configurable thresholds |
| **Edge computing** | Not supported | MiNiFi for IoT and low-resource devices |
| **Monitoring** | Web UI, StatsD/Prometheus metrics, log files | Built-in status bar, bulletin board, REST API, processor-level counters |
| **Best for** | Python-heavy teams; complex scheduling logic; cloud-managed deployments | Rapid prototyping; visual pipeline design; real-time streaming; heterogeneous team skill levels |

NiFi's monitoring capabilities are notably rich: the status bar shows real-time throughput, the bulletin board captures processor-level errors, and custom counters (via `UpdateCounter` processor) track domain-specific metrics. The NiFi REST API exposes all of this programmatically, enabling custom dashboards built with Python's `requests` library. NiFi also integrates with Slack via the `PutSlack` processor for failure alerting.

---

## Monitoring and Observability

### Built-In Airflow Monitoring

- **Web UI** -- Tree View and Graph View show task status per run; click any task to view logs.
- **Task logs** -- each task instance produces a log file viewable through the UI or on disk.
- **SLA misses** -- configurable alerts when tasks exceed expected duration.

### External Monitoring Integration

- **StatsD/Prometheus** -- Airflow emits metrics (task duration, success/failure counts, scheduler heartbeat) that can be scraped by Prometheus and visualised in Grafana.
- **Callbacks** -- `on_failure_callback`, `on_success_callback`, and `on_retry_callback` for Slack, PagerDuty, or email integration.
- **Tags** -- `tags=['production', 'warehouse']` for filtering DAGs in the UI.

### NiFi Monitoring (For Comparison)

NiFi provides a complementary monitoring model through:

- **Status bar** -- live bytes in/out, active threads, queue depth per connection
- **Bulletin board** -- aggregated error messages across all processors
- **Counters** -- custom `UpdateCounter` processors to track domain events (e.g., records processed, records failed)
- **REST API** -- programmatic access to system diagnostics, processor status, queue contents, and reporting tasks
- **PutSlack processor** -- inline alerting on failure relationships using NiFi Expression Language: `${id:append(': Record failed Upsert Elasticsearch')}`

---

## Testing DAGs

### Parse Validation

Validate that all DAGs load without import errors -- this is the minimum CI gate:

```python
from airflow.models import DagBag

def test_dag_loads_without_errors():
    dag_bag = DagBag(dag_folder="/opt/airflow/dags", include_examples=False)
    assert len(dag_bag.import_errors) == 0, f"Import errors: {dag_bag.import_errors}"
```

### Structural Tests

Verify expected tasks exist and no cycles are present:

```python
def test_expected_tasks():
    dag_bag = DagBag(dag_folder="/opt/airflow/dags", include_examples=False)
    dag = dag_bag.get_dag("daily_warehouse_refresh")
    task_ids = [t.task_id for t in dag.tasks]
    assert "extract_source_data" in task_ids
    assert "transform_data" in task_ids
```

### Integration Tests

Test operators against real or sandboxed external systems to verify connectivity and query correctness. See [[Apache Airflow & Orchestration Patterns]] for CI/CD pipeline patterns.

---

## Best Practices Summary

### DAG Design

- Keep one DAG per file for clarity and parse performance.
- Keep DAG parse time under 30 seconds; avoid heavy imports at module level.
- Use `catchup=False` unless historical reprocessing is explicitly needed.
- Use TaskGroups (not SubDAGs) for logical grouping.

### Task Design

- **Atomic tasks** -- each task should stand alone. If a function reads a database and inserts results, split it into two tasks. When it fails, you know immediately whether the read or the write caused the failure.
- **Idempotent tasks** -- every task must produce the same result when run multiple times for the same logical date.
- **Stage intermediate results** -- write task outputs to files or staging tables. This enables independent retry of each task and preserves data snapshots.

### File Handling Cautions

- Be careful with file operations: if multiple tasks or processes touch the same file concurrently, the pipeline may break.
- Use `cp` rather than `mv` in BashOperator to preserve source files for reruns.
- Ensure file paths are accessible from the Airflow worker process.

### XCom Discipline

- Small metadata only (paths, counts, flags).
- Never serialise DataFrames through XComs.
- Use object storage for intermediate datasets.

### Secrets Management

- Never store credentials in DAG code or `airflow.cfg`.
- Use a secrets backend (AWS SSM, GCP Secret Manager, HashiCorp Vault) in production.
- Manage connections via environment variables (`AIRFLOW_CONN_{ID}`) in containerised deployments.

---

## See Also

- [[Apache Airflow & Orchestration Patterns]] -- orchestration overview, executor types, Dagster/Prefect comparison, Cloud Composer/MWAA
- [[ETL Pipeline Templates & Patterns]] -- reusable pipeline architectures
- [[Data Ingestion Patterns]] -- source extraction strategies
- [[Data Quality & Testing Patterns]] -- Great Expectations integration
- [[Docker & Container Patterns]] -- containerised Airflow deployments
- [[Kubernetes for Data Workloads]] -- KubernetesExecutor and KubernetesPodOperator
- [[Jenkins Pipeline Patterns for Data Engineering]] -- CI/CD for DAG deployment
- [[GitHub Actions for Python Projects]] -- automated DAG validation and deployment
