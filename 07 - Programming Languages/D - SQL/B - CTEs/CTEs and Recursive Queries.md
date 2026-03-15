
 #SQL #CTE #RecursiveCTE #DataEngineering #SQLPatterns #HierarchicalData #TimeSeries #DataTransformation #QueryOptimization #ETL #BestPractices

## Common Table Expressions (CTEs)

A Common Table Expression (CTE) in SQL is a temporary result set that you can reference within a SELECT, INSERT, UPDATE, or DELETE statement. It's defined using the `WITH` keyword and can be used to make queries more readable and manageable.

### Key Points

- **Temporary Result Set:** Exists only during the execution of the query
- **Readability:** Improves readability of complex queries by breaking them into logical building blocks
- **Reusability:** Once defined, can be referred to multiple times within a query
- **Recursive CTEs:** Can reference itself, useful for hierarchical data
- **Chaining CTEs:** Multiple CTEs can be chained together in a single WITH clause

### Basic CTE Syntax

```sql
WITH CTE_name AS (
    -- Your query here
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
-- Main query using the CTE
SELECT *
FROM CTE_name;
```

---

## CTE Examples

### Example 1: Basic CTE

Get employees whose salary is above the average salary:

```sql
WITH AverageSalary AS (
    SELECT AVG(Salary) AS AvgSalary
    FROM Employees
)
SELECT Name, Salary
FROM Employees, AverageSalary
WHERE Employees.Salary > AverageSalary.AvgSalary;
```

**Explanation:** The CTE `AverageSalary` calculates the average salary, then the main query uses that result to filter employees whose salary is above the average.

### Example 2: Recursive CTE

Find all employees in the hierarchy of a specific manager:

```sql
WITH RECURSIVE EmployeeHierarchyCTE AS (
    -- Base case: Select the manager
    SELECT EmployeeID, Name, ManagerID
    FROM EmployeeHierarchy
    WHERE ManagerID = 1
    
    UNION ALL
    
    -- Recursive case: Select employees reporting to those already selected
    SELECT e.EmployeeID, e.Name, e.ManagerID
    FROM EmployeeHierarchy e
    INNER JOIN EmployeeHierarchyCTE cte
       ON e.ManagerID = cte.EmployeeID
)
SELECT * FROM EmployeeHierarchyCTE;
```

**Explanation:** The recursive CTE starts with the manager (base case) and recursively adds employees reporting to them, building the entire reporting hierarchy.

### Example 3: Chaining CTEs Together

Calculate average salary, filter employees, and rank them:

```sql
WITH AverageSalary AS (
    SELECT AVG(Salary) AS AvgSalary
    FROM Employees
),
FilteredEmployees AS (
    SELECT EmployeeID, Name, Salary
    FROM Employees
    WHERE Salary > (SELECT AvgSalary FROM AverageSalary)
),
RankedEmployees AS (
    SELECT EmployeeID, Name, Salary,
           RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM FilteredEmployees
)
SELECT * FROM RankedEmployees;
```

**Process Flow:**

1. **AverageSalary:** Calculate the average salary
2. **FilteredEmployees:** Select employees whose salary is above the average
3. **RankedEmployees:** Rank these employees by salary in descending order

Each CTE builds on the previous one, keeping the query modular and clean.

---

## When to Use CTEs

### Use Cases for Standard CTEs

|Use Case|Description|Benefits|
|---|---|---|
|**Simplifying Complex Queries**|Break complex subqueries or joins into understandable parts|Improved maintainability|
|**Improving Readability**|Organize and structure queries logically|Easier debugging and review|
|**Avoiding Redundancy**|Define logic once and reference multiple times|Reduced code duplication|
|**Step-by-Step Processing**|Build complex logic incrementally|Modular approach|

### Use Cases for Recursive CTEs

|Use Case|Description|Example|
|---|---|---|
|**Hierarchical Data**|Organizational structures, reporting chains|Employee management hierarchy|
|**Tree Structures**|Category trees, nested data|Product category hierarchies|
|**Graph Traversal**|Connected data relationships|Social network connections|
|**Bill of Materials**|Manufacturing component relationships|Product assembly structures|

### Use Cases for Chaining CTEs

|Use Case|Description|Benefits|
|---|---|---|
|**Building Step-by-Step Logic**|Series of operations like filtering, aggregating, ranking|Clear separation of concerns|
|**Modular Queries**|Multiple stages of data transformation|Easier to debug and maintain|
|**Complex Data Pipeline**|Multi-step data processing|Improved readability and testing|


 Q: When and how do you use CTEs effectively? Provide examples of recursive CTE patterns.

 Answer:

1. Complex Query Organization

**Purpose:** Break down complex logic into readable, modular steps.

```sql
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
```
 2. Recursive Hierarchical Queries

**Purpose:** Traverse hierarchical structures, such as employee-manager relationships.

```sql
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
    WHERE h.level < 10  -- Prevent infinite recursion
)
SELECT 
    employee_id,
    REPEAT('  ', level) || employee_name AS indented_name,
    level,
    hierarchy_path
FROM employee_hierarchy
ORDER BY hierarchy_path;
```
 3. Recursive Data Generation

**Purpose:** Generate sequences or fill gaps in datasets.

```sql

WITH RECURSIVE date_series AS (
    SELECT DATE('2024-01-01') AS date_value
    UNION ALL
    SELECT date_value + INTERVAL 1 DAY
    FROM date_series
    WHERE date_value < DATE('2024-12-31')
),
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
```