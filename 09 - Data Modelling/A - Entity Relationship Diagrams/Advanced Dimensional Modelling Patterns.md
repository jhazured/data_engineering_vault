---
tags:
  - data-modelling
  - kimball
  - dimensional-modelling
  - star-schema
  - advanced-patterns
---

# Advanced Dimensional Modelling Patterns

This note covers advanced techniques from Kimball's dimensional modelling methodology that go beyond the basic star schema. Each pattern addresses a specific modelling challenge -- multi-valued relationships, rapidly changing attributes, sparse flags, variable-depth hierarchies, and performance optimisation. For foundational concepts, see [[Dimensional Modelling (Kimball)]].

---

## Factless Fact Tables

### Definition

A factless fact table contains no numeric measures. It records relationships between dimensions at a point in time -- the fact that a set of dimensional entities came together constitutes the measurement itself.

### When To Use

- **Event tracking** -- recording that an event occurred (student attended a class, customer received a communication) where the event has no associated metric.
- **Coverage analysis** -- capturing all *possible* events so you can identify what *did not* happen by subtracting actual activity from the coverage set.

### Schema Example

```
-- Coverage factless fact table (what promotions are available)
fact_promotion_coverage (
    date_key           INT  REFERENCES dim_date,
    product_key        INT  REFERENCES dim_product,
    store_key          INT  REFERENCES dim_store,
    promotion_key      INT  REFERENCES dim_promotion,
    promotion_count    INT  DEFAULT 1   -- dummy fact to simplify counting
)

-- Activity fact table (what actually sold)
fact_pos_sales (
    date_key           INT  REFERENCES dim_date,
    product_key        INT  REFERENCES dim_product,
    store_key          INT  REFERENCES dim_store,
    promotion_key      INT  REFERENCES dim_promotion,
    sales_quantity     INT,
    sales_amount       DECIMAL(12,2)
)
```

The coverage table has one row per product on promotion per store per day, regardless of whether the product sold.

### SQL Example -- Finding Products on Promotion That Did Not Sell

```sql
-- Step 1: products on promotion
WITH promoted AS (
    SELECT date_key, product_key, store_key
    FROM fact_promotion_coverage
    WHERE date_key = 20260315
),
-- Step 2: products that actually sold
sold AS (
    SELECT date_key, product_key, store_key
    FROM fact_pos_sales
    WHERE date_key = 20260315
)
-- Set difference: promoted but not sold
SELECT p.date_key, p.product_key, p.store_key
FROM promoted p
LEFT JOIN sold s
    ON  p.date_key    = s.date_key
    AND p.product_key = s.product_key
    AND p.store_key   = s.store_key
WHERE s.product_key IS NULL;
```

> **Tip:** Include a dummy fact column (e.g. `promotion_count = 1`) so BI tools can aggregate without counting foreign keys directly.

---

## Junk Dimensions

### Definition

A junk dimension groups miscellaneous low-cardinality flags and indicators into a single dimension table, replacing multiple boolean or short-code columns in the fact table with one surrogate foreign key. Kimball also calls this a *transaction profile dimension*.

### When To Use

- The fact table would otherwise contain many two- or three-value flag columns (e.g. `is_expedited`, `is_hazardous`, `payment_type`).
- The Cartesian product of all flag values is manageably small (typically fewer than ~100,000 rows).
- Avoids creating a "centipede" fact table with dozens of tiny dimension foreign keys.

### Schema Example

```
dim_order_profile (
    order_profile_key     INT  PRIMARY KEY,  -- surrogate
    payment_type          VARCHAR(20),       -- 'Cash', 'Visa', 'MasterCard'
    order_type            VARCHAR(20),       -- 'Inbound', 'Outbound'
    commission_flag       VARCHAR(20),       -- 'Commissionable', 'Non-Commissionable'
    gift_wrap_flag        VARCHAR(10),       -- 'Yes', 'No'
    return_authorised     VARCHAR(10)        -- 'Yes', 'No'
)

fact_order_line (
    date_key              INT  REFERENCES dim_date,
    customer_key          INT  REFERENCES dim_customer,
    product_key           INT  REFERENCES dim_product,
    order_profile_key     INT  REFERENCES dim_order_profile,  -- single FK
    order_number          INT,               -- degenerate dimension
    quantity              INT,
    extended_amount       DECIMAL(12,2)
)
```

### Population Strategy

