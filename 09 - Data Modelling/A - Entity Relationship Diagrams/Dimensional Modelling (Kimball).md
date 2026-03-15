# Dimensional Modelling (Kimball)

The Kimball methodology designs data warehouses for **usability and query performance** using star schemas with fact and dimension tables.

## Four-Step Design Process

1. **Select the business process** — what are you measuring? (e.g., shipments, sales, claims)
2. **Declare the grain** — what does one row represent? (e.g., one shipment, one line item)
3. **Identify the dimensions** — who/what/where/when context (customer, product, date, location)
4. **Identify the facts** — numeric measurements at the grain (revenue, weight, duration)

**The grain is the most critical decision** — get it wrong and everything downstream breaks.

## Star Schema

```
              ┌──────────────┐
              │  dim_date    │
              └──────┬───────┘
                     │
┌──────────────┐     │     ┌──────────────┐
│ dim_customer ├─────┼─────┤ dim_vehicle  │
└──────────────┘     │     └──────────────┘
                     │
              ┌──────┴───────┐
              │ fact_shipments│
              └──────┬───────┘
                     │
┌──────────────┐     │     ┌──────────────┐
│ dim_location ├─────┘─────┤  dim_route   │
└──────────────┘           └──────────────┘
```

- **Fact table** at the center — contains foreign keys to dimensions + numeric measures
- **Dimension tables** surround the fact — contain descriptive attributes
- **Simple joins** — fact to dimension via single FK (no multi-hop joins)

### Star vs Snowflake

| Aspect | Star Schema | Snowflake Schema |
|--------|------------|------------------|
| Structure | Flat dimensions | Normalized dimensions (sub-tables) |
| Query complexity | Simple (one join per dim) | More joins |
| Query performance | Faster (fewer joins) | Slower |
| Storage | More redundancy | Less redundancy |
| Recommendation | **Preferred for analytics** | Use only when dimension is very large |

## Fact Table Types

### Transaction Facts

One row per event/transaction at the lowest grain:

```
fact_shipments:
  shipment_id, date_key, customer_key, vehicle_key,
  revenue, weight_kg, distance_km, fuel_cost, is_on_time
```

- Most common type
- Sparse — rows only exist when events occur
- Additive measures (revenue, weight) can be summed across any dimension

### Periodic Snapshot Facts

One row per entity per time period (daily, weekly, monthly):

```
fact_daily_inventory:
  date_key, warehouse_key, product_key,
  quantity_on_hand, quantity_on_order, days_of_supply
```

