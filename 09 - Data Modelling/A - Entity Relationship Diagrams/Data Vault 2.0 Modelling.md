# Data Vault 2.0 Modeling

Data Vault is a **detail-oriented, history-tracking, and uniquely linked** methodology for building enterprise data warehouses that prioritizes auditability, agility, and parallel loading.

## What Is Data Vault

Dan Linstedt introduced Data Vault 1.0 in the early 2000s as a modeling methodology that sits between [[Dimensional Modelling (Kimball)|Kimball star schemas]] and 3NF (Inmon). Data Vault 2.0 (published 2013) added hash keys, NoSQL compatibility, and formal loading patterns.

**Purpose:** Build a single, auditable, scalable enterprise data warehouse that can absorb source system changes without redesign.

### Data Vault 1.0 vs 2.0

| Aspect | DV 1.0 | DV 2.0 |
|--------|--------|--------|
| Surrogate keys | Sequences | **Hash keys** (deterministic) |
| Loading | Sequential | **Parallel and pattern-based** |
| Scope | Relational only | Relational + NoSQL + unstructured |
| Methodology | Modeling pattern | Full methodology (modeling + architecture + loading) |
| Business rules | Mixed into raw vault | Isolated in **Business Vault** layer |

### Positioning vs Other Approaches

| Concern | 3NF (Inmon) | Data Vault 2.0 | Kimball |
|---------|------------|----------------|---------|
| Primary goal | Enterprise consistency | Auditability + agility | Query performance |
| Schema style | Normalized | Hub-Link-Satellite | Star / snowflake |
| History handling | Varies | Built-in everywhere | [[SCD Type 2 Patterns|SCD Types]] |
| Source integration | ETL-heavy transforms | Insert-only, minimal transforms | Transform on load |
| Query complexity | High | High (needs Business Vault) | Low |

## Core Building Blocks

Data Vault uses three core entity types plus reference tables:

```
  ┌───────────┐         ┌───────────────┐         ┌───────────┐
  │  HUB      │         │     LINK      │         │  HUB      │
  │ Customer  ├────────►│ Order_Customer│◄────────┤  Order    │
  └─────┬─────┘         └──────┬────────┘         └─────┬─────┘
        │                      │                        │
  ┌─────▼─────┐         ┌──────▼────────┐         ┌─────▼─────┐
  │ SATELLITE │         │  SATELLITE    │         │ SATELLITE │
  │ Cust_Name │         │ Link_Effectiv │         │ Ord_Detail│
  └───────────┘         └───────────────┘         └───────────┘
```

- **Hubs** — represent core business concepts (unique business keys)
- **Links** — capture relationships and transactions between hubs
- **Satellites** — store descriptive attributes and all historical changes
- **Reference tables** — shared lookup data (country codes, currency codes) not tied to a specific business key

## Hub Design

A hub represents a **unique business concept** identified by its business key. Hubs contain no descriptive data — they are anchors for everything else.

### Hub Structure

```sql
CREATE TABLE hub_customer (
    customer_hk     BINARY(32)    PRIMARY KEY,  -- hash of business key
    customer_bk     VARCHAR(50)   NOT NULL,      -- business key (natural key)
    load_date       TIMESTAMP     NOT NULL,      -- first seen timestamp
    record_source   VARCHAR(100)  NOT NULL       -- originating source system
);
```

**Design rules:**
- Business key is the grain — one row per unique business key, ever
- `customer_hk` is a hash of `customer_bk` for join performance
- No business attributes — those belong in satellites
- Insert-only — once a hub row exists, it never changes
- `load_date` records when the key was **first observed**, not when data changed
- `record_source` identifies which source system first provided the key

### Composite Business Keys

When a business concept requires multiple columns to be unique:

```sql
-- Hub for order line items (order_number + line_number = business key)
CREATE TABLE hub_order_line (
    order_line_hk   BINARY(32)    PRIMARY KEY,
    order_number    VARCHAR(20)   NOT NULL,
    line_number     INT           NOT NULL,
    load_date       TIMESTAMP     NOT NULL,
    record_source   VARCHAR(100)  NOT NULL
);
```

