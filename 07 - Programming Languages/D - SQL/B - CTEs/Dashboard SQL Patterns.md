# Dashboard SQL Patterns

SQL patterns for building executive dashboards and BI-ready presentation views — rolling metrics, period-over-period comparisons, trend classification, and alert indicators.

## Layered CTE Architecture

Dashboard views follow a CTE pipeline: raw aggregation → rolling metrics → final presentation:

```sql
WITH daily_kpis AS (
    -- Step 1: Aggregate facts to daily grain per dimension
    SELECT
        fs.shipment_date,
        dc.customer_type,
        dv.vehicle_type,

        COUNT(*) AS daily_deliveries,
        SUM(fs.revenue) AS daily_revenue,
        SUM(fs.delivery_cost + fs.fuel_cost) AS daily_total_cost,
        SUM(fs.revenue - fs.delivery_cost - fs.fuel_cost) AS daily_profit,
        AVG(CASE WHEN fs.is_on_time THEN 1.0 ELSE 0.0 END) AS daily_on_time_rate,
        AVG(fs.route_efficiency_score) AS daily_efficiency

    FROM tbl_fact_shipments fs
    JOIN tbl_dim_customer dc ON fs.customer_id = dc.customer_id AND dc.is_current = true
    JOIN tbl_dim_vehicle dv ON fs.vehicle_id = dv.vehicle_id
    WHERE fs.shipment_date >= CURRENT_DATE() - 90
      AND fs.is_delivered = true
    GROUP BY 1, 2, 3
),

rolling_metrics AS (
    -- Step 2: Add rolling averages and period comparisons
    SELECT *,
        -- Rolling averages
        AVG(daily_revenue) OVER (PARTITION BY customer_type, vehicle_type
            ORDER BY shipment_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS revenue_7d_avg,
        AVG(daily_revenue) OVER (PARTITION BY customer_type, vehicle_type
            ORDER BY shipment_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS revenue_30d_avg,

        -- Week-over-week (same day last week)
        LAG(daily_revenue, 7) OVER (PARTITION BY customer_type, vehicle_type
            ORDER BY shipment_date) AS revenue_prev_week,

        -- Year-over-year (same date last year)
        LAG(daily_revenue, 365) OVER (PARTITION BY customer_type, vehicle_type
            ORDER BY shipment_date) AS revenue_prev_year
    FROM daily_kpis
)

-- Step 3: Final presentation with derived classifications
SELECT
    shipment_date,
    customer_type,
    vehicle_type,
    daily_deliveries,
    ROUND(daily_revenue, 2) AS revenue,
    ROUND(daily_profit, 2) AS profit,
    ROUND(daily_profit / NULLIF(daily_revenue, 0) * 100, 1) AS profit_margin_pct,
    ROUND(daily_on_time_rate * 100, 1) AS on_time_pct,

    -- Rolling context
    ROUND(revenue_7d_avg, 2) AS revenue_7d_avg,
    ROUND(revenue_30d_avg, 2) AS revenue_30d_avg,

    -- Period comparisons
    ROUND((daily_revenue - revenue_prev_week) / NULLIF(revenue_prev_week, 0) * 100, 1)
        AS revenue_wow_change_pct,
    ROUND((daily_revenue - revenue_prev_year) / NULLIF(revenue_prev_year, 0) * 100, 1)
        AS revenue_yoy_change_pct,

    -- Trend classification
    CASE
        WHEN revenue_7d_avg > revenue_30d_avg * 1.05 THEN 'growing'
        WHEN revenue_7d_avg < revenue_30d_avg * 0.95 THEN 'declining'
        ELSE 'stable'
    END AS revenue_trend,

    -- Performance rating
    CASE
        WHEN daily_on_time_rate >= 0.95 THEN 'excellent'
        WHEN daily_on_time_rate >= 0.90 THEN 'good'
        WHEN daily_on_time_rate >= 0.80 THEN 'acceptable'
        ELSE 'needs_improvement'
    END AS performance_rating,

    -- Alert indicators
    CASE
        WHEN daily_on_time_rate < 0.8 THEN 'performance_alert'
        WHEN daily_efficiency < 7 THEN 'efficiency_alert'
        WHEN (daily_revenue - revenue_7d_avg) / NULLIF(revenue_7d_avg, 0) < -0.2
            THEN 'revenue_alert'
        ELSE 'normal'
    END AS alert_status

FROM rolling_metrics
WHERE shipment_date >= CURRENT_DATE() - 90
```

## Rolling Averages (7/30/90-day)

```sql
AVG(metric) OVER (
    PARTITION BY dimension_columns
    ORDER BY date_column
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW     -- 7-day
) AS metric_7d_avg,

AVG(metric) OVER (
    PARTITION BY dimension_columns
    ORDER BY date_column
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW    -- 30-day
) AS metric_30d_avg,

AVG(metric) OVER (
    PARTITION BY dimension_columns
    ORDER BY date_column
    ROWS BETWEEN 89 PRECEDING AND CURRENT ROW    -- 90-day
) AS metric_90d_avg
```

