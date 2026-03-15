# Quick Reference Cards

> Condensed cheat sheets for everyday data engineering tools. Copy-paste ready.

---

## 1. dbt Commands

| Command | Description |
|---|---|
| `dbt run` | Execute models |
| `dbt test` | Run tests |
| `dbt build` | Run + test in dependency order |
| `dbt seed` | Load CSVs from `seeds/` |
| `dbt snapshot` | Execute SCD Type 2 snapshots |
| `dbt docs generate` | Build documentation site |
| `dbt compile` | Compile SQL without executing |
| `dbt debug` | Test connection and config |
| `dbt ls` | List resources in project |

**Common Flags:**
```bash
dbt run --select model_name           # Single model
dbt run --select +model_name          # Model + all upstream
dbt run --select model_name+          # Model + all downstream
dbt run --select +model_name+         # Full lineage
dbt run --select tag:daily            # By tag
dbt run --exclude model_name          # Everything except
dbt run --select path:models/staging  # By directory
dbt run --full-refresh                # Force full rebuild (incremental)
dbt run --vars '{"date": "2026-01-01"}'  # Pass variables
dbt build --select state:modified+    # Slim CI: modified + downstream
```

**Advanced Usage:**
```bash
# Docs
dbt docs generate                     # Generate catalogue
dbt docs serve --port 8080            # Serve docs locally

# Debugging
dbt debug                             # Verify profiles.yml + connectivity
dbt compile --select model_name       # Inspect compiled SQL
dbt --debug run --select model_name   # Verbose logging

# Listing resources
dbt ls --resource-type model          # List all models
dbt ls --select tag:finance           # List by tag
dbt ls --output json                  # JSON output for scripting

# Snapshots
dbt snapshot --select snap_orders     # Run specific snapshot
dbt snapshot --full-refresh           # Rebuild snapshot from scratch

# Testing
dbt test --select model_name         # Tests for one model
dbt test --select test_type:singular  # Only singular tests
dbt test --select source:*           # All source freshness tests

# Build (run + test combined)
dbt build --fail-fast                 # Stop on first failure
dbt build --select +model_name+ --exclude tag:slow
```

> [!tip] See [[dbt Project Structure]] and [[dbt Advanced Patterns]] for project-level guidance.

---

## 2. SQL Essentials

### JOIN Types
```sql
SELECT * FROM a INNER JOIN b ON a.id = b.id;      -- Matching rows only
SELECT * FROM a LEFT JOIN b ON a.id = b.id;        -- All left, NULLs if no match
SELECT * FROM a FULL OUTER JOIN b ON a.id = b.id;  -- All rows, NULLs both sides
SELECT * FROM a CROSS JOIN b;                       -- Cartesian product
-- ANTI JOIN
SELECT * FROM a LEFT JOIN b ON a.id = b.id WHERE b.id IS NULL;
```

### Window Functions
```sql
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)
RANK()       OVER (PARTITION BY dept ORDER BY salary DESC)    -- gaps on ties
DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC)    -- no gaps
LAG(col, 1)  OVER (ORDER BY date_col)                        -- previous row
LEAD(col, 1) OVER (ORDER BY date_col)                        -- next row
SUM(amt)     OVER (PARTITION BY cust ORDER BY dt ROWS UNBOUNDED PRECEDING)
```

### CTE and MERGE
```sql
WITH filtered AS (
    SELECT id, amount FROM orders WHERE status = 'complete'
),
aggregated AS (
    SELECT id, SUM(amount) AS total FROM filtered GROUP BY id
)
SELECT * FROM aggregated WHERE total > 100;

MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED AND s.updated_at > t.updated_at
    THEN UPDATE SET t.value = s.value, t.updated_at = s.updated_at
WHEN NOT MATCHED
    THEN INSERT (id, value, updated_at) VALUES (s.id, s.value, s.updated_at);
```

| Function | Example |
|---|---|
| `COALESCE(a, b, c)` | `COALESCE(phone, email, 'N/A')` — first non-NULL |
| `NULLIF(a, b)` | `NULLIF(count, 0)` — avoid division by zero |
| `TRY_CAST(x AS type)` | `TRY_CAST('abc' AS INT)` — NULL on failure |
| `CASE WHEN` | `CASE WHEN x > 0 THEN 'pos' ELSE 'neg' END` |
| `IFF(cond, t, f)` | Snowflake shorthand for simple CASE |

