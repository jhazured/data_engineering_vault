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