## Week-over-Week and Year-over-Year

```sql
-- Same day of week last week
LAG(daily_deliveries, 7) OVER (PARTITION BY segment ORDER BY date) AS prev_week

-- Same date last year
LAG(daily_revenue, 365) OVER (PARTITION BY segment ORDER BY date) AS prev_year

-- Percentage change
ROUND(
    (daily_revenue - LAG(daily_revenue, 7) OVER (ORDER BY date))
    / NULLIF(LAG(daily_revenue, 7) OVER (ORDER BY date), 0) * 100, 1
) AS wow_pct_change
```

## Trend Classification

Compare short-term (7d) to longer-term (30d) averages:

```sql
CASE
    WHEN avg_7d > avg_30d * 1.05 THEN 'increasing'     -- 7d > 30d by 5%+
    WHEN avg_7d < avg_30d * 0.95 THEN 'decreasing'     -- 7d < 30d by 5%+
    ELSE 'stable'
END AS trend
```

Variations:
- **Volume:** increasing / decreasing / stable
- **Performance:** improving / declining / stable
- **Revenue:** growing / declining / stable

## Composite Performance Scores

Weighted multi-metric scores:

```sql
ROUND(
    (on_time_rate * 0.4) +              -- 40% weight: delivery reliability
    (capacity_utilization * 0.3) +       -- 30% weight: fleet efficiency
    (route_efficiency / 10.0 * 0.3),     -- 30% weight: route optimization
    2
) AS overall_performance_score
```

## Productivity Metrics

Per-unit metrics for fleet analysis:

```sql
ROUND(daily_deliveries / NULLIF(active_vehicles, 0), 1) AS deliveries_per_vehicle,
ROUND(daily_revenue / NULLIF(active_vehicles, 0), 2) AS revenue_per_vehicle,
ROUND(daily_distance / NULLIF(active_vehicles, 0), 1) AS distance_per_vehicle
```

## AI Recommendation Patterns (UNION ALL)

Generate different recommendation types in a single view using UNION ALL:

```sql
-- Route optimization recommendations
SELECT
    'route_optimization' AS recommendation_type,
    route_id AS entity_id,
    'high' AS priority_level,
    ROUND(avg_duration_ratio * 100 - 100, 1) || '% over planned' AS issue,
    'Optimize route planning' AS recommendation,
    on_time_rate AS confidence_score
FROM route_analysis
WHERE avg_duration_ratio > 1.2 AND total_trips >= 5

UNION ALL

-- Vehicle assignment recommendations
SELECT
    'vehicle_assignment' AS recommendation_type,
    vehicle_id AS entity_id,
    'medium' AS priority_level,
    'Underutilized at ' || ROUND(capacity_util * 100, 1) || '%' AS issue,
    'Consider route consolidation' AS recommendation,
    1 - capacity_util AS confidence_score
FROM vehicle_analysis
WHERE capacity_util < 0.6 AND trips >= 10

UNION ALL

-- Maintenance schedule recommendations
SELECT
    'maintenance_schedule' AS recommendation_type,
    vehicle_id AS entity_id,
    CASE WHEN service_status = 'overdue' THEN 'urgent' ELSE 'high' END,
    'Service ' || service_status || ' — ' || days_since_service || ' days',
    'Schedule maintenance',
    CASE WHEN service_status = 'overdue' THEN 0.9 ELSE 0.8 END
FROM vehicle_analysis
WHERE service_status IN ('overdue', 'due_soon')
```

## Sustainability / ESG Metrics

Carbon emissions calculation by fuel type:

```sql
SUM(distance_km) * AVG(fuel_efficiency_l_per_100km) / 100 *
CASE
    WHEN vehicle_type = 'TRUCK' THEN 2.68       -- Diesel: 2.68 kg CO2/L
    WHEN vehicle_type = 'VAN' THEN 2.31          -- Petrol: 2.31 kg CO2/L
    WHEN vehicle_type = 'MOTORCYCLE' THEN 2.31
    ELSE 2.5
END AS co2_emissions_kg
```

YoY carbon tracking:

```sql
LAG(co2_emissions_kg, 365) OVER (PARTITION BY city ORDER BY date) AS co2_prev_year,

CASE
    WHEN co2_yoy_change <= -20 THEN 'exceeding_target'
    WHEN co2_yoy_change <= -10 THEN 'on_target'
    WHEN co2_yoy_change <= 0 THEN 'below_target'
    ELSE 'increasing_emissions'
END AS carbon_target_status
```

## Materialization

Dashboard views should be `materialized='view'` — no storage cost, always fresh:

```sql
{{ config(
    materialized='view',
    tags=['consumption', 'dashboard', 'business_intelligence']
) }}
```

Query via a dedicated consumption warehouse (`OUT_` prefix) to isolate BI query costs from ETL.
