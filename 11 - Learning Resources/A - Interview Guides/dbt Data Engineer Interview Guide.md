# DBT Knowledge Base

## Table of Contents

1. [[Core dbt Fundamentals]]
2. [[Model Development & Best Practices]]
3. [[Testing & Data Quality]]
4. [[Macros & Advanced SQL]]
5. [[Performance Optimization]]
6. [[Data Sources & Integration]]
7. [[Incremental Models & Strategies]]
8. [[Documentation & Collaboration]]
9. [[Debugging & Troubleshooting]]
10. [[CI/CD & Deployment]]
11. [[Cloud Platform Specifics]]
12. [[Real-World Scenarios]]

---

## Core dbt Fundamentals

### What is dbt and how does it fit into the modern data stack?

**Answer:** dbt (Data Build Tool) is an open-source transformation tool that enables analytics engineers to transform data in their warehouse using SQL and software engineering best practices.

**Key Components:**

- **Extract:** Tools like [[Fivetran]], [[Stitch]], [[Airbyte]]
- **Load:** Cloud warehouses ([[Snowflake]], [[BigQuery]], [[Redshift]])
- **Transform:** dbt handles the "T" in ELT
- **Visualize:** BI tools like [[Looker]], [[Tableau]], [[Power BI]]

**dbt's Core Value Propositions:**

- Version control for analytics code
- Built-in [[Testing Framework]]
- Automatic [[Documentation Generation]]
- [[Dependency Management]]
- Modular, reusable code

**Modern Data Stack Benefits:**

- Leverages warehouse compute power
- Treats analytics code like software
- Enables collaboration between data teams
- Provides [[Data Lineage]] and documentation

### dbt Project Structure and Directory Purpose

```
my_dbt_project/
├── analysis/          # Ad-hoc analytical queries (not materialized)
├── data/             # CSV seed files for static reference data
├── macros/           # Reusable Jinja SQL functions
├── models/           # SQL transformation files
│   ├── staging/      # 1:1 with source tables, basic cleaning
│   ├── intermediate/ # Complex business logic, reusable components
│   └── marts/        # Final business-ready models
├── snapshots/        # Type-2 slowly changing dimensions
├── tests/            # Custom singular tests
├── dbt_project.yml   # Project configuration
└── profiles.yml      # Database connection settings
```

**Directory Purposes:**

**[[Staging Models]]**: Source system cleanup

```sql
-- models/staging/stg_customers.sql
SELECT 
    customer_id,
    TRIM(UPPER(first_name)) as first_name,
    TRIM(UPPER(last_name)) as last_name,
    LOWER(email) as email,
    created_at
FROM {{ source('crm', 'customers') }}
WHERE customer_id IS NOT NULL
```

**[[Intermediate Models]]**: Reusable business logic

```sql
-- models/intermediate/int_customer_metrics.sql
SELECT 
    customer_id,
    COUNT(order_id) as total_orders,
    SUM(order_amount) as lifetime_value,
    MAX(order_date) as last_order_date
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

**[[Mart Models]]**: Business-facing final models

```sql
-- models/marts/dim_customers.sql
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    m.total_orders,
    m.lifetime_value
FROM {{ ref('stg_customers') }} c
LEFT JOIN {{ ref('int_customer_metrics') }} m
    ON c.customer_id = m.customer_id
```

### Materialization Types in dbt

**1. [[View Materialization]]** (Default for staging)

```sql
{{ config(materialized='view') }}

SELECT * FROM {{ source('raw', 'customers') }}
```

- **Use for:** [[Staging Models]], simple transformations
- **Pros:** Always fresh, no storage cost
- **Cons:** Query performance impact

**2. [[Table Materialization]]** (Common for marts)

```sql
{{ config(materialized='table') }}

SELECT 
    customer_id,
    SUM(order_amount) as total_revenue
FROM {{ ref('fct_orders') }}
GROUP BY customer_id
```

- **Use for:** Final business models, frequently queried data
- **Pros:** Fast query performance
- **Cons:** Storage cost, potential staleness

**3. [[Incremental Materialization]]** (For large datasets)

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

SELECT * FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

- **Use for:** Large datasets that grow over time
- **Pros:** Efficient processing, reduced cost
- **Cons:** More complex logic

**4. [[Ephemeral Materialization]]** (For shared logic)

```sql
{{ config(materialized='ephemeral') }}

SELECT 
    customer_id,
    CASE 
        WHEN total_orders > 10 THEN 'High Value'
        WHEN total_orders > 5 THEN 'Medium Value'
        ELSE 'Low Value'
    END as customer_segment