- Dense — a row exists for every entity in every period (even if nothing changed)
- Semi-additive measures (quantity_on_hand can't be summed across time, only across warehouses)

### Accumulating Snapshot Facts

One row per process instance, updated as milestones occur:

```
fact_order_fulfillment:
  order_key, order_date_key, ship_date_key, delivery_date_key,
  order_to_ship_days, ship_to_delivery_days, total_days
```

- Row is updated multiple times as the process progresses
- Multiple date keys (one per milestone)
- Least common type

## Dimension Types

### Slowly Changing Dimensions (SCD)

| Type | Behavior | Use Case |
|------|----------|----------|
| **Type 0** | Never changes | Original value preserved forever (birth date) |
| **Type 1** | Overwrite | No history needed (fix typos) |
| **Type 2** | New row per change | Full history (customer tier, address) |
| **Type 3** | Previous + current columns | Track one prior value only |
| **Type 4** | Mini-dimension | Rapidly changing attributes split into separate table |
| **Type 5** | Mini-dimension + Type 1 outrigger | Type 4 + current mini-dim FK in base dimension |
| **Type 6** | Hybrid (1+2+3) | Type 2 rows with current-value columns added |
| **Type 7** | Dual Type 1 and Type 2 | Fact has surrogate key (as-was) + durable key (as-is) |

**Type 2 is the most common for analytics** — see [[SCD Type 2 Patterns]] for implementation with dbt snapshots.

### Conformed Dimensions

Dimensions shared across multiple fact tables / business processes:

```
dim_date     → used by fact_shipments, fact_maintenance, fact_telemetry
dim_customer → used by fact_shipments, fact_returns, fact_billing
```

**Why it matters:** Enables cross-process analysis ("show me customers whose shipment volume dropped but maintenance costs rose").

### Degenerate Dimensions

Dimension attributes that live in the fact table (no separate dimension table):

```
fact_shipments:
  ...,
  invoice_number,    -- Degenerate dimension (no dim_invoice table needed)
  tracking_number    -- Just a reference, no descriptive attributes
```

Use when the "dimension" has no additional attributes beyond its ID.

### Role-Playing Dimensions

Same physical dimension used multiple times in one fact with different meanings:

```
fact_shipments:
  pickup_date_key    → dim_date (as pickup date)
  delivery_date_key  → dim_date (as delivery date)
  invoice_date_key   → dim_date (as invoice date)
```

Implemented as views or aliases: `dim_pickup_date`, `dim_delivery_date`.

### Junk Dimensions

Combine low-cardinality flags into a single dimension to reduce fact table width:

```
-- Instead of 5 boolean columns in the fact table:
dim_shipment_flags:
  flag_key, is_expedited, is_hazardous, is_refrigerated, is_insured, is_fragile

fact_shipments:
  ..., shipment_flag_key → dim_shipment_flags
```

### Bridge Tables

Handle many-to-many relationships between facts and dimensions:

```
fact_shipments ──→ bridge_shipment_product ──→ dim_product
  shipment_key       shipment_key
                     product_key
                     allocation_factor (weighting)
```

## Kimball's Bus Architecture

A matrix mapping business processes to conformed dimensions:

| Business Process | dim_date | dim_customer | dim_vehicle | dim_location | dim_route |
|-----------------|----------|-------------|-------------|-------------|-----------|
| Shipments | X | X | X | X | X |
| Maintenance | X | | X | X | |
| Telemetry | X | | X | | X |
| Billing | X | X | | | |

**Purpose:** Ensures dimensions are designed once and reused — prevents "stovepipe" marts that can't be joined.

## Design Guidelines

- **Grain first** — always declare the grain before adding dimensions/facts
- **Prefer star over snowflake** — denormalization is intentional for query performance
- **Use surrogate keys** — integer keys for joins (not business keys which can change)
- **Additive facts preferred** — measures that can be summed across all dimensions
- **Avoid nulls in FK columns** — use "Unknown" or "Not Applicable" dimension rows instead
- **Date dimension is mandatory** — never join on raw dates; always use a date dimension with business calendar attributes

---

## Inmon vs Kimball

### Inmon's Corporate Information Factory (CIF)

Bill Inmon's approach takes a **top-down** view: build a single, enterprise-wide data warehouse in third normal form (3NF) first, then derive departmental data marts from it.

```
Source Systems
      │
      ▼
┌─────────────────────────────┐
│  Enterprise Data Warehouse  │  ◄── 3NF, subject-oriented, integrated
│  (Single Source of Truth)   │
└──────────┬──────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
  ┌─────┐ ┌─────┐ ┌─────┐
  │Mart │ │Mart │ │Mart │  ◄── Dimensional (star/snowflake), department-specific
  │Sales│ │ HR  │ │Ops  │
  └─────┘ └─────┘ └─────┘
```

**Key principles of the Inmon approach:**

- **Subject-oriented** — organised around core business subjects (customer, product, transaction), not source systems
- **Integrated** — data from disparate sources is cleansed, conformed, and merged into a single 3NF model
- **Non-volatile** — data is never updated or deleted once loaded; history is preserved through insert-only patterns
- **Time-variant** — every record carries a time dimension, enabling point-in-time analysis
- **Data marts are derived** — star schemas are built as read-optimised projections of the 3NF warehouse, not as independent silos

### Comparison: Inmon vs Kimball vs Data Vault

| Aspect | Inmon (3NF EDW) | Kimball (Dimensional) | Data Vault 2.0 |
|--------|-----------------|----------------------|----------------|
| **Design philosophy** | Top-down, enterprise-first | Bottom-up, process-first | Hybrid, hub-and-spoke |
| **Core model** | 3NF relational | Star schema (facts + dimensions) | Hubs, links, satellites |
| **Build sequence** | EDW first → derive marts | Build marts iteratively → integrate via bus | Build raw vault → derive business vault → marts |
| **Normalisation** | Fully normalised (3NF) | Denormalised (star) | Normalised core, denormalised presentation |
| **Time to first delivery** | Longer (enterprise model required) | Shorter (single business process) | Moderate (vault infrastructure first) |
| **Flexibility to change** | Harder (3NF refactoring) | Moderate (conformed dims help) | Highest (additive by design) |
| **Historical tracking** | Insert-only patterns | SCD types (1-7) | Satellites with load timestamps |
| **Query performance** | Slower (more joins in 3NF) | Fastest (star schema optimised) | Slower in raw vault; fast in marts |
| **Complexity** | High (enterprise modelling) | Moderate (dimensional design) | High (vault patterns, automation) |
| **Best for** | Large enterprises, regulatory environments | BI-focused, analyst-friendly warehouses | Agile environments, multiple sources, auditability |

### When to Choose Each Approach

**Choose Inmon (3NF EDW) when:**
- Regulatory requirements demand a single, auditable enterprise model
- The organisation has a mature data governance function and the patience for upfront design
- Data integration across many disparate sources is the primary challenge

**Choose Kimball (Dimensional) when:**
- Speed to value matters — analysts need queryable data quickly
- The warehouse primarily serves BI and reporting use cases
- The team is smaller and prefers iterative delivery of business-process-aligned marts

**Choose Data Vault when:**
- Sources change frequently and the model must absorb change without refactoring
- Full auditability and historisation of every data point is required
- The team can invest in automation tooling (Data Vault modelling is repetitive but automatable)

**Modern reality:** Many organisations use a hybrid — a Data Vault or 3NF raw/staging layer feeding Kimball-style dimensional marts for consumption. The [[Data Vault 2.0 Methodology]] note covers the vault patterns in detail.

---

## Lakehouse-Era Dimensional Modelling

### How Open Table Formats Change Dimensional Design

Delta Lake, Apache Iceberg, and Apache Hudi introduce capabilities that alter traditional Kimball patterns:

| Traditional Pattern | Lakehouse Equivalent | Impact |
|--------------------|---------------------|--------|
| Stage-and-swap table loads | `MERGE INTO` (upsert) | Eliminates staging tables; atomic upserts directly into fact/dimension tables |
| SCD Type 2 snapshot tables | Time travel (`VERSION AS OF`, `AT TIMESTAMP`) | Point-in-time queries without maintaining explicit SCD rows (for short-term history) |
| Full table rebuilds | Schema evolution (`ALTER TABLE ADD COLUMN`) | Add columns without rewriting data; backward-compatible changes are automatic |
| Partition pruning via date keys | Partition evolution (Iceberg) / Z-ordering (Delta) | Optimised without rigid upfront partitioning decisions |
| Batch-only fact loading | Streaming + batch convergence | Delta/Iceberg support both batch `MERGE` and streaming appends to the same table |

**Important caveat:** Time travel has limited retention (default 7 days in Delta Lake, configurable in Iceberg). SCD Type 2 is still necessary for long-term historical tracking beyond the time travel window. Use time travel for debugging and short-term rollback, not as a replacement for proper dimensional history.

### Medallion Architecture as Dimensional Equivalent

The medallion pattern (bronze/silver/gold) maps naturally to Kimball concepts:

| Medallion Layer | Kimball Equivalent | Purpose |
|----------------|-------------------|---------|
| **Bronze** | Staging / raw | Ingested data, minimal transformation, append-only |
| **Silver** | Conformed dimensions + cleansed facts | Deduplicated, typed, business keys resolved, SCD applied |
| **Gold** | Dimensional marts (star schemas) | Aggregated, business-ready facts and dimensions for consumption |

The key difference: in a lakehouse, all three layers live in the same storage system (e.g., cloud object store with Delta/Iceberg metadata), whereas traditional Kimball often implied separate databases for staging and presentation.

### Materialised Views vs Aggregation Tables

| Approach | Freshness | Storage Cost | Query Performance | Maintenance |
|----------|-----------|-------------|-------------------|-------------|
| **Materialised views** | Auto-refreshed (platform-dependent) | Managed by engine | Fast (pre-computed) | Low (engine manages refresh) |
| **Aggregation tables** (gold layer) | Refreshed by pipeline (scheduled) | Explicit storage cost | Fast (pre-computed) | Higher (pipeline code required) |
| **Live aggregation** (query-time) | Always current | None | Slower (computed on read) | None |

**Guidance:**
- Use **materialised views** for simple, frequently-queried aggregations where the platform supports efficient incremental refresh (Snowflake, Databricks SQL, BigQuery)
- Use **aggregation tables** in the gold layer when aggregation logic is complex, involves multiple joins, or needs to be version-controlled in [[dbt Fundamentals|dbt]]
- Use **live aggregation** only for ad-hoc queries or when data volumes are small enough that query-time computation is acceptable

### MERGE Pattern for Dimension Loading

The `MERGE` statement replaces the traditional stage-load-swap pattern for SCD Type 1 updates:

```sql
MERGE INTO gold.dim_customer AS target
USING silver.customer_cleansed AS source
ON target.customer_business_key = source.customer_id
WHEN MATCHED AND (
    target.email <> source.email
    OR target.address <> source.address
) THEN UPDATE SET
    target.email = source.email,
    target.address = source.address,
    target.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT (
    customer_business_key, email, address, created_at, updated_at
) VALUES (
    source.customer_id, source.email, source.address,
    CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP()
);
```

For SCD Type 2 in a lakehouse context, see [[SCD Type 2 Patterns]] which covers both the dbt snapshot approach and the Delta Lake `MERGE` with row expiry pattern.
