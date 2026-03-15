# Python Core Patterns for Data Engineering

Design patterns, idioms, and standard library techniques for building production data pipelines in Python. Code examples adapted from a GCP ETL migration framework.

See also: [[Python Testing with pytest]], [[Python Data Generation Patterns]]

---

## 1. Factory Pattern

Use a **registry dict** mapping string keys to classes. The caller provides a config with a `type` field; the factory looks it up and instantiates the right class.

```python
from abc import ABC, abstractmethod
from typing import Dict, Any
import pandas as pd

class BaseExtractor(ABC):
    def __init__(self, config: Dict[str, Any]):
        self.config = config

    @abstractmethod
    def extract(self) -> pd.DataFrame:
        pass

    def _validate_config(self, required_fields: list) -> None:
        missing = [f for f in required_fields if f not in self.config]
        if missing:
            raise ValueError(f"Missing required config fields: {missing}")

class BigQueryExtractor(BaseExtractor):
    def __init__(self, config):
        super().__init__(config)
        self._validate_config(['query'])

    def extract(self) -> pd.DataFrame:
        ...

class DataExtractor:
    """Front-door class that delegates to the correct implementation."""
    def __init__(self, source_config: Dict[str, Any]):
        self.extractor = self._create_extractor(source_config)

    def _create_extractor(self, config) -> BaseExtractor:
        registry = {
            'bigquery':   BigQueryExtractor,
            'cloudsql':   CloudSQLExtractor,
            'gcs':        GCSExtractor,
            'local_file': LocalFileExtractor,
            'api':        APIExtractor,
        }
        source_type = config.get('type')
        if source_type not in registry:
            raise ValueError(f"Unsupported source type: {source_type}")
        return registry[source_type](config)

    def extract(self) -> pd.DataFrame:
        return self.extractor.extract()
```

**When to use which pattern:**
- **Factory** (above) — choose a concrete class at runtime from a fixed set of implementations. Best when each variant has different construction logic.
- **Strategy** — swap an algorithm at runtime via composition. Best when the object is the same but the behaviour varies (e.g. different compression strategies).
- **Template Method** — define a skeleton in a base class, let subclasses override specific steps. Best when the overall flow is identical but individual steps differ.

---

## 2. Configuration Management

### Jinja2 template rendering for YAML configs

Render environment variables into YAML before parsing:

```python
from jinja2 import Environment, FileSystemLoader
from pathlib import Path
import yaml

class ConfigManager:
    def __init__(self, environment: str = 'dev'):
        self.environment = environment
        self.config_cache: Dict[str, Any] = {}

    def load_config(self, config_path: str, variables: Optional[Dict] = None) -> Dict[str, Any]:
        path = Path(config_path)
        cache_key = f"{path}_{self.environment}"
        if cache_key in self.config_cache:
            return self.config_cache[cache_key]

        env_vars = self._load_environment_variables()
        template_vars = {**env_vars, **(variables or {})}
        config = self._render(path, template_vars)
        self._validate(config)
        self.config_cache[cache_key] = config
        return config

    def _render(self, path: Path, variables: dict) -> dict:
        env = Environment(loader=FileSystemLoader(path.parent))
        rendered = env.get_template(path.name).render(**variables)
        return yaml.safe_load(rendered)
```

A matching YAML template (`etl_job.yaml.j2`):

```yaml
job_name: sales_summary
source:
  type: bigquery
  query: "SELECT * FROM {{ PROJECT_ID }}.sales.transactions WHERE region = '{{ REGION }}'"
destination:
  type: gcs
  bucket: "{{ GCS_TEMP_BUCKET }}"
```

### Environment variable injection

```python
def _load_environment_variables(self) -> Dict[str, Any]:
    env_vars = {
        'ENVIRONMENT': self.environment,
        'PROJECT_ID': os.getenv('PROJECT_ID'),
        'REGION': os.getenv('REGION', 'us-central1'),
    }
    # Optionally load from env/<environment>.env file
    env_file = Path(f"env/{self.environment}.env")
    if env_file.exists():
        for line in env_file.read_text().splitlines():
            if line.strip() and not line.startswith('#'):
                key, value = line.strip().split('=', 1)
                env_vars[key] = value
    return env_vars
```

