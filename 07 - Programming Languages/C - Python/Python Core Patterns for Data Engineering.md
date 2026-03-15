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
