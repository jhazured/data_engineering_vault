# Snowflake Native dbt Workspace

Run dbt inside Snowflake's browser-based IDE — no local machine or dbt Cloud license required. The Workspace connects to your Git repository and runs dbt jobs via a built-in executor.

## How It Works

- The Workspace runs dbt via a **built-in executor**, not a terminal
- It connects as **`DBT_RUNTIME_ROLE`** — NOT the role in `profiles.yml`
- `profiles.yml` role settings are **ignored** by the job runner
- `DBT_RUNTIME_ROLE` is auto-created when the Workspace project is first initialised
- OAuth session token is injected automatically — no username/password needed
- `env_var()` calls in `profiles.yml` fail because there's no shell — use hardcoded values instead

## Setup (6 Steps)

### Step 1 — Create a GitLab Personal Access Token

GitLab > Preferences > Access Tokens:
- Scope: `read_repository`
- Copy the token immediately (shown once)

### Step 2 — Create Git Secret and Integration

```sql
USE ROLE ACCOUNTADMIN;

-- Store GitLab credentials
CREATE OR REPLACE SECRET DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS
    TYPE = PASSWORD
    USERNAME = '<gitlab_username>'
    PASSWORD = '<gitlab_pat>';

-- Create API integration for HTTPS git access
CREATE OR REPLACE API INTEGRATION GITLAB_INTEGRATION
    API_PROVIDER = git_https_api
    API_ALLOWED_PREFIXES = ('https://gitlab.example.com/')
    ALLOWED_AUTHENTICATION_SECRETS = ALL
    ENABLED = TRUE;

-- Grant access to ENGINEER role
GRANT USAGE ON INTEGRATION GITLAB_INTEGRATION TO ROLE ENGINEER;
GRANT READ ON SECRET DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS TO ROLE ENGINEER;

-- Create Git repository link
CREATE OR REPLACE GIT REPOSITORY DEV_T0_CONTROL.SECURITY.MY_DBT_REPO
    API_INTEGRATION = GITLAB_INTEGRATION
    GIT_CREDENTIALS = 'DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS'
    ORIGIN = 'https://gitlab.example.com/team/my-dbt-project.git';

-- Verify
ALTER GIT REPOSITORY DEV_T0_CONTROL.SECURITY.MY_DBT_REPO FETCH;
SHOW GIT BRANCHES IN GIT REPOSITORY DEV_T0_CONTROL.SECURITY.MY_DBT_REPO;
```

### Step 3 — Create the Workspace Project

Snowsight > Projects > + Project:
1. Name: `my_dbt_project`
2. Git repository: select `DEV_T0_CONTROL.SECURITY.MY_DBT_REPO`
3. Branch: `main`
4. Click Create

### Step 4 — Workspace Profile

Replace `profiles.yml` — remove all `env_var()` calls and use `authenticator: oauth`:

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: xxxxxxx-xxxxxxx
      authenticator: oauth          # No password needed
      role: ENGINEER                # Ignored by executor, but required for compile
      database: DEV_T2_PERSISTENT_STAGING
      warehouse: TRN_DEV_CENTRAL_WH
      schema: LOGISTICS
      threads: 4
      query_tag: dbt_dev
    prod:
      type: snowflake
      account: xxxxxxx-xxxxxxx
      authenticator: oauth
      role: CHANGE_CONTROL
      database: PROD_T2_PERSISTENT_STAGING
      warehouse: TRN_PROD_CENTRAL_WH
      schema: LOGISTICS
      threads: 12
      query_tag: dbt_prod
```

### Step 5 — Grant DBT_RUNTIME_ROLE

This role exists only after the Workspace project is created in Step 3:

```sql
USE ROLE ACCOUNTADMIN;

-- Critical — without this, every model fails with privilege errors
GRANT ROLE ENGINEER TO ROLE DBT_RUNTIME_ROLE;

-- Allow Workspace to read the Git secret and integration
GRANT USAGE ON INTEGRATION GITLAB_INTEGRATION TO ROLE DBT_RUNTIME_ROLE;
GRANT READ ON SECRET DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS TO ROLE DBT_RUNTIME_ROLE;
```

### Step 6 — Run dbt Jobs

Use the Profile dropdown in the Workspace UI (not a terminal). Run each as a separate job:

```
seed --select data_classification_tags
run --select tag:t0
run --select tag:load_priority_1
snapshot
run --select tag:load_priority_2a
run --select tag:load_priority_2b
run --select tag:load_priority_3
test
```

First run: `run --full-refresh --target dev`

## Key Differences from Local CLI

| Aspect | Local CLI | Snowflake Workspace |
|--------|-----------|-------------------|
| Auth | `env_var('SF_USER')` + password | OAuth (automatic) |
| Execution role | Uses `profiles.yml` role | Always `DBT_RUNTIME_ROLE` |
| `env_var()` | Works (shell environment) | Fails (no shell) |
| `dbt deps` | Required (downloads packages) | Not needed if `dbt_packages/` is committed |
| `dbt docs serve` | Works (local browser) | Not supported |

## Avoiding `dbt deps` in Workspace

Commit the `dbt/dbt_packages/` directory to Git. This avoids needing an External Access Integration (EAI) for outbound network calls to the dbt package hub.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `Insufficient privileges` | `GRANT ROLE ENGINEER TO ROLE DBT_RUNTIME_ROLE` |
| `env_var() is not defined` | Use Workspace profile with hardcoded values |
| `Secret does not exist` | PAT expired — refresh with `ALTER SECRET ... SET PASSWORD` |
| `Git repository does not exist` | Run `SHOW GIT REPOSITORIES IN ACCOUNT` to find actual location |
| Models write to wrong database | Check `SF_ENV` variable in `dbt_project.yml` |

## Refreshing an Expired GitLab PAT

```sql
USE ROLE ACCOUNTADMIN;
ALTER SECRET DEV_T0_CONTROL.SECURITY.GITLAB_CREDENTIALS
    SET USERNAME = '<username>'
        PASSWORD = '<new-glpat-token>';
```