### Alternatives

- **python-dotenv** — `load_dotenv()` reads `.env` into `os.environ` automatically. Good for simple projects.
- **Hydra / OmegaConf** — hierarchical config composition, CLI overrides (`python main.py db.host=prod-db`), config groups for environment variants. Better for ML/complex pipelines.

---

## 3. CLI Design

### argparse for data tools

Standard flags for ETL CLIs: config path, environment, dry-run, log level.

```python
import argparse, sys
from pathlib import Path

def parse_arguments():
    parser = argparse.ArgumentParser(
        description='GCP Data Migration ETL Framework',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
    python -m framework.main --config etls/sales.yaml.j2 --env dev
    python -m framework.main --config etls/sales.yaml.j2 --env dev --dry-run
        """
    )
    parser.add_argument('--config', required=True, help='Path to ETL config file')
    parser.add_argument('--env', default='dev', choices=['dev', 'uat', 'prod'])
    parser.add_argument('--dry-run', action='store_true',
                        help='Validate config without executing')
    parser.add_argument('--log-level', default='INFO',
                        choices=['DEBUG', 'INFO', 'WARNING', 'ERROR'])
    return parser.parse_args()

def main():
    args = parse_arguments()
    config_path = Path(args.config)
    if not config_path.exists():
        print(f"Error: config not found: {config_path}")
        sys.exit(1)
    processor = ETLProcessor(str(config_path), args.env, args.dry_run)
    sys.exit(0 if processor.execute() else 1)
```

### Click as alternative

Click is cleaner for complex CLIs with subcommands:

```python
import click

@click.group()
def cli():
    pass

@cli.command()
@click.option('--config', required=True, type=click.Path(exists=True))
@click.option('--env', default='dev', type=click.Choice(['dev', 'uat', 'prod']))
@click.option('--dry-run', is_flag=True)
def run(config, env, dry_run):
    """Execute an ETL job."""
    ETLProcessor(config, env, dry_run).execute()
```

---

## 4. Structured Logging

### Basic setup with file + stream handlers

```python
import logging
from pathlib import Path

class Logger:
    @staticmethod
    def setup_logging(level: int = logging.INFO, log_file: Optional[str] = None):
        handlers = [logging.StreamHandler()]
        if log_file:
            Path(log_file).parent.mkdir(parents=True, exist_ok=True)
            handlers.append(logging.FileHandler(log_file))

        logging.basicConfig(
            level=level,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            datefmt='%Y-%m-%d %H:%M:%S',
            handlers=handlers,
        )
```

### structlog for JSON output

For machine-parseable logs (shipped to BigQuery, Datadog, etc.):

```python
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.BoundLogger,
)

log = structlog.get_logger()
log = log.bind(job_name="sales_summary", environment="prod")
log.info("extraction_complete", records=50000)
# {"job_name":"sales_summary","environment":"prod","event":"extraction_complete","records":50000,...}
```

### Log-level conventions for pipelines

| Level | Use |
|-------|-----|
| DEBUG | Row-level detail, SQL queries |
| INFO  | Phase transitions (extract/transform/load), record counts |
| WARNING | Data quality issues that do not halt the pipeline |
| ERROR | Stage failures, exceptions |

---

## 5. Metrics Collection

### Lightweight metrics class

Emit structured JSON metrics through the logging system:

```python
import json
from datetime import datetime

class MetricsCollector:
    def __init__(self):
        self.logger = logging.getLogger(f"{__name__}.metrics")

    def record_job_start(self, job_name: str):
        self._emit({'event': 'job_start', 'job_name': job_name})

    def record_extraction_complete(self, count: int):
        self._emit({'event': 'extraction_complete', 'records_extracted': count})

    def record_job_complete(self, job_name: str, duration: float, records: int):
        self._emit({
            'event': 'job_complete', 'job_name': job_name,
            'duration_seconds': duration, 'records_processed': records,
        })

    def _emit(self, data: Dict[str, Any]):
        data['timestamp'] = datetime.utcnow().isoformat()
        self.logger.info(json.dumps(data))
```

