# Bash Deployment Patterns

Patterns from production data platform deployment scripts.

## Environment Variable Loading

The standard pattern for loading `.env` files in bash:

```bash
#!/bin/bash
set -e  # Exit on any error

# Load environment variables from .env
if [[ -f ".env" ]]; then
    set -a          # Automatically export all variables
    source .env
    set +a          # Stop auto-exporting
    echo "Environment variables loaded from .env"
fi
```

**Why `set -a` / `set +a`?** Without `set -a`, variables sourced from `.env` are local to the script. With `set -a`, they're exported to subprocesses (Python scripts, dbt commands, etc.).

### Environment-Specific Defaults

Use `${VAR:-default}` for cascading defaults:

```bash
case $ENVIRONMENT in
  dev)
    export SF_DATABASE="${SF_DATABASE:-DEV_T2_PERSISTENT_STAGING}"
    export SF_WAREHOUSE="${SF_WAREHOUSE:-IN_DEV_CENTRAL_WH}"
    export SF_ROLE="${SF_ROLE:-ENGINEER}"
    export DBT_THREADS="${DBT_THREADS:-4}"
    ;;
  prod)
    export SF_DATABASE="${SF_DATABASE:-PROD_T2_PERSISTENT_STAGING}"
    export SF_WAREHOUSE="${SF_WAREHOUSE:-IN_PROD_CENTRAL_WH}"
    export SF_ROLE="${SF_ROLE:-CHANGE_CONTROL}"
    export DBT_THREADS="${DBT_THREADS:-12}"
    ;;
  *)
    echo "Invalid environment. Use: dev, staging, or prod"
    exit 1
    ;;
esac
```

## Required Variable Validation

Fail fast if critical variables are missing:

```bash
required_vars=("SF_ACCOUNT" "SF_USER" "SF_PASSWORD" "SF_ROLE" "SF_WAREHOUSE")
missing_vars=()

for var in "${required_vars[@]}"; do
    if [[ -z "${!var}" ]]; then
        missing_vars+=("$var")
    fi
done

if [[ ${#missing_vars[@]} -gt 0 ]]; then
    echo "ERROR: Missing required environment variables: ${missing_vars[*]}"
    echo "Please check your .env file"
    exit 1
fi
```

**Key pattern:** `${!var}` is bash indirect expansion — it dereferences the variable whose name is stored in `$var`.

## Phased Deployment with Status Caching

Track completed phases in a status file to support idempotent re-runs:

```bash
DEPLOYMENT_STATUS_FILE="$PROJECT_ROOT/.deployment_status"

is_phase_completed() {
    local phase="$1"
    if [[ -f "$DEPLOYMENT_STATUS_FILE" ]]; then
        grep -q "^$phase:completed:" "$DEPLOYMENT_STATUS_FILE"
    else
        return 1
    fi
}

mark_phase_completed() {
    local phase="$1"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "$phase:completed:$timestamp" >> "$DEPLOYMENT_STATUS_FILE"
}

# Usage in deployment
setup_environment() {
    if is_phase_completed "1"; then
        echo "Phase 1 already completed - skipping"
        return 0
    fi

    # ... do phase 1 work ...

    mark_phase_completed "1"
}
```

**Benefits:**
- Re-running `deploy_all.sh` skips completed phases
- `--reset` flag clears the status file for a clean run
- Timestamps provide audit trail

## Script Directory Resolution

Reliably find the script's own directory (works from any `cwd`):

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/../../.." && pwd)"
```

Then reference all paths relative to these anchors:

```bash
SQL_TEMPLATE="$SCRIPT_DIR/../tasks/1.2 Create GitLab Integration.sql"
source "$PROJECT_ROOT/.env"
cd "$PROJECT_ROOT/dbt"
```

## dbt Command Wrapper

A wrapper script that loads `.env` and sets `DBT_PROFILES_DIR` before invoking dbt:

```bash
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
set -a
source "$SCRIPT_DIR/.env"
set +a
export DBT_PROFILES_DIR="$SCRIPT_DIR/dbt"
cd "$SCRIPT_DIR/dbt"
dbt "$@"
```

**Usage:** `./dbt_cmd.sh run --select tag:load_priority_1`

This avoids users needing to remember `set -a && source .env && set +a` and `cd dbt` every time.

## Coloured Output

Helper functions for deployment status messages:

```bash
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'  # No Color

print_status()  { echo -e "${BLUE}[INFO]${NC} $1"; }
print_success() { echo -e "${GREEN}[SUCCESS]${NC} $1"; }
print_warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
print_error()   { echo -e "${RED}[ERROR]${NC} $1"; exit 1; }
```

## SQL Execution via Python

When shell-based SQL runners (`snowsql`) aren't available, use a Python helper:

```bash
execute_sql_script() {
    local sql_file="$1"
    local description="$2"

    if [[ -f "$sql_file" ]]; then
        echo "Executing: $description"
        cd "$PROJECT_ROOT/scripts/01_setup/handlers"
        python3 execute_sql_python.py "$sql_file"
        local exit_code=$?
        cd "$PROJECT_ROOT"

        if [[ $exit_code -eq 0 ]]; then
            echo "SUCCESS: $description"
        else
            echo "ERROR: $description failed (exit $exit_code)"
            exit 1
        fi
    else
        echo "WARNING: SQL script not found: $sql_file"
    fi
}
```

## Dry Run Support

Enable a dry-run mode for testing deployment logic without executing:

```bash
if [[ "${DRY_RUN:-false}" == true ]]; then
    echo "[DRY RUN] Would execute SQL: $description"
    echo "          File: $sql_file"
    return 0