---

## 3. Git for Data Engineering

```bash
# Daily workflow
git status                            # What's changed?
git diff                              # Unstaged changes
git add -p                            # Stage interactively
git commit -m "feat: add order model" # Commit
git push origin feature-branch        # Push

# Branching
git checkout -b feature/new-model     # Create and switch
git branch -d feature/merged          # Delete merged branch

# Sync and rebase
git pull --rebase origin main         # Pull + rebase onto main
git rebase main                       # Rebase current onto main
git cherry-pick <hash>                # Apply single commit

# Stash and undo
git stash && git stash pop            # Save / restore changes
git reset --soft HEAD~1               # Undo commit, keep staged
git log --oneline -20                 # Last 20 commits, compact
git diff main...HEAD                  # Changes since divergence
```

**Advanced Git:**
```bash
# Rebase interactively (squash, reorder, edit commits)
git rebase -i HEAD~5                  # Last 5 commits
git rebase -i main                    # All commits since main
# In editor: pick, squash, reword, edit, drop

# Cherry-pick
git cherry-pick <hash>                # Apply single commit
git cherry-pick <hash1> <hash2>       # Apply multiple commits
git cherry-pick --no-commit <hash>    # Apply without committing
git cherry-pick --abort               # Cancel in-progress cherry-pick

# Stash
git stash                             # Stash tracked changes
git stash -u                          # Include untracked files
git stash list                        # View all stashes
git stash show -p stash@{0}           # View stash diff
git stash pop                         # Apply + remove latest stash
git stash apply stash@{2}             # Apply specific, keep in list
git stash drop stash@{0}              # Remove specific stash
git stash branch new-branch stash@{0} # Create branch from stash

# Bisect (find which commit introduced a bug)
git bisect start
git bisect bad                        # Current commit is broken
git bisect good <known-good-hash>     # This commit was fine
# Git checks out midpoint — test, then:
git bisect good                       # or git bisect bad
# Repeat until culprit found
git bisect reset                      # Return to original HEAD

# Reflog (recover "lost" commits)
git reflog                            # Show all HEAD movements
git checkout <reflog-hash>            # Recover lost commit
git reflog expire --expire=90.days    # Default retention: 90 days

# Worktree (multiple working directories, one repo)
git worktree add ../hotfix main       # Check out main in ../hotfix
git worktree add ../feature feat-branch
git worktree list                     # Show all worktrees
git worktree remove ../hotfix         # Clean up
```

> [!tip] `git bisect` is invaluable for tracking down when a [[dbt Advanced Patterns|dbt test]] started failing.

---

## 4. Docker

```bash
docker build -t myapp:1.0 .                        # Build image
docker run -d --name myapp -p 8080:80 myapp:1.0     # Run detached
docker exec -it myapp /bin/bash                      # Shell into container
docker ps -a                                         # All containers
docker logs -f myapp                                 # Follow logs
docker stop myapp && docker rm myapp                 # Stop and remove
docker system prune -a                               # Remove all unused
docker compose up -d                                 # Start all services
docker compose down -v                               # Stop + remove volumes
docker compose logs -f service                       # Follow one service
```

**Multi-Stage Build:**
```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /install /usr/local
COPY . /app
CMD ["python", "main.py"]
```

