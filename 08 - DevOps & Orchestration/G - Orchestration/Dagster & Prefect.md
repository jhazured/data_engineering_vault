tags: #dagster #prefect #orchestration #pipelines #scheduling #data-engineering #assets #workflows

# Dagster & Prefect

This note provides a comprehensive treatment of Dagster and Prefect as modern alternatives to [[Apache Airflow & Orchestration Patterns|Apache Airflow]]. Both represent a generational shift in orchestration philosophy -- Dagster towards asset-centric declarative pipelines, Prefect towards Pythonic simplicity and dynamic workflows. For Airflow internals, operators, and production patterns, see [[Apache Airflow Deep Dive]].

---

## Dagster

### Philosophy: Software-Defined Assets

Dagster's core innovation is the **asset graph** -- a departure from Airflow's task graph. Rather than defining "what to do" (tasks), you define "what should exist" (assets). The execution plan is derived from the dependency relationships between assets.

| Paradigm | Focus | Example |
|---|---|---|
| **Task graph** (Airflow) | Steps to execute | "Run extract, then transform, then load" |
| **Asset graph** (Dagster) | Data artefacts to materialise | "I need `clean_orders`, which depends on `raw_orders`" |

This shift provides several advantages:

- **Lineage is automatic** -- dependencies between data artefacts are visible without separate cataloguing tools.
- **Selective materialisation** -- materialise only the assets that are stale or requested, rather than running an entire pipeline.
- **Observability by default** -- every materialisation is tracked with metadata, partition information, and upstream provenance.

### Core Concepts

#### Assets

An asset is a persistent data artefact (a table, a file, a model) defined as a decorated Python function:

```python
from dagster import asset, AssetExecutionContext

@asset(
    description="Raw orders extracted from the source API.",
    group_name="ingestion",
)
def raw_orders(context: AssetExecutionContext):
    """Extract orders from source system."""
    data = extract_from_api("orders")
    context.log.info(f"Extracted {len(data)} orders")
    return data

@asset(
    description="Cleaned and validated orders.",
    group_name="transformation",
)
def clean_orders(raw_orders):
    """Apply data quality rules to raw orders."""
    return (
        raw_orders
        .dropna(subset=["order_id", "customer_id"])
        .drop_duplicates(subset=["order_id"])
    )

@asset(
    description="Daily order aggregates by region.",
    group_name="analytics",
)
def orders_summary(clean_orders):
    """Aggregate orders by region and day."""
    return clean_orders.groupby(["region", "order_date"]).agg(
        total_orders=("order_id", "count"),
        total_revenue=("amount", "sum"),
    ).reset_index()
```

Dependencies are inferred from function parameter names. Dagster builds the asset graph automatically.

#### Ops, Jobs, and Graphs

Dagster also supports a lower-level abstraction for cases where the asset model does not fit:

| Concept | Role | When to Use |
|---|---|---|
| **Op** | A single unit of computation (analogous to an Airflow operator) | Fine-grained control over execution steps |
| **Graph** | A composition of ops with explicit dependency wiring | Reusable sub-pipelines |
| **Job** | An executable instance of a graph, bound to resources and config | Scheduling and triggering |
| **Asset** | A declarative data artefact (built on ops internally) | Most use cases -- prefer assets by default |

```python
from dagster import op, job, graph, In, Out

@op(out=Out(int))
def extract_count():
    return query_source_count()

@op(ins={"count": In(int)})
def validate_count(count):
    if count == 0:
        raise Exception("No records extracted")

@op(ins={"count": In(int)})
def log_count(count):
    print(f"Extracted {count} records")

@graph
def extraction_validation():
    count = extract_count()
    validate_count(count)
    log_count(count)

# Bind to resources and configuration
extraction_job = extraction_validation.to_job(
    name="extraction_validation_job",
    description="Extract and validate source data.",
)
```

In practice, the asset API covers most orchestration needs. Ops and graphs are useful for utility operations that do not produce a persistent data artefact (e.g., sending notifications, clearing caches).

### Dagster UI (Dagit)

The Dagster UI (historically called Dagit) provides a rich development and monitoring experience:

| Feature | Description |
|---|---|
| **Asset Graph** | Interactive visualisation of all assets and their dependencies. Click any asset to see materialisation history, metadata, and upstream lineage. |
| **Asset Catalogue** | Searchable inventory of every asset, grouped by code location. Includes descriptions, types, and partition status. |
| **Run History** | Timeline of all job and asset materialisation runs with logs, duration, and status. |
| **Launchpad** | Configure and launch ad-hoc runs with typed configuration forms. |
| **Sensor and Schedule Monitor** | View tick history, evaluation results, and skip reasons for sensors and schedules. |
| **Global Asset Lineage** | Trace data provenance across the entire asset graph -- upstream sources through to downstream consumers. |

Unlike Airflow's web server (which is primarily a monitoring tool), Dagit is designed as a development environment. Engineers can test materialisation, inspect intermediate outputs, and debug failures without leaving the browser.

### IO Managers

IO Managers abstract the storage and retrieval of asset values. They separate "what to compute" from "where to store it", enabling environment-specific storage without changing asset code.

```python
from dagster import asset, Definitions, IOManager, io_manager
import pandas as pd

class LocalParquetIOManager(IOManager):
    def __init__(self, base_path: str):
        self.base_path = base_path

    def handle_output(self, context, obj: pd.DataFrame):
        path = f"{self.base_path}/{context.asset_key.path[-1]}.parquet"
        obj.to_parquet(path)
        context.log.info(f"Wrote {len(obj)} rows to {path}")

    def load_input(self, context) -> pd.DataFrame:
        path = f"{self.base_path}/{context.asset_key.path[-1]}.parquet"
        return pd.read_parquet(path)

@io_manager
def local_parquet_io_manager():
    return LocalParquetIOManager(base_path="/tmp/dagster_data")
```

Built-in and community IO managers cover common storage targets:

| IO Manager | Package | Target |
|---|---|---|
| `SnowflakeIOManager` | `dagster-snowflake-pandas` | [[Snowflake SQL Pipeline Patterns|Snowflake]] tables |
| `BigQueryIOManager` | `dagster-gcp-pandas` | [[GCP Data Services for Data Engineering|BigQuery]] tables |
| `S3PickleIOManager` | `dagster-aws` | S3 objects (pickle serialisation) |
| `DuckDBIOManager` | `dagster-duckdb-pandas` | DuckDB tables |
| `FilesystemIOManager` | `dagster` (built-in) | Local filesystem (default) |

Switching between local development and production is a matter of swapping the IO manager in the `Definitions` object:

```python
from dagster import Definitions

defs = Definitions(
    assets=[raw_orders, clean_orders, orders_summary],
    resources={
        "io_manager": local_parquet_io_manager,          # Dev
        # "io_manager": snowflake_io_manager,            # Prod
    },
)
```

### Resources and Configuration

Resources are shared dependencies (database connections, API clients, configuration) injected into assets and ops. Dagster provides typed configuration via Pydantic-style schemas.

```python
from dagster import asset, ConfigurableResource

class SnowflakeConfig(ConfigurableResource):
    account: str
    user: str
    password: str
    database: str
    schema_name: str
    warehouse: str

@asset
def raw_orders(snowflake: SnowflakeConfig):
    conn = connect_snowflake(
        account=snowflake.account,
        user=snowflake.user,
        password=snowflake.password,
        database=snowflake.database,
        schema=snowflake.schema_name,
        warehouse=snowflake.warehouse,
    )
    return pd.read_sql("SELECT * FROM raw.orders", conn)
```

Environment-specific configuration is managed through `Definitions`:

```python
# Development
dev_defs = Definitions(
    assets=[raw_orders, clean_orders],
    resources={
        "snowflake": SnowflakeConfig(
            account="dev-account",
            user="dev_user",
            password=EnvVar("SNOWFLAKE_DEV_PASSWORD"),
            database="DEV_DB",
            schema_name="RAW",
            warehouse="DEV_WH",
        ),
    },
)
```

### Partitions and Backfills

Partitions divide assets into discrete segments (typically by date) for incremental processing. Dagster tracks which partitions have been materialised and enables selective backfills.

```python
from dagster import asset, DailyPartitionsDefinition

daily_partitions = DailyPartitionsDefinition(start_date="2025-01-01")

@asset(partitions_def=daily_partitions)
def daily_orders(context):
    partition_date = context.partition_key
    return extract_orders_for_date(partition_date)

@asset(partitions_def=daily_partitions)
def daily_order_metrics(daily_orders):
    return compute_metrics(daily_orders)
```

Built-in partition types:

| Type | Use Case |
|---|---|
| `DailyPartitionsDefinition` | One partition per calendar day |
| `HourlyPartitionsDefinition` | One partition per hour |
| `WeeklyPartitionsDefinition` | One partition per week |
| `MonthlyPartitionsDefinition` | One partition per month |
| `StaticPartitionsDefinition` | Fixed set of named partitions (e.g., regions, categories) |
| `MultiPartitionsDefinition` | Combination of multiple partition dimensions (e.g., date x region) |

Backfills are launched from the UI or CLI -- select a range of partitions and Dagster materialises them in dependency order, respecting concurrency limits. This is considerably more ergonomic than Airflow's `airflow dags backfill` CLI command. See [[Apache Airflow & Orchestration Patterns]] for the Airflow equivalent.

### Sensors and Schedules

```python
from dagster import schedule, sensor, RunRequest, SkipReason

# Time-based schedule
@schedule(cron_schedule="0 6 * * *", job=daily_refresh_job)
def daily_refresh_schedule():
    return RunRequest()

# Event-driven sensor
@sensor(job=process_new_files_job, minimum_interval_seconds=30)
def new_file_sensor(context):
    new_files = check_for_new_files("/data/incoming/")
    if not new_files:
        yield SkipReason("No new files found")
        return
    for file_path in new_files:
        yield RunRequest(
            run_key=file_path,
            run_config={"ops": {"process_file": {"config": {"path": file_path}}}},
        )
```

Sensors can also trigger asset materialisations based on external events (file arrivals, API webhooks, database changes), providing event-driven orchestration without polling from within an asset.

### Testing

Dagster is designed for testability. Assets and ops are ordinary Python functions that can be unit-tested without a running Dagster instance.

```python
import pytest
from dagster import materialize_to_memory, build_asset_context

def test_clean_orders_removes_nulls():
    """Unit test an asset function directly."""
    raw = pd.DataFrame({
        "order_id": [1, 2, None, 4],
        "customer_id": [10, None, 30, 40],
        "amount": [100, 200, 300, 400],
    })
    result = clean_orders(raw)
    assert result["order_id"].notna().all()
    assert result["customer_id"].notna().all()
    assert len(result) == 2  # Only rows 1 and 4 survive

def test_asset_materialisation():
    """Integration test using in-process execution."""
    result = materialize_to_memory(
        assets=[raw_orders, clean_orders, orders_summary],
        resources={"snowflake": mock_snowflake_resource},
    )
    assert result.success
    output = result.output_for_node("orders_summary")
    assert "total_revenue" in output.columns

def test_asset_with_context():
    """Test an asset that uses context."""
    context = build_asset_context(partition_key="2025-03-15")
    result = daily_orders(context)
    assert len(result) > 0
```

This testability is a significant advantage over Airflow, where testing individual operators in isolation requires substantial boilerplate. See [[Apache Airflow Deep Dive]] for Airflow testing patterns.

### dbt Integration (dagster-dbt)

The `dagster-dbt` package maps dbt models to Dagster assets automatically, providing unified lineage across Python and SQL transformations. See [[Core dbt Fundamentals]] and [[dbt Advanced Patterns & Cost Optimisation]] for dbt concepts.

```python
from dagster_dbt import DbtCliResource, dbt_assets
from pathlib import Path

dbt_project_dir = Path("/opt/dbt/my_project")

@dbt_assets(manifest=dbt_project_dir / "target" / "manifest.json")
def my_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()

defs = Definitions(
    assets=[my_dbt_assets, raw_orders, clean_orders],
    resources={
        "dbt": DbtCliResource(project_dir=dbt_project_dir),
    },
)
```

Each dbt model appears as a Dagster asset with full lineage -- upstream Python assets feed into dbt sources, and dbt models feed into downstream Python assets. This creates a unified asset graph across both paradigms, with [[dbt Tag & Execution Strategy|dbt tags]] preserved as Dagster metadata.

### Deployment

| Option | Description | Best For |
|---|---|---|
| **Dagster Cloud (Serverless)** | Fully managed. No infrastructure to operate. Pay per materialisation. | Small-to-medium teams wanting zero-ops |
| **Dagster Cloud (Hybrid)** | Control plane managed by Dagster; compute runs in your infrastructure (K8s, ECS) | Enterprises with data residency requirements |
| **Self-Hosted (K8s)** | Helm chart deploys Dagit, daemon, and user code servers to [[Kubernetes for Data Workloads|Kubernetes]] | Teams with existing K8s infrastructure |
| **Self-Hosted (Docker)** | Docker Compose for smaller deployments. See [[Docker & Container Patterns]]. | Development and small production workloads |
| **Local** | Single-process execution via `dagster dev` | Development and testing |

