---
tags:
  - sql
  - analytics
  - query-patterns
  - data-engineering
---

# Analytical SQL Patterns

A reference for common analytical SQL patterns used in data warehousing and business intelligence. These patterns apply across Snowflake, BigQuery, Databricks SQL, PostgreSQL, and other ANSI SQL-compliant engines.

---

## Cohort Analysis

Cohort analysis groups users by a shared characteristic (typically first activity date) and tracks behaviour over subsequent periods.

```sql
WITH user_cohort AS (
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY user_id
),
cohort_activity AS (
    SELECT
        uc.cohort_month,
        DATEDIFF('month', uc.cohort_month, DATE_TRUNC('month', o.order_date)) AS period_offset,
        COUNT(DISTINCT o.user_id) AS active_users
    FROM orders o
    JOIN user_cohort uc ON o.user_id = uc.user_id
    GROUP BY uc.cohort_month, period_offset
),
cohort_size AS (
    SELECT cohort_month, COUNT(DISTINCT user_id) AS cohort_users
    FROM user_cohort
    GROUP BY cohort_month
)
SELECT
    ca.cohort_month,
    ca.period_offset,
    ca.active_users,
    cs.cohort_users,
    ROUND(ca.active_users * 100.0 / cs.cohort_users, 2) AS retention_pct
FROM cohort_activity ca
JOIN cohort_size cs ON ca.cohort_month = cs.cohort_month
ORDER BY ca.cohort_month, ca.period_offset;
```

---

## Funnel Analysis

Track conversion through a sequence of steps. Each step filters to users who completed the previous step.

```sql
WITH step_events AS (
    SELECT
        session_id,
        MAX(CASE WHEN event_name = 'page_view'    THEN 1 ELSE 0 END) AS hit_step_1,
        MAX(CASE WHEN event_name = 'add_to_cart'   THEN 1 ELSE 0 END) AS hit_step_2,
        MAX(CASE WHEN event_name = 'begin_checkout' THEN 1 ELSE 0 END) AS hit_step_3,
        MAX(CASE WHEN event_name = 'purchase'      THEN 1 ELSE 0 END) AS hit_step_4
    FROM events
    GROUP BY session_id
)
SELECT
    COUNT(*)                                               AS total_sessions,
    SUM(hit_step_1)                                        AS page_views,
    SUM(CASE WHEN hit_step_1 = 1 AND hit_step_2 = 1 THEN 1 END) AS add_to_cart,
    SUM(CASE WHEN hit_step_1 = 1 AND hit_step_2 = 1
              AND hit_step_3 = 1 THEN 1 END)               AS begin_checkout,
    SUM(CASE WHEN hit_step_1 = 1 AND hit_step_2 = 1
              AND hit_step_3 = 1 AND hit_step_4 = 1 THEN 1 END) AS purchase,
    ROUND(SUM(CASE WHEN hit_step_1 = 1 AND hit_step_2 = 1
              AND hit_step_3 = 1 AND hit_step_4 = 1 THEN 1 END)
          * 100.0 / NULLIF(SUM(hit_step_1), 0), 2)        AS overall_conversion_pct
FROM step_events;
```

---

## Sessionisation

Assign session IDs to clickstream data using an inactivity gap (commonly 30 minutes).

```sql
WITH lagged AS (
    SELECT
        user_id,
        event_timestamp,
        event_name,
        LAG(event_timestamp) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS prev_event_ts
    FROM events
),
flagged AS (
    SELECT
        *,
        CASE
            WHEN prev_event_ts IS NULL
              OR DATEDIFF('minute', prev_event_ts, event_timestamp) > 30
            THEN 1 ELSE 0
        END AS new_session_flag
    FROM lagged
)
SELECT
    user_id,
    event_timestamp,
    event_name,
    SUM(new_session_flag) OVER (
        PARTITION BY user_id ORDER BY event_timestamp
        ROWS UNBOUNDED PRECEDING
    ) AS session_id
FROM flagged;
```

---

## Retention Curves

Calculate N-day retention (e.g., Day 1, Day 7, Day 30) for a product.