FROM {{ ref('int_customer_metrics') }}
```

- **Use for:** Shared calculations that don't need persistence
- **Pros:** No storage overhead
- **Cons:** Recalculated for each dependent model

### dbt Dependencies and the ref() Function

**[[Dependency Management]]**: dbt automatically builds a Directed Acyclic Graph ([[DAG]]) based on `ref()` and `source()` functions.

```sql
-- models/marts/customer_summary.sql
SELECT 
    c.customer_id,
    c.email,
    o.total_orders,
    p.total_payments
FROM {{ ref('stg_customers') }} c  -- Dependency on staging model
LEFT JOIN {{ ref('order_summary') }} o  -- Dependency on intermediate model
    ON c.customer_id = o.customer_id
LEFT JOIN {{ ref('payment_summary') }} p  -- Another dependency
    ON c.customer_id = p.customer_id
```

**Key Benefits:**

- **Automatic execution order:** dbt runs models in correct dependency sequence
- **[[Impact Analysis]]:** See downstream effects of changes
- **[[Documentation]]:** Automatic lineage graphs
- **Development safety:** Prevents circular dependencies

**[[ref() vs source()]]:**

```sql
-- Reference another dbt model
FROM {{ ref('model_name') }}

-- Reference raw source data
FROM {{ source('database_name', 'table_name') }}
```

---

## Model Development & Best Practices

### SQL Structure in dbt Models

**[[dbt SQL Style Guide]]:**

**1. Model Structure:**

```sql
{{ config(
    materialized='table',
    indexes=[{'columns': ['customer_id'], 'type': 'hash'}]
) }}

-- Import CTEs (import data from other models)
WITH customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

-- Logical CTEs (business logic transformations)
customer_orders AS (
    SELECT 
        customer_id,
        COUNT(order_id) as total_orders,
        SUM(order_amount) as total_revenue,
        MAX(order_date) as last_order_date
    FROM orders
    GROUP BY customer_id
),

-- Final CTE (prepare for output)
final AS (
    SELECT 
        c.customer_id,
        c.first_name,
        c.last_name,
        c.email,
        c.created_date,
        COALESCE(co.total_orders, 0) as total_orders,
        COALESCE(co.total_revenue, 0) as total_revenue,
        co.last_order_date
    FROM customers c
    LEFT JOIN customer_orders co
        ON c.customer_id = co.customer_id
)

-- Always end with simple select
SELECT * FROM final
```

**2. [[Naming Conventions]]:**

- **Staging models:** `stg_{source}_{table}`
    - `stg_salesforce_accounts.sql`
    - `stg_stripe_payments.sql`
- **Intermediate models:** `int_{description}`
    - `int_customer_order_history.sql`
    - `int_revenue_by_month.sql`
- **Fact tables:** `fct_{business_process}`
    - `fct_orders.sql`
    - `fct_customer_sessions.sql`
- **Dimension tables:** `dim_{entity}`
    - `dim_customers.sql`
    - `dim_products.sql`

**3. [[Column Ordering]]:**

```sql
SELECT 
    -- Primary keys first
    customer_id,
    -- Foreign keys second
    account_id,
    region_id,
    -- Descriptive fields
    first_name,
    last_name,
    email,
    phone,
    -- Metrics
    total_orders,
    lifetime_value,
    -- Booleans
    is_active,
    is_verified,
    -- Dates (chronological order)
    created_date,
    updated_date,
    last_login_date
FROM customer_base
```

### Data Type Conversions and Cleaning

**[[Data Cleaning Patterns]]:**

**1. [[String Cleaning]]:**

```sql
-- models/staging/stg_customers.sql
SELECT 
    customer_id,
    -- Standardize text fields
    TRIM(UPPER(first_name)) as first_name,
    TRIM(UPPER(last_name)) as last_name,
    TRIM(LOWER(email)) as email,
    -- Remove extra whitespace
    REGEXP_REPLACE(phone, '[^0-9]', '') as phone_clean,
    -- Handle nulls and empty strings
    NULLIF(TRIM(company_name), '') as company_name,
    created_at