**Advanced Docker:**
```bash
# Build
docker build -t myapp:1.0 .                         # Build from Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .      # Specific Dockerfile
docker build --no-cache -t myapp:1.0 .               # Rebuild without cache
docker build --target builder -t myapp:build .        # Build specific stage
docker build --build-arg VERSION=3.11 -t myapp .      # Pass build argument

# Run
docker run -d --name app -p 8080:80 myapp:1.0        # Detached, port mapping
docker run --rm -it myapp:1.0 /bin/bash               # Interactive, auto-remove
docker run -v /host/data:/container/data myapp        # Bind mount
docker run --env-file .env myapp                      # Load environment file
docker run --network my-net --name app myapp          # Attach to network
docker run --cpus=2 --memory=4g myapp                 # Resource limits

# Exec
docker exec -it myapp /bin/bash                       # Shell into running container
docker exec myapp cat /app/config.yml                 # Run command in container

# Compose
docker compose up -d                                  # Start all services
docker compose up -d --build                          # Rebuild before starting
docker compose down -v                                # Stop + remove volumes
docker compose ps                                     # Status of services
docker compose logs -f --tail=100 service_name        # Follow logs, last 100
docker compose exec service_name /bin/bash            # Shell into service
docker compose pull                                   # Pull latest images
docker compose config                                 # Validate compose file

# Housekeeping
docker ps -a                                          # All containers
docker images                                         # List images
docker logs -f --since=1h myapp                       # Logs from last hour
docker system prune -a --volumes                      # Full cleanup
docker volume ls                                      # List volumes
docker network ls                                     # List networks
docker inspect myapp                                  # Container details (JSON)
docker stats                                          # Live resource usage
docker cp myapp:/app/output.csv ./output.csv          # Copy file out
```

> [!tip] See [[Docker Fundamentals]] and [[Docker Compose and Networking]] for deeper coverage.

---

## 5. Terraform

```bash
terraform init                        # Initialise providers/backend
terraform fmt && terraform validate   # Format + check syntax
terraform plan                        # Preview changes
terraform apply                       # Apply changes
terraform destroy                     # Tear down resources

# State
terraform state list                  # List managed resources
terraform state show <resource>       # Show details
terraform state rm <resource>         # Remove from state only
terraform import <resource> <id>      # Import existing

# Workspaces
terraform workspace new dev           # Create workspace
terraform workspace select prod       # Switch workspace
```

**Advanced Terraform:**
```bash
# Init
terraform init                                 # Download providers + modules
terraform init -upgrade                        # Upgrade providers to latest
terraform init -backend-config=prod.hcl        # Override backend config
terraform init -migrate-state                  # Migrate state backend

# Plan
terraform plan                                 # Preview all changes
terraform plan -target=aws_s3_bucket.data      # Plan single resource
terraform plan -var="env=prod"                 # Override variable
terraform plan -var-file=prod.tfvars           # Variables from file
terraform plan -out=plan.tfplan                # Save plan to file
terraform plan -destroy                        # Preview destroy

# Apply
terraform apply                                # Apply with confirmation
terraform apply plan.tfplan                    # Apply saved plan (no prompt)
terraform apply -auto-approve                  # Skip confirmation
terraform apply -target=aws_s3_bucket.data     # Apply single resource
terraform apply -replace=aws_instance.web      # Force recreate resource

# Destroy
terraform destroy                              # Destroy all resources
terraform destroy -target=aws_instance.web     # Destroy single resource

# State management
terraform state list                           # All managed resources
terraform state show aws_s3_bucket.data        # Resource details
terraform state rm aws_s3_bucket.old           # Remove from state (keep infra)
terraform state mv aws_s3_bucket.a aws_s3_bucket.b  # Rename in state
terraform state pull                           # Download remote state
terraform state push                           # Upload local state

# Import
terraform import aws_s3_bucket.data my-bucket  # Import existing resource
terraform import 'aws_iam_role.role["admin"]' arn:aws:iam::123:role/admin

# Other
terraform fmt -recursive                       # Format all .tf files
terraform validate                             # Syntax and reference check
terraform output                               # Show all outputs
terraform output -json                         # Outputs as JSON
terraform graph | dot -Tpng > graph.png        # Dependency graph
terraform providers                            # List required providers
terraform force-unlock <lock-id>               # Release stuck state lock
```

> [!tip] See [[Terraform AWS Modules]] for module patterns and remote state configuration.

---

## 6. Snowflake

```sql
-- Warehouse operations
CREATE WAREHOUSE IF NOT EXISTS loading_wh
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
ALTER WAREHOUSE loading_wh SET WAREHOUSE_SIZE = 'SMALL';
ALTER WAREHOUSE loading_wh SUSPEND;

-- Stage and file format
CREATE OR REPLACE STAGE my_s3_stage
  URL = 's3://bucket/path/' STORAGE_INTEGRATION = my_integration;
CREATE OR REPLACE FILE FORMAT csv_fmt
  TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null', '') FIELD_OPTIONALLY_ENCLOSED_BY = '"';

-- Load data
COPY INTO raw.orders FROM @my_s3_stage/orders/
  FILE_FORMAT = csv_fmt ON_ERROR = 'CONTINUE';

-- Access control
GRANT USAGE ON DATABASE analytics TO ROLE transformer;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.public TO ROLE transformer;
GRANT ROLE transformer TO USER dbt_service;
```