### Timing decorator

```python
import time, functools

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        logging.getLogger(func.__module__).info(
            f"{func.__name__} completed in {elapsed:.2f}s"
        )
        return result
    return wrapper
```

### Prometheus client (optional)

```python
from prometheus_client import Counter, Histogram, start_http_server

RECORDS_PROCESSED = Counter('etl_records_processed', 'Total records', ['job', 'stage'])
DURATION = Histogram('etl_stage_duration_seconds', 'Stage duration', ['job', 'stage'])

start_http_server(8000)  # Expose /metrics endpoint
```

---

## 6. Error Handling Patterns

### Retry with tenacity

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type((ConnectionError, TimeoutError)),
    reraise=True,
)
def fetch_from_api(url: str) -> dict:
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

### Custom exceptions for pipeline stages

```python
class PipelineError(Exception):
    """Base exception for all pipeline errors."""

class ExtractionError(PipelineError):
    """Raised when data extraction fails."""

class ValidationError(PipelineError):
    """Raised when data quality checks fail."""

class LoadError(PipelineError):
    """Raised when loading to destination fails."""
```

### Fail-fast vs graceful degradation

- **Fail-fast:** raise immediately on bad config, missing credentials, schema mismatch. Use in `initialize()`.
- **Graceful degradation:** log a warning and continue when a non-critical step fails (e.g. one of N files fails to parse, optional enrichment unavailable). Use in batch-processing loops.

```python
for blob in blobs:
    try:
        df = parse_blob(blob)
        dataframes.append(df)
    except Exception as e:
        self.logger.warning(f"Skipping {blob.name}: {e}")
        continue
```

---

## 7. Type Hints and Pydantic

### Annotating pipeline functions

```python
from typing import Dict, Any, Optional, List
import pandas as pd

def extract(self) -> pd.DataFrame: ...
def load(self, data: pd.DataFrame) -> Dict[str, Any]: ...
def _validate_config(self, required_fields: List[str]) -> None: ...
```

### Pydantic models for config validation

Replace manual `if key not in config` checks with Pydantic:

```python
from pydantic import BaseModel, Field, validator

class SourceConfig(BaseModel):
    type: str
    query: Optional[str] = None
    bucket: Optional[str] = None
    file_pattern: Optional[str] = None

    @validator('type')
    def validate_type(cls, v):
        allowed = {'bigquery', 'cloudsql', 'gcs', 'local_file', 'api'}
        if v not in allowed:
            raise ValueError(f"Unsupported source type: {v}")
        return v

class ETLConfig(BaseModel):
    job_name: str
    source: SourceConfig
    destination: Dict[str, Any]
    transformations: List[Dict[str, Any]] = Field(default_factory=list)
```

---

## 8. Package Structure

### `__init__.py` exports

Control the public API with explicit imports and `__all__`:

```python
# framework/__init__.py
__version__ = "1.0.0"

from .etl import ETLProcessor
from .config import ConfigManager
from .extract import DataExtractor
from .load import DataLoader
from .utils import Logger, ValidationUtils

__all__ = [
    'ETLProcessor', 'ConfigManager', 'DataExtractor',
    'DataLoader', 'Logger', 'ValidationUtils',
]
```

### Requirements per environment

```
requirements/
    base.txt          # Core: pandas, pyyaml, jinja2
    dev.txt           # -r base.txt + pytest, black, mypy
    prod.txt          # -r base.txt + google-cloud-bigquery, gunicorn
```

### Wheel packaging

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "etl-framework"
version = "1.0.0"
dependencies = ["pandas>=2.0", "pyyaml", "jinja2"]

[project.scripts]
etl-run = "framework.main:main"
```

Build with `python -m build`; install with `pip install dist/etl_framework-1.0.0-py3-none-any.whl`.

---

## 9. Decorators and Generators

### Retry decorator (simple)

```python
import functools, time

def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    time.sleep(delay * attempt)
        return wrapper
    return decorator
```

### Generator-based batch processing

Process large datasets without loading everything into memory:

```python
from typing import Iterator