FROM {{ source('crm', 'customers') }}
```

**2. [[Date Time Conversions]]:**

```sql
SELECT 
    order_id,
    -- Standardize date formats
    DATE(created_at) as order_date,
    -- Handle timezone conversions
    CONVERT_TIMEZONE('UTC', 'America/New_York', created_at) as created_at_est,
    -- Extract date parts
    EXTRACT(YEAR FROM created_at) as order_year,
    EXTRACT(MONTH FROM created_at) as order_month,
    EXTRACT(DOW FROM created_at) as day_of_week,
    -- Create fiscal periods
    CASE 
        WHEN EXTRACT(MONTH FROM created_at) >= 4 THEN EXTRACT(YEAR FROM created_at)
        ELSE EXTRACT(YEAR FROM created_at) - 1
    END as fiscal_year
FROM {{ source('orders', 'raw_orders') }}
```

**3. [[Numeric Data Cleaning]]:**

```sql
SELECT 
    order_id,
    -- Handle division by zero
    CASE 
        WHEN quantity = 0 THEN NULL
        ELSE revenue / quantity 
    END as unit_price,
    -- Round monetary values
    ROUND(revenue, 2) as revenue,
    -- Convert data types safely
    TRY_CAST(discount_percentage AS DECIMAL(5,2)) as discount_rate,
    -- Handle outliers
    CASE 
        WHEN revenue > 1000000 THEN NULL  -- Cap extreme values
        WHEN revenue < 0 THEN 0           -- Handle negative values
        ELSE revenue 
    END as revenue_clean
FROM {{ source('orders', 'raw_orders') }}
```

**4. [[Boolean Status Standardization]]:**

```sql
SELECT 
    customer_id,
    -- Standardize boolean flags
    CASE 
        WHEN UPPER(is_active) IN ('TRUE', 'YES', '1', 'Y') THEN TRUE
        WHEN UPPER(is_active) IN ('FALSE', 'NO', '0', 'N') THEN FALSE
        ELSE NULL
    END as is_active,
    -- Standardize status values
    CASE 
        WHEN UPPER(status) IN ('ACTIVE', 'CURRENT', 'LIVE') THEN 'active'
        WHEN UPPER(status) IN ('INACTIVE', 'CLOSED', 'TERMINATED') THEN 'inactive'
        WHEN UPPER(status) IN ('PENDING', 'PROCESSING') THEN 'pending'
        ELSE 'unknown'
    END as status_standardized
FROM {{ source('crm', 'customers') }}
```

**5. [[Reusable Cleaning Macros]]:**

```sql
-- macros/clean_phone.sql
{% macro clean_phone(phone_column) %}
    REGEXP_REPLACE(
        REGEXP_REPLACE({{ phone_column }}, '[^0-9]', ''),
        '^1', ''
    )
{% endmacro %}

-- Usage in model
SELECT 
    customer_id,
    {{ clean_phone('phone') }} as phone_clean
FROM {{ ref('stg_customers') }}
```

### Implementing Business Logic and Calculations

**1. [[Customer Segmentation Logic]]:**

```sql
-- models/intermediate/int_customer_segments.sql
WITH customer_metrics AS (
    SELECT 
        customer_id,
        SUM(order_amount) as total_revenue,
        COUNT(order_id) as total_orders,
        AVG(order_amount) as avg_order_value,
        MAX(order_date) as last_order_date,
        MIN(order_date) as first_order_date
    FROM {{ ref('fct_orders') }}
    GROUP BY customer_id
),

segment_logic AS (
    SELECT 
        *,
        -- RFM Analysis components
        CASE 
            WHEN last_order_date >= CURRENT_DATE() - 30 THEN 'Recent'
            WHEN last_order_date >= CURRENT_DATE() - 90 THEN 'Moderate'
            ELSE 'Distant'
        END as recency_segment,
        CASE 
            WHEN total_orders >= 10 THEN 'High'
            WHEN total_orders >= 5 THEN 'Medium'
            ELSE 'Low'
        END as frequency_segment,
        CASE 
            WHEN total_revenue >= 1000 THEN 'High'
            WHEN total_revenue >= 500 THEN 'Medium'
            ELSE 'Low'
        END as monetary_segment,
        -- Customer lifecycle stage
        CASE 
            WHEN DATEDIFF('day', first_order_date, CURRENT_DATE()) <= 30 THEN 'New'
            WHEN last_order_date < CURRENT_DATE() - 365 THEN 'Churned'
            WHEN total_orders = 1 THEN 'One-time'
            ELSE 'Returning'
        END as lifecycle_stage
    FROM customer_metrics
)

SELECT 
    *,
    -- Combined segment
    CONCAT(recency_segment, '-', frequency_segment, '-', monetary_segment) as rfm_segment,
    -- Business value score (0-100)
    LEAST(100, 
        (total_revenue / 10) + 
        (total_orders * 5) + 
        (CASE WHEN recency_segment = 'Recent' THEN 20 ELSE 0 END)
    ) as customer_score