| Command | Purpose |
|---|---|
| `SHOW DATABASES / SCHEMAS / TABLES` | List objects |
| `DESCRIBE TABLE db.schema.table` | Column details |
| `SHOW WAREHOUSES` | Status and sizes |
| `SHOW GRANTS TO ROLE x` | Role permissions |

**Advanced Snowflake SQL:**
```sql
-- COPY INTO variants
COPY INTO raw.events FROM @my_s3_stage/events/
  FILE_FORMAT = (TYPE = 'JSON')
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
  ON_ERROR = 'SKIP_FILE';

COPY INTO raw.orders FROM @my_s3_stage/orders/
  FILE_FORMAT = csv_fmt
  PATTERN = '.*2026-03.*\\.csv'                -- Regex file filter
  FORCE = TRUE;                                 -- Reload already-loaded files

-- Unload (export) data
COPY INTO @my_s3_stage/export/orders_
  FROM (SELECT * FROM analytics.orders WHERE order_date >= '2026-01-01')
  FILE_FORMAT = (TYPE = 'PARQUET')
  HEADER = TRUE
  OVERWRITE = TRUE;

-- MERGE (upsert)
MERGE INTO target t USING staging s ON t.id = s.id
WHEN MATCHED AND s.updated_at > t.updated_at
    THEN UPDATE SET t.name = s.name, t.updated_at = s.updated_at
WHEN NOT MATCHED
    THEN INSERT (id, name, updated_at) VALUES (s.id, s.name, s.updated_at);

-- Stage operations
CREATE OR REPLACE STAGE my_internal_stage;
PUT file:///data/file.csv @my_internal_stage AUTO_COMPRESS = TRUE;
LIST @my_s3_stage/orders/;
REMOVE @my_internal_stage/file.csv.gz;

-- SHOW commands
SHOW DATABASES;
SHOW SCHEMAS IN DATABASE analytics;
SHOW TABLES IN SCHEMA analytics.public;
SHOW COLUMNS IN TABLE analytics.public.orders;
SHOW STAGES IN SCHEMA raw.public;
SHOW FILE FORMATS IN SCHEMA raw.public;
SHOW WAREHOUSES;
SHOW ROLES;
SHOW GRANTS TO ROLE transformer;
SHOW GRANTS ON TABLE analytics.public.orders;

-- DESCRIBE commands
DESCRIBE TABLE analytics.public.orders;
DESCRIBE STAGE my_s3_stage;
DESCRIBE FILE FORMAT csv_fmt;
DESCRIBE WAREHOUSE loading_wh;

-- ALTER WAREHOUSE
ALTER WAREHOUSE loading_wh SET WAREHOUSE_SIZE = 'MEDIUM';
ALTER WAREHOUSE loading_wh SET AUTO_SUSPEND = 120;
ALTER WAREHOUSE loading_wh SET MAX_CLUSTER_COUNT = 3;   -- Multi-cluster
ALTER WAREHOUSE loading_wh SUSPEND;
ALTER WAREHOUSE loading_wh RESUME;

-- Useful metadata queries
SELECT * FROM INFORMATION_SCHEMA.LOAD_HISTORY
  WHERE TABLE_NAME = 'ORDERS' ORDER BY LAST_LOAD_TIME DESC LIMIT 10;

SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
  WHERE EXECUTION_STATUS = 'FAIL' ORDER BY START_TIME DESC LIMIT 20;
```

> [!tip] See [[Snowflake Architecture]], [[Snowflake Data Loading]], and [[Snowflake Pipeline Patterns]] for deeper patterns.

---

## 7. Python One-Liners

