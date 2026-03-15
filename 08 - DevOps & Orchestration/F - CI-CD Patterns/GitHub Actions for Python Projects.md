# GitHub Actions for Python Projects

CI/CD patterns for Python data engineering projects using GitHub Actions.

## Matrix Testing Across Python Versions

```yaml
name: Verify

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.12']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements-test.txt

      - name: Download NLTK data
        run: python -c "import nltk; nltk.download('punkt_tab', quiet=True)"

      - name: Run tests
        run: pytest tests/ -v --tb=short
```

**Key patterns:**
- **Matrix strategy**: Tests across 3.9, 3.10, 3.12 to catch version-specific issues
- **requirements-test.txt**: Lean dependency file (no torch/ML libs) for fast CI
- **NLTK data**: Downloaded at CI time — not committed to repo

## Lean Requirements for CI

Split requirements to keep CI fast:

```
requirements.txt              # Full: ingestion + UI + ML (~2GB with torch)
requirements-test.txt         # CI: pytest + parsing libs (~200MB, no torch)
requirements-agent-only.txt   # Slim: retriever + connector only (~100MB)
```

```
# requirements-test.txt
snowflake-connector-python>=3.0
pandas>=2.0
python-dotenv>=1.0
langchain-core>=0.3
pytest>=7.0
pypdf>=4.0
beautifulsoup4>=4.12
unstructured>=0.20,<1.0
python-docx
nltk
```

## Verification Job (Separate from Tests)

Run setup verification and dry-run validation as a separate job:

```yaml
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install agent-only dependencies
        run: pip install -r requirements-agent-only.txt

      - name: Verify setup (no Snowflake connection)
        run: python scripts/setup/verify_setup.py

      - name: Dry-run ingestion (no Snowflake connection)
        run: python scripts/load_documents.py --dry-run --source-dir sample_docs/
```

**Dry-run validation**: Exercises the full partition/chunk pipeline without a Snowflake connection — catches import errors, schema issues, and parsing bugs in CI.

## GitHub Actions vs GitLab CI