def batch_iter(df: pd.DataFrame, batch_size: int = 10_000) -> Iterator[pd.DataFrame]:
    for start in range(0, len(df), batch_size):
        yield df.iloc[start:start + batch_size]

for batch in batch_iter(large_df, batch_size=5000):
    loader.load(batch)
```

### Context managers for resource cleanup

```python
from contextlib import contextmanager

@contextmanager
def gcs_temp_file(bucket_name: str, blob_name: str):
    """Upload temp data, yield path, delete on exit."""
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    try:
        yield blob
    finally:
        if blob.exists():
            blob.delete()
```

---

## 10. Key Standard Library

| Module | Use in data engineering |
|--------|----------------------|
| `pathlib.Path` | Cross-platform file paths, `path.parent.mkdir(parents=True, exist_ok=True)` |
| `dataclasses` | Lightweight data containers without Pydantic overhead |
| `enum.Enum` | Constrain values: `class Environment(Enum): DEV='dev'; UAT='uat'; PROD='prod'` |
| `functools.lru_cache` | Cache expensive lookups (schema fetches, API metadata) |
| `functools.partial` | Pre-fill function args: `bq_query = partial(client.query, project=project_id)` |
| `collections.defaultdict` | Accumulate metrics: `counts = defaultdict(int)` |
| `collections.Counter` | Frequency counts: `Counter(df['status'])` |
| `typing` | `Dict`, `Any`, `Optional`, `List`, `Iterator` for readable signatures |

### Example: caching with lru_cache

```python
from functools import lru_cache

@lru_cache(maxsize=32)
def get_table_schema(project: str, dataset: str, table: str) -> list:
    client = bigquery.Client(project=project)
    ref = client.get_table(f"{project}.{dataset}.{table}")
    return [{'name': f.name, 'type': f.field_type} for f in ref.schema]
```

### Example: Enum for environment safety

```python
from enum import Enum

class Environment(Enum):
    DEV = 'dev'
    UAT = 'uat'
    PROD = 'prod'

def get_bucket(env: Environment) -> str:
    buckets = {
        Environment.DEV: 'dev-data-bucket',
        Environment.UAT: 'uat-data-bucket',
        Environment.PROD: 'prod-data-bucket',
    }
    return buckets[env]
```

---

## Async Patterns for Data Engineering

### asyncio Fundamentals

Python's `asyncio` module provides cooperative multitasking via an **event loop**. Functions declared with `async def` are coroutines; `await` yields control back to the loop while waiting for I/O.

```python
import asyncio

async def fetch_data(source: str) -> dict:
    """Simulate an I/O-bound operation."""
    await asyncio.sleep(1)  # Non-blocking wait
    return {"source": source, "rows": 1000}

async def main():
    result = await fetch_data("api_endpoint")
    print(result)

asyncio.run(main())  # Entry point — creates and runs the event loop
```

Key concepts:
- **Event loop** — the scheduler that runs coroutines, handles I/O callbacks, and manages tasks
- **Coroutine** — an `async def` function; does nothing until awaited or wrapped in a task
- **Task** — a coroutine scheduled on the loop via `asyncio.create_task()`; runs concurrently

### Parallel API Calls with aiohttp

[[Python Core Patterns for Data Engineering#6. Error Handling Patterns|Retry patterns]] apply here too. Use `aiohttp` for non-blocking HTTP:

```python
import aiohttp
import asyncio
from typing import List, Dict, Any

async def fetch_page(session: aiohttp.ClientSession, url: str) -> Dict[str, Any]:
    async with session.get(url, timeout=aiohttp.ClientTimeout(total=30)) as resp:
        resp.raise_for_status()
        return await resp.json()

async def fetch_all_pages(base_url: str, total_pages: int) -> List[Dict]:
    results = []
    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_page(session, f"{base_url}?page={p}")
            for p in range(1, total_pages + 1)
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if not isinstance(r, Exception)]
```

### Async Database Queries with asyncpg

`asyncpg` provides a high-performance async driver for PostgreSQL:

```python
import asyncpg