```python
# Comprehensions
squares = [x**2 for x in range(10)]
evens   = [x for x in items if x % 2 == 0]
lookup  = {row["id"]: row for row in rows}
flat    = [x for sub in nested for x in sub]

# f-strings
f"Loaded {count:,} rows in {elapsed:.2f}s"

# Enumerate and zip
for i, item in enumerate(items, start=1): ...
merged = dict(zip(keys, values))

# Pathlib
from pathlib import Path
files = list(Path("data").glob("*.csv"))
text  = Path("file.txt").read_text()

# Collections
from collections import defaultdict, Counter
groups = defaultdict(list)
for item in items: groups[item.category].append(item)
freq = Counter(words).most_common(10)

# Dataclass
from dataclasses import dataclass
@dataclass
class PipelineResult:
    table: str
    rows_loaded: int
    success: bool = True

# Patterns
value  = data.get("key", "default")              # Safe access
status = "active" if count > 0 else "empty"       # Ternary
first, *middle, last = sorted(scores)              # Unpack
```

---

## 8. Python Packaging

### pip and venv

```bash
# Virtual environments
python -m venv .venv                              # Create virtual environment
source .venv/bin/activate                         # Activate (Linux/macOS)
.venv\Scripts\activate                            # Activate (Windows)
deactivate                                        # Deactivate

# pip basics
pip install requests                              # Install package
pip install "requests>=2.28,<3.0"                 # Version constraints
pip install -r requirements.txt                   # Install from file
pip install -e .                                  # Editable install (local package)
pip install --upgrade pip                         # Upgrade pip itself

# Dependency management
pip freeze > requirements.txt                     # Export installed packages
pip list --outdated                                # Show upgradable packages
pip show requests                                 # Package metadata
pip uninstall requests                            # Remove package
pip cache purge                                   # Clear download cache
```

### Poetry

```bash
# Project setup
poetry new my-project                             # Scaffold new project
poetry init                                       # Initialise in existing directory

# Dependencies
poetry add requests                               # Add dependency
poetry add "requests>=2.28"                       # With version constraint
poetry add --group dev pytest black ruff          # Dev dependencies
poetry remove requests                            # Remove dependency
poetry update                                     # Update all to latest allowed

# Environment
poetry install                                    # Install all dependencies
poetry install --no-dev                           # Production only
poetry shell                                      # Activate virtual environment
poetry env info                                   # Show environment details
poetry env remove python3.11                      # Remove environment

# Build and publish
poetry build                                      # Create sdist + wheel
poetry publish                                    # Publish to PyPI

# Lock file
poetry lock                                       # Regenerate lock file
poetry lock --no-update                           # Lock without upgrading
poetry export -f requirements.txt -o requirements.txt  # Export for pip
```

### uv (Fast Python Package Manager)

```bash
# Virtual environments
uv venv                                           # Create .venv
uv venv --python 3.12                             # Specific Python version

# Installing packages (pip-compatible)
uv pip install requests                           # Install package
uv pip install -r requirements.txt                # From requirements file
uv pip install -e .                               # Editable install
uv pip compile requirements.in -o requirements.txt  # Lock dependencies
uv pip sync requirements.txt                      # Sync environment exactly

# Project management (uv native)
uv init my-project                                # Scaffold project
uv add requests                                   # Add dependency
uv add --dev pytest ruff                          # Dev dependency
uv remove requests                                # Remove dependency
uv lock                                           # Generate lock file
uv sync                                           # Install from lock file
uv run python main.py                             # Run within environment
uv run pytest                                     # Run tool within environment

# Tool management
uv tool install ruff                              # Install CLI tool globally
uv tool run black .                               # Run tool without installing
```

### pyproject.toml Reference

```toml
[project]
name = "my-pipeline"
version = "1.0.0"
description = "Data ingestion pipeline"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28",
    "pandas>=2.0",
    "sqlalchemy>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "ruff>=0.1", "black>=23.0"]

[project.scripts]
ingest = "my_pipeline.cli:main"

[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.backends._legacy:_Backend"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

| Tool | Speed | Lock File | Resolver | Best For |
|---|---|---|---|---|
| **pip + venv** | Moderate | Manual (`freeze`) | Basic | Simple projects, CI |
| **Poetry** | Moderate | `poetry.lock` | Advanced | Libraries, published packages |
| **uv** | Very fast | `uv.lock` | Advanced | All use cases (Rust-based, drop-in replacement) |

> [!tip] `uv` is rapidly becoming the standard for Python packaging in data engineering due to its speed and compatibility with both pip and Poetry workflows. See [[Python Development Environment]] for setup guidance.

---

## PySpark Quick Reference

### Read and Write

```python
# Read
df = spark.read.parquet("s3://bucket/data/")
df = spark.read.csv("path/", header=True, inferSchema=True)
df = spark.read.json("path/")
df = spark.read.format("delta").load("path/")
df = spark.read.jdbc(url, "schema.table", properties=props)
df = spark.table("catalogue.schema.table")