## Link Design

Links capture **relationships between business concepts** (hubs). They are always modeled as many-to-many, even if the source is one-to-many, since relationships can change.

### Standard Link

```sql
CREATE TABLE link_order_customer (
    order_customer_hk   BINARY(32)    PRIMARY KEY,  -- hash of parent HKs
    customer_hk         BINARY(32)    NOT NULL,      -- FK to hub_customer
    order_hk            BINARY(32)    NOT NULL,      -- FK to hub_order
    load_date           TIMESTAMP     NOT NULL,
    record_source       VARCHAR(100)  NOT NULL
);
```

**Design rules:**
- Composite hash key is generated from the combination of parent hub hash keys
- Insert-only — links are never deleted or updated
- No descriptive attributes — those go in link satellites

### Link Variants

| Variant | Purpose | Example |
|---------|---------|---------|
| **Same-as link** | Tracks when two business keys represent the same entity | `link_same_as_customer` (merging CRM + ERP customer IDs) |
| **Hierarchical link** | Self-referencing relationship on a single hub | `link_employee_manager` (employee hub to itself) |
| **Non-historized link** | Relationship that never changes, no satellites needed | `link_product_category` (static classification) |
| **Transactional link** | Records transactions (has degenerate attributes) | `link_payment` with amount, timestamp on the link itself |

### Link Satellites (Effectivity Satellites)

Track when a relationship was **active or inactive** over time:

```sql
CREATE TABLE sat_order_customer_eff (
    order_customer_hk   BINARY(32)    NOT NULL,
    load_date           TIMESTAMP     NOT NULL,
    record_source       VARCHAR(100)  NOT NULL,
    is_active           BOOLEAN       NOT NULL,
    PRIMARY KEY (order_customer_hk, load_date)
);
```

## Satellite Design

Satellites hold **all descriptive attributes and history**. They attach to either a hub or a link and are the only entity type that stores temporal change.

### Satellite Structure

```sql
CREATE TABLE sat_customer_details (
    customer_hk     BINARY(32)    NOT NULL,      -- FK to hub_customer
    load_date       TIMESTAMP     NOT NULL,      -- when this version was loaded
    end_date        TIMESTAMP,                   -- NULL = current; set when superseded
    record_source   VARCHAR(100)  NOT NULL,
    hash_diff       BINARY(32)    NOT NULL,      -- hash of all descriptive columns
    customer_name   VARCHAR(200),
    email           VARCHAR(200),
    tier            VARCHAR(20),
    PRIMARY KEY (customer_hk, load_date)
);
```

**Design rules:**
- Primary key is always `(parent_hk, load_date)`
- `hash_diff` is a hash of all descriptive columns — used for **change detection** during loading
- `end_date` is optional (can be derived) but useful for query performance
- Insert a new row only when `hash_diff` differs from the current record
- No updates or deletes — append-only

### Satellite Splitting Strategy

**One satellite per source system per rate of change:**

```
hub_customer
  ├── sat_customer_crm        (from CRM, changes daily)
  ├── sat_customer_erp        (from ERP, changes monthly)
  ├── sat_customer_address    (from CRM, changes rarely)
  └── sat_customer_compliance (from compliance system, changes yearly)
```

Splitting prevents unnecessary row inserts — if only the email changes in CRM data, you do not need to write a new row for compliance attributes that have not changed.

## Hash Keys

Data Vault 2.0 replaces sequence-based surrogate keys with **deterministic hash keys**, enabling parallel and idempotent loading.

### Hash Generation Rules

```sql
-- Hub hash key: hash of business key
customer_hk = MD5(UPPER(TRIM(customer_bk)))

-- Link hash key: hash of sorted parent hub hash keys
order_customer_hk = MD5(CONCAT(customer_hk, '||', order_hk))

-- Hash diff: hash of all descriptive columns for change detection
hash_diff = MD5(CONCAT(
    COALESCE(customer_name, '^^'),  '||',
    COALESCE(email, '^^'),          '||',
    COALESCE(tier, '^^')
))
```