fi
```

Run with: `DRY_RUN=true ./deploy.sh`

## Temp File Cleanup with trap

Always clean up temporary files, even on error:

```bash
TEMP_SQL=$(mktemp /tmp/gitlab_integration_XXXXXX.sql)
trap 'rm -f "$TEMP_SQL"' EXIT

# ... use $TEMP_SQL ...
# Automatically cleaned up when script exits
```

## Deployment Entry Point Pattern

A simple entry script delegates to the main orchestrator:

```bash
#!/bin/bash
set -e
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

if [[ -f "$SCRIPT_DIR/.env" ]]; then
    set -a && source "$SCRIPT_DIR/.env" && set +a
fi

echo "Options:"
echo "  --skip-data    Skip data generation and loading"
echo "  --reset        Reset deployment status and run all phases"

exec "$SCRIPT_DIR/scripts/02_deployment/handlers/deploy_all.sh" "$@"
```

`exec` replaces the current shell with the target script — cleaner process tree and the exit code propagates correctly.

## Makefile Patterns

Use Make as a convenience layer over complex commands:

```makefile
.PHONY: load verify test teardown bundle dry-run workbook

load:
	python scripts/load_books_to_snowflake.py --mode incremental

dry-run:
	python scripts/load_books_to_snowflake.py --dry-run

verify:
	python scripts/setup/verify_setup.py

test:
	pytest tests/ -v --tb=short

workbook:
	python scripts/queries_to_workbook.py

teardown:
	python scripts/setup/snowflake_teardown.py

bundle:
	@echo "Building review_bundle.txt..."
	@echo "" > review_bundle.txt
	@git ls-files | while read f; do \
		echo "\n\n### FILE: ./$$f\n" >> review_bundle.txt; \
		cat "$$f" >> review_bundle.txt; \
	done
	@echo "Done: review_bundle.txt"
```

**Key patterns:**
- `.PHONY` — ensures targets always run (not file-based)
- `bundle` — concatenates all git-tracked files into a single review file (useful for AI code review)
- Simple wrappers — `make load` is easier to remember than the full python command with flags

---

## Production Deployment Scripts

Patterns from `gcp_datamigration` for deploying ETL jobs across environments.

### GCS Staging with gsutil

Upload job files to environment-specific [[Google Cloud Storage]] paths:

```bash
#!/usr/bin/env bash
set -e

while [[ "$#" -gt 0 ]]; do
  case $1 in
    --env) ENV="$2"; shift ;;
    --script) ETL_SCRIPT="$2"; shift ;;
    *) echo "Unknown parameter passed: $1"; exit 1 ;;
  esac
  shift
done

if [ -z "$ENV" ] || [ -z "$ETL_SCRIPT" ]; then
  echo "Usage: deploy_etl.sh --env ENV --script ETL_SCRIPT"
  exit 1
fi

gsutil cp "etl_jobs/${ETL_SCRIPT}.py" "gs://my-etl-bucket/${ENV}/jobs/"
```

The `--env` flag drives the GCS path, so the same script works for `dev`, `test`, `uat`, and `prod` without modification.

### Cloud Composer DAG Triggering

Trigger an [[Apache Airflow]] DAG through [[Cloud Composer]] after deployment:

```bash
gcloud composer environments run my-composer-env \
  --location us-central1 \
  trigger_dag -- "my_${ETL_SCRIPT}_dag"
```

This couples the deploy and run steps: `deploy_etl.sh` stages the file, then `run_etl.sh` kicks off the DAG. Both accept the same `--env` / `--script` interface, keeping invocation consistent.

### Cleanup Scripts

Remove deployed artifacts when decommissioning a job:

```bash
gsutil rm -r "gs://my-etl-bucket/${ENV}/jobs/${ETL_SCRIPT}/"
```

The `-r` flag handles recursive deletion for jobs that produce subdirectories. The delete script uses the same argument parsing as deploy/run, so the three scripts form a symmetric lifecycle: **deploy, run, delete**.

## Development Environment Setup

From `setup_environment.sh` — a task-runner pattern where a single script handles the full local workflow.

### Virtualenv Creation and Dependency Installation

```bash
setup_python_env() {
    if [ ! -d "venv" ]; then
        python3 -m venv venv
    fi
    source venv/bin/activate

    if [ -f "requirements/dev.txt" ]; then
        pip install -r requirements/dev.txt
    elif [ -f "requirements.txt" ]; then
        pip install -r requirements.txt
    else
        pip install pytest pytest-cov pytest-html
    fi
}
```

The cascading `if/elif/else` on requirements files makes the script portable across projects that structure dependencies differently.

### Prerequisite Checking

Guard against missing tools before any real work happens:

```bash
command_exists() {
    command -v "$1" >/dev/null 2>&1
}