# Write
df.write.parquet("s3://bucket/output/", mode="overwrite")
df.write.format("delta").mode("append").partitionBy("date").save("path/")
df.write.saveAsTable("catalogue.schema.table")
df.write.csv("path/", header=True, mode="overwrite")
```

### Transformations

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Select, filter, alias
df.select("col1", F.col("col2").alias("renamed"))
df.filter(F.col("status") == "active")
df.filter(F.col("amount").between(100, 500))
df.where("date >= '2026-01-01'")

# Add and modify columns
df.withColumn("upper_name", F.upper("name"))
df.withColumn("year", F.year("event_date"))
df.withColumnRenamed("old_name", "new_name")
df.drop("unwanted_col")

# Aggregations
df.groupBy("category").agg(
    F.count("*").alias("cnt"),
    F.sum("amount").alias("total"),
    F.avg("amount").alias("avg_amount"),
)

# Deduplication
df.dropDuplicates(["id", "event_date"])
df.distinct()
```

### Actions

```python
df.show(20, truncate=False)          # Display rows
df.count()                           # Row count
df.collect()                         # All rows to driver (use with care)
df.take(5)                           # First 5 rows to driver
df.describe("amount").show()         # Summary statistics
df.printSchema()                     # Column names and types
df.explain(True)                     # Physical + logical plan
```

### Window Functions

```python
w = Window.partitionBy("customer_id").orderBy(F.desc("order_date"))

df.withColumn("row_num", F.row_number().over(w))
df.withColumn("rank", F.rank().over(w))
df.withColumn("dense_rank", F.dense_rank().over(w))
df.withColumn("prev_amount", F.lag("amount", 1).over(w))
df.withColumn("next_amount", F.lead("amount", 1).over(w))

# Running total
w_running = Window.partitionBy("account").orderBy("txn_date").rowsBetween(
    Window.unboundedPreceding, Window.currentRow
)
df.withColumn("running_total", F.sum("amount").over(w_running))
```

### Joins

```python
joined = left.join(right, on="id", how="inner")
joined = left.join(right, left.key == right.key, "left")
joined = left.join(right, ["col1", "col2"], "full_outer")

# Anti join (rows in left not in right)
anti = left.join(right, on="id", how="left_anti")

# Broadcast small table
from pyspark.sql.functions import broadcast
joined = large.join(broadcast(small), on="id")
```

### UDFs

```python
from pyspark.sql.types import StringType

# Standard UDF (serialises to Python — slower)
@F.udf(returnType=StringType())
def clean_phone(phone):
    return phone.replace("-", "").replace(" ", "") if phone else None

df.withColumn("clean_phone", clean_phone("phone"))

# Pandas UDF (vectorised — much faster)
import pandas as pd
from pyspark.sql.functions import pandas_udf

@pandas_udf(StringType())
def normalise_name(s: pd.Series) -> pd.Series:
    return s.str.strip().str.title()

df.withColumn("normalised", normalise_name("name"))
```

### spark-submit

```bash
spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --num-executors 10 \
    --executor-memory 8g \
    --executor-cores 4 \
    --driver-memory 4g \
    --conf spark.sql.shuffle.partitions=200 \
    --conf spark.dynamicAllocation.enabled=true \
    --py-files libs.zip \
    main_job.py --date 2026-03-15
```

> [!tip] See [[PySpark Architecture and Core Concepts]] and [[PySpark DataFrame Operations]] for in-depth coverage.

---

## Airflow Quick Reference

### CLI Commands

