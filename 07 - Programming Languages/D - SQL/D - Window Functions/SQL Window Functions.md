# SQL Window Functions

## Fundamentals

Window functions perform calculations across a set of rows related to the current row, without collapsing rows like GROUP BY.

```sql
function_name() OVER (
  [PARTITION BY column]
  [ORDER BY column]
  [frame_clause]
)
```

## Ranking Functions

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT
  warehouse_name,
  daily_credits,
  ROW_NUMBER() OVER (ORDER BY daily_credits DESC) AS row_num,      -- 1, 2, 3, 4
  RANK()       OVER (ORDER BY daily_credits DESC) AS rank_val,      -- 1, 2, 2, 4 (gaps)
  DENSE_RANK() OVER (ORDER BY daily_credits DESC) AS dense_rank_val -- 1, 2, 2, 3 (no gaps)
FROM warehouse_usage
```

### NTILE — Distribute into Buckets

```sql
SELECT
  customer_id,
  lifetime_value,
  NTILE(4) OVER (ORDER BY lifetime_value DESC) AS value_quartile
FROM customers
```

## Analytical Functions

### LAG / LEAD — Access Adjacent Rows

```sql
SELECT
  usage_date,
  daily_credits,
  LAG(daily_credits, 1) OVER (ORDER BY usage_date) AS prev_day_credits,
  LEAD(daily_credits, 1) OVER (ORDER BY usage_date) AS next_day_credits,
  daily_credits - LAG(daily_credits, 1) OVER (ORDER BY usage_date) AS day_over_day_change
FROM daily_usage
```

### FIRST_VALUE / LAST_VALUE

```sql
SELECT
  shipment_date,
  revenue,
  FIRST_VALUE(revenue) OVER (
    PARTITION BY customer_id ORDER BY shipment_date
  ) AS first_order_revenue
FROM shipments
```

## Aggregate Window Functions

Standard aggregates (SUM, AVG, COUNT, MIN, MAX) work as window functions:

```sql
SELECT
  shipment_date,
  revenue,
  SUM(revenue) OVER (PARTITION BY customer_id) AS customer_total,
  AVG(revenue) OVER (PARTITION BY customer_id) AS customer_avg,
  COUNT(*) OVER (PARTITION BY customer_id) AS customer_order_count
FROM shipments
```

## Rolling Averages & Running Totals

### Rolling 7/30/90-Day Averages

```sql
SELECT
  date_key,
  customer_id,
  revenue,
  AVG(revenue) OVER (
    PARTITION BY customer_id
    ORDER BY date_key
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS revenue_7d_avg,
  AVG(revenue) OVER (
    PARTITION BY customer_id
    ORDER BY date_key
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
  ) AS revenue_30d_avg,
  AVG(revenue) OVER (
    PARTITION BY customer_id
    ORDER BY date_key
    ROWS BETWEEN 89 PRECEDING AND CURRENT ROW
  ) AS revenue_90d_avg
FROM daily_metrics
```

### Running Total

```sql
SELECT
  usage_date,
  daily_credits,
  SUM(daily_credits) OVER (
    ORDER BY usage_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_credits
FROM daily_usage
```

## Frame Specifications

| Frame | Meaning |
|-------|---------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | All rows from start to current |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | Current + 6 previous rows (7-day window) |
| `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | Previous, current, and next row |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | Current to end |
| `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` | Date-based range (handles gaps) |

**ROWS vs RANGE:**
- `ROWS`: Physical row count — predictable but ignores date gaps
- `RANGE`: Logical value range — handles missing dates but requires compatible ORDER BY type

## PERCENTILE_CONT — Statistical Analysis

```sql
SELECT
  customer_id,
  revenue,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY revenue) OVER (
    PARTITION BY customer_id
  ) AS median_revenue,
  PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY revenue) OVER (
    PARTITION BY customer_id
  ) AS p90_revenue
FROM shipments
```

## Coefficient of Variation (Volatility)

```sql
SELECT
  date_key,
  revenue,
  STDDEV(revenue) OVER (
    PARTITION BY customer_id
    ORDER BY date_key
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
  ) / NULLIF(AVG(revenue) OVER (
    PARTITION BY customer_id
    ORDER BY date_key
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
  ), 0) AS revenue_volatility_30d
FROM daily_metrics
```

## Week-over-Week & Year-over-Year Comparisons

```sql
SELECT
  report_date,
  daily_revenue,
  -- WoW comparison
  LAG(daily_revenue, 7) OVER (ORDER BY report_date) AS revenue_prev_week,
  (daily_revenue / NULLIF(LAG(daily_revenue, 7) OVER (ORDER BY report_date), 0) - 1) * 100
    AS wow_pct_change,
  -- YoY comparison
  LAG(daily_revenue, 365) OVER (ORDER BY report_date) AS revenue_prev_year,
  (daily_revenue / NULLIF(LAG(daily_revenue, 365) OVER (ORDER BY report_date), 0) - 1) * 100
    AS yoy_pct_change
FROM daily_revenue_summary
```

## Trend Classification

Combine rolling averages with CASE logic for automated trend detection:

```sql
SELECT
  date_key,
  metric_value,
  AVG(metric_value) OVER (ORDER BY date_key ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_7d,
  AVG(metric_value) OVER (ORDER BY date_key ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS avg_30d,
  CASE
    WHEN avg_7d > avg_30d * 1.05 THEN 'increasing'
    WHEN avg_7d < avg_30d * 0.95 THEN 'decreasing'
    ELSE 'stable'
  END AS trend
FROM metrics
```

## Common Patterns

### Deduplication (Keep Latest)

```sql
SELECT * FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) AS rn
  FROM raw_customers
) WHERE rn = 1
```

### Gap Detection

```sql
SELECT
  event_date,
  LAG(event_date) OVER (PARTITION BY customer_id ORDER BY event_date) AS prev_date,
  DATEDIFF(day, prev_date, event_date) AS gap_days
FROM events
WHERE gap_days > 30  -- Flag gaps > 30 days
```

### Cumulative Distribution

```sql
SELECT
  customer_id,
  revenue,
  CUME_DIST() OVER (ORDER BY revenue) AS cumulative_pct,
  PERCENT_RANK() OVER (ORDER BY revenue) AS percent_rank
FROM customer_revenue
```