check_prerequisites() {
    if ! command_exists docker; then
        print_error "Docker is not installed or not in PATH"
        exit 1
    fi
    if ! command_exists gcloud; then
        print_error "gcloud CLI is not installed or not in PATH"
        exit 1
    fi
}
```

`command -v` is preferred over `which` — it is a [[Bash]] builtin and works consistently across shells.

### Case-Based Task Dispatch

Use `case` as a subcommand router at the bottom of the script:

```bash
case "${1:-help}" in
    setup)    setup_python_env ;;
    lint)     setup_python_env; run_linting ;;
    test)     setup_python_env; run_tests ;;
    build)    check_prerequisites; build_docker_images ;;
    deploy)   check_prerequisites; deploy_to_gcp ;;
    pipeline) run_full_pipeline ;;
    cleanup)  cleanup ;;
    help)     show_help ;;
    *)        print_error "Unknown task: $1"; show_help; exit 1 ;;
esac
```

`${1:-help}` defaults to `help` when no argument is given, so running the script bare always prints usage.

## Test Runner Scripts

### Pytest with Coverage and Reports

From `run_pytest.sh` — a comprehensive test runner that builds a pytest command dynamically:

```bash
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

build_pytest_command() {
    local cmd="python3 -m pytest"

    if [[ -n "${TEST_PATH}" ]]; then
        cmd="${cmd} ${TEST_PATH}"
    else
        cmd="${cmd} tests/"
    fi

    if [[ "${COVERAGE}" == "true" ]]; then
        cmd="${cmd} --cov=framework --cov=etls --cov-report=term-missing"
        cmd="${cmd} --cov-report=xml:${COVERAGE_DIR}/coverage.xml"
    fi

    if [[ "${PARALLEL}" == "true" ]]; then
        cmd="${cmd} -n auto"
    fi

    echo "${cmd}"
}
```

The function accumulates flags as strings, then returns the full command via `echo`. The caller runs it with `eval "$(build_pytest_command)"`.

### Docker-Compose Test Profiles

Run tests inside a container with a dependent database:

```bash
docker-compose --profile test up -d test_db

docker-compose --profile test run --rm \
    -e TESTING=true \
    -e ENV="${ENVIRONMENT}" \
    etl_test bash -c "${pytest_cmd}"

docker-compose --profile test down
```

[[Docker Compose]] profiles (`test`, `dev`, `ci`, `jupyter`) keep service definitions in one file while only starting what each workflow needs.

### Interactive Debug Shell

From `run_bash.sh` — drop into a container for manual debugging:

```bash
docker-compose run --rm --name "$CONTAINER_NAME" \
    -v "$PROJECT_ROOT:/app" \
    -w /app \
    app bash
```

The `--rm` flag ensures the container is removed on exit. Volume mounts allow live editing of source files on the host while running inside the container.

## Bash Scripting Fundamentals for DE

### Strict Mode

Always start scripts with strict error handling:

```bash
set -euo pipefail
```

| Flag | Effect |
|------|--------|
| `-e` | Exit immediately on non-zero exit code |
| `-u` | Treat unset variables as errors |
| `-o pipefail` | Propagate failures through pipes (not just last command) |

Most project scripts use at least `set -e`. The full `set -euo pipefail` (as in `run_bash.sh`) is the safest default.

### Trap for Cleanup

Register a cleanup function that runs on script exit, regardless of success or failure:

```bash
trap cleanup_containers EXIT
```

Combine with temp files:

```bash
TEMP_FILE=$(mktemp /tmp/etl_XXXXXX)
trap 'rm -f "$TEMP_FILE"' EXIT
```

### Parameter Validation

The `while/case/shift` pattern for named arguments:

```bash
while [[ "$#" -gt 0 ]]; do
  case $1 in
    --env) ENV="$2"; shift ;;
    --script) ETL_SCRIPT="$2"; shift ;;
    *) echo "Unknown parameter: $1"; exit 1 ;;
  esac
  shift
done
```

Follow with a guard clause:

```bash
if [ -z "$ENV" ] || [ -z "$ETL_SCRIPT" ]; then
  echo "Usage: script.sh --env ENV --script ETL_SCRIPT"
  exit 1
fi
```

### Logging Functions

Standardise output with severity-level helpers:

```bash
log_info()    { echo -e "${BLUE}[INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[SUCCESS]${NC} $1"; }
log_warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
log_error()   { echo -e "${RED}[ERROR]${NC} $1"; }
```

These appear across every script in the project. Consistent prefixes make log output easy to scan and filter with [[grep]].

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error (missing args, unknown command) |
| `2` | Prerequisite not met (Docker not running, gcloud not authenticated) |

Always `exit 1` on validation failure rather than letting the script continue into undefined state.

## Common Patterns

### Environment Variable Guards

```bash
IS_DOCKER=${IS_DOCKER:-false}
IS_JENKINS=${JENKINS_URL:+true}
```

`${VAR:-default}` supplies a fallback. `${VAR:+value}` returns `value` only if `VAR` is set and non-empty — useful for detecting CI environments by their injected variables.

### Conditional Execution

```bash
if [[ -f "requirements/dev.txt" ]]; then
    pip install -r requirements/dev.txt
