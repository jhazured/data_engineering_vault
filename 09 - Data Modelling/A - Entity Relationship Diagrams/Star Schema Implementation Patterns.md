# Star Schema Implementation Patterns

Practical patterns for implementing star schemas in Snowflake, dbt, and Microsoft Fabric, drawn from the [[Dimensional Modelling (Kimball)]] methodology.

---

## 1. Star Schema Anatomy

A star schema places a **fact table** at the centre, surrounded by **dimension tables** joined via foreign keys. Every query fans out from the fact through single-hop joins.

```
            DIM_DATE
                │
DIM_AIRPORT ────┼──── DIM_AIRLINE
 (origin)       │
          FACT_FLIGHTS
                │
DIM_AIRPORT ────┘──── DIM_AIRCRAFT
 (dest)
```

**Grain** — one row in `FACT_FLIGHTS` = one scheduled flight segment. One row in `tbl_fact_shipments` = one shipment transaction. Get the grain wrong and everything downstream breaks.

| Measure Type | Example | Behaviour |
|-------------|---------|-----------|
| **Additive** | `DISTANCE`, `revenue_usd` | SUM across any dimension |
| **Semi-additive** | `fuel_level_pct` | SUM across non-time dims only |
| **Non-additive** | `profit_margin_pct` | Must re-derive from components |

---

## 2. Fact Table Design

### Transaction Facts (Flights)

From the US-flights pipeline — surrogate dimension keys, audit columns, degenerate dims:

```sql
CREATE OR REPLACE TABLE "FACT_FLIGHTS" (
    "TRANSACTIONID" VARCHAR PRIMARY KEY,
    "AIRLINE_KEY" NUMBER, "AIRCRAFT_KEY" NUMBER,
    "ORIGIN_AIRPORT_KEY" NUMBER, "DEST_AIRPORT_KEY" NUMBER, "DATE_KEY" NUMBER,
    "DEPDELAY" NUMBER, "ARRDELAY" NUMBER, "DISTANCE" NUMBER,
    "CANCELLED" BOOLEAN DEFAULT FALSE, "DIVERTED" BOOLEAN DEFAULT FALSE,
    "DEPDELAYGT15" NUMBER(1) DEFAULT 0, "DISTANCEGROUP" VARCHAR(50),
    "LOAD_BATCH_ID" NUMBER, "SOURCE_FILE" VARCHAR,
    "CREATED_AT" TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP
);
```

- **Surrogate keys** (`AIRLINE_KEY`, `DATE_KEY`) for join performance
- **Degenerate dimension** — `FLIGHTNUM` in the fact (no separate table)
- **Audit columns** — `LOAD_BATCH_ID`, `SOURCE_FILE`, `CREATED_AT`

**Periodic snapshots** — one row per vehicle per day; semi-additive measures like `capacity_utilization_pct`. **Accumulating snapshots** — one row per process instance updated at milestones; multiple date FKs are role-playing dims. **Factless facts** — row existence is the measure (no numeric payload).

---

## 3. Dimension Design

### Date Dimension (YYYYMMDD Key)

```sql
CREATE OR REPLACE TABLE "DIM_DATE" (
    "DATE_KEY" NUMBER PRIMARY KEY,  -- 20260315
    "FULL_DATE" DATE NOT NULL, "YEAR" NUMBER(4), "QUARTER" NUMBER(1),
    "MONTH" NUMBER(2), "MONTH_NAME" VARCHAR(20), "DAY_OF_MONTH" NUMBER(2),
    "DAY_OF_WEEK" NUMBER(1), "DAY_NAME" VARCHAR(20),
    "IS_WEEKEND" BOOLEAN NOT NULL, UNIQUE ("FULL_DATE")
);
```

Include fiscal calendar, holiday flags, and `IS_WEEKEND` for business-hour filtering.

### Role-Playing Dimensions (Origin / Dest Airport)

Same physical table joined twice with different aliases:

```sql
JOIN "DIM_AIRPORT" o ON f.ORIGIN_AIRPORT_KEY = o.AIRPORT_KEY AND o.IS_CURRENT = TRUE
JOIN "DIM_AIRPORT" d ON f.DEST_AIRPORT_KEY   = d.AIRPORT_KEY AND d.IS_CURRENT = TRUE
```

The view exposes `ORIGAIRPORTNAME` / `DESTAIRPORTNAME` — same table, different role.

### Junk Dimensions

Combine low-cardinality flags into one table: `dim_flight_flags (flag_key, is_cancelled, is_diverted, is_delayed_gt15, is_next_day_arrival)`.

### Degenerate Dimensions

`FLIGHTNUM`, `invoice_number`, `tracking_number` — identifiers with no additional attributes, stored directly in the fact row.

### Outrigger Dimensions

A dimension hanging off another dimension (e.g., `dim_geography` off `dim_airport`). Introduces a snowflake join — use sparingly.

---

## 4. Conformed Dimensions