Production self-hosted deployments typically separate **user code servers** (containing asset/op definitions) from the **Dagster daemon** (responsible for scheduling, sensors, and run queuing) and the **Dagit web server**. This allows code deployments without restarting core infrastructure.

---

## Prefect

### Philosophy: Pythonic Workflows

Prefect positions itself as "workflow orchestration done right" -- workflows are native Python with decorators, not configuration files or DSL constructs. Any Python function can become a flow or task with a single decorator.

### Flows and Tasks

```python
from prefect import flow, task
from prefect.logging import get_run_logger

@task(retries=3, retry_delay_seconds=60, log_prints=True)
def extract_data(source: str) -> pd.DataFrame:
    logger = get_run_logger()
    logger.info(f"Extracting from {source}")
    return run_extraction(source)

@task(retries=2, retry_delay_seconds=30)
def transform_data(raw_data: pd.DataFrame) -> pd.DataFrame:
    return apply_transforms(raw_data)

@task
def load_data(data: pd.DataFrame, target: str):
    write_to_warehouse(data, target)

@flow(name="daily-etl", log_prints=True)
def daily_etl(source: str = "orders", target: str = "warehouse.orders"):
    raw = extract_data(source)
    transformed = transform_data(raw)
    load_data(transformed, target)
```

Key characteristics:

- **Flows** are the top-level orchestration unit (analogous to an Airflow DAG or Dagster job). They can call tasks and other flows.
- **Tasks** are individual units of work within a flow. They get retry logic, caching, concurrency limits, and observability automatically.
- **No DAG compilation step** -- the execution order is determined by Python's natural control flow (if/else, for loops, try/except).
- **Type hints** are respected and validated at runtime.

### Task Runners

Task runners control how tasks within a flow are executed:

```python
from prefect import flow, task
from prefect.task_runners import ConcurrentTaskRunner
from prefect_dask import DaskTaskRunner
from prefect_ray import RayTaskRunner

@flow(task_runner=ConcurrentTaskRunner())
def parallel_etl():
    # Tasks submitted with .submit() run concurrently
    futures = [extract_data.submit(source) for source in sources]
    results = [f.result() for f in futures]
    return results
```

| Task Runner | Execution Model | Use Case |
|---|---|---|
| `SequentialTaskRunner` | One task at a time (default) | Simple pipelines, debugging |
| `ConcurrentTaskRunner` | Async concurrency via `asyncio` | I/O-bound tasks (API calls, database queries) |
| `DaskTaskRunner` | Distributed via Dask cluster | CPU-bound parallel processing |
| `RayTaskRunner` | Distributed via Ray cluster | ML workloads, large-scale parallelism |

### Prefect Cloud vs Self-Hosted Server

| Feature | Prefect Cloud | Self-Hosted Server |
|---|---|---|
| **Hosting** | Managed SaaS | Docker Compose or [[Kubernetes for Data Workloads|K8s]] |
| **UI** | Full-featured dashboard with RBAC | Open-source UI (subset of features) |
| **Authentication** | Built-in with SSO, API keys, service accounts | Community-managed |
| **Automations** | Event-driven triggers, notifications, webhooks | Limited to API-based triggers |
| **Work Pools** | Managed infrastructure pools (K8s, ECS, Cloud Run) | Self-managed work pools |
| **Audit Logs** | Full audit trail | Not available |
| **Cost** | Free tier available; paid plans for teams | Infrastructure costs only |

### Deployments and Work Pools

Deployments define how and where a flow runs in production. Work pools provide the compute infrastructure.

```yaml
# prefect.yaml
deployments:
  - name: daily-etl-prod
    entrypoint: flows/etl.py:daily_etl
    work_pool:
      name: k8s-prod
      job_variables:
        image: registry.company.com/etl:latest
        cpu: "2"
        memory: "4Gi"
    schedule:
      cron: "0 6 * * *"
      timezone: "UTC"
    parameters:
      source: "orders"
      target: "prod.warehouse.orders"
    tags:
      - production
      - warehouse
```

```bash
# Deploy from CLI
prefect deploy --all

# Or deploy a specific flow
prefect deploy flows/etl.py:daily_etl -n daily-etl-prod
```

