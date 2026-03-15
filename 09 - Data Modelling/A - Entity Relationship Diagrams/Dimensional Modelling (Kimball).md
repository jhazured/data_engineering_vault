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
