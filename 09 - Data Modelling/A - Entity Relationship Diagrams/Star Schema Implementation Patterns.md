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

## 11. Implementation in Microsoft Fabric Warehouse

### TRUNCATE + INSERT Load Pattern

Fabric Warehouse star schemas use a **full reload** pattern for presentation layer tables rather than MERGE. Each dim/fact has a dedicated stored procedure that truncates the target and inserts from the T3 integration layer:

```sql
CREATE OR ALTER PROCEDURE [presentation].[usp_load_dim_customer]
AS
BEGIN
    SET NOCOUNT ON;

    TRUNCATE TABLE [T4_presentation].[presentation].[dim_customer];

    INSERT INTO [T4_presentation].[presentation].[dim_customer]
    (
        customer_id, first_name, last_name, customer_name,
        location_key, location_id,                    -- surrogate + natural key
        is_active, start_date, end_date, etl_loaded_at
    )
    SELECT
        c.id, c.first_name, c.last_name, c.name,
        dl.location_key,                              -- surrogate key lookup
        c.location_id,                                -- natural key preserved
        CASE WHEN c.inactive = 1 THEN 0 ELSE 1 END,  -- business logic
        c.start_date, c.end_date, SYSUTCDATETIME()
    FROM [T3_integration].[{schema}].[customer] c
    LEFT JOIN [T4_presentation].[presentation].[dim_location] dl
        ON c.location_id = dl.location_id;
END;
```

**Why TRUNCATE + INSERT instead of MERGE?**
- Dimensions are small (thousands of rows) — full reload is fast and simple
- Eliminates complexity of detecting deletes
- `IDENTITY` surrogate keys reset naturally (Fabric IDENTITY continues from the max value after truncate, not from 1 — but this is acceptable since surrogate keys are meaningless outside the session)
- Facts use the same pattern because T3 already filters to current rows only

### Surrogate Key Lookups in Fact Loads

Fact load procedures join to dimension tables to resolve natural keys to surrogate keys:

```sql
CREATE OR ALTER PROCEDURE [presentation].[usp_load_fact_orders]
AS
BEGIN
    SET NOCOUNT ON;

    TRUNCATE TABLE [T4_presentation].[presentation].[fact_orders];

    INSERT INTO [T4_presentation].[presentation].[fact_orders]
    (
        order_id,
        date_key, employee_key, product_key,              -- surrogate FKs
        invoice_number, line_item_id,                      -- degenerate dims
        order_date, ship_date, quantity, total_amount,
        is_zero_quantity, is_approved, etl_loaded_at
    )
    SELECT
        o.id,
        CAST(FORMAT(o.order_date, 'yyyyMMdd') AS INT),    -- date_key: YYYYMMDD
        de.employee_key,                                    -- lookup from dim_employee
        dp.product_key,                                     -- lookup from dim_product
        o.invoice_number, o.line_item_id,
        o.order_date, o.ship_date, o.quantity,
        o.total_amount,
        CASE WHEN o.quantity = 0 THEN 1 ELSE 0 END,       -- derived flag
        o.is_approved, SYSUTCDATETIME()
    FROM [T3_integration].[{schema}].[orders] o
    LEFT JOIN [T4_presentation].[presentation].[dim_employee] de
        ON o.employee_id = de.employee_id
    LEFT JOIN [T4_presentation].[presentation].[dim_product] dp
        ON o.product_id = dp.product_id;
END;
```

**Design decisions:**
- **LEFT JOIN** to dimensions — if a dimension record is missing, the fact row still loads with a NULL surrogate key (caught by `vw_referential_integrity`)
- **`date_key` as `YYYYMMDD` integer** — `CAST(FORMAT(order_date, 'yyyyMMdd') AS INT)` produces a compact, sortable key that joins to `dim_date.date_key`
- **Degenerate dimensions** — `invoice_number` and `line_item_id` are natural keys stored directly on the fact (no separate dimension table)
- **Derived columns** — `is_zero_quantity`, `day_name`, `order_time_hhmm` are computed during the load rather than in BI tool measures

### Date Dimension Generation (Recursive CTE)

The date dimension is self-generated via a recursive CTE with `OPTION (MAXRECURSION 0)`:

```sql
CREATE OR ALTER PROCEDURE [presentation].[usp_load_dim_date]
    @start_date DATE = '2015-01-01',
    @end_date   DATE = '2040-12-31'
AS
BEGIN
    SET NOCOUNT ON;
    TRUNCATE TABLE [T4_presentation].[presentation].[dim_date];

    ;WITH dates AS (
        SELECT @start_date AS dt
        UNION ALL
        SELECT DATEADD(DAY, 1, dt) FROM dates WHERE dt < @end_date
    )
    INSERT INTO [T4_presentation].[presentation].[dim_date]
        (date_key, full_date, year, quarter, month, month_name,
         day_of_week, day_name, week_of_year, is_weekend,
         fiscal_year, fiscal_quarter, year_month)
    SELECT
        CAST(FORMAT(dt, 'yyyyMMdd') AS INT),           -- YYYYMMDD integer key
        dt,
        YEAR(dt), DATEPART(QUARTER, dt), MONTH(dt),
        DATENAME(MONTH, dt),
        (DATEPART(WEEKDAY, dt) + 5) % 7 + 1,          -- 1=Monday, 7=Sunday
        DATENAME(WEEKDAY, dt),
        DATEPART(ISO_WEEK, dt),
        CASE WHEN DATEPART(WEEKDAY, dt) IN (1, 7) THEN 1 ELSE 0 END,
        -- Australian fiscal year: Jul-Jun
        CASE WHEN MONTH(dt) >= 7 THEN YEAR(dt) + 1 ELSE YEAR(dt) END,
        CASE
            WHEN MONTH(dt) IN (7,8,9)   THEN 1
            WHEN MONTH(dt) IN (10,11,12) THEN 2
            WHEN MONTH(dt) IN (1,2,3)    THEN 3
            ELSE 4
        END,
        FORMAT(dt, 'yyyy-MM')
    FROM dates
    OPTION (MAXRECURSION 0);                           -- allow 25+ years of rows
END;
```

**Fiscal calendar note:** The Australian fiscal year runs July–June, so `MONTH(dt) >= 7` maps to FY+1. Adjust the CASE expression for different fiscal calendars.

### Load Order (Dims Before Facts)

The T4 pipeline loads tables in dependency order — dimensions first, then facts that reference them:

```
1. dim_date            ← independent (run once on first deployment)
2. dim_location        ← independent
3. dim_product         ← independent
4. dim_customer        ← depends on dim_location (location_key lookup)
5. dim_employee        ← depends on dim_location (default_location_id)
6. fact_orders         ← depends on dim_location, dim_product, dim_date
7. fact_order_lines    ← depends on dim_employee, dim_date
8. fact_timesheets     ← depends on dim_employee, dim_product, dim_date
```

The T4 parent pipeline calls each SP sequentially to enforce this order. Dims could be parallelised (steps 2–5) but sequential execution is simpler and fast enough for small dimension tables.

### Security Schema Pattern (OLS + RLS)

Load procedures are placed in a `security` schema to restrict day-to-day access — users query the `presentation` schema tables but cannot execute the load SPs directly.

**Object-Level Security (OLS)** restricts column access via `GRANT`/`DENY`:

```sql
-- Standard readers: deny access to sensitive columns
GRANT SELECT ON [presentation].[dim_employee] TO [StandardReader];
DENY SELECT ON [presentation].[dim_employee]
    ([payroll_id], [hourly_rate], [pay_level]) TO [StandardReader];

-- Payroll readers: full access including sensitive columns
GRANT SELECT ON [presentation].[dim_employee] TO [PayrollReader];
```

**Row-Level Security (RLS)** filters rows by location/region via an Entra group mapping table:

```sql
-- Mapping table: Entra group → location access
CREATE TABLE presentation.rls_user_location_map (
    entra_group_id  NVARCHAR(100) NOT NULL,  -- Entra group GUID
    location_id     VARCHAR(100)  NOT NULL,   -- FK to dim_location
    is_admin        BIT           NOT NULL DEFAULT 0  -- 1 = sees all locations
);

-- Predicate function using IS_MEMBER()
CREATE FUNCTION presentation.fn_rls_location_filter(@location_id VARCHAR(100))
RETURNS TABLE WITH SCHEMABINDING
AS RETURN
    SELECT 1 AS result
    WHERE EXISTS (
        SELECT 1 FROM presentation.rls_user_location_map m
        WHERE m.is_admin = 1 AND IS_MEMBER(m.entra_group_id) = 1
    )
    OR EXISTS (
        SELECT 1 FROM presentation.rls_user_location_map m
        WHERE m.location_id = @location_id AND IS_MEMBER(m.entra_group_id) = 1
    );

-- Apply to location-scoped tables; cascades via Power BI relationships
CREATE SECURITY POLICY presentation.policy_rls_location
    ADD FILTER PREDICATE presentation.fn_rls_location_filter(location_id)
        ON presentation.dim_location,
    ADD FILTER PREDICATE presentation.fn_rls_location_filter(location_id)
        ON presentation.dim_customer
WITH (STATE = ON);
```

**RLS cascading:** Applying the filter only to `dim_location` and `dim_customer` is sufficient — Power BI's single-direction cross-filter relationships propagate the location filter from dims to facts automatically.

---

## Related Notes

- [[Dimensional Modelling (Kimball)]] — four-step design process, bus architecture
- [[SCD Type 2 Patterns]] — MERGE-based SCD2, dbt snapshots
- [[Snowflake SQL Pipeline Patterns]] — pipeline orchestration, MERGE patterns
- [[Microsoft Fabric & Azure Data Services]] — pipeline orchestration, T0-T5 architecture