Shared dimensions enable cross-process analysis. Design once, reuse everywhere.

| Business Process | dim_date | dim_customer | dim_vehicle | dim_location | dim_route |
|-----------------|----------|-------------|-------------|-------------|-----------|
| Shipments       | X | X | X | X | X |
| Maintenance     | X |   | X | X |   |
| Telemetry       | X |   | X |   | X |
| Billing         | X | X |   |   |   |

From the logistics platform: `tbl_dim_date` joins to `tbl_fact_shipments`, `tbl_fact_vehicle_telemetry`, and `tbl_fact_route_performance`. This prevents stovepipe marts that cannot be joined.

---

## 5. Naming Conventions

| Prefix | Usage | Example |
|--------|-------|---------|
| `tbl_fact_*` | Fact tables | `tbl_fact_shipments` |
| `tbl_dim_*` | Dimension tables | `tbl_dim_customer`, `tbl_dim_date` |
| `tbl_stg_*` | Staging views | `tbl_stg_customers` |
| `vw_*` | Consumption views | `vw_consolidated_dashboard` |

### Schema Organisation (Snowflake / dbt)

`ANALYTICS_RAW` (landing) → `DBT_ANALYTICS_STAGING` (8 views) → `DBT_ANALYTICS_MARTS` (8 dims + 5 facts) → `DBT_CONSUMPTION` (5 views). Plus `DBT_ANALYTICS_ML_FEATURES` (6 tables), `MONITORING`, `GOVERNANCE`, `SNAPSHOTS`.

### T0-T5 Layer Naming (Microsoft Fabric)

| Layer | Purpose | Key Objects |
|-------|---------|-------------|
| **T0** | Control & orchestration | Pipeline metadata, watermarks, error logs |
| **T1** | Raw landing (VARIANT) | Schema-agnostic ingestion, transient |
| **T2** | Historical record | `t2.dim_*` (SCD2), `t2.fact_*`, MERGE procs |
| **T3** | Business transformations | `t3.ref_*`, `t3.*_enriched`, star schema |
| **T3._FINAL** | Validated snapshots | Zero-copy clones (`*_FINAL`) |
| **T5** | Presentation | `t5.vw_*` (views only, no base tables) |

Flow: T1 → T2 (SCD2) → T3 (star schema) → T3._FINAL (stable clones) → T5 (views) → Semantic Layer.

---

## 6. Implementation in Snowflake

### Surrogate Keys with AUTOINCREMENT

```sql
CREATE TABLE tbl_dim_customer (
    customer_key   NUMBER AUTOINCREMENT PRIMARY KEY,
    customer_id    VARCHAR NOT NULL,       -- natural key
    customer_name  VARCHAR(255), customer_tier VARCHAR(20),
    effective_date DATE, expiry_date DATE, is_current BOOLEAN DEFAULT TRUE,
    UNIQUE (customer_id, effective_date)   -- SCD2 uniqueness
);
```

See [[SCD Type 2 Patterns]] for the full MERGE-based SCD2 implementation.

### Staging View, MERGE Load, Analytical View

```sql
-- Staging: clean and standardise before dimension load
CREATE VIEW tbl_stg_customers AS
SELECT customer_id, TRIM(customer_name) AS customer_name,
       UPPER(customer_tier) AS customer_tier
FROM analytics_raw.customers;

-- MERGE for fact upsert
MERGE INTO tbl_fact_shipments tgt
USING dbt_analytics_staging.tbl_stg_shipments src ON tgt.shipment_id = src.shipment_id
WHEN MATCHED THEN UPDATE SET tgt.delivery_date = src.delivery_date
WHEN NOT MATCHED THEN INSERT (shipment_id, customer_key, date_key, revenue_usd, created_at)
VALUES (src.shipment_id, src.customer_key, src.date_key, src.revenue_usd, CURRENT_TIMESTAMP);

-- Analytical view: fact joined to all dimensions (flights pipeline)
CREATE OR REPLACE VIEW "VW_FLIGHTS" AS
SELECT f."TRANSACTIONID", a."AIRLINENAME",
       o."ORIGAIRPORTNAME", d."DESTAIRPORTNAME",
       dt."FULL_DATE" AS FLIGHTDATE, f."DISTANCE", f."DEPDELAY", f."ARRDELAY"
FROM "FACT_FLIGHTS" f
JOIN "DIM_AIRLINE"  a  ON f.AIRLINE_KEY        = a.AIRLINE_KEY  AND a.IS_CURRENT = TRUE
JOIN "DIM_AIRPORT"  o  ON f.ORIGIN_AIRPORT_KEY = o.AIRPORT_KEY  AND o.IS_CURRENT = TRUE
JOIN "DIM_AIRPORT"  d  ON f.DEST_AIRPORT_KEY   = d.AIRPORT_KEY  AND d.IS_CURRENT = TRUE
JOIN "DIM_DATE"     dt ON f.DATE_KEY           = dt.DATE_KEY;
```