async def fetch_records(dsn: str, query: str) -> list:
    conn = await asyncpg.connect(dsn)
    try:
        rows = await conn.fetch(query)
        return [dict(row) for row in rows]
    finally:
        await conn.close()

async def bulk_fetch(dsn: str, queries: List[str]) -> List[list]:
    pool = await asyncpg.create_pool(dsn, min_size=2, max_size=10)
    async with pool:
        tasks = [pool.fetch(q) for q in queries]
        return await asyncio.gather(*tasks)
```

### Semaphore for Rate Limiting

Prevent overwhelming upstream APIs or databases with `asyncio.Semaphore`:

```python
async def rate_limited_fetch(
    urls: List[str], max_concurrent: int = 10
) -> List[dict]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async def _fetch(session: aiohttp.ClientSession, url: str) -> dict:
        async with semaphore:  # At most max_concurrent coroutines proceed
            async with session.get(url) as resp:
                return await resp.json()

    async with aiohttp.ClientSession() as session:
        tasks = [_fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)
```

### gather for Concurrent Tasks

`asyncio.gather()` runs multiple coroutines concurrently and collects results in order:

```python
async def ingest_from_multiple_sources():
    api_data, db_data, file_data = await asyncio.gather(
        fetch_from_api("https://api.example.com/orders"),
        fetch_from_database("postgresql://host/db", "SELECT * FROM events"),
        fetch_from_s3("s3://bucket/data.parquet"),
    )
    # All three run concurrently; results are returned in declaration order
    return merge_datasets(api_data, db_data, file_data)
