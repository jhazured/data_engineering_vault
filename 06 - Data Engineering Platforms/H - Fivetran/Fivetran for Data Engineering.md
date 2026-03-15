#fivetran #data-engineering #snowflake #etl #ingestion

# Fivetran for Data Engineering

Fivetran is a managed ELT platform that replicates data from source systems into a cloud data warehouse. It handles extraction and loading automatically via pre-built connectors, leaving transformation to downstream tools like [[dbt Macro Patterns|dbt]].

## Key Terminology

- **Connector** -- the source system (e.g. Salesforce, Google Ads, a database). Fivetran reads from this.
- **Destination** -- the target data warehouse (e.g. [[Snowflake]]). Fivetran writes to this.
- **Schema prefix** -- the naming prefix Fivetran applies to all schemas it creates in the destination database. This is permanent and cannot be changed after the first sync.
- **MAR (Monthly Active Row)** -- the billing metric. A row is "active" if it was inserted or updated during the billing period.

---

## Snowflake Destination Setup

### Provisioning the Service Account

Fivetran requires a dedicated service user in Snowflake with its own role, warehouse, and database. Never use a personal or shared account -- credential changes on a shared user will break syncs silently.

Run this provisioning script as `SECURITYADMIN` / `SYSADMIN`:

```sql
begin;

  set role_name      = 'FIVETRAN_ROLE';
  set user_name      = 'FIVETRAN_USER';
  set warehouse_name = 'FIVETRAN_WAREHOUSE';
  set database_name  = 'FIVETRAN_DATABASE';

  use role securityadmin;

  create role if not exists identifier($role_name);
  grant role identifier($role_name) to role SYSADMIN;

  create user if not exists identifier($user_name)
    default_role      = $role_name
    default_warehouse = $warehouse_name;

  grant role identifier($role_name) to user identifier($user_name);

  -- Required input format defaults
  alter user identifier($user_name) set BINARY_INPUT_FORMAT = 'BASE64';
  alter user identifier($user_name) set TIMESTAMP_INPUT_FORMAT = 'AUTO';

  use role sysadmin;

  create warehouse if not exists identifier($warehouse_name)
    warehouse_size      = xsmall
    warehouse_type      = standard
    auto_suspend        = 60
    auto_resume         = true
    initially_suspended = true;

  create database if not exists identifier($database_name);

  grant usage          on warehouse identifier($warehouse_name) to role identifier($role_name);
  grant monitor        on warehouse identifier($warehouse_name) to role identifier($role_name);
  grant usage          on database  identifier($database_name)  to role identifier($role_name);
  grant monitor        on database  identifier($database_name)  to role identifier($role_name);
  grant create schema  on database  identifier($database_name)  to role identifier($role_name);

commit;
```

### Role Hierarchy

The Fivetran role slots into the standard Snowflake hierarchy beneath `SYSADMIN`:

```
ACCOUNTADMIN
  +-- SYSADMIN
        +-- FIVETRAN_ROLE    (owns FIVETRAN_WAREHOUSE, FIVETRAN_DATABASE)
              +-- FIVETRAN_USER
```

`FIVETRAN_ROLE` is granted to `SYSADMIN` so that objects Fivetran creates remain visible to account administrators. The Fivetran role should never be `ACCOUNTADMIN` or `SYSADMIN` directly.

### Privileges Granted to FIVETRAN_ROLE

| Object | Privileges |
|---|---|
| `FIVETRAN_WAREHOUSE` | `USAGE`, `MONITOR` |
| `FIVETRAN_DATABASE` | `USAGE`, `MONITOR`, `CREATE SCHEMA` |
| Schemas (created by Fivetran) | `OWNERSHIP` (Fivetran creates and owns these) |
| Tables (created by Fivetran) | `OWNERSHIP` (Fivetran creates and owns these) |

### Exposing Fivetran Tables to Downstream Roles

Use `FUTURE GRANTS` to give downstream roles (e.g. `ANALYST_ROLE`, `DBT_ROLE`) read access:

```sql
grant usage  on future schemas in database FIVETRAN_DATABASE to role ANALYST_ROLE;
grant select on future tables  in database FIVETRAN_DATABASE to role ANALYST_ROLE;

-- Backfill for existing objects
grant usage  on all schemas in database FIVETRAN_DATABASE to role ANALYST_ROLE;
grant select on all tables  in database FIVETRAN_DATABASE to role ANALYST_ROLE;
```

### Warehouse Configuration

`XSMALL` is sufficient for most Fivetran workloads -- Fivetran loads incrementally and uses minimal compute. Key settings:

| Setting | Value | Rationale |
|---|---|---|
| `warehouse_size` | `xsmall` | Incremental loads are lightweight |
| `auto_suspend` | `60` (seconds) | Avoids idle credit burn between syncs |
| `auto_resume` | `true` | Wakes on demand when Fivetran triggers a sync |
| `initially_suspended` | `true` | No cost until the first sync runs |

A shared warehouse is possible but not recommended -- Fivetran operations will then compete with query workloads.

---

## Key-Pair Authentication

Snowflake is deprecating username/password authentication for service accounts. Use RSA key-pair authentication for the Fivetran user.

### RSA Key Generation

```bash
# Generate 2048-bit private key in PKCS#8 format (no passphrase)
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt

# Derive public key
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

### Assign Public Key to Snowflake User

```sql
alter user FIVETRAN_USER set RSA_PUBLIC_KEY='<paste_public_key_contents_here>';
```

Store the private key (`rsa_key.p8`) securely -- its contents are pasted into the Fivetran destination setup form.

### Additional Account Parameters

For **AWS PrivateLink** or **GCP Private Service Connect** deployments:

```sql
alter account set PREVENT_LOAD_FROM_INLINE_URL = FALSE;
alter account set REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_OPERATION = FALSE;
```

To preserve source naming conventions (original casing):

```sql
alter user FIVETRAN_USER set QUOTED_IDENTIFIERS_IGNORE_CASE = FALSE;
```

---

## Fivetran RBAC Roles

Fivetran has its own platform-level RBAC system, separate from Snowflake's.

### Role Definitions

| Role | Scope | Capabilities |
|---|---|---|
| **Account Administrator** | Account | Full access: destinations, connectors, users, billing, API keys |
| **Account Analyst** | Account | Read-only: view connectors and sync history, no edits |
| **Destination Creator** | Account | Can create new destinations; manages ones they created |
| **Manage Destination** | Destination | Manage a specific destination and its connectors |
| **View Destination** | Destination | Read-only access to a specific destination |
| **Connector Administrator** | Connector | Manage a specific connector |
| **Manage Connection** | Connector | Edit, pause, and re-sync a specific connection |

Custom roles are available on **Enterprise and Business Critical** plans only.

### Recommended Assignments

| Persona | Recommended Role |
|---|---|
| Data platform / infrastructure team | Account Administrator |
| Data analysts (monitoring only) | Account Analyst or View Destination |
| Connector owners (source team leads) | Manage Connection (per connector) |
| CI/CD service accounts using the API | Account Administrator (API key scoped) |

Assign roles via: **Fivetran Dashboard --> Settings --> Users --> Invite User --> Assign Role**.

---

## REST API Patterns

The Fivetran REST API enables programmatic inspection of connectors, destinations, and metadata. API access requires the **Standard, Enterprise, or Business Critical** plan and an Account Administrator role.

### Authentication (Base64 Basic Auth)

```bash
echo -n "your_api_key:your_api_secret" | base64
```

All requests use HTTP Basic Auth with the Base64-encoded `key:secret` string in the `Authorization` header.

### Connector Type Metadata

Fetch the schema, configuration options, and supported features for a connector type:

```bash
curl -X GET \
  "https://api.fivetran.com/v1/metadata/connector-types/salesforce" \
  -H "Authorization: Basic <BASE64_CREDENTIALS>" \
  -H "Accept: application/json"
```

The response includes `config.fields` (required parameters), `supported_features` (e.g. `history_mode`, `re_sync_table_support`), and documentation links.

### Querying a Live Connector Instance

Retrieve actual configuration and sync state for a running connector:

```bash
curl -X GET \
  "https://api.fivetran.com/v1/connections/{connection_id}" \
  -H "Authorization: Basic <BASE64_CREDENTIALS>" \
  -H "Accept: application/json"
```

The `connection_id` appears in the Fivetran dashboard URL: `https://fivetran.com/dashboard/connectors/{connection_id}`.

Key fields in the response:

| Field | Description |
|---|---|
| `schema` | The destination schema prefix in Snowflake |
| `sync_frequency` | Sync interval in minutes |
| `status.setup_state` | `connected`, `incomplete`, `broken` |
| `status.sync_state` | `scheduled`, `syncing`, `paused` |
| `status.is_historical_sync` | Whether the initial historical sync is still in progress |
| `succeeded_at` | Timestamp of the last successful sync |

### Pagination with next_cursor

All list endpoints are paginated. Always handle the `next_cursor` field:

```python
import requests, base64, json

API_KEY    = "your_api_key"
API_SECRET = "your_api_secret"

credentials = base64.b64encode(f"{API_KEY}:{API_SECRET}".encode()).decode()
headers = {
    "Authorization": f"Basic {credentials}",
    "Accept": "application/json"
}

def get_all_connections():
    url = "https://api.fivetran.com/v1/connections"
    connections = []
    cursor = None

    while True:
        params = {"cursor": cursor, "limit": 100} if cursor else {"limit": 100}
        response = requests.get(url, headers=headers, params=params)
        response.raise_for_status()
        data = response.json()

        connections.extend(data.get("data", {}).get("items", []))

        next_cursor = data.get("data", {}).get("next_cursor")
        if not next_cursor:
            break
        cursor = next_cursor

    return connections
```

The same `next_cursor` pattern applies to `/v1/metadata/connector-types` and other list endpoints.

---

## Cost Optimisation

### Warehouse Cost Reduction

- Set `auto_suspend = 60` on the Fivetran warehouse so it does not sit idle between syncs.
- Use `XSMALL` warehouse size -- incremental loads are lightweight.
- Consider a shared warehouse only if Fivetran sync windows do not overlap with heavy query workloads.

### MAR (Monthly Active Row) Reduction Strategies

Fivetran bills based on Monthly Active Rows -- rows inserted or updated during the billing period. Strategies to reduce MAR:

- **Exclude unnecessary tables** -- only sync tables you actually consume downstream. Disable sync for tables not referenced in your dbt models.
- **Column hashing** -- Fivetran detects changes at the row level. If a source system updates a timestamp column on every read (e.g. `last_accessed_at`), exclude that column to avoid false-positive row activations.
- **Sync frequency tuning** -- reduce sync frequency for low-priority connectors. A table synced every 24 hours generates fewer active rows than one synced every 5 minutes if the source updates in bursts.
- **History mode awareness** -- enabling history mode (where available) creates additional rows for every change. Use it selectively for tables where full change history is required.
- **Schema prefix consolidation** -- avoid creating duplicate connectors to the same source. Each connector instance counts MAR independently.

### Network and Collation Considerations

- If Snowflake has a network policy, add [Fivetran's published IP ranges](https://fivetran.com/docs/using-fivetran/ips) to the `allowed_ip_list`.
- Do not set custom collation at the database, schema, or table level -- it reduces maximum string column storage from 128 MB to 64 MB and can cause load failures on large text columns. Apply collation in queries instead.

---

## Connector Configuration Best Practices

### Destination Setup Checklist

| Field | Value |
|---|---|
| **Host** | `<org-name>-<account-name>.snowflakecomputing.com` |
| **Port** | `443` (default) |
| **Database** | `FIVETRAN_DATABASE` |
| **Warehouse** | `FIVETRAN_WAREHOUSE` |
| **User** | `FIVETRAN_USER` |
| **Role** | `FIVETRAN_ROLE` |
| **Auth Method** | Key-pair (paste private key content) |

Fivetran runs four validation checks on save:

1. **Host Connection** -- validates credentials and host accessibility
2. **Database Connection** -- confirms the database exists and is accessible
3. **Permission** -- verifies `CREATE SCHEMA` and `CREATE TEMPORARY TABLES`
4. **Workspace** -- confirms Fivetran can create a temporary table

### Schema Prefix Naming

The schema prefix is permanent -- it cannot be changed after the first sync. Choose a naming convention that aligns with your data warehouse layer structure (e.g. `salesforce_prod`, `hubspot_marketing`).

### Operational Guidelines

- **Do not manually alter Fivetran-created objects** -- schema ownership sits with `FIVETRAN_ROLE`. Use `FUTURE GRANTS` to expose data to downstream consumers.
- **Monitor sync health** via the Fivetran dashboard or the REST API `status` fields.
- **Audit connector configurations** periodically by exporting all connector configs to JSON via the REST API (see pagination example above).
- **Source freshness tests** in [[dbt Macro Patterns|dbt]] -- configure `freshness` blocks in your `sources.yml` to alert when Fivetran data is stale.

---

## Related Notes

- [[dbt Macro Patterns]] -- transformation patterns applied to Fivetran-landed data
- [[Snowflake]] -- destination platform configuration
- [[Data Ingestion Patterns]] -- broader ingestion architecture context

## References

- [Fivetran Snowflake Destination Setup Guide](https://fivetran.com/docs/destinations/snowflake/setup-guide)
- [Fivetran REST API Reference](https://fivetran.com/docs/rest-api)
- [Fivetran Role-Based Access Control](https://fivetran.com/docs/using-fivetran/fivetran-dashboard/account-settings/role-based-access-control)
- [Snowflake Key-Pair Authentication](https://docs.snowflake.com/en/user-guide/key-pair-auth)
- [Fivetran IP Addresses](https://fivetran.com/docs/using-fivetran/ips)