fi
```

```bash
docker image inspect ${IMAGE}:latest >/dev/null 2>&1 || build_image
```

The `||` short-circuit is cleaner than a full `if/fi` for single fallback actions.

### Timestamp Generation

```bash
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
```

Used for log file names, output directories, and deployment audit trails.

### Heredocs for Multi-Line Strings

Generate config files or display help text without escaping:

```bash
cat > "$env_file" << EOF
GCP_PROJECT_ID=${GCP_PROJECT_ID:-my-gcp-project}
GCP_REGION=${GCP_REGION:-australia-southeast1}
ENVIRONMENT=${ENVIRONMENT:-dev}
EOF
```

Variables inside the heredoc are expanded. Use `<< 'EOF'` (quoted delimiter) to suppress expansion when writing literal template content.

### Colour-Coded Output

Define colours once at the top of the script:

```bash
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'
```

Then use via `echo -e` or the logging functions above. Every script in the project follows this convention.

## Makefile as Script Orchestrator

From the project `makefile` — [[Make]] wraps [[Docker Compose]] commands into memorable targets.

### Common Targets

```makefile
DC=docker compose

build:
	$(DC) build

test:
	$(DC) run --rm etl_test

up:
	$(DC) up

stop:
	$(DC) down

clean:
	$(DC) down -v --remove-orphans
```

### Aggressive Cleanup Target

```makefile
clean-volumes:
	@echo "Stopping containers and removing volumes..."
	docker compose down -v --remove-orphans
	@echo "Pruning dangling volumes..."
	docker volume prune -f
```

The `@` prefix suppresses command echo, keeping output clean.

### PHONY Targets

Every target in a script-orchestrator Makefile should be `.PHONY` since none produce files:

```makefile
.PHONY: build test up stop clean clean-volumes help
```

Without `.PHONY`, Make would skip a target if a file with the same name existed in the directory.

### Help Target Pattern

A self-documenting help target using comments:

```makefile
.PHONY: help
help:  ## Show available targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'
```

Then annotate each target: `build: ## Build all Docker images`. Running `make help` prints a formatted list.

### Variable Passing

Pass environment at invocation time:

```bash
make test ENV=prod
make build DC="docker-compose -f docker-compose.prod.yml"
```

Makefile variables set on the command line override those defined inside the file, making targets reusable across environments.

---

## Bash Scripting Fundamentals — Comprehensive Reference

The sections above cover deployment-specific patterns. What follows is a broader reference for the [[Bash]] language itself, focused on patterns data engineers use daily.

### Script Structure

Every production script should follow this skeleton:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# --- Constants ---------------------------------------------------------------
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/tmp/${SCRIPT_NAME%.sh}_$(date +%Y%m%d_%H%M%S).log"

# --- Cleanup -----------------------------------------------------------------
cleanup() {
    local exit_code=$?
    rm -f "$TEMP_FILE" 2>/dev/null
    [[ $exit_code -ne 0 ]] && echo "Script failed with exit code $exit_code" >&2
    exit "$exit_code"
}
trap cleanup EXIT

# --- Functions ---------------------------------------------------------------
main() {
    # Script logic here
    :
}

# --- Entry Point -------------------------------------------------------------
main "$@"
```

**Why `#!/usr/bin/env bash`** rather than `#!/bin/bash`? The `env` lookup finds bash wherever it is installed, which matters on macOS (where `/bin/bash` is ancient 3.2) and on NixOS where binaries live outside `/bin`.

**Why `IFS=$'\n\t'`?** The default `IFS` includes space, which causes word-splitting on filenames with spaces. Restricting to newline and tab prevents a large class of bugs when iterating over file lists or command output.

**Why wrap in `main()`?** If the script is sourced rather than executed, a bare top-level block runs immediately. Wrapping in `main "$@"` and calling it at the bottom means the entire file is parsed before execution begins, and sourcing the file without calling `main` is safe.

### Variables and Parameter Expansion

Beyond `${VAR:-default}` (covered above), bash offers a rich set of parameter expansion operators.

#### Substitution Operators

| Syntax | Behaviour |
|--------|-----------|
| `${var:-default}` | Use `default` if `var` is unset or empty |
| `${var-default}` | Use `default` only if `var` is unset (empty string is kept) |
| `${var:=default}` | Assign `default` if `var` is unset or empty |
| `${var:+alt}` | Use `alt` only if `var` is set and non-empty |
| `${var:?error msg}` | Exit with error if `var` is unset or empty |

```bash
# Practical example: mandatory config
DB_HOST="${DB_HOST:?DB_HOST must be set in .env}"
DB_PORT="${DB_PORT:-5432}"
SCHEMA="${TARGET_SCHEMA:+--schema $TARGET_SCHEMA}"  # flag only if set
```

#### String Trimming Operators

| Syntax | Behaviour |
|--------|-----------|
| `${var%pattern}` | Remove shortest match from the end |
| `${var%%pattern}` | Remove longest match from the end |
| `${var#pattern}` | Remove shortest match from the beginning |
| `${var##pattern}` | Remove longest match from the beginning |

```bash
filepath="/data/warehouse/raw/customers_20250315.csv.gz"

echo "${filepath##*/}"      # customers_20250315.csv.gz  (basename)
echo "${filepath%.*}"       # /data/warehouse/raw/customers_20250315.csv  (strip .gz)
echo "${filepath%%.*}"      # /data/warehouse/raw/customers_20250315     (strip all extensions)
echo "${filepath#/data/}"   # warehouse/raw/customers_20250315.csv.gz

# Extract date from filename
filename="${filepath##*/}"          # customers_20250315.csv.gz
date_part="${filename%.*}"          # customers_20250315.csv
date_part="${date_part%.*}"         # customers_20250315
date_part="${date_part##*_}"        # 20250315
```