Work pool types include:

| Pool Type | Infrastructure | Best For |
|---|---|---|
| **Process** | Local subprocesses | Development |
| **Kubernetes** | K8s pods via a worker | Production workloads on K8s |
| **Docker** | Docker containers | Isolated execution without K8s |
| **ECS** | AWS ECS/Fargate tasks | AWS-native deployments |
| **Cloud Run** | GCP Cloud Run jobs | [[GCP Data Services for Data Engineering|GCP]]-native deployments |
| **Azure Container Instances** | ACI containers | Azure-native deployments |

### Blocks

Blocks are typed, reusable configuration objects for connecting to external systems. They are stored centrally (in Prefect Cloud or the self-hosted database) and can be shared across flows.

```python
from prefect_snowflake import SnowflakeConnector

# Load a pre-configured block
snowflake = SnowflakeConnector.load("prod-snowflake")

@task
def query_snowflake(query: str) -> pd.DataFrame:
    with snowflake.get_connection() as conn:
        return pd.read_sql(query, conn)
```

Common block types:

| Block | Purpose |
|---|---|
| `SnowflakeConnector` | [[Snowflake SQL Pipeline Patterns|Snowflake]] connections |
| `GcsBucket` | GCS storage operations |
| `S3Bucket` | S3 storage operations |
| `SlackWebhook` | Slack notifications |
| `Email` | Email notifications |
| `Secret` | Encrypted secret storage |
| `JSON` | Arbitrary JSON configuration |

Blocks can be created via the UI, CLI, or Python code. They serve a similar role to Airflow Connections but with stronger typing and versioning.

### State Handlers and Retries

Prefect provides granular control over task state transitions:

```python
from prefect import task, flow
from prefect.states import Failed, Completed

@task(
    retries=3,
    retry_delay_seconds=[60, 120, 300],   # Progressive backoff
    retry_jitter_factor=0.5,               # Add randomness to avoid thundering herd
)
def flaky_api_call(endpoint: str):
    response = requests.get(endpoint)
    response.raise_for_status()
    return response.json()

@flow
def resilient_pipeline():
    result = flaky_api_call("https://api.vendor.com/data")
    if result is None:
        return Failed(message="API returned no data")
    return Completed(message=f"Processed {len(result)} records")
```

State handlers can also be attached to flows for custom failure logic:

```python
from prefect import flow
from prefect.states import State

def notify_on_failure(flow, flow_run, state: State):
    if state.is_failed():
        send_slack_alert(f"Flow {flow.name} failed: {state.message}")

@flow(on_failure=[notify_on_failure])
def monitored_pipeline():
    extract_data()
    transform_data()
```

### Caching and Result Persistence

Prefect caches task results to avoid redundant computation:

```python
from prefect import task
from prefect.tasks import task_input_hash
from datetime import timedelta

@task(
    cache_key_fn=task_input_hash,
    cache_expiration=timedelta(hours=1),
    result_storage_key="{flow_run.name}/{task_run.name}.json",
    persist_result=True,
)
def expensive_computation(data: pd.DataFrame) -> pd.DataFrame:
    return run_heavy_transform(data)
```

- **`cache_key_fn=task_input_hash`** -- cache is keyed on the hash of input arguments. Same inputs produce a cache hit.
- **`cache_expiration`** -- time after which the cached result is invalidated.
- **`persist_result`** -- store results durably (in the configured result storage) rather than only in memory.

### Subflows and Flow Composition

Flows can call other flows, creating a hierarchical orchestration structure:

```python
@flow(name="extract-sources")
def extract_all_sources():
    orders = extract_data("orders")
    customers = extract_data("customers")
    products = extract_data("products")
    return {"orders": orders, "customers": customers, "products": products}

@flow(name="transform-and-load")
def transform_and_load(datasets: dict):
    for name, data in datasets.items():
        transformed = transform_data(data)
        load_data(transformed, f"warehouse.{name}")

@flow(name="master-pipeline")
def master_pipeline():
    datasets = extract_all_sources()       # Subflow call
    transform_and_load(datasets)           # Subflow call
```

Subflows appear as nested runs in the Prefect UI, with independent state tracking and retry logic. This is more flexible than Airflow's deprecated SubDAGs and more composable than TaskGroups.

### dbt Integration (prefect-dbt)

The `prefect-dbt` package provides tasks for running dbt commands within Prefect flows. See [[Core dbt Fundamentals]] for dbt concepts.