See [[Snowflake SQL Pipeline Patterns]] for full pipeline orchestration.

---

## 7. Implementation in dbt

### ref() for Dimension Lookups

```sql
SELECT s.shipment_id, c.customer_key, v.vehicle_key, d.date_key, s.revenue_usd
FROM {{ ref('tbl_stg_shipments') }} s
JOIN {{ ref('tbl_dim_customer') }} c ON s.customer_id = c.customer_id AND c.is_current = TRUE
JOIN {{ ref('tbl_dim_vehicle') }}  v ON s.vehicle_id  = v.vehicle_id  AND v.is_current = TRUE
JOIN {{ ref('tbl_dim_date') }}     d ON s.pickup_date = d.full_date
```

### Incremental Fact Loading

```sql
{{ config(materialized='incremental', unique_key='shipment_id') }}
SELECT ... FROM {{ ref('tbl_stg_shipments') }} s
{% if is_incremental() %}
WHERE s.updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

### schema.yml Relationship Tests

```yaml
models:
  - name: tbl_fact_shipments
    columns:
      - name: customer_key
        tests: [not_null, {relationships: {to: ref('tbl_dim_customer'), field: customer_key}}]
      - name: date_key
        tests: [not_null, {relationships: {to: ref('tbl_dim_date'), field: date_key}}]
```

### dbt_utils.star Macro

```sql
SELECT {{ dbt_utils.star(from=ref('tbl_dim_customer'), except=['_loaded_at']) }}
FROM {{ ref('tbl_dim_customer') }}
```

---

## 8. Schema Architecture

### Multi-Schema Design

`ANALYTICS_RAW` (landing) → `DBT_ANALYTICS_STAGING` (8 views) → `DBT_ANALYTICS_MARTS` (8 dims + 5 facts) → `DBT_CONSUMPTION` (5 views), plus `DBT_ANALYTICS_ML_FEATURES` (6 tables), `MONITORING`, `GOVERNANCE`, `SNAPSHOTS`.

### Access Control Per Schema

| Schema | Role | Permissions |
|--------|------|-------------|
| `ANALYTICS_RAW` | `LOADER_ROLE` | INSERT, SELECT |
| `DBT_ANALYTICS_MARTS` | `TRANSFORMER_ROLE` | CREATE TABLE, SELECT |
| `DBT_CONSUMPTION` | `ANALYST_ROLE` | SELECT only |

Raw → Staging (type validation) → Marts (star schema joins) → Consumption (KPIs) and ML Features (feature engineering).

---

## 9. Data Dictionary Patterns

### Field-Level Definitions

| Column | Type | Business Definition | Valid Values |
|--------|------|-------------------|--------------|
| `customer_tier` | VARCHAR | Business classification | PLATINUM, GOLD, SILVER, BRONZE |
| `on_time_delivery_flag` | BOOLEAN | SLA compliance | delivery_date <= requested_date |
| `profit_margin_pct` | DECIMAL | Profit percentage | (revenue - cost) / revenue * 100 |

### Quality Metrics & Business Glossary

| Table | SLA | Completeness | Accuracy |
|-------|-----|-------------|----------|
| `tbl_fact_shipments` | 1 hour | 99.5% | Revenue +/- $0.01 |
| `tbl_fact_vehicle_telemetry` | 5 min | 95% | Distance +/- 1% |

Maintain a glossary: **On-Time Delivery** (delivery_date <= requested_date), **Route Efficiency** ((optimal / actual) * 100), **Carbon Footprint** (distance * fuel_per_100km * 2.31 EPA factor).

---

## 10. Architecture Decision Records

ADRs capture the **why** behind design choices. Format:

```
**Status**: Proposed | Accepted | Deprecated | Superseded
**Context**: What problem? **Decision**: What did we decide?
**Rationale**: Why? **Alternatives**: What else? **Consequences**: Trade-offs?
```

### Example ADRs (from Fabric wiki)

**ADR-001 — Data Factory for T1 Ingestion**: Accepted. Use Data Factory exclusively for ingestion (optimised for data movement, supports ADLS/SQL/APIs). Alternatives rejected: Dataflows Gen2 (better for transforms), Notebooks (too complex). Consequence: clear ingestion/transformation separation.

**ADR-005 — Zero-Copy Clones for T3._FINAL**: Accepted. Use clones for stable snapshots (no storage overhead, fast metadata op, pipeline isolation). Alternatives rejected: physical copy (duplication), views (unstable). Consequence: requires clone refresh procedure.

The Fabric wiki maintains 12 ADRs covering ingestion, transformations, SCD2, VARIANT storage, cloning, Direct Lake, and error handling.

---

## Related Notes

- [[Dimensional Modelling (Kimball)]] — four-step design process, bus architecture
- [[SCD Type 2 Patterns]] — MERGE-based SCD2, dbt snapshots
- [[Snowflake SQL Pipeline Patterns]] — pipeline orchestration, MERGE patterns