#### Length, Substring, and Case

```bash
var="Hello World"
echo "${#var}"              # 11  (string length)
echo "${var:6}"             # World  (substring from offset 6)
echo "${var:0:5}"           # Hello  (substring offset 0, length 5)
echo "${var^^}"             # HELLO WORLD  (uppercase)
echo "${var,,}"             # hello world  (lowercase)
echo "${var^}"              # Hello World  (capitalise first char)
```

#### Array Parameter Expansion

```bash
tables=("customers" "orders" "products" "inventory")

echo "${#tables[@]}"        # 4  (array length)
echo "${tables[@]:1:2}"     # orders products  (slice from index 1, 2 elements)
echo "${tables[-1]}"        # inventory  (last element, bash 4.3+)
```

### Conditionals and Loops

#### `[[ ]]` vs `[ ]`

Always prefer `[[ ]]` in bash scripts. It is a bash keyword (not an external command), supports pattern matching, regex, and logical operators without quoting pitfalls.

| Feature | `[ ]` (POSIX) | `[[ ]]` (Bash) |
|---------|---------------|-----------------|
| Word splitting on variables | Yes (must quote) | No |
| Pattern matching | No | `[[ $x == glob* ]]` |
| Regex | No | `[[ $x =~ regex ]]` |
| Logical AND/OR | `-a` / `-o` | `&&` / `\|\|` |
| Portability | All POSIX shells | Bash, zsh, ksh |

```bash
# Pattern matching
[[ "$ENV" == prod* ]] && echo "Production environment detected"

# Regex matching
if [[ "$filename" =~ ^[0-9]{8}_(.+)\.csv$ ]]; then
    table_name="${BASH_REMATCH[1]}"
    echo "Importing into table: $table_name"
fi

# Compound conditions
if [[ -f "$config_file" && -r "$config_file" ]]; then
    source "$config_file"
fi
```

#### For Loops

```bash
# Iterate over array
schemas=("raw" "staging" "mart")
for schema in "${schemas[@]}"; do
    echo "Processing schema: $schema"
done

# C-style for loop
for ((i = 0; i < ${#schemas[@]}; i++)); do
    echo "Schema $i: ${schemas[$i]}"
done

# Iterate over files (never parse ls output)
for csv_file in /data/landing/*.csv; do
    [[ -e "$csv_file" ]] || continue   # guard against no matches
    process_file "$csv_file"
done

# Iterate over command output (line by line)
while IFS= read -r line; do
    echo "Processing: $line"
done < <(find /data/raw -name "*.parquet" -mtime -1)
```

#### While and Until

```bash
# Wait for a service to become available
max_retries=30
attempt=0
until pg_isready -h "$DB_HOST" -p "$DB_PORT" -q 2>/dev/null; do
    ((attempt++))
    if [[ $attempt -ge $max_retries ]]; then
        echo "Database not ready after $max_retries attempts" >&2
        exit 1
    fi
    echo "Waiting for database... (attempt $attempt/$max_retries)"
    sleep 2
done

# Read a file line by line
while IFS=',' read -r id name email; do
    echo "User: $name ($email)"
done < users.csv
```

#### Case Statements — Advanced

```bash
# Route log levels
case "${LOG_LEVEL:-INFO}" in
    DEBUG|TRACE)
        set -x
        verbose=true
        ;;
    INFO)
        verbose=false
        ;;
    WARN|ERROR)
        verbose=false
        quiet=true
        ;;
    *)
        echo "Unknown log level: $LOG_LEVEL" >&2
        exit 1
        ;;
esac

# Match file types for processing
case "$filename" in
    *.csv)       load_csv "$filename" ;;
    *.json)      load_json "$filename" ;;
    *.parquet)   load_parquet "$filename" ;;
    *.csv.gz)    zcat "$filename" | load_csv /dev/stdin ;;
    *)           echo "Unsupported format: $filename" >&2; return 1 ;;
esac
```

### Functions

#### Local Variables and Return Codes

```bash
calculate_partition_key() {
    local input_date="$1"
    local format="${2:-"%Y/%m/%d"}"

    # Validate input
    if ! date -d "$input_date" +"%s" &>/dev/null; then
        echo "Invalid date: $input_date" >&2
        return 1
    fi

    local partition
    partition="$(date -d "$input_date" +"$format")"
    echo "$partition"
    return 0
}

# Usage: capture output, check return code
if partition=$(calculate_partition_key "2025-03-15"); then
    echo "Partition: $partition"
else
    echo "Failed to calculate partition" >&2
fi
```

**Always declare variables `local`** inside functions. Without `local`, variables leak into the global scope and cause subtle bugs in larger scripts.

#### Passing Arrays to Functions

Bash cannot pass arrays by reference (before bash 4.3 namerefs). The common workaround is to pass the array elements as positional parameters:

```bash
process_tables() {
    local env="$1"
    shift
    local tables=("$@")

    for table in "${tables[@]}"; do
        echo "Loading $table in $env"
    done
}

tables=("customers" "orders" "products")
process_tables "prod" "${tables[@]}"
```