```python
from prefect import flow
from prefect_dbt.cli.commands import DbtCoreOperation

@flow(name="dbt-daily-run")
def dbt_daily():
    # Run dbt build (run + test)
    result = DbtCoreOperation(
        commands=["dbt build --select tag:daily"],
        project_dir="/opt/dbt/my_project",
        profiles_dir="/opt/dbt",
    )
    result.run()

@flow(name="dbt-with-prefect-tasks")
def dbt_with_custom_logic():
    # Pre-dbt validation
    validate_sources()

    # dbt run
    DbtCoreOperation(
        commands=["dbt run --select +orders_mart"],
        project_dir="/opt/dbt/my_project",
    ).run()

    # Post-dbt quality checks
    run_quality_checks()
```

For dbt Cloud users, `prefect-dbt` also supports triggering dbt Cloud jobs via the API, similar to Airflow's `DbtCloudRunJobOperator`. See [[dbt Tag & Execution Strategy]] for selective execution patterns.

### Webhooks and Automations

Prefect Cloud provides event-driven automations that respond to flow run state changes, work pool events, and external webhooks:

```python
# Trigger a flow via webhook (Prefect Cloud)
# POST https://api.prefect.cloud/hooks/<webhook-id>
# Body: {"source": "github", "event": "push", "branch": "main"}

@flow
def webhook_triggered_pipeline(source: str, event: str, branch: str):
    if branch == "main" and event == "push":
        run_deployment_pipeline()
```

Automation triggers available in Prefect Cloud:

| Trigger | Example |
|---|---|
| **Flow run state change** | Notify Slack when any production flow fails |
| **Work pool status** | Alert when a work pool has no healthy workers |
| **Deployment schedule** | Pause deployments during maintenance windows |
| **Webhook** | Trigger a flow when a GitHub push event fires |
| **Metric threshold** | Alert when flow duration exceeds historical average by 2x |

Actions include sending notifications (Slack, email, PagerDuty), pausing/resuming deployments, cancelling runs, and calling external webhooks.

---

## Comparison: Airflow vs Dagster vs Prefect

### Feature Comparison

| Dimension | Apache Airflow | Dagster | Prefect |
|---|---|---|---|
| **Core paradigm** | Task graph (imperative DAGs) | Asset graph (declarative assets) | Flow/task decorators (Pythonic) |
| **Workflow definition** | Python DAG files with operators | Python functions with `@asset` / `@op` | Python functions with `@flow` / `@task` |
| **Dependency model** | Explicit `>>` operator chains | Inferred from function signatures | Implicit from Python control flow |
| **Dynamic workflows** | Dynamic task mapping (2.3+) | Dynamic partitions, asset factories | Native Python loops and conditionals |
| **Data awareness** | Datasets (2.4+) -- limited | First-class asset lineage and metadata | Result persistence, caching |
| **Testing** | DagBag import tests, integration tests | In-process materialisation, direct function calls | Direct function calls, mock task runners |
| **UI** | Monitoring-focused (Graph View, logs) | Development + monitoring (asset lineage, catalogue) | Monitoring-focused (flow runs, radar) |
| **Configuration** | Connections, Variables, `airflow.cfg` | Typed resources, `ConfigurableResource` (Pydantic) | Blocks, variables, profiles |
| **Scheduling** | Cron, timetables, datasets | Schedules, sensors, freshness policies | Cron schedules, event triggers, automations |
| **Retry logic** | Task-level retries with exponential backoff | Op/asset-level retries | Task-level retries with progressive backoff and jitter |
| **dbt integration** | `BashOperator`, `DbtCloudRunJobOperator` | `dagster-dbt` (model-to-asset mapping) | `prefect-dbt` (CLI and Cloud operations) |
| **Managed offering** | Cloud Composer (GCP), MWAA (AWS) | Dagster Cloud (serverless/hybrid) | Prefect Cloud (free tier available) |
| **Self-hosted** | Mature; extensive community support | Helm chart, Docker Compose | Docker Compose, K8s |
| **Community size** | Largest (est. 2014, CNCF graduated) | Growing rapidly (est. 2018) | Significant (est. 2018, Prefect 2.0 in 2022) |
| **Learning curve** | Moderate -- many concepts, provider ecosystem | Moderate -- asset model requires mental shift | Low -- natural Python, minimal boilerplate |
| **Backfill support** | CLI-based (`airflow dags backfill`) | UI-driven partition backfills | Re-run via UI or API (less structured) |
| **Plugin ecosystem** | 80+ provider packages | Growing; `dagster-dbt`, `dagster-k8s`, etc. | Growing; `prefect-dbt`, `prefect-dask`, etc. |
| **Secrets management** | Secrets backends (Vault, SSM, GCP SM) | Environment variables, `EnvVar` in resources | Blocks (`Secret`), environment variables |