**Rules:**
- Use a consistent delimiter (`||`) between concatenated values
- Apply `UPPER()` and `TRIM()` to business keys before hashing
- Handle NULLs with a sentinel value (`^^`) — raw NULL in any position would produce identical hashes for different data
- MD5 is common for performance; SHA-256 provides lower collision risk for very large datasets
- Same input always produces the same hash — no sequence dependency

### MD5 vs SHA-256

| Factor | MD5 (128-bit) | SHA-256 (256-bit) |
|--------|--------------|-------------------|
| Storage | 16 bytes | 32 bytes |
| Speed | Faster | Slower |
| Collision risk | Theoretical (acceptable at DW scale) | Negligible |
| Recommendation | Default choice | Use for billions+ of keys or regulatory requirements |

## Loading Patterns

Data Vault loading follows a strict order with deterministic, idempotent operations.

### Hub-First Loading

```sql
-- 1. Load hubs first (insert new business keys only)
INSERT INTO hub_customer (customer_hk, customer_bk, load_date, record_source)
SELECT DISTINCT
    MD5(UPPER(TRIM(src.customer_id))),
    src.customer_id,
    CURRENT_TIMESTAMP(),
    'CRM_SYSTEM'
FROM staging.customers src
WHERE NOT EXISTS (
    SELECT 1 FROM hub_customer h
    WHERE h.customer_hk = MD5(UPPER(TRIM(src.customer_id)))
);
```

### Link Loading

```sql
-- 2. Load links (insert new relationships only)
INSERT INTO link_order_customer (order_customer_hk, customer_hk, order_hk, load_date, record_source)
SELECT DISTINCT
    MD5(CONCAT(cust_hk, '||', ord_hk)),
    cust_hk,
    ord_hk,
    CURRENT_TIMESTAMP(),
    'ORDER_SYSTEM'
FROM staging.orders_prepared src
WHERE NOT EXISTS (
    SELECT 1 FROM link_order_customer l
    WHERE l.order_customer_hk = MD5(CONCAT(src.cust_hk, '||', src.ord_hk))
);
```

### Satellite Loading with Hash Diff

```sql
-- 3. Load satellites (insert only when data has changed)
INSERT INTO sat_customer_details
    (customer_hk, load_date, record_source, hash_diff, customer_name, email, tier)
SELECT
    src.customer_hk,
    CURRENT_TIMESTAMP(),
    'CRM_SYSTEM',
    src.hash_diff,
    src.customer_name,
    src.email,
    src.tier
FROM staging.customers_prepared src
LEFT JOIN sat_customer_details sat
    ON  src.customer_hk = sat.customer_hk
    AND sat.end_date IS NULL  -- current record
WHERE sat.customer_hk IS NULL              -- new hub, no satellite yet
   OR sat.hash_diff != src.hash_diff;      -- attributes changed
```

**Key properties:**
- Hub, link, and satellite loads can run **in parallel across different source systems**
- Every load is **idempotent** — rerunning the same data produces no duplicates thanks to hash key checks
- No updates, no deletes — insert-only pattern

## Business Vault

The raw vault stores data **as-is from source systems** with no business logic. The Business Vault applies transformations and business rules in a separate layer.

### Bridge Tables

Pre-join hubs and links to simplify downstream [[Star Schema Implementation Patterns|star schema]] queries:

```sql
CREATE TABLE bridge_customer_orders AS
SELECT
    h_c.customer_hk,
    h_c.customer_bk,
    h_o.order_hk,
    h_o.order_bk,
    s_c.customer_name,
    s_o.order_total
FROM hub_customer h_c
JOIN link_order_customer l   ON h_c.customer_hk = l.customer_hk
JOIN hub_order h_o           ON l.order_hk = h_o.order_hk
JOIN sat_customer_details s_c ON h_c.customer_hk = s_c.customer_hk AND s_c.end_date IS NULL
JOIN sat_order_details s_o    ON h_o.order_hk = s_o.order_hk AND s_o.end_date IS NULL;
```