| Approach | When To Use |
|----------|-------------|
| Pre-build full Cartesian product | Few attributes with few values (e.g. 10 binary flags = 1,024 rows) |
| Build on encounter during ETL | Many possible combinations but only a fraction appears in data |

### SQL Example -- Filtering by Junk Dimension

```sql
SELECT
    d.calendar_month,
    SUM(f.extended_amount) AS total_amount
FROM fact_order_line f
JOIN dim_order_profile p ON f.order_profile_key = p.order_profile_key
JOIN dim_date d          ON f.date_key = d.date_key
WHERE p.payment_type = 'Visa'
  AND p.order_type   = 'Inbound'
GROUP BY d.calendar_month;
```

---

## Degenerate Dimensions

### Definition

A degenerate dimension is a dimension key that lives directly in the fact table with no corresponding dimension table. It typically represents a transaction control number (invoice number, order number, tracking number) whose only content is the identifier itself -- there are no additional descriptive attributes worth storing in a separate table.

### When To Use

- The identifier groups fact rows (e.g. all line items on an invoice) but carries no descriptive attributes beyond the number itself.
- Most common with transaction fact tables and accumulating snapshot fact tables.
- Marked with `(DD)` notation in Kimball diagrams.

### Schema Example

```
fact_order_line (
    date_key              INT  REFERENCES dim_date,
    customer_key          INT  REFERENCES dim_customer,
    product_key           INT  REFERENCES dim_product,
    order_number          INT,       -- degenerate dimension (DD)
    line_number           INT,       -- degenerate dimension (DD)
    quantity              INT,
    unit_price            DECIMAL(10,2),
    extended_amount       DECIMAL(12,2)
)
```

No `dim_order` table exists because `order_number` has no attributes beyond itself.

### SQL Example -- Grouping by Degenerate Dimension

```sql
-- Reconstruct an order header from line-level facts
SELECT
    f.order_number,
    c.customer_name,
    d.full_date           AS order_date,
    COUNT(*)              AS line_count,
    SUM(f.extended_amount) AS order_total
FROM fact_order_line f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date d     ON f.date_key = d.date_key
GROUP BY f.order_number, c.customer_name, d.full_date;
```

### When To Promote to a Full Dimension

Consider assigning a surrogate key and creating a dimension table when:

- Transaction numbers are not unique across locations or get reused.
- The identifier is a bulky alphanumeric string (a surrogate saves storage in the fact table).
- You need to drill across on the transaction number using BI tools that require dimension joins.

---

## Bridge Tables

### Definition

A bridge table resolves many-to-many relationships or variable-depth hierarchies that cannot be modelled with simple foreign keys in a star schema. It sits between the fact table and a dimension, containing one row for every valid pairing (or path).

### When To Use

- **Multi-valued dimensions** -- a patient with multiple simultaneous diagnoses, an account with multiple owners.
- **Ragged/variable-depth hierarchies** -- an organisational tree of indeterminate depth where SQL recursion is insufficient (no support for alternative hierarchies, shared ownership, or time-variance).

### Schema Example -- Multi-Valued Dimension Bridge

```
fact_treatment (
    date_key              INT  REFERENCES dim_date,
    patient_key           INT  REFERENCES dim_patient,
    diagnosis_group_key   INT,           -- FK to bridge
    treatment_charge      DECIMAL(12,2)
)

bridge_patient_diagnosis (
    diagnosis_group_key   INT,           -- groups diagnoses for one treatment
    diagnosis_key         INT  REFERENCES dim_diagnosis,
    allocation_factor     DECIMAL(5,4)   -- weighting (sums to 1.0 per group)
)

dim_diagnosis (
    diagnosis_key         INT  PRIMARY KEY,
    diagnosis_code        VARCHAR(10),
    diagnosis_description VARCHAR(200),
    diagnosis_category    VARCHAR(50)
)
```

### Schema Example -- Hierarchy Bridge

```
dim_organisation (
    organisation_key      INT  PRIMARY KEY,
    organisation_name     VARCHAR(100),
    ...
)

bridge_org_hierarchy (
    parent_organisation_key  INT  REFERENCES dim_organisation,
    child_organisation_key   INT  REFERENCES dim_organisation,
    depth_from_parent        INT,
    is_highest_parent        BOOLEAN,
    is_lowest_child          BOOLEAN
)
```