FROM segment_logic
```

**2. [[Financial Calculations]]:**

```sql
-- models/marts/fct_monthly_revenue.sql
WITH daily_revenue AS (
    SELECT 
        DATE_TRUNC('day', order_date) as date,
        SUM(order_amount) as daily_revenue,
        COUNT(DISTINCT customer_id) as daily_customers,
        COUNT(order_id) as daily_orders
    FROM {{ ref('fct_orders') }}
    GROUP BY DATE_TRUNC('day', order_date)
),

monthly_aggregations AS (
    SELECT 
        DATE_TRUNC('month', date) as month,
        SUM(daily_revenue) as monthly_revenue,
        AVG(daily_revenue) as avg_daily_revenue,
        -- Growth calculations
        LAG(SUM(daily_revenue)) OVER (ORDER BY DATE_TRUNC('month', date)) as prev_month_revenue,
        -- Running totals
        SUM(SUM(daily_revenue)) OVER (
            ORDER BY DATE_TRUNC('month', date) 
            ROWS UNBOUNDED PRECEDING
        ) as cumulative_revenue
    FROM daily_revenue
    GROUP BY DATE_TRUNC('month', date)
)

SELECT 
    month,
    monthly_revenue,
    avg_daily_revenue,
    prev_month_revenue,
    cumulative_revenue,
    -- Month-over-month growth
    CASE 
        WHEN prev_month_revenue > 0 
        THEN (monthly_revenue - prev_month_revenue) / prev_month_revenue * 100
        ELSE NULL 
    END as mom_growth_percent,
    -- Revenue variance from average
    monthly_revenue - AVG(monthly_revenue) OVER () as revenue_variance
FROM monthly_aggregations
```

---

## Testing & Data Quality

### Comprehensive Testing in dbt

**1. [[Schema Tests]] (Generic Tests):**

```yaml
# models/schema.yml
version: 2

models:
  - name: dim_customers
    description: "Customer dimension table"
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - customer_id
            - effective_date
    columns:
      - name: customer_id
        description: "Unique customer identifier"
        tests:
          - not_null
          - unique
      - name: email
        description: "Customer email address"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
      - name: lifetime_value
        description: "Total customer value"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 1000000
      - name: status
        description: "Customer status"
        tests:
          - accepted_values:
              values: ['active', 'inactive', 'pending']

sources:
  - name: raw_crm
    tables:
      - name: customers
        tests:
          - dbt_utils.recency:
              datepart: hour
              field: updated_at
              interval: 24
        columns:
          - name: customer_id
            tests:
              - not_null
              - unique
```

**2. [[Singular Tests]] (Custom SQL Tests):**

```sql
-- tests/test_revenue_consistency.sql
-- Test that aggregated revenue matches source totals
WITH source_revenue AS (
    SELECT SUM(amount) as total
    FROM {{ source('payments', 'transactions') }}
    WHERE status = 'completed'
),

model_revenue AS (
    SELECT SUM(revenue) as total  
    FROM {{ ref('fct_revenue') }}
)

SELECT 
    s.total as source_total,
    m.total as model_total,
    ABS(s.total - m.total) as difference
FROM source_revenue s
CROSS JOIN model_revenue m
WHERE ABS(s.total - m.total) > 100  -- Fail if difference > $100
```

**3. [[Custom Generic Tests]]:**

```sql
-- macros/test_valid_date_range.sql
{% test valid_date_range(model, column_name, start_date, end_date) %}

    SELECT {{ column_name }}
    FROM {{ model }}
    WHERE {{ column_name }} < '{{ start_date }}'
       OR {{ column_name }} > '{{ end_date }}'

{% endtest %}

-- macros/test_referential_integrity.sql
{% test referential_integrity(model, column_name, to, field) %}

    SELECT {{ column_name }}
    FROM {{ model }}
    WHERE {{ column_name }} IS NOT NULL
      AND {{ column_name }} NOT IN (
          SELECT {{ field }}
          FROM {{ to }}
          WHERE {{ field }} IS NOT NULL
      )