With bash 4.3+ namerefs:

```bash
process_tables_v2() {
    local env="$1"
    local -n table_ref="$2"    # nameref

    for table in "${table_ref[@]}"; do
        echo "Loading $table in $env"
    done
}

tables=("customers" "orders" "products")
process_tables_v2 "prod" tables
```

#### Function Return Patterns

```bash
# Pattern 1: echo the result, capture with $()
get_row_count() {
    local table="$1"
    local count
    count=$(psql -t -c "SELECT count(*) FROM $table" 2>/dev/null)
    echo "${count// /}"   # trim whitespace
}

# Pattern 2: set a global variable (avoid in complex scripts)
RESULT=""
compute_hash() {
    RESULT=$(sha256sum "$1" | cut -d' ' -f1)
}

# Pattern 3: return 0/1 for boolean checks
is_table_empty() {
    local count
    count=$(get_row_count "$1")
    [[ "$count" -eq 0 ]]
}

if is_table_empty "staging.raw_events"; then
    echo "No data to process"
fi
```

### Error Handling

#### Trap ERR and Error Propagation

```bash
#!/usr/bin/env bash
set -euo pipefail

on_error() {
    local exit_code=$?
    local line_no=$1
    echo "ERROR: Script failed at line $line_no with exit code $exit_code" >&2
    echo "ERROR: Command: ${BASH_COMMAND}" >&2

    # Optional: send alert
    # curl -X POST "$SLACK_WEBHOOK" -d "{\"text\": \"Pipeline failed: $BASH_COMMAND\"}"
}
trap 'on_error $LINENO' ERR

# Now any failing command prints the line number and command before exiting
```

**Important:** `trap ERR` fires on any command that returns non-zero. If you intentionally allow a command to fail, use `|| true`:

```bash
# This will NOT trigger ERR trap
grep -q "HEADER" "$file" || true

# Alternatively, temporarily disable
set +e
risky_command
rc=$?
set -e
[[ $rc -ne 0 ]] && echo "risky_command failed with $rc, continuing..."
```

#### Structured Logging Pattern

```bash
readonly LOG_FILE="/var/log/etl/pipeline_$(date +%Y%m%d).log"

log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp
    timestamp="$(date '+%Y-%m-%d %H:%M:%S')"
    echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"
}

log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@"; }
log_error() { log "ERROR" "$@" >&2; }

# Usage
log_info "Starting load for table: $table"
log_warn "Row count below threshold: $count < $min_expected"
log_error "Connection to $DB_HOST failed after $max_retries attempts"
```

#### Retry with Exponential Back-Off

```bash
retry() {
    local max_attempts="${1:-3}"
    local delay="${2:-2}"
    shift 2
    local cmd=("$@")

    local attempt=1
    while true; do
        if "${cmd[@]}"; then
            return 0
        fi

        if [[ $attempt -ge $max_attempts ]]; then
            log_error "Command failed after $max_attempts attempts: ${cmd[*]}"
            return 1
        fi

        log_warn "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
        delay=$((delay * 2))
    done
}

# Usage
retry 5 3 curl -sf "https://api.example.com/health"
retry 3 5 psql -h "$DB_HOST" -c "SELECT 1"
```

### String Manipulation

#### Substring Replacement

```bash
connection_string="host=prod-db.internal;port=5432;db=warehouse"

# Replace first occurrence
echo "${connection_string/prod/dev}"
# host=dev-db.internal;port=5432;db=warehouse

# Replace all occurrences
path="s3://bucket/raw/data/raw/archive"
echo "${path//raw/processed}"
# s3://bucket/processed/data/processed/archive
```

#### String Splitting

```bash
# Split on a delimiter
IFS=',' read -ra columns <<< "id,name,email,created_at"
for col in "${columns[@]}"; do
    echo "Column: $col"
done

# Split a path
IFS='/' read -ra path_parts <<< "s3://bucket/env/schema/table"
bucket="${path_parts[2]}"
schema="${path_parts[4]}"
```

#### Regex Matching with BASH_REMATCH

```bash
log_line='2025-03-15 14:32:01 [ERROR] Connection timeout after 30s (host=db-prod-01)'

if [[ "$log_line" =~ ^([0-9]{4}-[0-9]{2}-[0-9]{2})\ ([0-9:]+)\ \[([A-Z]+)\]\ (.+)$ ]]; then
    date="${BASH_REMATCH[1]}"
    time="${BASH_REMATCH[2]}"
    level="${BASH_REMATCH[3]}"
    message="${BASH_REMATCH[4]}"
    echo "Date=$date Time=$time Level=$level Message=$message"
fi
```

### File Operations

#### Test Operators

| Operator | Tests |
|----------|-------|
| `-f file` | Regular file exists |
| `-d dir` | Directory exists |
| `-s file` | File exists and is non-empty |
| `-r file` | File is readable |
| `-w file` | File is writable |
| `-x file` | File is executable |
| `-L file` | File is a symbolic link |
| `-e path` | Path exists (any type) |
| `-nt` / `-ot` | Newer than / older than (by modification time) |

