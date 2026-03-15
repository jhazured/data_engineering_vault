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