The bridge contains one row for every path from a parent to each descendant, including a self-referencing row (`parent = child, depth = 0`). For the 13-node tree in Kimball's accounting example, this produces 43 rows.

### SQL Example -- Rolling Up a Hierarchy

```sql
-- Total budget for organisation node 1 and all descendants
SELECT SUM(f.amount) AS total_budget
FROM fact_general_ledger f
JOIN bridge_org_hierarchy b
    ON f.organisation_key = b.child_organisation_key
WHERE b.parent_organisation_key = 1;

-- Leaf nodes only
SELECT SUM(f.amount) AS leaf_budget
FROM fact_general_ledger f
JOIN bridge_org_hierarchy b
    ON f.organisation_key = b.child_organisation_key
WHERE b.parent_organisation_key = 1
  AND b.is_lowest_child = TRUE;
```

> **Caution:** When constraining the parent dimension to multiple rows (e.g. `state = 'Victoria'`), use a subquery to avoid double-counting descendants that appear under more than one matching parent.

### Time-Varying Bridge Tables

When the many-to-many relationship changes over time (e.g. bank account ownership), add `effective_date` and `expiration_date` columns to the bridge and constrain to a specific point in time at query time. See [[SCD Type 2 Patterns]] for related time-variance techniques.

---

## Mini-Dimensions (SCD Type 4)

### Definition

A mini-dimension splits rapidly changing or frequently analysed attributes out of a large base dimension into a separate, much smaller dimension table. The fact table carries foreign keys to both the base dimension and the mini-dimension.

### When To Use

- A dimension has millions of rows and a subset of attributes changes frequently (e.g. customer demographics: age band, income bracket, purchase frequency score).
- Type 2 tracking on the base dimension would cause unacceptable row explosion.
- The volatile attributes are commonly used for filtering and grouping.

### Schema Example

```
dim_customer (
    customer_key          INT  PRIMARY KEY,   -- surrogate
    customer_id           VARCHAR(20),        -- natural key
    customer_name         VARCHAR(100),
    customer_address      VARCHAR(200),
    ...
    current_demographics_key INT              -- Type 5 outrigger (see below)
)

dim_demographics (
    demographics_key      INT  PRIMARY KEY,   -- surrogate
    age_band              VARCHAR(10),        -- '21-25', '26-30', ...
    income_level          VARCHAR(20),        -- '<$30,000', '$30,000-39,999', ...
    purchase_frequency    VARCHAR(10)         -- 'Low', 'Medium', 'High'
)

fact_customer_activity (
    date_key              INT  REFERENCES dim_date,
    customer_key          INT  REFERENCES dim_customer,
    demographics_key      INT  REFERENCES dim_demographics,  -- as-was profile
    ...
    order_amount          DECIMAL(12,2)
)
```

Continuously variable attributes (income, age) are converted to **banded ranges** in the mini-dimension to keep the row count manageable.

### How Change Tracking Works

Each time a fact row is loaded, the ETL process looks up the customer's current demographic profile and assigns the matching `demographics_key`. If John Smith turns 26, his next fact row gets a different demographics key -- but older fact rows retain the prior key, preserving a history of profile changes without adding rows to the customer dimension.

---

## Slowly Changing Dimension Types Beyond Type 2

The base SCD types are covered in [[Dimensional Modelling (Kimball)]]. This section details the less common types and the hybrid techniques (Types 5, 6, 7) that combine them.

### Type 0 -- Retain Original

The attribute value is set once and never updated. Appropriate for values labelled "original" (original credit score, date of birth) and most date dimension attributes.

### Type 1 -- Overwrite

The current value overwrites the previous value, destroying history. Simple to implement but requires recomputation of any aggregate fact tables or OLAP cubes built on the changed attribute.

### Type 3 -- Add New Attribute (Alternate Reality)

A new column is added to hold the prior value while the main column is overwritten (Type 1 style). Enables users to group facts by either the current or prior attribute value.

```
dim_product (
    product_key           INT  PRIMARY KEY,
    sku                   VARCHAR(20),
    product_description   VARCHAR(100),
    current_department    VARCHAR(50),    -- overwritten (Type 1)
    prior_department      VARCHAR(50),    -- preserves previous value
    ...
)
```

**Best for:** Predictable, en-masse reorganisations (sales territory realignment, product line restructure) where users need both views temporarily. **Not suitable for** unpredictable attribute changes (e.g. customer address moves).

