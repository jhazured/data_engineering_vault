# SQL Stored Procedures

Server-side programs that encapsulate SQL logic, control flow, and error handling — executed within the database engine.

## Snowflake Stored Procedures

### JavaScript (Classic)

```sql
CREATE OR REPLACE PROCEDURE check_data_freshness()
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS
$$
    var result = snowflake.execute({
        sqlText: `SELECT COUNT(*) AS stale_count
                  FROM INFORMATION_SCHEMA.TABLES
                  WHERE LAST_ALTERED < DATEADD(hour, -24, CURRENT_TIMESTAMP())`
    });
    result.next();
    var stale = result.getColumnValue(1);
    return stale > 0 ? 'WARNING: ' + stale + ' stale tables' : 'OK: all tables fresh';
$$;

CALL check_data_freshness();
```

### SQL Scripting (Snowflake)

```sql
CREATE OR REPLACE PROCEDURE refresh_staging(table_name VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
BEGIN
    -- Validate input (prevent SQL injection)
    IF (table_name NOT REGEXP '^[A-Za-z_][A-Za-z0-9_]*$') THEN
        RETURN 'ERROR: Invalid table name';
    END IF;

    -- Truncate and reload
    EXECUTE IMMEDIATE 'TRUNCATE TABLE staging.' || :table_name;
    EXECUTE IMMEDIATE 'INSERT INTO staging.' || :table_name ||
                      ' SELECT * FROM raw.' || :table_name;

    RETURN 'OK: ' || :table_name || ' refreshed';
EXCEPTION
    WHEN OTHER THEN
        RETURN 'ERROR: ' || SQLERRM;
END;
$$;
```

### Python (Snowpark)

```sql
CREATE OR REPLACE PROCEDURE generate_quality_report()
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas')
HANDLER = 'main'
AS
$$
import pandas as pd

def main(session):
    df = session.sql("""
        SELECT table_name, row_count, last_altered
        FROM information_schema.tables
        WHERE table_schema = 'LOGISTICS'
    """).to_pandas()

    stale = df[df['LAST_ALTERED'] < pd.Timestamp.now() - pd.Timedelta(hours=24)]
    return f"OK: {len(stale)} stale tables out of {len(df)}"
$$;
```

## Use Cases in Data Engineering

### Scheduled Data Quality Checks

```sql
CREATE OR REPLACE PROCEDURE run_quality_checks()
RETURNS TABLE(check_name VARCHAR, status VARCHAR, details VARCHAR)
LANGUAGE SQL
AS
$$
DECLARE
    results RESULTSET;
BEGIN
    results := (
        SELECT 'null_check' AS check_name,
               CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END AS status,
               COUNT(*) || ' null customer_ids' AS details
        FROM fact_shipments WHERE customer_id IS NULL

        UNION ALL

        SELECT 'freshness_check',
               CASE WHEN MAX(shipment_date) >= CURRENT_DATE() - 2 THEN 'PASS' ELSE 'FAIL' END,
               'Latest: ' || MAX(shipment_date)
        FROM fact_shipments
    );
    RETURN TABLE(results);
END;
$$;
```

### Email Alert Integration

```sql
CREATE OR REPLACE PROCEDURE send_alert_if_stale()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    stale_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO stale_count
    FROM information_schema.tables
    WHERE table_schema = 'LOGISTICS'
      AND last_altered < DATEADD(hour, -24, CURRENT_TIMESTAMP());

    IF (stale_count > 0) THEN
        CALL SYSTEM$SEND_EMAIL(
            'email_integration',
            'oncall@example.com',
            'Data Freshness Alert',
            stale_count || ' tables are stale (>24h since update)'
        );
        RETURN 'ALERT SENT: ' || stale_count || ' stale tables';
    END IF;

    RETURN 'OK: all tables fresh';
END;
$$;
```

### Pipeline Orchestration

```sql
CREATE OR REPLACE PROCEDURE run_daily_pipeline()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    -- Phase 1: Staging
    EXECUTE IMMEDIATE 'CALL refresh_staging(''customers'')';
    EXECUTE IMMEDIATE 'CALL refresh_staging(''shipments'')';

    -- Phase 2: Dimensions
    EXECUTE IMMEDIATE 'INSERT INTO dim_customer SELECT ... FROM staging.customers';

    -- Phase 3: Facts
    EXECUTE IMMEDIATE 'INSERT INTO fact_shipments SELECT ... FROM staging.shipments';

    -- Phase 4: Quality checks
    CALL run_quality_checks();

    RETURN 'Pipeline complete';
EXCEPTION
    WHEN OTHER THEN
        -- Log error and re-raise
        INSERT INTO pipeline_errors (error_message, occurred_at)
        VALUES (SQLERRM, CURRENT_TIMESTAMP());
        RAISE;
END;
$$;
```

## Scheduling with Snowflake Tasks

```sql
-- Create a task that calls the procedure on a schedule
CREATE OR REPLACE TASK daily_pipeline_task
    WAREHOUSE = TRN_PROD_CENTRAL_WH
    SCHEDULE = 'USING CRON 0 6 * * * Australia/Sydney'
AS
    CALL run_daily_pipeline();

-- Tasks are created suspended — must be resumed
ALTER TASK daily_pipeline_task RESUME;

-- Check task history
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
ORDER BY SCHEDULED_TIME DESC LIMIT 10;
```

## Security Considerations

### EXECUTE AS CALLER vs OWNER

| Mode | Runs As | Use When |
|------|---------|----------|
| `EXECUTE AS CALLER` | The calling user's privileges | General-purpose, user needs own access |
| `EXECUTE AS OWNER` | The procedure owner's privileges | Elevated access (e.g., procedure reads tables the caller can't) |

### Input Validation

Always validate dynamic identifiers to prevent SQL injection:

```sql
-- Validate table names
IF (table_name NOT REGEXP '^[A-Za-z_][A-Za-z0-9_]*$') THEN
    RETURN 'ERROR: Invalid identifier';
END IF;

-- Use bind variables for values (not string concatenation)
EXECUTE IMMEDIATE 'SELECT * FROM orders WHERE id = ?' USING (order_id);
```

## Stored Procedures vs dbt

| Aspect | Stored Procedures | dbt |
|--------|-------------------|-----|
| Version control | Harder (lives in database) | Native (SQL files in git) |
| Testing | Manual or custom | Built-in (unique, not_null, etc.) |
| Lineage | Manual documentation | Automatic DAG |
| Orchestration | Snowflake Tasks | CI/CD pipeline or Airflow |
| Best for | Alerts, email, admin tasks, Tasks | Data transformation, models |

**Recommendation:** Use dbt for transformation logic. Use stored procedures for operational tasks (alerts, cleanup, monitoring) that need to run inside Snowflake.