```

Use `return_exceptions=True` to prevent one failure from cancelling all tasks — essential for batch ingestion where partial results are acceptable.

### When Async Beats Threading

| Scenario | Async Recommended | Why |
|----------|-------------------|-----|
| Parallel API ingestion (100s of endpoints) | Yes | One thread per request wastes memory; async handles thousands of connections on a single thread |
| Concurrent file downloads (S3, GCS) | Yes | I/O-bound; `aiobotocore` or `gcloud-aio-storage` avoids thread overhead |
| Paginated API extraction | Yes | Natural fit for semaphore-controlled concurrency |
| Fan-out webhook delivery | Yes | Fire-and-forget pattern maps cleanly to tasks |
| Database connection pooling | Yes | `asyncpg` outperforms synchronous drivers for high-concurrency reads |

### When NOT to Use Async

| Scenario | Why Not |
|----------|---------|
| CPU-bound transforms (pandas, NumPy) | Async does not bypass the GIL; use `multiprocessing` or [[PySpark Architecture and Core Concepts\|PySpark]] |
| Spark jobs | Spark has its own parallelism model; wrapping it in async adds complexity for no gain |
| Simple sequential scripts | Async adds cognitive overhead; a straightforward `for` loop is clearer |
| Library ecosystem gaps | If your database driver or SDK lacks async support, forced wrapping with `run_in_executor()` negates the benefits |

> [!tip] Rule of thumb: if your pipeline spends most of its time **waiting for network responses**, async will help. If it spends most of its time **computing**, use multiprocessing or a distributed framework.

---

## Packaging and Dependency Management

### pyproject.toml Anatomy

`pyproject.toml` is the standard Python project configuration file ([[Python Core Patterns for Data Engineering#8. Package Structure|see also section 8]]). Key sections:

```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "my-etl-pipeline"
version = "2.1.0"
description = "Production data ingestion framework"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [{name = "Data Team", email = "data@example.com"}]
dependencies = [
    "pandas>=2.0,<3.0",
    "sqlalchemy>=2.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "ruff>=0.1", "mypy>=1.0", "pre-commit"]
aws = ["boto3>=1.28", "aiobotocore>=2.5"]
gcp = ["google-cloud-bigquery>=3.0", "google-cloud-storage>=2.0"]

[project.scripts]
etl-run = "my_pipeline.cli:main"
etl-validate = "my_pipeline.validate:main"

[tool.ruff]
line-length = 100
target-version = "py311"
select = ["E", "F", "I", "N", "W"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short --strict-markers"

[tool.mypy]
python_version = "3.11"
strict = true
```

### Poetry Workflow

Poetry provides dependency resolution, virtual environment management, and publishing in a single tool:

```bash
# Project lifecycle
poetry new my-pipeline                        # Scaffold with src layout
poetry init                                   # Initialise in existing project

# Dependency management
poetry add pandas sqlalchemy                  # Add production dependencies
poetry add --group dev pytest ruff mypy       # Dev-only dependencies
poetry add "boto3>=1.28,<2.0"                 # With version constraints
poetry remove boto3                           # Remove a dependency

# Lock and install
poetry lock                                   # Resolve and lock all versions
poetry lock --no-update                       # Re-lock without upgrading
poetry install                                # Install everything from lock file
poetry install --only main                    # Production dependencies only

# Build and publish
poetry build                                  # Create sdist + wheel in dist/
poetry publish --repository pypi              # Publish to PyPI
poetry export -f requirements.txt -o requirements.txt  # Export for pip-based CI
```

### uv — Fast pip Replacement

`uv` is a Rust-based tool that replaces pip, pip-tools, and virtualenv with dramatically faster performance:

```bash
# Virtual environment management
uv venv                                       # Create .venv (auto-detects Python)
uv venv --python 3.12                         # Specific Python version
uv python install 3.12                        # Download and install Python

# pip-compatible interface
uv pip install pandas                         # Install single package
uv pip install -r requirements.txt            # Install from requirements
uv pip install -e ".[dev]"                    # Editable install with extras

# Native project management
uv init my-pipeline                           # Scaffold project
uv add pandas sqlalchemy                      # Add to pyproject.toml + lock
uv add --dev pytest ruff                      # Dev dependencies
uv lock                                       # Generate uv.lock
uv sync                                       # Install exactly what lock specifies
uv run pytest                                 # Run within managed environment
```

### pip-tools (pip-compile)

`pip-tools` bridges the gap between raw `requirements.txt` and full dependency managers:

```bash
# Install pip-tools
pip install pip-tools

# Create requirements.in with top-level dependencies
# requirements.in:
#   pandas>=2.0
#   sqlalchemy>=2.0

pip-compile requirements.in                   # Resolve → requirements.txt (pinned)
pip-compile --upgrade                         # Upgrade all to latest compatible
pip-compile --generate-hashes                 # Add hashes for supply-chain security
pip-sync requirements.txt                     # Make environment match exactly
```

### requirements.txt vs Lock Files

| Approach | Reproducible | Flexible | Best For |
|----------|-------------|----------|----------|
| `requirements.txt` (unpinned) | No | Yes | Quick experiments, tutorials |
| `pip freeze > requirements.txt` | Snapshot only | No | Simple CI, Docker builds |
| `pip-compile` (pip-tools) | Yes | Moderate | Teams using pip without Poetry |
| `poetry.lock` | Yes | Yes | Libraries, published packages |
| `uv.lock` | Yes | Yes | All use cases (fastest resolver) |

### When to Use Each Tool

| Tool | Best For | Avoid When |
|------|----------|------------|
| **pip + venv** | Simple scripts, Docker layers, CI without extras | Large dependency trees with conflicts |
| **pip-tools** | Teams already on pip wanting reproducibility | Publishing libraries to PyPI |
| **Poetry** | Library development, publishing, monorepos with extras | Extremely large projects (slower resolver) |
| **uv** | Everything — fastest resolver, drop-in replacement | Ecosystem is very new; some edge cases remain |

### Editable Installs for Development

Editable installs let you modify source code without reinstalling:

```bash
# With pip
pip install -e .                              # Install current project in dev mode
pip install -e ".[dev]"                       # With optional dev dependencies

# With uv
uv pip install -e ".[dev]"                    # Same semantics, faster

# With Poetry
poetry install                                # Always editable by default
```

Editable installs are essential when:
- Developing a shared library used by multiple pipelines
- Running [[Python Testing with pytest|tests]] against the installed package
- Using `entry_points` / `[project.scripts]` during local development

> [!tip] Prefer `uv` for new projects — it handles virtual environments, dependency resolution, and locking in a single tool with speed comparable to Cargo or npm.