### Point-in-Time (PIT) Tables

Snapshot tables that pre-calculate which satellite record was active at each point in time — avoids expensive temporal joins:

```sql
CREATE TABLE pit_customer (
    customer_hk         BINARY(32),
    snapshot_date        DATE,
    sat_details_ldts     TIMESTAMP,   -- load_date of active sat_customer_details row
    sat_address_ldts     TIMESTAMP,   -- load_date of active sat_customer_address row
    PRIMARY KEY (customer_hk, snapshot_date)
);
```

PIT tables are rebuilt on a schedule (daily, hourly) and make querying satellites at a specific point in time performant.

### Computed Satellites

Derived attributes calculated from raw vault data — stored as satellites with a different `record_source` to distinguish them from raw data:

```sql
-- sat_customer_lifetime_value: computed from raw order satellites
INSERT INTO sat_customer_lifetime_value
    (customer_hk, load_date, record_source, hash_diff, lifetime_value, order_count)
SELECT
    customer_hk, CURRENT_TIMESTAMP(), 'BUSINESS_VAULT',
    MD5(CONCAT(lifetime_value, '||', order_count)),
    SUM(order_total), COUNT(*)
FROM bridge_customer_orders
GROUP BY customer_hk;
```

## Data Vault with dbt

dbt is a natural fit for Data Vault because of its incremental materialization, testing framework, and modularity. See [[Core dbt Fundamentals]] for dbt basics.

### Hub/Link/Satellite as dbt Models

```yaml
# dbt_project.yml
models:
  my_project:
    raw_vault:
      hubs:
        materialized: incremental
        tags: ['raw_vault', 'hub']
      links:
        materialized: incremental
        tags: ['raw_vault', 'link']
      satellites:
        materialized: incremental
        tags: ['raw_vault', 'satellite']
    business_vault:
      materialized: table
      tags: ['business_vault']
```

### automate-dv (formerly dbtvault)

The `automate-dv` package provides macros that generate hub, link, and satellite SQL from YAML metadata:

```sql
-- models/raw_vault/hubs/hub_customer.sql
{%- set source_model = 'stg_crm_customers' -%}
{%- set src_pk = 'customer_hk' -%}
{%- set src_nk = 'customer_bk' -%}
{%- set src_ldts = 'load_date' -%}
{%- set src_source = 'record_source' -%}

{{ automate_dv.hub(src_pk=src_pk, src_nk=src_nk,
                   src_ldts=src_ldts, src_source=src_source,
                   source_model=source_model) }}
```

```sql
-- models/raw_vault/satellites/sat_customer_details.sql
{%- set source_model = 'stg_crm_customers' -%}
{%- set src_pk = 'customer_hk' -%}
{%- set src_hashdiff = 'hash_diff' -%}
{%- set src_payload = ['customer_name', 'email', 'tier'] -%}
{%- set src_ldts = 'load_date' -%}
{%- set src_source = 'record_source' -%}

{{ automate_dv.sat(src_pk=src_pk, src_hashdiff=src_hashdiff,
                   src_payload=src_payload, src_ldts=src_ldts,
                   src_source=src_source,
                   source_model=source_model) }}
```

### Testing Patterns

```yaml
# schema.yml for hub_customer
models:
  - name: hub_customer
    columns:
      - name: customer_hk
        tests:
          - unique
          - not_null
      - name: customer_bk
        tests:
          - unique
          - not_null
  - name: sat_customer_details
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns: ['customer_hk', 'load_date']
```

## Data Vault vs Kimball

See [[Dimensional Modelling (Kimball)]] for the full Kimball approach.

