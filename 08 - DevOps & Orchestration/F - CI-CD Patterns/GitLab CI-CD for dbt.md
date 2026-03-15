# GitLab CI/CD for dbt

## Pipeline Architecture

A production dbt pipeline uses six stages with progressively stricter gates:

```
secret-detection → build → test → staging → production → docs
```

| Stage | Job | Trigger | What It Does |
|-------|-----|---------|--------------|
| **secret-detection** | `secret_detection` | All pushes/MRs | Scans for leaked credentials using GitLab template |
| **build** | `dbt:build` | MRs, pushes to main/develop | `dbt parse` with dummy creds — validates syntax without a warehouse |
| **test** | `dbt:lint-and-test` | MRs + dbt changes, nightly | sqlfluff lint + `dbt test` against CI environment |
| **test** | `dbt:data-quality-monitor` | Nightly schedule only | `dbt test --select tag:data_quality` against prod |
| **staging** | `dbt:deploy-staging` | Pushes to develop | `dbt run` + `dbt test` against staging |
| **production** | `dbt:deploy-production` | Pushes to main (manual gate) | `dbt snapshot` + `dbt run` + `dbt test` against prod |
| **docs** | `pages` | Pushes to main (model/macro changes) | Publish dbt docs to GitLab Pages |

## Base Templates

Use YAML anchors to avoid repeating setup across jobs:

```yaml
.python-base:
  image: python:3.11
  before_script:
    - pip install --upgrade pip

.dbt-base:
  extends: .python-base
  before_script:
    - pip install --upgrade pip
    - pip install dbt-snowflake~=1.8
    - pip install -r requirements.txt
    - |
      mkdir -p dbt
      cat > dbt/profiles.yml <<EOF
      my_project:
        target: ${DBT_TARGET:-ci}
        outputs:
          ci:
            type: snowflake
            account: "${SF_ACCOUNT}"
            user: "${SF_USER}"
            password: "${SF_PASSWORD}"
            role: "${SF_ROLE:-ENGINEER}"
            database: "${SF_DATABASE}"
            warehouse: "${SF_WAREHOUSE}"
            schema: "${SF_SCHEMA:-LOGISTICS}"
            threads: 4
      EOF
    - dbt deps --project-dir dbt
```

**Key pattern:** `profiles.yml` is generated at runtime from CI/CD variables — never committed.

## Build Stage (Syntax Validation)

Validate SQL parses without a live warehouse connection:

```yaml
dbt:build:
  extends: .python-base
  stage: build
  variables:
    SF_ACCOUNT: "dummy"
    SF_USER: "dummy"
    SF_PASSWORD: "dummy"
  script:
    - pip install dbt-snowflake~=1.8
    - dbt deps --project-dir dbt
    - dbt parse --project-dir dbt
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH =~ /^(main|develop)$/
```

`dbt parse` validates syntax, ref/source resolution, and Jinja compilation without executing anything.

## Test Stage (Lint + Data Tests)

```yaml
dbt:lint-and-test:
  extends: .dbt-base
  stage: test
  script:
    - pip install sqlfluff sqlfluff-templater-dbt
    - cd dbt && sqlfluff lint models/ --config .sqlfluff --templater dbt --ignore templating --ignore parsing
    # Fail fast if credentials are missing
    - |
      missing_vars=""
      [ -z "$SF_ACCOUNT" ]  && missing_vars="$missing_vars SF_ACCOUNT"
      [ -z "$SF_USER" ]     && missing_vars="$missing_vars SF_CI_USER"
      [ -z "$SF_PASSWORD" ] && missing_vars="$missing_vars SF_CI_PASSWORD"
      if [ -n "$missing_vars" ]; then
        echo "ERROR: Missing credentials:$missing_vars"
        exit 1
      fi
    - dbt debug --project-dir dbt --target ci
    - dbt test --project-dir dbt --target ci --store-failures
  artifacts:
    when: always
    paths: [dbt/target/, dbt/logs/]
    expire_in: 7 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: [dbt/**/*]
      allow_failure: true      # Lint-only if no credentials
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

## Production Deployment

Manual gate ensures human approval before prod changes:

```yaml
dbt:deploy-production:
  extends: .dbt-base
  stage: production
  script:
    # Hard fail if credentials missing (prod must never silently skip)
    - |
      [ -z "$SF_ACCOUNT" ] || [ -z "$SF_USER" ] && exit 1
    - dbt snapshot --project-dir dbt --target prod
    - dbt run --project-dir dbt --target prod --exclude tag:dev_only
    - dbt test --project-dir dbt --target prod --select tag:critical
    - dbt docs generate --project-dir dbt --target prod
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

**Execution order in prod:**
1. `dbt snapshot` — capture SCD2 state before transforms
2. `dbt run --exclude tag:dev_only` — all models except dev-only
3. `dbt test --select tag:critical` — critical tests only (fast feedback)
4. `dbt docs generate` — update documentation

## CI/CD Variables

Configure in GitLab > Settings > CI/CD > Variables:

| Variable | Scope | Purpose |
|----------|-------|---------|
| `SF_ACCOUNT` | All | Snowflake account identifier |
| `SF_CI_USER` | Test | CI service account |
| `SF_CI_PASSWORD` | Test | CI service account password |
| `SF_STAGING_USER` | Staging | Staging deployment account |
| `SF_STAGING_PASSWORD` | Staging | Staging deployment password |
| `SF_PROD_USER` | Production | Prod deployment account (CHANGE_CONTROL role) |
| `SF_PROD_PASSWORD` | Production | Prod deployment password |

## dbt Docs on GitLab Pages

```yaml
pages:
  extends: .dbt-base
  stage: docs
  script:
    - dbt docs generate --project-dir dbt --target prod
    - mkdir -p public
    - cp -r dbt/target/* public/
  artifacts:
    paths: [public]
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes: [dbt/models/**/*, dbt/macros/**/*]
```

Docs are auto-published at `https://<group>.gitlab.io/<project>`.

## Nightly Data Quality

Scheduled pipeline runs quality tests against production data:

```yaml
dbt:data-quality-monitor:
  extends: .dbt-base
  stage: test
  script:
    - pip install pandas great-expectations
    - dbt test --project-dir dbt --target prod --select tag:data_quality
    - python scripts/generate_quality_report.py
  artifacts:
    paths: [reports/quality_report.html]
    expire_in: 30 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

## Credential Handling Patterns

**Test stage:** Warn on missing creds (lint still runs)
```yaml
allow_failure: true  # MR rule — lint passes even without warehouse
```

**Staging:** Soft skip (exit 0) if creds not configured yet
```bash
if [ -z "$SF_ACCOUNT" ]; then echo "Skipping"; exit 0; fi
```

**Production:** Hard fail (exit 1) — prod must never silently succeed without running
```bash
if [ -z "$SF_ACCOUNT" ]; then echo "ERROR: credentials missing"; exit 1; fi
```