```sql
WITH first_seen AS (
    SELECT user_id, MIN(activity_date) AS signup_date
    FROM user_activity
    GROUP BY user_id
)
SELECT
    fs.signup_date,
    COUNT(DISTINCT fs.user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN DATEDIFF('day', fs.signup_date, ua.activity_date) = 1
                        THEN ua.user_id END) AS day_1,
    COUNT(DISTINCT CASE WHEN DATEDIFF('day', fs.signup_date, ua.activity_date) = 7
                        THEN ua.user_id END) AS day_7,
    COUNT(DISTINCT CASE WHEN DATEDIFF('day', fs.signup_date, ua.activity_date) = 30
                        THEN ua.user_id END) AS day_30,
    ROUND(COUNT(DISTINCT CASE WHEN DATEDIFF('day', fs.signup_date, ua.activity_date) = 7
                              THEN ua.user_id END)
          * 100.0 / COUNT(DISTINCT fs.user_id), 2) AS day_7_retention_pct
FROM first_seen fs
LEFT JOIN user_activity ua
    ON fs.user_id = ua.user_id
   AND ua.activity_date > fs.signup_date
GROUP BY fs.signup_date
ORDER BY fs.signup_date;
```

---

## Time-Series Aggregation

Fill gaps in sparse time-series data using a date spine and window functions.

```sql
WITH date_spine AS (
    SELECT date_day
    FROM (
        SELECT DATEADD('day', SEQ4(), '2024-01-01'::DATE) AS date_day
        FROM TABLE(GENERATOR(ROWCOUNT => 365))
    )
),
daily_metrics AS (
    SELECT
        ds.date_day,
        COALESCE(SUM(o.revenue), 0) AS daily_revenue,
        COALESCE(COUNT(o.order_id), 0) AS daily_orders
    FROM date_spine ds
    LEFT JOIN orders o ON ds.date_day = o.order_date::DATE
    GROUP BY ds.date_day
)
SELECT
    date_day,
    daily_revenue,
    daily_orders,
    AVG(daily_revenue) OVER (
        ORDER BY date_day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7d_avg_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY date_day ROWS BETWEEN 27 PRECEDING AND CURRENT ROW
    ) AS rolling_28d_total_revenue
FROM daily_metrics
ORDER BY date_day;
```

---

## Running Totals and Cumulative Sums

```sql
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY order_date
        ROWS UNBOUNDED PRECEDING
    ) AS cumulative_revenue,
    SUM(daily_revenue) OVER (
        PARTITION BY DATE_TRUNC('month', order_date)
        ORDER BY order_date
        ROWS UNBOUNDED PRECEDING
    ) AS mtd_revenue
FROM daily_revenue_summary
ORDER BY order_date;
```

For month-over-month comparison:

```sql
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(revenue) AS monthly_revenue,
    LAG(SUM(revenue)) OVER (ORDER BY DATE_TRUNC('month', order_date)) AS prev_month_revenue,
    ROUND((SUM(revenue) - LAG(SUM(revenue)) OVER (ORDER BY DATE_TRUNC('month', order_date)))
          * 100.0 / NULLIF(LAG(SUM(revenue)) OVER (ORDER BY DATE_TRUNC('month', order_date)), 0),
          2) AS mom_growth_pct
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

---

## Percentile Calculations

```sql
-- Exact percentiles (use for smaller datasets)
SELECT
    product_category,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY order_amount) AS median_amount,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY order_amount) AS p90_amount,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY order_amount) AS p95_amount,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY order_amount) AS p99_amount
FROM orders
GROUP BY product_category;

-- Approximate percentiles (use for large datasets — faster, slightly less accurate)
SELECT
    product_category,
    APPROX_PERCENTILE(order_amount, 0.50) AS approx_median,
    APPROX_PERCENTILE(order_amount, 0.95) AS approx_p95
FROM orders
GROUP BY product_category;

-- NTILE for bucketing into quantile groups
SELECT
    user_id,
    total_spend,
    NTILE(10) OVER (ORDER BY total_spend DESC) AS decile
FROM customer_summary;
```

---

## Pivot and Unpivot

### Pivot (Rows to Columns)

```sql
-- Standard PIVOT syntax (Snowflake, SQL Server, Databricks)
SELECT *
FROM monthly_sales
PIVOT (
    SUM(revenue)
    FOR sale_month IN ('2024-01', '2024-02', '2024-03')
) AS pivoted;

-- CASE-based pivot (works on all engines)
SELECT
    product_category,
    SUM(CASE WHEN sale_month = '2024-01' THEN revenue END) AS jan_2024,
    SUM(CASE WHEN sale_month = '2024-02' THEN revenue END) AS feb_2024,
    SUM(CASE WHEN sale_month = '2024-03' THEN revenue END) AS mar_2024