### When to Choose Each

**Choose [[Apache Airflow & Orchestration Patterns|Airflow]] when:**

- You have an existing Airflow investment with many DAGs and trained operators.
- Your organisation uses a managed service (Cloud Composer, MWAA) and values vendor support.
- You need the largest ecosystem of pre-built operators and provider packages.
- Your team is comfortable with the DAG-as-code paradigm and does not need asset lineage.

**Choose Dagster when:**

- You are starting a greenfield data platform and want asset-centric orchestration.
- Data lineage and cataloguing are first-class requirements (reducing the need for a separate data catalogue).
- You have a significant dbt investment and want unified lineage across Python and SQL -- `dagster-dbt` is the most mature integration.
- Your team values testability and wants to unit-test individual assets without infrastructure.
- You need partition-aware backfills with a visual interface.

**Choose Prefect when:**

- Your team prioritises Pythonic simplicity and fast time-to-value.
- Workflows are highly dynamic -- lots of conditional logic, loops, and runtime decisions.
- You want managed orchestration without operating infrastructure (Prefect Cloud's free tier is generous).
- You need distributed task execution via Dask or Ray for compute-heavy workloads.
- Your pipelines are Python-heavy and do not benefit from the asset abstraction.

### Migration Patterns

#### Airflow to Dagster

1. **Map DAGs to asset groups** -- each Airflow DAG typically corresponds to a group of related Dagster assets.
2. **Convert operators to assets** -- `PythonOperator` callables become `@asset` functions. `BashOperator` commands become `@op` functions wrapping `subprocess`.
3. **Replace Connections with Resources** -- Airflow Connections map to Dagster `ConfigurableResource` instances with typed configuration.
4. **Replace XComs with IO Managers** -- data passed between tasks via XComs is naturally handled by the asset dependency graph and IO managers.
5. **Migrate schedules** -- Airflow cron schedules map directly to Dagster `@schedule` decorators.
6. **Run in parallel** -- Dagster can coexist with Airflow during migration. Use Airflow to trigger Dagster jobs via the Dagster API during the transition period.

```python
# Before (Airflow)
extract = PythonOperator(task_id="extract", python_callable=extract_orders)
transform = PythonOperator(task_id="transform", python_callable=transform_orders)
extract >> transform

# After (Dagster)
@asset
def raw_orders():
    return extract_orders()

@asset
def clean_orders(raw_orders):
    return transform_orders(raw_orders)
```

#### Airflow to Prefect

1. **Convert DAGs to flows** -- each Airflow DAG becomes a `@flow`-decorated function.
2. **Convert operators to tasks** -- `PythonOperator` callables become `@task` functions. Replace `BashOperator` with `subprocess.run()` inside a task.
3. **Replace Connections with Blocks** -- Airflow Connections map to Prefect Blocks (e.g., `SnowflakeConnector`, `S3Bucket`).
4. **Replace XComs with return values** -- tasks return values directly; Prefect handles serialisation and result persistence.
5. **Replace sensors with automations** -- Airflow sensors become Prefect automations or polling tasks.
6. **Deploy incrementally** -- migrate one DAG at a time. Use Prefect's webhook triggers to integrate with remaining Airflow DAGs during transition.

```python
# Before (Airflow)
with DAG("daily_etl", schedule="0 6 * * *") as dag:
    extract = PythonOperator(task_id="extract", python_callable=extract_fn)
    transform = PythonOperator(task_id="transform", python_callable=transform_fn)
    load = PythonOperator(task_id="load", python_callable=load_fn)
    extract >> transform >> load

# After (Prefect)
@flow(name="daily-etl")
def daily_etl():
    raw = extract_fn()
    transformed = transform_fn(raw)
    load_fn(transformed)
```

---

## Deployment Architecture Patterns

### Dagster on Kubernetes

```
                    ┌─────────────────────┐
                    │    Dagster Daemon    │
                    │  (schedules, sensors,│
                    │   run coordination)  │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───┐  ┌──────▼─────┐  ┌─────▼──────┐
     │  Dagit UI  │  │  User Code │  │  User Code  │
     │  (web)     │  │  Server A  │  │  Server B   │
     └────────────┘  └────────────┘  └─────────────┘
                             │
                    ┌────────▼────────┐
                    │  PostgreSQL     │
                    │  (run storage,  │
                    │   event log)    │
                    └─────────────────┘
```

Each user code server is an independent deployment containing asset and job definitions. This enables independent code deployments per team or domain -- a pattern that scales well for platform teams supporting multiple data engineering squads.

### Prefect on Kubernetes

```
                    ┌─────────────────────┐
                    │  Prefect Cloud /    │
                    │  Self-Hosted Server │
                    │  (API, UI, DB)      │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───┐  ┌──────▼─────┐  ┌─────▼──────┐
     │  K8s Worker│  │  K8s Worker│  │  K8s Worker │
     │  Pool A    │  │  Pool B    │  │  Pool C     │
     │  (ETL)     │  │  (ML)      │  │  (dbt)      │
     └────────────┘  └────────────┘  └─────────────┘
```

Work pools can be segmented by workload type, team, or environment, each with independent scaling policies and resource limits. See [[Kubernetes for Data Workloads]] for K8s patterns and [[Terraform for Data Infrastructure]] for IaC provisioning.

---

## Practical Recommendations

### For Teams Currently on Airflow

- **Do not migrate for the sake of migrating.** Airflow is mature, well-supported, and has the largest ecosystem. If your current setup works, invest in improving it (TaskFlow API, dynamic task mapping, deferrable operators) rather than switching.
- **Evaluate Dagster if** you find yourself building custom lineage tracking, maintaining a separate data catalogue, or struggling with partition-based backfills.
- **Evaluate Prefect if** your team spends more time fighting Airflow's abstractions than writing pipeline logic, or if you need dynamic workflows that do not fit the DAG model.

### For Greenfield Projects

- **Default to Dagster** if your platform is asset-centric (warehouse tables, ML models, dbt models) and you want lineage out of the box.
- **Default to Prefect** if your workloads are diverse (ETL, ML training, data science notebooks, API integrations) and you value flexibility over structure.
- **Default to Airflow** if your cloud provider offers a managed service (Composer, MWAA) and your team has existing Airflow experience.

### Common Anti-Patterns

| Anti-Pattern | Impact | Remedy |
|---|---|---|
| Using the orchestrator for heavy computation | Resource exhaustion, slow scheduling | Orchestrate external compute ([[PySpark Production Engineering|Spark]], [[Snowflake SQL Pipeline Patterns|Snowflake]], BigQuery) |
| Passing large data between tasks/assets | Memory pressure, serialisation failures | Write to storage, pass references |
| Monolithic pipelines (single massive flow/job) | Difficult debugging, no partial reruns | Decompose into focused assets or subflows |
| Ignoring idempotency | Duplicate data on reruns | Design all writes as upserts or partition replacements. See [[dbt Incremental Loading Patterns]]. |
| Skipping tests | Silent failures in production | Unit-test assets/tasks, validate DAG structure in CI |

---

## See Also

- [[Apache Airflow & Orchestration Patterns]] -- Airflow architecture, DAG design, Cloud Composer, MWAA
- [[Apache Airflow Deep Dive]] -- Airflow internals, operators, XComs, production patterns
- [[Core dbt Fundamentals]] -- dbt basics for orchestrator integration
- [[dbt Advanced Patterns & Cost Optimisation]] -- advanced dbt patterns orchestrated by Dagster or Prefect
- [[dbt Tag & Execution Strategy]] -- selective dbt execution from orchestrators
- [[dbt Incremental Loading Patterns]] -- incremental strategies relevant to partition-based orchestration
- [[Kubernetes for Data Workloads]] -- K8s deployment for Dagster and Prefect workers
- [[Terraform for Data Infrastructure]] -- IaC for orchestrator infrastructure
- [[Docker & Container Patterns]] -- containerised orchestrator deployments
- [[Snowflake SQL Pipeline Patterns]] -- Snowflake queries from orchestrator tasks
- [[GCP Data Services for Data Engineering]] -- GCP-native orchestration services
- [[ETL Pipeline Templates & Patterns]] -- reusable pipeline architectures
- [[Pipeline Observability & Monitoring]] -- monitoring patterns for orchestrated pipelines