```bash
# Guard against missing data files
data_dir="/data/landing/$(date +%Y%m%d)"
if [[ ! -d "$data_dir" ]]; then
    log_error "Landing directory not found: $data_dir"
    exit 1
fi

# Check files arrived and are non-empty
for expected in customers.csv orders.csv products.csv; do
    if [[ ! -s "$data_dir/$expected" ]]; then
        log_error "Missing or empty file: $expected"
        exit 1
    fi
done

# Only reprocess if source is newer than target
if [[ "$source_file" -nt "$target_file" ]]; then
    log_info "Source updated — reprocessing"
    transform "$source_file" > "$target_file"
fi
```

#### Find with Exec

```bash
# Delete files older than 30 days
find /data/archive -name "*.csv.gz" -mtime +30 -exec rm {} +

# Process all parquet files modified today
find /data/raw -name "*.parquet" -mtime 0 -exec ./load_file.sh {} \;

# Count lines across all CSVs (using + for batching)
find /data/landing -name "*.csv" -exec wc -l {} +

# Find large files that may need splitting
find /data/staging -name "*.csv" -size +1G -printf '%s %p\n' | sort -rn
```

#### Process Substitution

Process substitution (`<()` and `>()`) lets you treat command output as a file. Invaluable for comparing datasets:

```bash
# Compare sorted output of two queries without temp files
diff <(psql -t -c "SELECT id FROM prod.customers ORDER BY id") \
     <(psql -t -c "SELECT id FROM staging.customers ORDER BY id")

# Feed multiple inputs to a command
paste <(cut -d',' -f1 file_a.csv) <(cut -d',' -f3 file_b.csv)

# Iterate over output without creating a subshell (unlike piping)
while IFS= read -r file; do
    count=$((count + 1))
    process "$file"
done < <(find /data -name "*.json")
# $count is available here because the while loop ran in the current shell
```

### Process Management

#### Background Jobs and Wait

```bash
# Run multiple independent loads in parallel
load_table "customers" &
pid_customers=$!
load_table "orders" &
pid_orders=$!
load_table "products" &
pid_products=$!

# Wait for all, capture failures
failed=0
for pid in $pid_customers $pid_orders $pid_products; do
    if ! wait "$pid"; then
        ((failed++))
    fi
done

if [[ $failed -gt 0 ]]; then
    log_error "$failed table loads failed"
    exit 1
fi
```

#### Controlled Parallel Execution

Limit concurrency to avoid overwhelming a database or API:

```bash
max_parallel=4
running=0

for table in "${tables[@]}"; do
    load_table "$table" &
    ((running++))

    if [[ $running -ge $max_parallel ]]; then
        wait -n    # wait for ANY one job to finish (bash 4.3+)
        ((running--))
    fi
done

# Wait for remaining jobs
wait
```

#### Named Pipes (FIFOs)

Named pipes are useful for streaming data between processes without intermediate files:

```bash
fifo="/tmp/etl_pipe_$$"
mkfifo "$fifo"
trap 'rm -f "$fifo"' EXIT

# Producer: extract from API, write to pipe
curl -sS "https://api.example.com/export" > "$fifo" &

# Consumer: load from pipe into database
psql -c "\COPY staging.events FROM '$fifo' WITH CSV HEADER"

# The two processes run concurrently — no intermediate file on disc
```

#### Subshell Isolation

Run a block in a subshell to prevent `cd` or variable changes from affecting the parent:

```bash
(
    cd /opt/dbt_project
    source .env
    dbt run --select tag:nightly
)
# Back in the original directory, .env variables not leaked
```

### Log Parsing Patterns

Common one-liners for data engineering log analysis using [[grep]], [[AWK]], and [[sed]].

#### Extract Errors from Application Logs

```bash
# All ERROR lines with timestamps
grep -E '^\d{4}-\d{2}-\d{2}.*\[ERROR\]' /var/log/etl/pipeline.log

# Errors from the last hour
awk -v cutoff="$(date -d '1 hour ago' '+%Y-%m-%d %H:%M')" \
    '$0 ~ /\[ERROR\]/ && $1" "$2 >= cutoff' /var/log/etl/pipeline.log

# Unique error messages with counts, sorted
grep '\[ERROR\]' pipeline.log | sed 's/^.*\[ERROR\] //' | sort | uniq -c | sort -rn
```

#### Parse Timestamps and Measure Duration

```bash
# Extract start and end times from a log
start=$(grep -m1 'Pipeline started' pipeline.log | awk '{print $1, $2}')
end=$(grep -m1 'Pipeline completed' pipeline.log | awk '{print $1, $2}')

start_epoch=$(date -d "$start" +%s)
end_epoch=$(date -d "$end" +%s)
duration_mins=$(( (end_epoch - start_epoch) / 60 ))
echo "Pipeline ran for $duration_mins minutes"
```

#### Count Patterns and Summarise

```bash
# Requests per minute from an access log
awk '{print $4}' access.log | cut -d: -f1-3 | sort | uniq -c | sort -rn | head -20

# Row counts per table from dbt output
grep -oP 'SUCCESS \d+ of \d+.*CREATE TABLE.*?(\S+)' dbt.log

# Summarise HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Extract slow queries (over 10s) from Postgres log
awk '/duration:/ {
    match($0, /duration: ([0-9.]+) ms/, arr)
    if (arr[1]+0 > 10000) print $0
}' postgresql.log
```

#### AWK for Structured Log Processing