```bash
# DAG management
airflow dags list                             # List all DAGs
airflow dags trigger my_dag                   # Trigger DAG run
airflow dags trigger my_dag --conf '{"key":"value"}'  # With config
airflow dags pause my_dag                     # Pause scheduling
airflow dags unpause my_dag                   # Resume scheduling
airflow dags test my_dag 2026-03-15           # Test DAG for a date (no DB writes)
airflow dags backfill my_dag -s 2026-01-01 -e 2026-03-01  # Backfill date range

# Task management
airflow tasks list my_dag                     # List tasks in DAG
airflow tasks test my_dag my_task 2026-03-15  # Test single task (no DB writes)
airflow tasks run my_dag my_task 2026-03-15   # Run single task
airflow tasks clear my_dag -t my_task -s 2026-03-15  # Clear task for re-run
airflow tasks failed-deps my_dag my_task 2026-03-15  # Show unmet dependencies

# Database and config
airflow db init                               # Initialise metadata database
airflow db upgrade                            # Apply migrations
airflow config list                           # Show all config settings

# Connections and variables
airflow connections list                      # List connections
airflow connections add my_conn --conn-type postgres --conn-host localhost
airflow variables set MY_VAR "value"          # Set variable
airflow variables get MY_VAR                  # Get variable
```

### DAG Structure

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.dates import days_ago
from datetime import timedelta

default_args = {
    "owner": "data-team",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": True,
    "email": ["data-alerts@example.com"],
}

with DAG(
    dag_id="daily_ingestion",
    default_args=default_args,
    schedule_interval="0 6 * * *",            # 06:00 UTC daily
    start_date=days_ago(1),
    catchup=False,
    tags=["ingestion", "production"],
    max_active_runs=1,
) as dag:
    extract = PythonOperator(task_id="extract", python_callable=extract_fn)
    transform = PythonOperator(task_id="transform", python_callable=transform_fn)
    load = PythonOperator(task_id="load", python_callable=load_fn)

    extract >> transform >> load
```

### Common Operators

| Operator | Purpose |
|----------|---------|
| `PythonOperator` | Run a Python callable |
| `BashOperator` | Execute a shell command |
| `DbtCloudRunJobOperator` | Trigger a dbt Cloud job |
| `SnowflakeOperator` | Execute SQL on Snowflake |
| `S3ToSnowflakeOperator` | Load S3 files into Snowflake |
| `HttpSensor` | Wait for an HTTP endpoint to return success |
| `ExternalTaskSensor` | Wait for a task in another DAG |
| `EmailOperator` | Send an email notification |
| `TriggerDagRunOperator` | Trigger another DAG |
| `BranchPythonOperator` | Conditional branching based on return value |

### Connections and Variables

```python
from airflow.hooks.base import BaseHook
from airflow.models import Variable

# Connections — stored credentials (UI, CLI, or env vars)
conn = BaseHook.get_connection("my_postgres")
host = conn.host
password = conn.password

# Environment variable format:
# AIRFLOW_CONN_MY_POSTGRES='postgresql://user:pass@host:5432/db'

# Variables — key-value config
env = Variable.get("ENVIRONMENT", default_var="dev")
config = Variable.get("pipeline_config", deserialize_json=True)
```

### Trigger Rules

| Rule | Fires When |
|------|-----------|
| `all_success` (default) | All upstream tasks succeeded |
| `all_failed` | All upstream tasks failed |
| `all_done` | All upstream tasks completed (any state) |
| `one_success` | At least one upstream succeeded |
| `one_failed` | At least one upstream failed |
| `none_failed` | No upstream task failed (success or skipped) |
| `none_skipped` | No upstream task was skipped |

### XCom (Cross-Communication)

```python
# Push — return value from PythonOperator is auto-pushed
def extract_fn(**context):
    row_count = run_extraction()
    return row_count  # Automatically pushed as XCom

# Pull — retrieve in downstream task
def transform_fn(**context):
    row_count = context["ti"].xcom_pull(task_ids="extract")
    print(f"Processing {row_count} rows")

# Explicit push
def custom_push(**context):
    context["ti"].xcom_push(key="file_path", value="s3://bucket/output.parquet")

# Pull specific key
def custom_pull(**context):
    path = context["ti"].xcom_pull(task_ids="custom_push", key="file_path")
```

> [!tip] See [[Apache Airflow Fundamentals]] and [[Apache Airflow Deep Dive]] for DAG design patterns and production configuration.

---