FROM monthly_sales
GROUP BY product_category;
```

### Unpivot (Columns to Rows)

```sql
-- Standard UNPIVOT syntax
SELECT product_id, month_name, revenue
FROM quarterly_report
UNPIVOT (
    revenue FOR month_name IN (jan_revenue, feb_revenue, mar_revenue)
);

-- UNION ALL-based unpivot (works on all engines)
SELECT product_id, 'January' AS month_name, jan_revenue AS revenue FROM quarterly_report
UNION ALL
SELECT product_id, 'February', feb_revenue FROM quarterly_report
UNION ALL
SELECT product_id, 'March', mar_revenue FROM quarterly_report;
```

---

## GROUPING SETS, CUBE, and ROLLUP

These extensions produce multiple levels of aggregation in a single pass over the data.

### GROUPING SETS

Specify exactly which grouping combinations to compute:

```sql
SELECT
    region,
    product_category,
    SUM(revenue) AS total_revenue,
    COUNT(*) AS order_count,
    GROUPING(region) AS is_region_total,
    GROUPING(product_category) AS is_category_total
FROM orders
GROUP BY GROUPING SETS (
    (region, product_category),   -- region + category breakdown
    (region),                      -- region subtotal
    (product_category),            -- category subtotal
    ()                             -- grand total
)
ORDER BY region NULLS LAST, product_category NULLS LAST;
```

### ROLLUP

Produces a hierarchy of subtotals — useful for drill-down reports:

```sql
-- Generates: (year, quarter, month), (year, quarter), (year), ()
SELECT
    EXTRACT(YEAR FROM order_date) AS yr,
    EXTRACT(QUARTER FROM order_date) AS qtr,
    EXTRACT(MONTH FROM order_date) AS mth,
    SUM(revenue) AS total_revenue
FROM orders
GROUP BY ROLLUP (yr, qtr, mth)
ORDER BY yr NULLS LAST, qtr NULLS LAST, mth NULLS LAST;
```

### CUBE

Produces all possible combinations of the grouped columns:

```sql
-- Generates all 2^3 = 8 grouping combinations
SELECT
    region,
    product_category,
    channel,
    SUM(revenue) AS total_revenue
FROM orders
GROUP BY CUBE (region, product_category, channel)
ORDER BY region NULLS LAST, product_category NULLS LAST, channel NULLS LAST;
```

### When to Use Each

| Pattern | Use Case | Number of Groupings |
|---------|----------|---------------------|
| `GROUPING SETS` | Custom combinations, sparse reports | Exactly what you specify |
| `ROLLUP` | Hierarchical subtotals (year > quarter > month) | n + 1 groupings |
| `CUBE` | Cross-tabulation, all dimension slices | 2^n groupings |

---

## Pattern Selection Guide

| Analysis Need | Pattern | Key Functions |
|---------------|---------|---------------|
| User lifecycle tracking | Cohort analysis | `DATE_TRUNC`, `DATEDIFF`, `COUNT(DISTINCT)` |
| Conversion measurement | Funnel analysis | Conditional `MAX`/`SUM`, `CASE WHEN` |
| Clickstream grouping | Sessionisation | `LAG`, cumulative `SUM` window |
| Product stickiness | Retention curves | `DATEDIFF`, conditional `COUNT(DISTINCT)` |
| Trend analysis | Time-series aggregation | Date spine, rolling `AVG`/`SUM` windows |
| YTD / MTD metrics | Running totals | `SUM OVER (ROWS UNBOUNDED PRECEDING)` |
| Distribution analysis | Percentiles | `PERCENTILE_CONT`, `NTILE`, `APPROX_PERCENTILE` |
| Report reshaping | Pivot / Unpivot | `PIVOT`, `UNPIVOT`, `CASE WHEN` |
| Multi-level reporting | Grouping sets | `GROUPING SETS`, `ROLLUP`, `CUBE` |

---

## See Also

- [[RAG Patterns]] -- retrieval-augmented generation with SQL-based vector search
- [[Dimensional Modelling (Kimball)]] -- star schema design that these patterns query against
- [[dbt Testing & Data Quality]] -- testing analytical query outputs
- [[Snowflake SQL Scripting & Stored Procedures]] -- procedural SQL for complex analytical workflows