| Aspect | Data Vault 2.0 | Kimball Dimensional |
|--------|---------------|---------------------|
| **Auditability** | Full — every record traceable to source + timestamp | Limited — SCDs track history but transforms lose lineage |
| **Agility** | High — add new sources without redesign | Medium — new sources may require schema changes |
| **Query complexity** | High — multi-hop joins across hubs/links/satellites | Low — star schema with simple FK joins |
| **Loading complexity** | Pattern-based, automatable | Custom ETL per fact/dimension |
| **History handling** | Built-in on every satellite | Explicit SCD choice per attribute |
| **Source integration** | Multiple sources per hub (one sat per source) | Must reconcile sources before loading |
| **End-user access** | Requires Business Vault / presentation layer | Direct BI tool access |
| **Schema evolution** | Add hub/link/satellite — no destructive changes | May require altering fact/dim tables |

### When to Use Each

- **Data Vault** — multiple source systems, regulatory/audit requirements, frequent source changes, large data engineering teams
- **Kimball** — single or few sources, direct BI access priority, smaller teams, well-understood business processes
- **Hybrid** — Data Vault as the integration layer (raw vault), Kimball star schemas as the presentation layer (built from Business Vault). This is the most common enterprise pattern.

## Implementation on Snowflake

Snowflake's architecture aligns well with Data Vault's parallel loading and hash-based patterns.

### Hash Functions

```sql
-- MD5 returns a 32-char hex string in Snowflake
SELECT MD5(UPPER(TRIM(customer_id))) AS customer_hk FROM raw.customers;

-- SHA2 for higher collision resistance
SELECT SHA2(UPPER(TRIM(customer_id)), 256) AS customer_hk FROM raw.customers;

-- Multi-column hash with NULL handling
SELECT MD5(CONCAT(
    COALESCE(UPPER(TRIM(col1)), '^^'), '||',
    COALESCE(UPPER(TRIM(col2)), '^^')
)) AS composite_hk;
```

### MERGE for Satellite Loading

```sql
MERGE INTO sat_customer_details tgt
USING (
    SELECT customer_hk, hash_diff, customer_name, email, tier,
           CURRENT_TIMESTAMP() AS load_date, 'CRM' AS record_source
    FROM staging.customers_prepared
) src
ON tgt.customer_hk = src.customer_hk AND tgt.end_date IS NULL
WHEN MATCHED AND tgt.hash_diff != src.hash_diff THEN
    UPDATE SET tgt.end_date = src.load_date
WHEN NOT MATCHED THEN
    INSERT (customer_hk, load_date, record_source, hash_diff, customer_name, email, tier)
    VALUES (src.customer_hk, src.load_date, src.record_source, src.hash_diff,
            src.customer_name, src.email, src.tier);
```

### Snowflake-Specific Optimizations

| Feature | Data Vault Application |
|---------|----------------------|
| **Clustering keys** | Cluster satellites on `(parent_hk, load_date)` for temporal lookups |
| **Streams** | Capture CDC from staging tables to drive incremental hub/link/satellite loads |
| **Tasks** | Schedule hub-first, then link/satellite loading in dependency chains |
| **Time Travel** | Acts as a safety net — recover from bad loads without backup tables |
| **Zero-copy clones** | Clone the raw vault for testing or development at no additional storage cost |
| **Variant columns** | Store semi-structured satellite payloads (JSON/XML) natively |

### Streams for CDC Loading

```sql
-- Create a stream on the staging table
CREATE OR REPLACE STREAM stg_customers_stream ON TABLE staging.customers;

-- Use the stream in satellite loading (only processes new/changed rows)
INSERT INTO sat_customer_details (customer_hk, load_date, record_source, hash_diff, customer_name, email, tier)
SELECT
    MD5(UPPER(TRIM(customer_id))),
    CURRENT_TIMESTAMP(),
    'CRM',
    MD5(CONCAT(COALESCE(customer_name,'^^'),'||',COALESCE(email,'^^'),'||',COALESCE(tier,'^^'))),
    customer_name, email, tier
FROM stg_customers_stream
WHERE METADATA$ACTION = 'INSERT';
```

Data Vault 2.0 works best as the **integration and historization layer** of an enterprise warehouse, with [[Dimensional Modelling (Kimball)|Kimball star schemas]] built on top as the presentation layer for BI consumption.
