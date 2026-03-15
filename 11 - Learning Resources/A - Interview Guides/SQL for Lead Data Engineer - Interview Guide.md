**

## Table of Contents

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#11-execution-plans-and-performance-tuning)
    
- [1.2 Index Design and Strategy](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#12-index-design-and-strategy)
    
- [1.3 Query Rewriting Techniques](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#13-query-rewriting-techniques)
    

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#21-window-functions-and-analytics)
    
- [2.2 CTEs and Recursive Queries](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#22-ctes-and-recursive-queries)
    
- [2.3 Advanced Joins and Subqueries](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#23-advanced-joins-and-subqueries)
    

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#31-dimensional-modeling)
    
- [3.2 Slowly Changing Dimensions](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#32-slowly-changing-dimensions)
    
- [3.3 Schema Evolution Strategies](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#33-schema-evolution-strategies)
    

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#41-incremental-loading-strategies)
    
- [4.2 Change Data Capture (CDC)](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#42-change-data-capture-cdc)
    
- [4.3 Data Quality and Validation](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#43-data-quality-and-validation)
    

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#51-cloud-data-warehouses)
    
- [5.2 Columnar vs Row-Based Storage](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#52-columnar-vs-row-based-storage)
    

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#61-performance-monitoring-and-troubleshooting)
    
- [6.2 SQL Code Standards and Reviews](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#62-sql-code-standards-and-reviews)
    
- [6.3 Team Mentoring and Best Practices](https://claude.ai/chat/800ec0e5-ec46-4717-8e7e-db3040243168#63-team-mentoring-and-best-practices)
    

---

## 1. Advanced Query Optimization

### 1.1 Execution Plans and Performance Tuning

Q: How do you analyze and optimize query execution plans? What are the key indicators of performance issues?

Answer:

Reading Execution Plans:

-- PostgreSQL

EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 

SELECT c.customer_name, SUM(o.order_amount)

FROM customers c

JOIN orders o ON c.customer_id = o.customer_id

WHERE o.order_date >= '2024-01-01'

GROUP BY c.customer_name;

  

-- SQL Server

SET STATISTICS IO ON;

SET STATISTICS TIME ON;

SELECT c.customer_name, SUM(o.order_amount)

FROM customers c

JOIN orders o ON c.customer_id = o.customer_id

WHERE o.order_date >= '2024-01-01'

GROUP BY c.customer_name;

  

-- Oracle

SELECT /*+ GATHER_PLAN_STATISTICS */ 

       c.customer_name, SUM(o.order_amount)

FROM customers c

JOIN orders o ON c.customer_id = o.customer_id

WHERE o.order_date >= '2024-01-01'

GROUP BY c.customer_name;

  

Key Performance Indicators:

1. High Cost Operations: Look for operations consuming >20% of total cost
    
2. Table Scans: Full table scans on large tables indicate missing indexes
    
3. Sort Operations: Large sorts suggest need for better indexing or query rewriting
    
4. Hash vs Nested Loop Joins: Hash joins better for large datasets, nested loops for small
    
5. Cardinality Estimates: Significant differences between estimated and actual rows
    

Common Optimization Techniques:

-- Before: Inefficient correlated subquery

SELECT customer_id, customer_name

FROM customers c

WHERE EXISTS (

    SELECT 1 FROM orders o 

    WHERE o.customer_id = c.customer_id 

    AND o.order_date >= '2024-01-01'

);

  

-- After: Optimized with JOIN

SELECT DISTINCT c.customer_id, c.customer_name

FROM customers c

INNER JOIN orders o ON c.customer_id = o.customer_id

WHERE o.order_date >= '2024-01-01';

  

-- Or even better with EXISTS if you just need existence check

SELECT c.customer_id, c.customer_name

FROM customers c

WHERE c.customer_id IN (

    SELECT DISTINCT o.customer_id 

    FROM orders o 

    WHERE o.order_date >= '2024-01-01'

);

  

### 1.2 Index Design and Strategy

Q: How do you design an effective indexing strategy? What are the trade-offs between different index types?

Answer:

Index Types and Use Cases:

1. Clustered Index:
    

-- Choose clustering key carefully - affects all queries

CREATE CLUSTERED INDEX IX_Orders_Date 

ON orders (order_date, customer_id);

-- Good: Range queries on date, point lookups with customer_id

-- Bad: Frequent INSERTs can cause page splits

  

2. Composite Indexes:
    

-- Order matters: most selective column first for equality, 

-- range column last

CREATE INDEX IX_Orders_Customer_Date_Status 

ON orders (customer_id, order_date, status);

  

-- Supports these queries efficiently:

-- WHERE customer_id = ? AND order_date >= ?

-- WHERE customer_id = ? AND order_date = ? AND status = ?

-- WHERE customer_id = ?

  

3. Covering Indexes:
    

-- Include frequently accessed columns to avoid key lookups

CREATE INDEX IX_Orders_Customer_Covering 

ON orders (customer_id, order_date) 

INCLUDE (order_amount, status, product_id);

  

4. Partial Indexes:
    

-- PostgreSQL: Index only active records

CREATE INDEX IX_Orders_Active 

ON orders (customer_id, order_date) 

WHERE status = 'ACTIVE';

  

-- SQL Server: Filtered index

CREATE INDEX IX_Orders_Recent 

ON orders (customer_id) 

WHERE order_date >= '2024-01-01';

  

Index Maintenance Strategy:

-- Monitor index usage

SELECT 

    i.name AS index_name,

    s.user_seeks,

    s.user_scans,

    s.user_lookups,

    s.user_updates

FROM sys.indexes i

JOIN sys.dm_db_index_usage_stats s 

    ON i.object_id = s.object_id AND i.index_id = s.index_id

WHERE s.user_seeks + s.user_scans + s.user_lookups = 0

AND s.user_updates > 0;  -- Unused but maintained indexes

  

-- Index fragmentation check

SELECT 

    object_name(object_id) AS table_name,

    index_id,

    avg_fragmentation_in_percent,

    page_count

FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED')

WHERE avg_fragmentation_in_percent > 10

AND page_count > 1000;

  

### 1.3 Query Rewriting Techniques

Q: What are the most effective query rewriting patterns for performance optimization?

Answer:

1. Subquery to JOIN Conversion:

-- Slow: Correlated subquery

SELECT p.product_name, p.price

FROM products p

WHERE p.price > (

    SELECT AVG(price) 

    FROM products p2 

    WHERE p2.category = p.category

);

  

-- Fast: Window function

SELECT product_name, price

FROM (

    SELECT 

        product_name, 

        price,

        AVG(price) OVER (PARTITION BY category) AS avg_category_price

    FROM products

) t

WHERE price > avg_category_price;

  

2. UNION to UNION ALL:

-- Slow: UNION removes duplicates (expensive sort/distinct)

SELECT customer_id FROM active_customers

UNION

SELECT customer_id FROM vip_customers;

  

-- Fast: UNION ALL if duplicates are acceptable or known not to exist

SELECT customer_id FROM active_customers

UNION ALL

SELECT customer_id FROM vip_customers;

  

3. EXISTS vs IN Optimization:

-- For large datasets, EXISTS often faster

SELECT c.customer_name

FROM customers c

WHERE EXISTS (

    SELECT 1 FROM orders o 

    WHERE o.customer_id = c.customer_id

);

  

-- IN can be faster for small, static lists

SELECT c.customer_name

FROM customers c

WHERE c.customer_id IN (SELECT customer_id FROM orders);

  

-- For NULL-safe comparisons, use EXISTS

SELECT c.customer_name

FROM customers c

WHERE EXISTS (

    SELECT 1 FROM orders o 

    WHERE o.customer_id = c.customer_id 

    AND o.status = c.preferred_status  -- Handles NULLs correctly

);

  

4. Conditional Aggregation:

-- Instead of multiple queries

SELECT 

    COUNT(CASE WHEN status = 'COMPLETED' THEN 1 END) as completed_orders,

    COUNT(CASE WHEN status = 'PENDING' THEN 1 END) as pending_orders,

    COUNT(CASE WHEN status = 'CANCELLED' THEN 1 END) as cancelled_orders,

    SUM(CASE WHEN status = 'COMPLETED' THEN order_amount ELSE 0 END) as completed_revenue

FROM orders

WHERE order_date >= '2024-01-01';

  

---

## 2. Complex SQL Patterns

### 2.1 Window Functions and Analytics

Q: How do you use window functions to solve complex analytical problems? Provide examples of advanced window function patterns.

Answer:

1. Running Totals and Moving Averages:

SELECT 

    order_date,

    daily_revenue,

    -- Running total

    SUM(daily_revenue) OVER (

        ORDER BY order_date 

        ROWS UNBOUNDED PRECEDING

    ) AS running_total,

    -- 7-day moving average

    AVG(daily_revenue) OVER (

        ORDER BY order_date 

        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

    ) AS moving_avg_7day,

    -- Month-to-date total

    SUM(daily_revenue) OVER (

        PARTITION BY YEAR(order_date), MONTH(order_date)

        ORDER BY order_date

        ROWS UNBOUNDED PRECEDING

    ) AS mtd_total

FROM daily_sales;

  

2. Ranking and Top-N Analysis:

-- Top 3 products per category by revenue

SELECT category, product_name, total_revenue

FROM (

    SELECT 

        category,

        product_name,

        SUM(revenue) AS total_revenue,

        ROW_NUMBER() OVER (

            PARTITION BY category 

            ORDER BY SUM(revenue) DESC

        ) AS rn

    FROM product_sales

    GROUP BY category, product_name

) ranked

WHERE rn <= 3;

  

-- Percentile analysis

SELECT 

    customer_id,

    total_spent,

    NTILE(4) OVER (ORDER BY total_spent) AS quartile,

    PERCENT_RANK() OVER (ORDER BY total_spent) AS percentile_rank,

    CUME_DIST() OVER (ORDER BY total_spent) AS cumulative_dist

FROM customer_spending;

  

3. Gap and Island Analysis:

-- Find consecutive days with sales

WITH sales_with_groups AS (

    SELECT 

        sale_date,

        ROW_NUMBER() OVER (ORDER BY sale_date) AS rn,

        sale_date - INTERVAL ROW_NUMBER() OVER (ORDER BY sale_date) DAY AS group_date

    FROM (SELECT DISTINCT sale_date FROM sales) dates

),

consecutive_periods AS (

    SELECT 

        group_date,

        MIN(sale_date) AS period_start,

        MAX(sale_date) AS period_end,

        COUNT(*) AS consecutive_days

    FROM sales_with_groups

    GROUP BY group_date

)

SELECT * FROM consecutive_periods

WHERE consecutive_days >= 7;  -- At least 7 consecutive days

  

4. Advanced Lead/Lag Analysis:

-- Customer purchase behavior analysis

SELECT 

    customer_id,

    order_date,

    order_amount,

    -- Days since last order

    DATEDIFF(

        order_date, 

        LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)

    ) AS days_since_last_order,

    -- Compare with previous order amount

    order_amount - LAG(order_amount) OVER (

        PARTITION BY customer_id ORDER BY order_date

    ) AS amount_change,

    -- Next order preview

    LEAD(order_date) OVER (

        PARTITION BY customer_id ORDER BY order_date

    ) AS next_order_date,

    -- First and last order values

    FIRST_VALUE(order_amount) OVER (

        PARTITION BY customer_id 

        ORDER BY order_date 

        ROWS UNBOUNDED PRECEDING

    ) AS first_order_amount,

    LAST_VALUE(order_amount) OVER (

        PARTITION BY customer_id 

        ORDER BY order_date 

        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

    ) AS last_order_amount

FROM orders;

  

### 2.2 CTEs and Recursive Queries

Q: When and how do you use CTEs effectively? Provide examples of recursive CTE patterns.

Answer:

1. Complex Query Organization:

-- Break down complex logic into readable steps

WITH monthly_sales AS (

    SELECT 

        YEAR(order_date) AS year,

        MONTH(order_date) AS month,

        SUM(order_amount) AS total_sales,

        COUNT(DISTINCT customer_id) AS unique_customers

    FROM orders

    WHERE order_date >= '2023-01-01'

    GROUP BY YEAR(order_date), MONTH(order_date)

),

sales_with_growth AS (

    SELECT 

        year,

        month,

        total_sales,

        unique_customers,

        LAG(total_sales) OVER (ORDER BY year, month) AS prev_month_sales,

        (total_sales - LAG(total_sales) OVER (ORDER BY year, month)) / 

        LAG(total_sales) OVER (ORDER BY year, month) * 100 AS growth_rate

    FROM monthly_sales

)

SELECT 

    year,

    month,

    total_sales,

    unique_customers,

    ROUND(growth_rate, 2) AS growth_rate_pct,

    CASE 

        WHEN growth_rate > 10 THEN 'High Growth'

        WHEN growth_rate > 0 THEN 'Positive Growth'

        WHEN growth_rate < -10 THEN 'Declining'

        ELSE 'Stable'

    END AS growth_category

FROM sales_with_growth

ORDER BY year, month;

  

2. Recursive Hierarchical Queries:

-- Employee hierarchy traversal

WITH RECURSIVE employee_hierarchy AS (

    -- Anchor: Top-level managers

    SELECT 

        employee_id,

        employee_name,

        manager_id,

        0 AS level,

        CAST(employee_name AS VARCHAR(1000)) AS hierarchy_path

    FROM employees

    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Add subordinates

    SELECT 

        e.employee_id,

        e.employee_name,

        e.manager_id,

        h.level + 1,

        CONCAT(h.hierarchy_path, ' -> ', e.employee_name)

    FROM employees e

    INNER JOIN employee_hierarchy h ON e.manager_id = h.employee_id

    WHERE h.level < 10  -- Prevent infinite recursion

)

SELECT 

    employee_id,

    REPEAT('  ', level) || employee_name AS indented_name,

    level,

    hierarchy_path

FROM employee_hierarchy

ORDER BY hierarchy_path;

  

3. Recursive Data Generation:

-- Generate date series

WITH RECURSIVE date_series AS (

    SELECT DATE('2024-01-01') AS date_value

    UNION ALL

    SELECT date_value + INTERVAL 1 DAY

    FROM date_series

    WHERE date_value < DATE('2024-12-31')

),

-- Fill gaps in sales data

complete_sales AS (

    SELECT 

        ds.date_value,

        COALESCE(s.daily_sales, 0) AS daily_sales

    FROM date_series ds

    LEFT JOIN (

        SELECT 

            order_date,

            SUM(order_amount) AS daily_sales

        FROM orders

        WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'

        GROUP BY order_date

    ) s ON ds.date_value = s.order_date

)

SELECT * FROM complete_sales;

  

### 2.3 Advanced Joins and Subqueries

Q: How do you handle complex join scenarios and optimize subqueries? What are the performance implications of different join types?

Answer:

1. Complex Multi-Table Joins:

-- Comprehensive customer analysis with multiple fact tables

SELECT 

    c.customer_id,

    c.customer_name,

    c.registration_date,

    COALESCE(o.total_orders, 0) AS total_orders,

    COALESCE(o.total_spent, 0) AS total_spent,

    COALESCE(s.support_tickets, 0) AS support_tickets,

    COALESCE(r.avg_rating, 0) AS avg_rating,

    CASE 

        WHEN o.total_spent > 10000 THEN 'VIP'

        WHEN o.total_spent > 1000 THEN 'Premium'

        ELSE 'Standard'

    END AS customer_tier

FROM customers c

LEFT JOIN (

    SELECT 

        customer_id,

        COUNT(*) AS total_orders,

        SUM(order_amount) AS total_spent,

        MAX(order_date) AS last_order_date

    FROM orders

    GROUP BY customer_id

) o ON c.customer_id = o.customer_id

LEFT JOIN (

    SELECT 

        customer_id,

        COUNT(*) AS support_tickets,

        AVG(CASE WHEN resolution_time <= 24 THEN 1 ELSE 0 END) AS quick_resolution_rate

    FROM support_tickets

    GROUP BY customer_id

) s ON c.customer_id = s.customer_id

LEFT JOIN (

    SELECT 

        customer_id,

        AVG(rating) AS avg_rating,

        COUNT(*) AS total_reviews

    FROM product_reviews

    GROUP BY customer_id

) r ON c.customer_id = r.customer_id;

  

2. Anti-Join Patterns:

-- Find customers who haven't ordered in the last 90 days

SELECT c.customer_id, c.customer_name, c.email

FROM customers c

LEFT JOIN orders o ON c.customer_id = o.customer_id 

    AND o.order_date >= CURRENT_DATE - INTERVAL 90 DAY

WHERE o.customer_id IS NULL

AND c.registration_date < CURRENT_DATE - INTERVAL 90 DAY;

  

-- Alternative using NOT EXISTS (often faster for large datasets)

SELECT c.customer_id, c.customer_name, c.email

FROM customers c

WHERE NOT EXISTS (

    SELECT 1 FROM orders o 

    WHERE o.customer_id = c.customer_id 

    AND o.order_date >= CURRENT_DATE - INTERVAL 90 DAY

)

AND c.registration_date < CURRENT_DATE - INTERVAL 90 DAY;

  

3. Lateral Joins (PostgreSQL/SQL Server CROSS APPLY):

-- PostgreSQL: Get top 3 orders for each customer

SELECT 

    c.customer_name,

    recent_orders.order_date,

    recent_orders.order_amount

FROM customers c

CROSS JOIN LATERAL (

    SELECT order_date, order_amount

    FROM orders o

    WHERE o.customer_id = c.customer_id

    ORDER BY order_date DESC

    LIMIT 3

) recent_orders;

  

-- SQL Server equivalent

SELECT 

    c.customer_name,

    recent_orders.order_date,

    recent_orders.order_amount

FROM customers c

CROSS APPLY (

    SELECT TOP 3 order_date, order_amount

    FROM orders o

    WHERE o.customer_id = c.customer_id

    ORDER BY order_date DESC

) recent_orders;

  

4. Self-Joins for Comparative Analysis:

-- Find products with declining sales trends

SELECT 

    current_month.product_id,

    current_month.product_name,

    current_month.current_sales,

    previous_month.previous_sales,

    (current_month.current_sales - previous_month.previous_sales) AS sales_change,

    ROUND(

        (current_month.current_sales - previous_month.previous_sales) * 100.0 / 

        previous_month.previous_sales, 2

    ) AS change_percentage

FROM (

    SELECT 

        p.product_id,

        p.product_name,

        SUM(o.order_amount) AS current_sales

    FROM products p

    JOIN orders o ON p.product_id = o.product_id

    WHERE o.order_date >= DATE_TRUNC('month', CURRENT_DATE)

    GROUP BY p.product_id, p.product_name

) current_month

JOIN (

    SELECT 

        p.product_id,

        SUM(o.order_amount) AS previous_sales

    FROM products p

    JOIN orders o ON p.product_id = o.product_id

    WHERE o.order_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'

    AND o.order_date < DATE_TRUNC('month', CURRENT_DATE)

    GROUP BY p.product_id

) previous_month ON current_month.product_id = previous_month.product_id

WHERE (current_month.current_sales - previous_month.previous_sales) < 0

ORDER BY change_percentage;

  

---

## 3. Data Modeling and Schema Design

### 3.1 Dimensional Modelling

Q: How do you design effective dimensional models? What are the key principles of star vs snowflake schemas?

Answer:

Star Schema Design:

-- Fact table: Sales transactions

CREATE TABLE fact_sales (

    sale_id BIGINT PRIMARY KEY,

    date_key INT NOT NULL,

    customer_key INT NOT NULL,

    product_key INT NOT NULL,

    store_key INT NOT NULL,

    quantity INT NOT NULL,

    unit_price DECIMAL(10,2) NOT NULL,

    total_amount DECIMAL(12,2) NOT NULL,

    discount_amount DECIMAL(10,2) DEFAULT 0,

    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),

    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),

    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),

    FOREIGN KEY (store_key) REFERENCES dim_store(store_key)

);

  

-- Date dimension with pre-calculated attributes

CREATE TABLE dim_date (

    date_key INT PRIMARY KEY,

    full_date DATE NOT NULL,

    day_of_week INT NOT NULL,

    day_name VARCHAR(20) NOT NULL,

    day_of_month INT NOT NULL,

    day_of_year INT NOT NULL,

    week_of_year INT NOT NULL,

    month_number INT NOT NULL,

    month_name VARCHAR(20) NOT NULL,

    quarter INT NOT NULL,

    year INT NOT NULL,

    is_weekend BOOLEAN NOT NULL,

    is_holiday BOOLEAN NOT NULL,

    fiscal_year INT NOT NULL,

    fiscal_quarter INT NOT NULL

);

  

-- Customer dimension

CREATE TABLE dim_customer (

    customer_key INT PRIMARY KEY,

    customer_id VARCHAR(50) NOT NULL, -- Natural key

    customer_name VARCHAR(200) NOT NULL,

    customer_type VARCHAR(50) NOT NULL,

    segment VARCHAR(50) NOT NULL,

    registration_date DATE NOT NULL,

    city VARCHAR(100),

    state VARCHAR(50),

    country VARCHAR(50),

    -- SCD Type 2 fields

    effective_date DATE NOT NULL,

    expiry_date DATE,

    is_current BOOLEAN NOT NULL DEFAULT TRUE,

    UNIQUE(customer_id, effective_date)

);

  

Snowflake Schema Example:

-- Normalized product dimension

CREATE TABLE dim_product (

    product_key INT PRIMARY KEY,

    product_id VARCHAR(50) NOT NULL,

    product_name VARCHAR(200) NOT NULL,

    brand_key INT NOT NULL,

    category_key INT NOT NULL,

    subcategory_key INT NOT NULL,

    unit_cost DECIMAL(10,2),

    FOREIGN KEY (brand_key) REFERENCES dim_brand(brand_key),

    FOREIGN KEY (category_key) REFERENCES dim_category(category_key),

    FOREIGN KEY (subcategory_key) REFERENCES dim_subcategory(subcategory_key)

);

  

CREATE TABLE dim_brand (

    brand_key INT PRIMARY KEY,

    brand_name VARCHAR(100) NOT NULL,

    brand_country VARCHAR(50)

);

  

CREATE TABLE dim_category (

    category_key INT PRIMARY KEY,

    category_name VARCHAR(100) NOT NULL,

    category_description TEXT

);

  

Design Principles:

- Star Schema: Denormalized dimensions, simpler queries, better performance
    
- Snowflake Schema: Normalized dimensions, reduced storage, more complex queries
    
- Conformed Dimensions: Shared across multiple fact tables for consistency
    
- Degenerate Dimensions: Dimension attributes stored in fact table (like invoice numbers)
    

### 3.2 Slowly Changing Dimensions

Q: How do you implement different types of slowly changing dimensions? Provide examples of SCD Types 1, 2, and 3.

Answer:

SCD Type 1 - Overwrite:

-- Update customer address (lose history)

UPDATE dim_customer 

SET 

    address = 'New Address',

    city = 'New City',

    state = 'New State',

    last_updated = CURRENT_TIMESTAMP

WHERE customer_id = 'CUST001';

  

-- Implementation pattern

MERGE dim_customer AS target

USING staging_customer AS source

ON target.customer_id = source.customer_id

WHEN MATCHED THEN

    UPDATE SET 

        customer_name = source.customer_name,

        address = source.address,

        city = source.city,

        state = source.state,

        last_updated = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN

    INSERT (customer_id, customer_name, address, city, state, created_date)

    VALUES (source.customer_id, source.customer_name, source.address, 

            source.city, source.state, CURRENT_TIMESTAMP);

  

SCD Type 2 - Add New Record:

-- Implementation for customer dimension changes

WITH customer_changes AS (

    SELECT 

        s.customer_id,

        s.customer_name,

        s.segment,

        s.address,

        s.city,

        s.state,

        CASE 

            WHEN d.customer_id IS NULL THEN 'INSERT'

            WHEN d.segment != s.segment 

                OR d.address != s.address 

                OR d.city != s.city 

                OR d.state != s.state THEN 'UPDATE'

            ELSE 'NO_CHANGE'

        END AS change_type

    FROM staging_customer s

    LEFT JOIN dim_customer d ON s.customer_id = d.customer_id 

        AND d.is_current = TRUE

)

-- Expire current records

UPDATE dim_customer 

SET 

    expiry_date = CURRENT_DATE - 1,

    is_current = FALSE

WHERE customer_id IN (

    SELECT customer_id FROM customer_changes WHERE change_type = 'UPDATE'

)

AND is_current = TRUE;

  

-- Insert new records

INSERT INTO dim_customer (

    customer_id, customer_name, segment, address, city, state,

    effective_date, expiry_date, is_current

)

SELECT 

    customer_id, customer_name, segment, address, city, state,

    CURRENT_DATE, '9999-12-31', TRUE

FROM customer_changes

WHERE change_type IN ('INSERT', 'UPDATE');

  

SCD Type 3 - Add New Column:

-- Track previous value in separate column

ALTER TABLE dim_customer 

ADD COLUMN previous_segment VARCHAR(50),

ADD COLUMN segment_change_date DATE;

  

-- Update with history

UPDATE dim_customer 

SET 

    previous_segment = segment,

    segment = 'Premium',

    segment_change_date = CURRENT_DATE

WHERE customer_id = 'CUST001';

  

SCD Type 4 - History Table:

-- Current table

CREATE TABLE dim_customer_current AS 

SELECT * FROM dim_customer WHERE is_current = TRUE;

  

-- History table

CREATE TABLE dim_customer_history AS

SELECT * FROM dim_customer WHERE is_current = FALSE;

  

-- Query across both for complete history

SELECT * FROM dim_customer_current

UNION ALL

SELECT * FROM dim_customer_history

WHERE customer_id = 'CUST001'

ORDER BY effective_date;

  

### 3.3 Schema Evolution Strategies

Q: How do you handle schema changes in production environments? What are the best practices for backward compatibility?

Answer:

1. Additive Changes (Safest):

-- Add new optional columns

ALTER TABLE customers 

ADD COLUMN middle_name VARCHAR(50),

ADD COLUMN preferred_contact_method VARCHAR(20) DEFAULT 'email',

ADD COLUMN created_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

  

-- Create new indexes

CREATE INDEX IX_customers_contact_method 

ON customers (preferred_contact_method);

  

2. Column Modifications (Requires Planning):

-- Step 1: Add new column

ALTER TABLE products ADD COLUMN product_code_new VARCHAR(20);

  

-- Step 2: Populate new column

UPDATE products 

SET product_code_new = LPAD(product_code, 20, '0');

  

-- Step 3: Update application to use new column

  

-- Step 4: Drop old column (after validation)

ALTER TABLE products DROP COLUMN product_code;

  

-- Step 5: Rename new column

ALTER TABLE products RENAME COLUMN product_code_new TO product_code;

  

3. Table Restructuring (Blue-Green Approach):

-- Create new table structure

CREATE TABLE orders_v2 (

    order_id BIGINT PRIMARY KEY,

    customer_id INT NOT NULL,

    order_date DATE NOT NULL,

    order_items JSONB NOT NULL,  -- Normalized structure

    total_amount DECIMAL(12,2) NOT NULL,

    currency_code VARCHAR(3) NOT NULL,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

);

  

-- Migrate data with transformation

INSERT INTO orders_v2 (order_id, customer_id, order_date, order_items, total_amount, currency_code)

SELECT 

    order_id,

    customer_id,

    order_date,

    JSON_OBJECT(

        'product_id', product_id,

        'quantity', quantity,

        'unit_price', unit_price

    ) AS order_items,

    total_amount,

    'USD' AS currency_code

FROM orders_v1;

  

-- Switch with minimal downtime

BEGIN;

    ALTER TABLE orders RENAME TO orders_old;

    ALTER TABLE orders_v2 RENAME TO orders;

COMMIT;

  

4. Version-Based Schema Management:

-- Schema version tracking

CREATE TABLE schema_versions (

    version_id INT PRIMARY KEY,

    version_name VARCHAR(50) NOT NULL,

    applied_date TIMESTAMP NOT NULL,

    description TEXT,

    rollback_script TEXT

);

  

-- Migration script template

-- Migration: v1.2.0 - Add customer segments

-- Date: 2024-08-13

-- Description: Add customer segmentation support

  

-- Forward migration

ALTER TABLE customers ADD COLUMN segment VARCHAR(50) DEFAULT 'STANDARD';

UPDATE customers SET segment = 'PREMIUM' WHERE total_orders > 100;

CREATE INDEX IX_customers_segment ON customers (segment);

  

-- Record migration

INSERT INTO schema_versions VALUES 

(120, 'v1.2.0', CURRENT_TIMESTAMP, 'Add customer segments', 

 'ALTER TABLE customers DROP COLUMN segment; DROP INDEX IX_customers_segment;');

  

---

## 4. ETL and Data Pipeline Patterns

### 4.1 Incremental Loading Strategies

Q: How do you implement efficient incremental data loading? What are the different patterns and their use cases?

Answer:

1. Timestamp-Based Incremental Loading:

-- Control table to track high water marks

CREATE TABLE etl_control (

    table_name VARCHAR(100) PRIMARY KEY,

    last_loaded_timestamp TIMESTAMP NOT NULL,

    last_loaded_id BIGINT,

    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP

);

  

-- Incremental load procedure

WITH new_records AS (

    SELECT *

    FROM source_orders

    WHERE updated_at > (

        SELECT last_loaded_timestamp 

        FROM etl_control 

        WHERE table_name = 'orders'

    )

),

upsert_data AS (

    -- UPSERT pattern

    INSERT INTO target_orders 

    SELECT * FROM new_records

    ON CONFLICT (order_id) 

    DO UPDATE SET

        customer_id = EXCLUDED.customer_id,

        order_amount = EXCLUDED.order_amount,

        status = EXCLUDED.status,

        updated_at = EXCLUDED.updated_at

)

-- Update control table

UPDATE etl_control 

SET 

    last_loaded_timestamp = (SELECT MAX(updated_at) FROM new_records),

    updated_at = CURRENT_TIMESTAMP

WHERE table_name = 'orders';

  

2. Hash-Based Change Detection:

-- Add hash column for change detection

ALTER TABLE staging_customers 

ADD COLUMN row_hash VARCHAR(64);

  

-- Calculate hash of business columns

UPDATE staging_customers 

SET row_hash = MD5(CONCAT(

    COALESCE(customer_name, ''),

    COALESCE(email, ''),

    COALESCE(phone, ''),

    COALESCE(address, ''),

    COALESCE(segment, '')

));

  

-- Identify changes

WITH changed_records AS (

    SELECT s.*

    FROM staging_customers s

    LEFT JOIN target_customers t ON s.customer_id = t.customer_id

    WHERE t.customer_id IS NULL  -- New records

       OR s.row_hash != t.row_hash  -- Changed records

)

-- Process only changed records

MERGE target_customers AS target

USING changed_records AS source

ON target.customer_id = source.customer_id

WHEN MATCHED THEN

    UPDATE SET 

        customer_name = source.customer_name,

        email = source.email,

        phone = source.phone,

        address = source.address,

        segment = source.segment,

        row_hash = source.row_hash,

        updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN

    INSERT VALUES (source.customer_id, source.customer_name, source.email,

                   source.phone, source.address, source.segment, 

                   source.row_hash, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

  

3. Partition-Based Incremental Loading:

-- Load only new partitions

CREATE TABLE sales_daily (

    sale_date DATE NOT NULL,

    customer_id INT NOT NULL,

    product_id INT NOT NULL,

    quantity INT NOT NULL,

    amount DECIMAL(10,2) NOT NULL

) PARTITION BY RANGE (sale_date);

  

-- Create partitions dynamically

DO $

DECLARE

    partition_date DATE;

BEGIN

    FOR partition_date IN 

        SELECT generate_series(

            DATE_TRUNC('month', CURRENT_DATE),

            DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '3 months',

            INTERVAL '1 day'

        )::DATE

    LOOP

        EXECUTE format('

            CREATE TABLE IF NOT EXISTS sales_daily_%s 

            PARTITION OF sales_daily 

            FOR VALUES FROM (%L) TO (%L)',

            to_char(partition_date, 'YYYY_MM_DD'),

            partition_date,

            partition_date + 1

        );

    END LOOP;

END $;

  

-- Load only yesterday's data

INSERT INTO sales_daily

SELECT * FROM staging_sales

WHERE sale_date = CURRENT_DATE - 1;

  

### 4.2 Change Data Capture (CDC)

Q: How do you implement Change Data Capture patterns? What are the different CDC approaches and their trade-offs?

Answer:

1. Trigger-Based CDC:

-- Create audit/change table

CREATE TABLE customer_changes (

    change_id SERIAL PRIMARY KEY,

    customer_id INT NOT NULL,

    operation CHAR(1) NOT NULL, -- I/U/D

    old_values JSONB,

    new_values JSONB,

    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    changed_by VARCHAR(100) DEFAULT USER

);

  

-- Create trigger function

CREATE OR REPLACE FUNCTION capture_customer_changes()

RETURNS TRIGGER AS $

BEGIN

    IF TG_OP = 'INSERT' THEN

        INSERT INTO customer_changes (customer_id, operation, new_values)

        VALUES (NEW.customer_id, 'I', row_to_json(NEW)::jsonb);

        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN

        INSERT INTO customer_changes (customer_id, operation, old_values, new_values)

        VALUES (NEW.customer_id, 'U', row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);

        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN

        INSERT INTO customer_changes (customer_id, operation, old_values)

        VALUES (OLD.customer_id, 'D', row_to_json(OLD)::jsonb);

        RETURN OLD;

    END IF;

    RETURN NULL;

END;

$ LANGUAGE plpgsql;

  

-- Attach trigger

CREATE TRIGGER customer_cdc_trigger

    AFTER INSERT OR UPDATE OR DELETE ON customers

    FOR EACH ROW EXECUTE FUNCTION capture_customer_changes();

  

2. Log-Based CDC (Conceptual Pattern):

-- Process transaction log entries

WITH log_entries AS (

    SELECT 

        table_name,

        operation,

        lsn,

        transaction_id,

        commit_time,

        old_values,

        new_values

    FROM transaction_log

    WHERE commit_time > (

        SELECT last_processed_time 

        FROM cdc_checkpoint 

        WHERE source_table = 'customers'

    )

    AND table_name = 'customers'

),

processed_changes AS (

    SELECT 

        CASE operation

            WHEN 'INSERT' THEN 

                JSON_EXTRACT_SCALAR(new_values, '$.customer_id')

            WHEN 'UPDATE' THEN 

                JSON_EXTRACT_SCALAR(new_values, '$.customer_id')

            WHEN 'DELETE' THEN 

                JSON_EXTRACT_SCALAR(old_values, '$.customer_id')

        END AS customer_id,

        operation,

        old_values,

        new_values,

        commit_time

    FROM log_entries

    ORDER BY lsn

)

-- Apply changes to target

SELECT * FROM processed_changes;

  

3. Timestamp-Based CDC:

-- Add CDC columns to source tables

ALTER TABLE customers 

ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;

  

-- Update trigger for timestamp

CREATE OR REPLACE FUNCTION update_timestamp()

RETURNS TRIGGER AS $

BEGIN

    NEW.updated_at = CURRENT_TIMESTAMP;

    RETURN NEW;

END;

$ LANGUAGE plpgsql;

  

CREATE TRIGGER customers_update_timestamp

    BEFORE UPDATE ON customers

    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

  

-- CDC extraction query

SELECT 

    customer_id,

    customer_name,

    email,

    CASE 

        WHEN is_deleted THEN 'DELETE'

        WHEN created_at = updated_at THEN 'INSERT'

        ELSE 'UPDATE'

    END AS operation,

    updated_at

FROM customers

WHERE updated_at > :last_sync_time

ORDER BY updated_at;

  

### 4.3 Data Quality and Validation

Q: How do you implement comprehensive data quality checks in SQL? What are the key validation patterns?

Answer:

1. Data Quality Framework:

-- Data quality rules table

CREATE TABLE dq_rules (

    rule_id SERIAL PRIMARY KEY,

    table_name VARCHAR(100) NOT NULL,

    column_name VARCHAR(100),

    rule_type VARCHAR(50) NOT NULL,

    rule_definition TEXT NOT NULL,

    severity VARCHAR(20) DEFAULT 'ERROR',

    is_active BOOLEAN DEFAULT TRUE

);

  

-- Sample rules

INSERT INTO dq_rules (table_name, column_name, rule_type, rule_definition, severity) VALUES

('customers', 'email', 'FORMAT', 'email ~ ''^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

  

*Focused on practical skills and leadership scenarios for lead data engineer interviews*'', 'ERROR'),

('customers', 'phone', 'FORMAT', 'phone ~ ''^[\+]?[1-9][\d]{0,15}# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

  

*Focused on practical skills and leadership scenarios for lead data engineer interviews*'', 'WARNING'),

('orders', 'order_amount', 'RANGE', 'order_amount > 0 AND order_amount < 100000', 'ERROR'),

('orders', 'order_date', 'RANGE', 'order_date >= ''2020-01-01'' AND order_date <= CURRENT_DATE', 'ERROR'),

('customers', NULL, 'UNIQUENESS', 'COUNT(*) = COUNT(DISTINCT email)', 'ERROR');

  

2. Comprehensive Validation Checks:

-- Data quality validation procedure

WITH validation_results AS (

    -- Completeness checks

    SELECT 

        'customers' AS table_name,

        'completeness' AS check_type,

        'customer_name' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(customer_name) AS valid_rows,

        COUNT(*) - COUNT(customer_name) AS null_count,

        ROUND(COUNT(customer_name) * 100.0 / COUNT(*), 2) AS completeness_pct

    FROM customers

    UNION ALL

    -- Uniqueness checks

    SELECT 

        'customers' AS table_name,

        'uniqueness' AS check_type,

        'email' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(DISTINCT email) AS valid_rows,

        COUNT(*) - COUNT(DISTINCT email) AS duplicate_count,

        ROUND(COUNT(DISTINCT email) * 100.0 / COUNT(*), 2) AS uniqueness_pct

    FROM customers

    UNION ALL

    -- Format validation

    SELECT 

        'customers' AS table_name,

        'format' AS check_type,

        'email' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(*) FILTER (WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

  

*Focused on practical skills and leadership scenarios for lead data engineer interviews*) AS valid_rows,

        COUNT(*) FILTER (WHERE email IS NOT NULL AND email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

  

*Focused on practical skills and leadership scenarios for lead data engineer interviews*) AS invalid_count,

        ROUND(COUNT(*) FILTER (WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

  

*Focused on practical skills and leadership scenarios for lead data engineer interviews*) * 100.0 / COUNT(*), 2) AS format_validity_pct

    FROM customers

    UNION ALL

    -- Range validation

    SELECT 

        'orders' AS table_name,

        'range' AS check_type,

        'order_amount' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(*) FILTER (WHERE order_amount > 0 AND order_amount <= 100000) AS valid_rows,

        COUNT(*) FILTER (WHERE order_amount <= 0 OR order_amount > 100000) AS out_of_range_count,

        ROUND(COUNT(*) FILTER (WHERE order_amount > 0 AND order_amount <= 100000) * 100.0 / COUNT(*), 2) AS range_validity_pct

    FROM orders

),

-- Business rule validation

business_validation AS (

    -- Referential integrity

    SELECT 

        'orders' AS table_name,

        'referential_integrity' AS check_type,

        'customer_id' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(*) - COUNT(c.customer_id) AS orphan_count,

        ROUND((COUNT(*) - COUNT(c.customer_id)) * 100.0 / COUNT(*), 2) AS orphan_pct

    FROM orders o

    LEFT JOIN customers c ON o.customer_id = c.customer_id

    UNION ALL

    -- Logical consistency

    SELECT 

        'orders' AS table_name,

        'logical_consistency' AS check_type,

        'dates' AS column_name,

        COUNT(*) AS total_rows,

        COUNT(*) FILTER (WHERE created_at <= shipped_at OR shipped_at IS NULL) AS valid_rows,

        COUNT(*) FILTER (WHERE created_at > shipped_at) AS inconsistent_count,

        ROUND(COUNT(*) FILTER (WHERE created_at <= shipped_at OR shipped_at IS NULL) * 100.0 / COUNT(*), 2) AS consistency_pct

    FROM orders

)

-- Combine all validation results

SELECT * FROM validation_results

UNION ALL

SELECT 

    table_name, check_type, column_name, total_rows,

    total_rows - orphan_count AS valid_rows,

    orphan_count AS invalid_count,

    100 - orphan_pct AS validity_pct

FROM business_validation;

  

3. Automated Data Quality Monitoring:

-- Data quality metrics tracking

CREATE TABLE dq_metrics (

    metric_id SERIAL PRIMARY KEY,

    table_name VARCHAR(100) NOT NULL,

    check_type VARCHAR(50) NOT NULL,

    column_name VARCHAR(100),

    total_rows BIGINT NOT NULL,

    valid_rows BIGINT NOT NULL,

    invalid_rows BIGINT NOT NULL,

    validity_percentage DECIMAL(5,2) NOT NULL,

    check_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    threshold_met BOOLEAN NOT NULL

);

  

-- Automated quality check procedure

CREATE OR REPLACE FUNCTION run_data_quality_checks()

RETURNS TABLE(

    table_name VARCHAR(100),

    check_type VARCHAR(50),

    column_name VARCHAR(100),

    status VARCHAR(20),

    message TEXT

) AS $

DECLARE

    rec RECORD;

    check_result RECORD;

BEGIN

    -- Loop through active DQ rules

    FOR rec IN SELECT * FROM dq_rules WHERE is_active = TRUE LOOP

        -- Execute dynamic quality check based on rule type

        CASE rec.rule_type

            WHEN 'COMPLETENESS' THEN

                EXECUTE format('

                    SELECT COUNT(*) as total, COUNT(%I) as valid

                    FROM %I', 

                    rec.column_name, rec.table_name) INTO check_result;

            WHEN 'UNIQUENESS' THEN

                EXECUTE format('

                    SELECT COUNT(*) as total, COUNT(DISTINCT %I) as valid

                    FROM %I', 

                    rec.column_name, rec.table_name) INTO check_result;

            WHEN 'FORMAT' THEN

                EXECUTE format('

                    SELECT COUNT(*) as total, 

                           COUNT(*) FILTER (WHERE %s) as valid

                    FROM %I', 

                    rec.rule_definition, rec.table_name) INTO check_result;

        END CASE;

        -- Return results

        table_name := rec.table_name;

        check_type := rec.rule_type;

        column_name := rec.column_name;

        IF check_result.valid::DECIMAL / check_result.total >= 0.95 THEN

            status := 'PASS';

            message := format('Quality check passed: %s/%s valid records', 

                            check_result.valid, check_result.total);

        ELSE

            status := 'FAIL';

            message := format('Quality check failed: %s/%s valid records (%.2f%%)', 

                            check_result.valid, check_result.total,

                            check_result.valid::DECIMAL * 100 / check_result.total);

        END IF;

        RETURN NEXT;

    END LOOP;

END;

$ LANGUAGE plpgsql;

  

---

## 5. Modern SQL Platforms

### 5.1 Cloud Data Warehouses

Q: What are the key differences between major cloud data warehouses? How do you optimize queries for each platform?

Answer:

Snowflake Optimization Patterns:

-- Clustering for better performance

CREATE TABLE sales_fact (

    sale_date DATE,

    customer_id NUMBER,

    product_id NUMBER,

    amount NUMBER(10,2)

) CLUSTER BY (sale_date, customer_id);

  

-- Zero-copy cloning for development

CREATE TABLE sales_fact_dev CLONE sales_fact;

  

-- Time travel queries

SELECT * FROM sales_fact 

AT (TIMESTAMP => '2024-08-01 10:00:00'::timestamp);

  

-- Automatic clustering monitoring

SELECT 

    table_name,

    clustering_key,

    total_cluster_keys,

    average_overlaps,

    average_depth

FROM table_clustering_information

WHERE table_name = 'SALES_FACT';

  

-- Warehouse scaling

ALTER WAREHOUSE compute_wh SET warehouse_size = 'LARGE';

ALTER WAREHOUSE compute_wh SUSPEND;

ALTER WAREHOUSE compute_wh RESUME;

  

BigQuery Optimization Patterns:

-- Partitioned and clustered table

CREATE TABLE sales_fact (

    sale_date DATE,

    customer_id INT64,

    product_id INT64,

    region STRING,

    amount NUMERIC

)

PARTITION BY sale_date

CLUSTER BY customer_id, region;

  

-- Optimize with partition pruning

SELECT 

    customer_id,

    SUM(amount) as total_amount

FROM sales_fact

WHERE sale_date >= '2024-01-01'  -- Partition pruning

    AND region = 'US'            -- Cluster pruning

GROUP BY customer_id;

  

-- Approximate aggregation for large datasets

SELECT 

    region,

    APPROX_COUNT_DISTINCT(customer_id) as unique_customers,

    APPROX_QUANTILES(amount, 4) as amount_quartiles

FROM sales_fact

GROUP BY region;

  

-- Materialized views for performance

CREATE MATERIALIZED VIEW daily_sales_summary AS

SELECT 

    sale_date,

    region,

    COUNT(*) as transaction_count,

    SUM(amount) as total_amount,

    AVG(amount) as avg_amount

FROM sales_fact

GROUP BY sale_date, region;

  

Redshift Optimization Patterns:

-- Distribution and sort keys

CREATE TABLE sales_fact (

    sale_id BIGINT IDENTITY(1,1),

    sale_date DATE,

    customer_id INTEGER,

    product_id INTEGER,

    amount DECIMAL(10,2)

)

DISTKEY(customer_id)  -- Distribute by customer for joins

SORTKEY(sale_date, customer_id);  -- Sort for range queries

  

-- Compression encoding

CREATE TABLE sales_fact_compressed (

    sale_date DATE ENCODE delta,

    customer_id INTEGER ENCODE mostly16,

    product_id INTEGER ENCODE mostly16,

    amount DECIMAL(10,2) ENCODE mostly32,

    region VARCHAR(50) ENCODE lzo

);

  

-- Vacuum and analyze maintenance

VACUUM sales_fact;

ANALYZE sales_fact;

  

-- Query performance monitoring

SELECT 

    query,

    total_time,

    rows,

    bytes,

    cpu_time,

    io_time

FROM stl_query

WHERE endtime >= DATEADD(hour, -1, GETDATE())

ORDER BY total_time DESC;

  

### 5.2 Columnar vs Row-Based Storage

Q: When should you choose columnar vs row-based storage? How does storage format affect query optimization?

Answer:

Columnar Storage Benefits:

-- Analytical query - benefits from columnar storage

-- Only scans needed columns, better compression

SELECT 

    product_category,

    SUM(sales_amount) as total_sales,

    AVG(sales_amount) as avg_sales,

    COUNT(*) as transaction_count

FROM sales_transactions

WHERE sale_date BETWEEN '2024-01-01' AND '2024-12-31'

GROUP BY product_category

ORDER BY total_sales DESC;

  

-- Column pruning optimization

-- In columnar storage, only these columns are read:

-- - product_category (for GROUP BY)

-- - sales_amount (for aggregations)

-- - sale_date (for WHERE clause)

-- - No unnecessary I/O for other columns

  

Row-Based Storage Benefits:

-- OLTP query - benefits from row-based storage

-- Needs most/all columns, single row access

SELECT 

    order_id,

    customer_id,

    customer_name,

    customer_email,

    product_id,

    product_name,

    quantity,

    unit_price,

    total_amount,

    order_date,

    shipping_address,

    order_status

FROM orders o

JOIN customers c ON o.customer_id = c.customer_id

JOIN products p ON o.product_id = p.product_id

WHERE order_id = 12345;

  

-- Single row updates - row storage more efficient

UPDATE orders 

SET order_status = 'SHIPPED',

    tracking_number = 'TRACK123',

    shipped_date = CURRENT_TIMESTAMP

WHERE order_id = 12345;

  

Hybrid Approaches:

-- PostgreSQL - Column store extension (cstore_fdw)

CREATE FOREIGN TABLE sales_analytics (

    sale_date DATE,

    customer_id INTEGER,

    product_id INTEGER,

    amount DECIMAL(10,2)

) SERVER cstore_server

OPTIONS (compression 'pglz');

  

-- SQL Server - Columnstore indexes

CREATE CLUSTERED COLUMNSTORE INDEX CCI_sales_fact 

ON sales_fact;

  

-- Add nonclustered columnstore for analytics

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_orders_analytics

ON orders (order_date, customer_id, product_id, order_amount);

  

-- Oracle - In-Memory Column Store

ALTER TABLE sales_fact INMEMORY MEMCOMPRESS FOR QUERY HIGH;

  

Storage Format Decision Matrix:

|   |   |   |   |
|---|---|---|---|
|Use Case|Row-Based|Columnar|Hybrid|
|OLTP operations|✓|✗|✓|
|Analytics/OLAP|✗|✓|✓|
|Point queries|✓|✗|✓|
|Aggregation queries|✗|✓|✓|
|Frequent updates|✓|✗|✓|
|Data compression|✗|✓|✓|
|Mixed workloads|✗|✗|✓|

---

## 6. Production and Leadership

### 6.1 Performance Monitoring and Troubleshooting

Q: How do you establish performance monitoring for SQL systems? What are your troubleshooting methodologies?

Answer:

1. Key Performance Metrics:

-- Query performance monitoring view

CREATE VIEW query_performance_summary AS

SELECT 

    DATE_TRUNC('hour', start_time) AS hour,

    database_name,

    user_name,

    COUNT(*) AS query_count,

    AVG(duration_ms) AS avg_duration_ms,

    MAX(duration_ms) AS max_duration_ms,

    AVG(rows_examined) AS avg_rows_examined,

    AVG(rows_sent) AS avg_rows_sent,

    SUM(CASE WHEN duration_ms > 10000 THEN 1 ELSE 0 END) AS slow_queries

FROM query_log

WHERE start_time >= CURRENT_DATE - INTERVAL 7 DAY

GROUP BY 1, 2, 3

ORDER BY hour DESC;

  

-- Resource utilization tracking

CREATE TABLE resource_metrics (

    metric_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    cpu_usage_percent DECIMAL(5,2),

    memory_usage_percent DECIMAL(5,2),

    disk_io_wait_percent DECIMAL(5,2),

    active_connections INT,

    blocked_queries INT,

    cache_hit_ratio DECIMAL(5,2)

);

  

-- Automated alerting conditions

SELECT 

    metric_timestamp,

    'HIGH_CPU' AS alert_type,

    cpu_usage_percent AS metric_value

FROM resource_metrics

WHERE cpu_usage_percent > 80

AND metric_timestamp >= CURRENT_TIMESTAMP - INTERVAL 5 MINUTE

  

UNION ALL

  

SELECT 

    metric_timestamp,

    'LOW_CACHE_HIT_RATIO' AS alert_type,

    cache_hit_ratio AS metric_value

FROM resource_metrics

WHERE cache_hit_ratio < 90

AND metric_timestamp >= CURRENT_TIMESTAMP - INTERVAL 5 MINUTE;

  

2. Systematic Troubleshooting Process:

-- Step 1: Identify problematic queries

WITH slow_queries AS (

    SELECT 

        query_hash,

        COUNT(*) AS execution_count,

        AVG(duration_ms) AS avg_duration,

        MAX(duration_ms) AS max_duration,

        SUM(duration_ms) AS total_duration,

        AVG(cpu_time_ms) AS avg_cpu_time,

        AVG(logical_reads) AS avg_logical_reads

    FROM query_performance_log

    WHERE log_date >= CURRENT_DATE - 1

    GROUP BY query_hash

    HAVING AVG(duration_ms) > 5000 OR COUNT(*) > 1000

),

-- Step 2: Get query plans and resource usage

query_analysis AS (

    SELECT 

        sq.*,

        qp.execution_plan,

        qp.estimated_cost,

        qp.actual_cost,

        qi.index_usage,

        qi.table_scans

    FROM slow_queries sq

    LEFT JOIN query_plans qp ON sq.query_hash = qp.query_hash

    LEFT JOIN query_indexes qi ON sq.query_hash = qi.query_hash

)

-- Step 3: Prioritize by business impact

SELECT 

    query_hash,

    execution_count,

    avg_duration,

    total_duration,

    (execution_count * avg_duration) AS business_impact_score,

    CASE 

        WHEN avg_logical_reads > 100000 THEN 'I/O Intensive'

        WHEN avg_cpu_time > avg_duration * 0.8 THEN 'CPU Intensive'

        WHEN table_scans > 0 THEN 'Full Table Scan'

        ELSE 'Other'

    END AS performance_category

FROM query_analysis

ORDER BY business_impact_score DESC;

  

3. Proactive Performance Management:

-- Index recommendation system

WITH missing_indexes AS (

    SELECT 

        table_name,

        column_names,

        equality_columns,

        inequality_columns,

        included_columns,

        user_seeks,

        avg_total_user_cost,

        (user_seeks * avg_total_user_cost) AS improvement_measure

    FROM sys.dm_db_missing_index_details d

    JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle

    JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle

    WHERE d.database_id = DB_ID()

)

SELECT 

    table_name,

    'CREATE INDEX IX_' + table_name + '_suggested ON ' + 

    table_name + ' (' + 

    ISNULL(equality_columns, '') + 

    CASE WHEN inequality_columns IS NOT NULL 

         THEN ', ' + inequality_columns 

         ELSE '' END + ')' +

    CASE WHEN included_columns IS NOT NULL 

         THEN ' INCLUDE (' + included_columns + ')'

         ELSE '' END AS suggested_index,

    improvement_measure,

    user_seeks

FROM missing_indexes

WHERE improvement_measure > 10000

ORDER BY improvement_measure DESC;

  

### 6.2 SQL Code Standards and Reviews

Q: How do you establish and enforce SQL coding standards for a data engineering team?

Answer:

1. SQL Coding Standards Framework:

-- Example of well-formatted SQL following standards

WITH monthly_aggregates AS (

    -- Calculate monthly sales metrics per region

    SELECT 

        r.region_name,

        DATE_TRUNC('month', s.sale_date) AS month_year,

        COUNT(DISTINCT s.customer_id) AS unique_customers,

        COUNT(*) AS total_transactions,

        SUM(s.sale_amount) AS total_revenue,

        AVG(s.sale_amount) AS avg_transaction_value

    FROM sales s

    INNER JOIN regions r 

        ON s.region_id = r.region_id

    WHERE s.sale_date >= DATE_TRUNC('year', CURRENT_DATE)

        AND s.sale_status = 'COMPLETED'

    GROUP BY 

        r.region_name,

        DATE_TRUNC('month', s.sale_date)

),

performance_metrics AS (

    -- Calculate month-over-month growth rates

    SELECT 

        region_name,

        month_year,

        total_revenue,

        LAG(total_revenue) OVER (

            PARTITION BY region_name 

            ORDER BY month_year

        ) AS prev_month_revenue,

        ROUND(

            (total_revenue - LAG(total_revenue) OVER (

                PARTITION BY region_name 

                ORDER BY month_year

            )) * 100# SQL for Lead Data Engineer - Interview Guide

  

## Table of Contents

  

### 1. Advanced Query Optimization

- [1.1 Execution Plans and Performance Tuning](#11-execution-plans-and-performance-tuning)

- [1.2 Index Design and Strategy](#12-index-design-and-strategy)

- [1.3 Query Rewriting Techniques](#13-query-rewriting-techniques)

  

### 2. Complex SQL Patterns

- [2.1 Window Functions and Analytics](#21-window-functions-and-analytics)

- [2.2 CTEs and Recursive Queries](#22-ctes-and-recursive-queries)

- [2.3 Advanced Joins and Subqueries](#23-advanced-joins-and-subqueries)

  

### 3. Data Modeling and Schema Design

- [3.1 Dimensional Modelling](#31-dimensional-modeling)

- [3.2 Slowly Changing Dimensions](#32-slowly-changing-dimensions)

- [3.3 Schema Evolution Strategies](#33-schema-evolution-strategies)

  

### 4. ETL and Data Pipeline Patterns

- [4.1 Incremental Loading Strategies](#41-incremental-loading-strategies)

- [4.2 Change Data Capture (CDC)](#42-change-data-capture-cdc)

- [4.3 Data Quality and Validation](#43-data-quality-and-validation)

  

### 5. Modern SQL Platforms

- [5.1 Cloud Data Warehouses](#51-cloud-data-warehouses)

- [5.2 Columnar vs Row-Based Storage](#52-columnar-vs-row-based-storage)

  

### 6. Production and Leadership

- [6.1 Performance Monitoring and Troubleshooting](#61-performance-monitoring-and-troubleshooting)

- [6.2 SQL Code Standards and Reviews](#62-sql-code-standards-and-reviews)

- [6.3 Team Mentoring and Best Practices](#63-team-mentoring-and-best-practices)

  

---

  

## Key Interview Focus Areas

  

### Technical Questions

- Optimizing slow-running queries in production

- Designing scalable data models

- Implementing efficient ETL patterns

- Platform-specific SQL optimizations

  

### Leadership Questions  

- Establishing team SQL standards

- Code review processes

- Mentoring junior engineers

- Technology decision making

  

### Problem-Solving Scenarios

- Diagnosing production performance issues

- Handling schema migrations

- Data quality incident response

- Capacity planning and scaling

  

---

*Focused on practical skills and leadership scenarios for lead data engineer interviews*

---

## Real-World Supplement: Production SQL Patterns

### CASE-Based Classification Tiers

Production monitoring views use CASE expressions to classify data into action categories:

```sql
SELECT
  warehouse_name,
  daily_credits,
  CASE
    WHEN daily_credits > 100 THEN 'HIGH_USAGE'
    WHEN daily_credits > 50  THEN 'MEDIUM_USAGE'
    WHEN daily_credits > 10  THEN 'LOW_USAGE'
    ELSE 'MINIMAL_USAGE'
  END AS usage_category,
  CASE
    WHEN projected_cost > 10000 THEN 'OVER_BUDGET_RISK'
    WHEN projected_cost > 5000  THEN 'BUDGET_WARNING'
    ELSE 'WITHIN_BUDGET'
  END AS budget_status
FROM warehouse_metrics
```

### Safe Division with NULLIF

Avoid divide-by-zero in KPI calculations:

```sql
-- Profit margin
ROUND((revenue - total_cost) / NULLIF(revenue, 0) * 100, 2) AS profit_margin_pct

-- Cost per km
total_cost / NULLIF(distance_km, 0) AS cost_per_km

-- Data freshness percentage
ROUND(fresh_tables_24h / NULLIF(total_tables, 0) * 100, 2) AS freshness_pct
```

### MD5 Surrogate Key Generation

Generate deterministic keys from composite natural keys:

```sql
MD5(CONCAT(
  COALESCE(CAST(origin_location_id AS VARCHAR), ''),
  '_',
  COALESCE(CAST(destination_location_id AS VARCHAR), '')
)) AS route_id
```

### LISTAGG for Summary Reports

```sql
SELECT LISTAGG(TABLE_SCHEMA || ': ' || TABLE_COUNT || ' objects', ', ')
FROM (
  SELECT TABLE_SCHEMA, COUNT(*) AS TABLE_COUNT
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
  GROUP BY TABLE_SCHEMA
  ORDER BY TABLE_COUNT DESC
  LIMIT 5
) AS top_schemas_by_objects
```

### DATE_TRUNC and DATEDIFF Patterns

```sql
-- Monthly cost projection
DATE_TRUNC('month', CURRENT_DATE()) AS forecast_month

-- Data freshness check
CASE
  WHEN LAST_ALTERED >= CURRENT_TIMESTAMP() - INTERVAL '24 hours' THEN 'FRESH'
  WHEN LAST_ALTERED >= CURRENT_TIMESTAMP() - INTERVAL '7 days' THEN 'STALE'
  ELSE 'VERY_STALE'
END AS data_freshness

-- Delivery delay calculation
DATEDIFF(day, planned_delivery_date, actual_delivery_date) AS delay_days
```

**