{% endtest %}
```

**4. [[Data Quality Monitoring]]:**

```sql
-- models/monitoring/data_quality_dashboard.sql
WITH test_results AS (
    SELECT 
        'dim_customers' as table_name,
        'customer_id_unique' as test_name,
        CASE 
            WHEN COUNT(customer_id) = COUNT(DISTINCT customer_id) 
            THEN 'PASS' 
            ELSE 'FAIL' 
        END as test_status,
        COUNT(customer_id) - COUNT(DISTINCT customer_id) as failure_count
    FROM {{ ref('dim_customers') }}
    UNION ALL
    SELECT 
        'fct_orders' as table_name,
        'revenue_positive' as test_name,
        CASE 
            WHEN COUNT(*) FILTER (WHERE revenue < 0) = 0 
            THEN 'PASS' 
            ELSE 'FAIL' 
        END as test_status,
        COUNT(*) FILTER (WHERE revenue < 0) as failure_count
    FROM {{ ref('fct_orders') }}
),

quality_summary AS (
    SELECT 
        table_name,
        COUNT(*) as total_tests,
        COUNT(*) FILTER (WHERE test_status = 'PASS') as passed_tests,
        COUNT(*) FILTER (WHERE test_status = 'FAIL') as failed_tests,
        ROUND(
            COUNT(*) FILTER (WHERE test_status = 'PASS') * 100.0 / COUNT(*), 2
        ) as pass_rate
    FROM test_results
    GROUP BY table_name
)

SELECT 
    table_name,
    total_tests,
    passed_tests,
    failed_tests,
    pass_rate,
    CASE 
        WHEN pass_rate = 100 THEN '✅ Excellent'
        WHEN pass_rate >= 90 THEN '⚠️ Good'
        WHEN pass_rate >= 75 THEN '🔥 Needs Attention'
        ELSE '🚨 Critical'
    END as quality_status
FROM quality_summary
ORDER BY pass_rate DESC
```

---

## Related Topics

- [[Pyspark Functions and Methods]]
- [[SQL]]
- [[Snowflake]]
- [[Azure]]
- [[Data Engineering]]
- [[Data Warehouse Design]]
- [[Analytics Engineering]]

## Tags

#dbt #data-engineering #sql #data-transformation #analytics #testing #documentation

---

## Real-World Supplement: Production Patterns

### Incremental Watermark Pattern

**Q: How do you implement incremental loading in a production dbt project?**

Use a `_loaded_at` / `_ingested_at` watermark chain:

```sql
{{ config(materialized='incremental', unique_key='shipment_id', incremental_strategy='merge') }}

WITH src AS (
  SELECT * FROM {{ source('RAW', 'SHIPMENTS') }}
  {% if is_incremental() %}
    WHERE _loaded_at > (SELECT COALESCE(MAX(_ingested_at), '1900-01-01') FROM {{ this }})
  {% endif %}
)
SELECT *, CURRENT_TIMESTAMP() AS _ingested_at FROM src
```

The source writes `_loaded_at`; the staging model adds `_ingested_at`. Downstream models chain: `WHERE _ingested_at > (SELECT MAX(_ingested_at) FROM {{ this }})`.

### Snapshot Strategy Trade-offs

**Q: When do you use timestamp vs check strategy for dbt snapshots?**

- **Timestamp:** Use when source has a reliable `updated_at`. Simple, fast (only compares one column).
- **Check:** Use when no reliable timestamp exists (e.g., `updated_at` is derived from `created_at`). Slower — compares all specified columns every run.

```sql
-- Check strategy example (traffic data with no real updated_at)
{{ config(strategy='check', check_cols=['traffic_level', 'congestion_delay_minutes']) }}
```

### Tag-Based Execution Ordering

**Q: How do you ensure models run in the right order in a multi-tier architecture?**

Assign load priority tags and run sequentially:

```bash
dbt snapshot                           # SCD2 state capture
dbt run --select tag:load_priority_1   # T2 staging
dbt run --select tag:load_priority_2a  # T3 dimensions
dbt run --select tag:load_priority_2b  # T3 facts (depend on dims)
dbt run --select tag:load_priority_3   # T4 presentation
```

### generate_schema_name for Multi-Database Routing

**Q: How do you route dbt models to different Snowflake databases?**

Override `generate_schema_name` to use only the custom schema (not prepend `target.schema`), then set `+database` per folder in `dbt_project.yml`:

```yaml
staging:
  +database: "{{ env_var('SF_ENV', 'DEV') | upper }}_T2_PERSISTENT_STAGING"
  +schema: "LOGISTICS"
```

### Pipeline Run Logging via on-run-end

**Q: How do you monitor dbt pipeline execution?**

Use an `on-run-end` hook that iterates `results` and inserts one row per model into an audit table (T0_CONTROL.JOB_CONTROL), capturing status, execution time, layer (derived from tags), and row counts from adapter response.