```bash
# Parse CSV-like logs: sum a numeric column
awk -F',' '{sum += $3} END {printf "Total rows loaded: %d\n", sum}' load_report.csv

# Group by and aggregate
awk -F',' '
NR > 1 {
    table = $1
    rows[table] += $3
    files[table]++
}
END {
    for (t in rows)
        printf "%-30s %8d rows  %4d files\n", t, rows[t], files[t]
}' load_report.csv

# Multi-line log parsing: extract stack traces
awk '/\[ERROR\]/{found=1; buf=$0; next}
     found && /^[[:space:]]/{buf = buf "\n" $0; next}
     found {print buf; found=0}
     END {if(found) print buf}' application.log
```

#### Sed for Log Transformation

```bash
# Mask sensitive data before sharing logs
sed -E 's/(password=)[^& ]+/\1*****/gi; s/(token=)[^& ]+/\1*****/gi' app.log

# Convert log timestamps to ISO 8601
sed -E 's|^([0-9]{2})/([A-Za-z]{3})/([0-9]{4}):|\3-\2-\1T|' access.log

# Extract SQL statements from verbose logs
sed -n '/Executing SQL:/,/^$/p' pipeline.log
```

### Best Practices

#### Quoting Rules

The cardinal rule: **always double-quote variable expansions** unless you specifically need word splitting or glob expansion.

```bash
# WRONG: breaks on filenames with spaces
for f in $(ls /data/raw/*.csv); do process "$f"; done

# RIGHT: glob directly, quote the variable
for f in /data/raw/*.csv; do
    [[ -e "$f" ]] || continue
    process "$f"
done

# WRONG: unquoted $@ loses argument boundaries
run_cmd() { some_command $@; }

# RIGHT: "$@" preserves each argument as a separate word
run_cmd() { some_command "$@"; }
```

Exceptions where quoting is unnecessary:
- Inside `[[ ]]` (no word splitting)
- Arithmetic context `$(( ))`
- Array index `${arr[0]}`

#### ShellCheck

[[ShellCheck]] is a static analysis tool for shell scripts. Run it on every script before committing:

```bash
# Install
apt-get install shellcheck    # Debian/Ubuntu
brew install shellcheck       # macOS

# Run
shellcheck deploy.sh

# Integrate with CI
shellcheck scripts/**/*.sh || exit 1
```

Common issues ShellCheck catches:
- Unquoted variables (`SC2086`)
- Useless use of `cat` (`SC2002`)
- Using `ls` in a for loop (`SC2045`)
- Variable used before assignment (`SC2154`)

Add directives to suppress false positives:

```bash
# shellcheck disable=SC2034  # variable appears unused but is exported
readonly CONFIG_VERSION="2.1"
```

#### Portability: Bash vs Sh

| Feature | `sh` (POSIX) | `bash` |
|---------|--------------|--------|
| `[[ ]]` | No | Yes |
| Arrays | No | Yes |
| `$()` command substitution | Yes | Yes |
| `${var//pat/repl}` | No | Yes |
| `<<<` here-strings | No | Yes |
| `set -o pipefail` | No (varies) | Yes |
| Process substitution `<()` | No | Yes |

If the script must run on Alpine (which uses `ash`/`busybox`), Debian minimal containers, or embedded systems, stick to POSIX sh. For data engineering work on controlled infrastructure, bash is the pragmatic choice — just ensure the shebang says `bash`, not `sh`.

#### When to Switch to [[Python]]

Bash is excellent for orchestration, file management, and glue. Switch to Python when you need:

- **Complex data transformation** — anything beyond simple column extraction. If you are writing nested `awk` with associative arrays, Python with [[Pandas]] or [[Polars]] will be clearer.
- **JSON/YAML/XML parsing** — `jq` handles simple JSON, but nested transformations or schema validation belong in Python.
- **API interaction** — authentication flows, pagination, error handling, and rate limiting are painful in curl-based scripts.
- **Database operations** — anything beyond a single query. Connection pooling, transactions, and parameterised queries need a proper driver.
- **Error handling with context** — Python exceptions carry stack traces and can be caught selectively; bash `set -e` is all-or-nothing.
- **Unit testing** — bash scripts can be tested with `bats`, but Python's `pytest` is vastly more capable.

A good rule of thumb: **if the script exceeds 200 lines or handles structured data, rewrite in Python.** Use bash to call the Python script, manage environment variables, and handle file logistics around it.

```bash
# Good pattern: bash for orchestration, Python for logic
#!/usr/bin/env bash
set -euo pipefail

source "$(dirname "$0")/.env"
export PYTHONPATH="$(dirname "$0")/src"

log_info "Starting daily load pipeline"
python3 -m pipeline.extract --date "$(date -d yesterday +%Y-%m-%d)"
python3 -m pipeline.transform
python3 -m pipeline.load --target "$TARGET_SCHEMA"
log_info "Pipeline complete"
```

#### Defensive Scripting Checklist

1. Start with `set -euo pipefail`
2. Use `readonly` for constants
3. Declare `local` in every function
4. Quote all variable expansions
5. Validate inputs before using them
6. Use `trap` for cleanup
7. Write to stderr for errors (`>&2`), stdout for data
8. Return meaningful exit codes
9. Run `shellcheck` before every commit
10. Add `--help` / usage output to every script
