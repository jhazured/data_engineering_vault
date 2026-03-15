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

---

## 5. Terraform

```bash
terraform init                        # Initialize providers/backend
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