Multiple Type 3 columns can be used when changes follow a regular rhythm (e.g. `2024_department`, `2025_department`, `current_department`).

### Type 5 -- Mini-Dimension + Type 1 Outrigger

Combines Type 4 (mini-dimension in the fact table) with a Type 1 current reference in the base dimension. The base dimension carries `current_demographics_key` as an outrigger FK that is overwritten whenever the profile changes. This enables:

- **As-was analysis** via the fact table's demographics FK.
- **As-is analysis** via the base dimension's current outrigger FK.
- **Profile counting without a fact table** (e.g. "how many customers are currently in the High income band?").

Logically presented to BI tools as a single combined view with prefixed column names (e.g. `current_age_band` vs `age_band`).

### Type 6 -- Hybrid (Type 1 + Type 2 + Type 3)

Each Type 2 dimension row carries both the historically accurate attribute value and a Type 1 "current" version of the same attribute. When a change occurs:

1. A new Type 2 row is inserted with the new value in both the historic and current columns.
2. All prior Type 2 rows for that entity have their current column overwritten to the new value.

```
dim_product (
    product_key           INT  PRIMARY KEY,
    sku                   VARCHAR(20),
    product_description   VARCHAR(100),
    historic_department   VARCHAR(50),    -- Type 2 (as-was)
    current_department    VARCHAR(50),    -- Type 1 (as-is, overwritten on all rows)
    row_effective_date    DATE,
    row_expiration_date   DATE,
    current_row_indicator VARCHAR(10)
)
```

Users filter on `historic_department` for as-was reporting or `current_department` for as-is reporting.

### Type 7 -- Dual Type 1 and Type 2 Dimensions

The fact table carries two keys to the same dimension:

- A **surrogate key** (for Type 2 as-was joins).
- A **durable natural key** (for Type 1 as-is joins).

```
fact_sales (
    date_key              INT,
    product_surrogate_key INT,   -- joins for as-was (Type 2)
    product_durable_key   INT,   -- joins for as-is  (Type 1)
    ...
)
```

Two views are deployed:

| View | Join Column | Constraint | Perspective |
|------|-------------|------------|-------------|
| `v_product_as_was` | `product_surrogate_key` | None on current flag | Historical profile at time of fact |
| `v_product_as_is` | `product_durable_key` | `current_row_indicator = 'Current'` | Latest profile applied to all facts |

### Choosing a Hybrid Technique

| Requirement | Recommended Type |
|-------------|-----------------|
| Rapidly changing attributes on a large dimension | Type 4 / Type 5 |
| As-was *and* as-is on the same attribute, simple implementation | Type 6 |
| As-was *and* as-is without modifying the dimension structure | Type 7 |
| Temporary dual view during a reorganisation | Type 3 |

> **Guidance:** Do not pursue hybrid types unless the business explicitly requires both as-was and as-is reporting. The added ETL complexity is significant. See [[SCD Type 2 Patterns]] for implementation details using dbt snapshots.

---

## Aggregate Fact Tables

### Definition

Aggregate fact tables are pre-computed numeric rollups of atomic fact table data, built to accelerate query performance. They function like database indexes -- invisible to end users, selected automatically by the BI layer through a process called *aggregate navigation*.

### When To Use

- Queries against the atomic fact table are too slow for acceptable dashboard response times.
- The rollup ratio is significant (e.g. daily to monthly reduces rows by ~30x; session-level to monthly demographic summary can achieve 100x+).
- The BI tool supports aggregate navigation (automatic aggregate selection at query time).

### Schema Example

```
-- Atomic fact table
fact_session (
    date_key              INT  REFERENCES dim_date,
    customer_key          INT  REFERENCES dim_customer,
    page_key              INT  REFERENCES dim_page,
    session_outcome_key   INT  REFERENCES dim_session_outcome,
    session_duration      INT,
    page_views            INT,
    order_amount          DECIMAL(12,2)
)

-- Aggregate fact table (monthly grain)
fact_session_monthly_agg (
    month_key             INT  REFERENCES dim_month,           -- shrunken dim
    demographic_key       INT  REFERENCES dim_demographic,     -- shrunken dim
    entry_page_key        INT  REFERENCES dim_page,
    session_outcome_key   INT  REFERENCES dim_session_outcome,
    session_count         INT,
    total_duration        INT,
    total_page_views      INT,
    total_order_amount    DECIMAL(12,2)
)
```

### Key Design Rules