| Aspect | GitHub Actions | GitLab CI |
|--------|---------------|-----------|
| Config file | `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| Runner | `runs-on: ubuntu-latest` | `image: python:3.11` |
| Matrix | `strategy.matrix` | `parallel:matrix` |
| Secrets | `${{ secrets.NAME }}` | `$NAME` (CI/CD variables) |
| Artifacts | `actions/upload-artifact` | `artifacts: paths:` |
| Manual gate | `workflow_dispatch` | `when: manual` |
| Pages | GitHub Pages action | `pages:` job |

## Caching pip Dependencies

Speed up repeated runs:

```yaml
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ matrix.python-version }}-${{ hashFiles('requirements-test.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-${{ matrix.python-version }}-

      - name: Install dependencies
        run: pip install -r requirements-test.txt
```

## Secrets Management

Never hardcode credentials — use GitHub Secrets for Snowflake connectivity tests:

```yaml
      - name: Run integration tests
        if: github.event_name != 'pull_request'  # Only on push, not PRs from forks
        env:
          SF_ACCOUNT: ${{ secrets.SF_ACCOUNT }}
          SF_USER: ${{ secrets.SF_CI_USER }}
          SF_PASSWORD: ${{ secrets.SF_CI_PASSWORD }}
        run: pytest tests/ -v -m integration
```

**Security note**: PRs from forks don't have access to secrets — skip integration tests on external PRs.

## Advanced CI/CD Patterns

### Artifact Management

CI/CD pipelines produce artifacts that need to be stored, versioned, and promoted across environments. Each artifact type has its own storage strategy:

#### Container Registries

Store Docker images for Spark jobs, API services, and Airflow task containers:

```yaml
      - name: Build and push container
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.ECR_REGISTRY }}/data-pipeline:${{ github.sha }}
            ${{ secrets.ECR_REGISTRY }}/data-pipeline:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Registry options:** AWS ECR (integrated with ECS/EKS IAM), GCP Artifact Registry (supports Docker + Python + Maven), Azure Container Registry, GitHub Container Registry (ghcr.io — free for public repos).

**Tagging strategy:** tag with both `git-sha` (immutable, traceable) and `latest` or `staging`/`production` (mutable, for deployment references). Never deploy `latest` to production — always use the SHA tag.

#### Package Repositories

Publish internal Python packages for shared utilities, connectors, and transformation libraries:

```yaml
      - name: Build and publish package
        run: |
          python -m build
          twine upload --repository-url ${{ secrets.PYPI_REPO_URL }} dist/*
        env:
          TWINE_USERNAME: ${{ secrets.PYPI_USERNAME }}
          TWINE_PASSWORD: ${{ secrets.PYPI_PASSWORD }}
```

**Options:** AWS CodeArtifact, GCP Artifact Registry (Python), Azure Artifacts, private PyPI (devpi), GitHub Packages.

#### dbt Artifact Storage

dbt produces `manifest.json`, `run_results.json`, and `catalog.json` after each run. These artifacts are essential for [[Core dbt Fundamentals|Slim CI]] (`state:modified`) and documentation:

```yaml
      - name: Upload dbt artifacts
        run: |
          aws s3 cp target/manifest.json \
            s3://dbt-artifacts/${{ github.ref_name }}/manifest.json
          aws s3 cp target/run_results.json \
            s3://dbt-artifacts/${{ github.ref_name }}/run_results.json

      - name: Slim CI — compare against production manifest
        run: |
          aws s3 cp s3://dbt-artifacts/main/manifest.json target/prod-manifest/manifest.json
          dbt run --select state:modified+ --defer --state target/prod-manifest/
          dbt test --select state:modified+ --defer --state target/prod-manifest/
```

### Blue/Green Deployments for Data Pipelines

Blue/green deployments maintain two identical environments and switch traffic between them, enabling zero-downtime releases for data services.

#### Strategy: Schema Swapping

For warehouse-based pipelines, deploy transformations to a shadow schema, validate, then swap:

```sql
-- 1. Build new models in a shadow schema
-- dbt run --target prod-green (targets PROD_GREEN schema)

-- 2. Run validation queries against the green schema
-- dbt test --target prod-green

-- 3. Swap schemas atomically
ALTER SCHEMA PROD_BLUE RENAME TO PROD_OLD;
ALTER SCHEMA PROD_GREEN RENAME TO PROD_BLUE;

-- 4. Rollback if needed
ALTER SCHEMA PROD_BLUE RENAME TO PROD_GREEN;
ALTER SCHEMA PROD_OLD RENAME TO PROD_BLUE;
```

#### Strategy: Zero-Downtime API Deployments

For data APIs served via [[Kubernetes for Data Workloads|Kubernetes]]:

```yaml
# Rolling update with readiness gates
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # zero downtime — new pods must be ready before old ones terminate
  template:
    spec:
      containers:
        - name: data-api
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
```

### Canary Deployments

Canary deployments gradually shift traffic to a new version, validating behaviour before full rollout:

#### Shadow Mode

Run the new pipeline version alongside the existing one, comparing outputs without serving the new results:

```yaml
      - name: Shadow pipeline execution
        run: |
          # Run both versions
          dbt run --target prod --select marts.finance
          dbt run --target prod-canary --select marts.finance

          # Compare outputs
          duckdb -c "
            SELECT COUNT(*) AS diff_count
            FROM (
              SELECT * FROM read_parquet('prod/finance/*.parquet')
              EXCEPT
              SELECT * FROM read_parquet('canary/finance/*.parquet')
            );
          "
```

#### Percentage Rollout

Route a fraction of traffic to the new version using weighted routing:

```yaml
# Istio VirtualService for percentage-based routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: data-api
spec:
  hosts:
    - data-api
  http:
    - route:
        - destination:
            host: data-api
            subset: stable
          weight: 90
        - destination:
            host: data-api
            subset: canary
          weight: 10
```

Monitor error rates and latency on the canary subset. If metrics degrade beyond thresholds, automatically roll back by setting canary weight to 0.

### Feature Flags for Data Models

Feature flags decouple deployment from activation, allowing new data models to be deployed but not exposed until explicitly enabled:

```sql
-- dbt model with feature flag
{{ config(materialized='table') }}

{% if var('enable_new_scoring_model', false) %}
    SELECT
        customer_id,
        new_score_v2 AS customer_score,
        score_components
    FROM {{ ref('stg_new_scoring') }}
{% else %}
    SELECT
        customer_id,
        legacy_score AS customer_score,
        NULL AS score_components
    FROM {{ ref('stg_legacy_scoring') }}
{% endif %}
```

```bash
# Enable the feature flag in production
dbt run --vars '{"enable_new_scoring_model": true}' --select customer_scores

# Roll back by re-running without the flag
dbt run --select customer_scores
```

**External feature flag services** (LaunchDarkly, Unleash, AWS AppConfig) provide more sophisticated controls: gradual rollout percentages, user-segment targeting, and automatic rollback on metric degradation. Integrate by reading the flag value in a pre-run hook or macro.