1. **Shrunken conformed dimensions** -- aggregate fact tables join to subset dimensions (e.g. `dim_month` is a shrunken rollup of `dim_date`). These must conform to the base dimensions.
2. **Additive facts only** -- only measures that can be summed are stored in aggregates. Semi-additive or non-additive measures require special handling.
3. **Type 1 ripple effect** -- any Type 1 overwrite on a dimension attribute used in an aggregate requires recomputation of that aggregate. Type 2 changes do not have this problem.
4. **Transparent to users** -- aggregates should never be queried directly by business users. The BI layer or materialised view system selects the appropriate level.

### SQL Example -- Building an Aggregate

```sql
INSERT INTO fact_session_monthly_agg
SELECT
    dm.month_key,
    dd.demographic_key,
    f.entry_page_key,
    f.session_outcome_key,
    COUNT(*)              AS session_count,
    SUM(f.session_duration) AS total_duration,
    SUM(f.page_views)     AS total_page_views,
    SUM(f.order_amount)   AS total_order_amount
FROM fact_session f
JOIN dim_date d           ON f.date_key = d.date_key
JOIN dim_month dm         ON d.month_key = dm.month_key
JOIN dim_customer c       ON f.customer_key = c.customer_key
JOIN dim_demographic dd   ON c.demographic_key = dd.demographic_key
GROUP BY dm.month_key, dd.demographic_key, f.entry_page_key, f.session_outcome_key;
```

> In modern cloud warehouses (Snowflake, BigQuery, Databricks), materialised views and automatic clustering often reduce the need for manually managed aggregate tables. See [[Star Schema Implementation Patterns]] for platform-specific guidance.

---

## Conformed Dimensions

### Definition

Dimensions conform when they share the same column names, domain values, and semantic meaning across multiple fact tables. Conformed dimensions are the mechanism by which independently designed star schemas can be integrated into a coherent enterprise data warehouse.

### When To Use

- Always. Conformed dimensions are not optional in Kimball's methodology -- they are the architectural backbone of the [[Dimensional Modelling (Kimball)|Enterprise Data Warehouse Bus Architecture]].

### How Conformance Works

| Conformance Type | Description | Example |
|------------------|-------------|---------|
| **Identical** | Same dimension table shared across fact tables | `dim_date` used by `fact_sales`, `fact_inventory`, `fact_returns` |
| **Shrunken rollup** | Subset of columns at a coarser grain | `dim_month` (month, quarter, year) derived from `dim_date` |
| **Shrunken row subset** | Subset of rows at the same grain | `dim_corporate_customer` containing only B2B rows from `dim_customer` |

### Drilling Across

Conformed dimensions enable *drilling across* -- running separate queries against different fact tables and merging results on shared dimension attributes:

```sql
-- Drill-across: compare shipment volume with return rate by customer
WITH shipments AS (
    SELECT c.customer_name, SUM(f.shipment_count) AS total_shipments
    FROM fact_shipments f
    JOIN dim_customer c ON f.customer_key = c.customer_key
    GROUP BY c.customer_name
),
returns AS (
    SELECT c.customer_name, SUM(f.return_count) AS total_returns
    FROM fact_returns f
    JOIN dim_customer c ON f.customer_key = c.customer_key
    GROUP BY c.customer_name
)
SELECT
    COALESCE(s.customer_name, r.customer_name) AS customer_name,
    s.total_shipments,
    r.total_returns
FROM shipments s
FULL OUTER JOIN returns r ON s.customer_name = r.customer_name;
```

### Governance

Conformed dimensions must be defined once in collaboration with business data stewards and reused across all fact tables. The Enterprise Data Warehouse Bus Matrix maps business processes (rows) to conformed dimensions (columns) to ensure nothing is built in isolation. See [[Dimensional Modelling (Kimball)#Kimball's Bus Architecture]].

---

## Cross-Reference Summary

| Pattern | Related Notes |
|---------|---------------|
| Foundational star schema design | [[Dimensional Modelling (Kimball)]] |
| Star schema DDL and implementation | [[Star Schema Implementation Patterns]] |
| SCD Type 2 with dbt snapshots | [[SCD Type 2 Patterns]] |
| Data Vault as alternative methodology | [[Data Vault 2.0 Modelling]] |

---

*Source: Kimball, R. & Ross, M. (2013). The Data Warehouse Toolkit, 3rd Edition.*
