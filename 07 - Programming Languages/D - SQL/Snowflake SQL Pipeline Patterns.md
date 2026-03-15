---
tags:
  - sql
  - snowflake
  - etl
  - data-engineering
created: 2026-03-15
---

# Snowflake SQL Pipeline Patterns

Patterns from a US flights ETL pipeline: S3 ingestion through bronze/silver/gold into a [[Dimensional Modelling (Kimball)|Kimball star schema]].

---

## 1. JavaScript Stored Procedures in Snowflake

See also [[SQL Stored Procedures]]. Snowflake procs use a JavaScript body with `snowflake.execute()` to run SQL.

```sql
CREATE OR REPLACE PROCEDURE "SP_LOAD_FLIGHTS_DATA_ETL"(
    FILE_PATH VARCHAR DEFAULT '@S3_FOLDER/flights.gz',
    BATCH_ID VARIANT DEFAULT NULL,      -- VARIANT accepts NULL naturally
    FORCE_RELOAD BOOLEAN DEFAULT FALSE
)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
try {
    var filename = FILE_PATH.split('/').pop();
    var rs = snowflake.execute({
        sqlText: "SELECT COALESCE(MAX(\"BATCH_ID\"), 0) + 1 FROM \"TBL_ETL_LOG\""
    });
    rs.next();
    var batch_id = rs.getColumnValue(1);
    snowflake.execute({
        sqlText: "INSERT INTO \"TBL_ETL_LOG\" VALUES (?, ?, ?)",
        binds: [batch_id, 'STARTED', 'ETL begun']
    });
    return 'Success: batch ' + batch_id;
} catch (err) { return 'Failed: ' + err.message; }
$$;
```

- `snowflake.execute()` returns a result set; `.next()` + `.getColumnValue(1)` to read rows
- Bind variables use `?` with `binds` array (SQL injection safe)
- Backtick template literals for multi-line SQL; quoted identifiers for case-sensitive names

---

## 2. Multi-Layer ETL in a Single Procedure

Five steps run sequentially, each in its own `try/catch` so failures log but don't halt downstream.

**Batch ID**: auto-incremented from `MAX(BATCH_ID) + 1` in the log table, or passed in as a parameter. **Idempotency**: each step checks whether work has already been done (e.g. `SOURCE_FILE` already loaded) and skips if so, unless `FORCE_RELOAD = TRUE`.

### Step Orchestration

| Step | Source | Target | Guard |
|------|--------|--------|-------|
| 1 Raw | S3 stage | `TBL_FLIGHTS_RAW` | File not loaded (or `FORCE_RELOAD`) |
| 2 Silver | `VW_FLIGHTS_SILVER` | `TBL_FLIGHTS_SILVER` | `NOT EXISTS` in silver |
| 3 Dims | `VW_*_STAGING` views | `DIM_*` tables | Staging count > 0 |
| 4 Fact | `VW_FACT_STAGING` | `FACT_FLIGHTS` | `NOT EXISTS` in fact |
| 5 Validate | `FACT_FLIGHTS` | `TBL_ETL_LOG` | Always; checks NULL FKs |

---

## 3. Complex Transformation Views

`VW_FLIGHTS_SILVER` uses a multi-CTE chain: **ranked_flights** (dedup + clean) -> **flights_with_derived** (business logic) -> final SELECT filtering on quality.

### ROW_NUMBER for Dedup + LAG for Change Detection

```sql
-- ranked_flights CTE: dedup + track changes
ROW_NUMBER() OVER (PARTITION BY r."TRANSACTIONID"
    ORDER BY r."UPDATED_AT" DESC, r."LOAD_BATCH_ID" DESC) as "RN",
LAG(r."UPDATED_AT") OVER (PARTITION BY r."TRANSACTIONID"
    ORDER BY r."UPDATED_AT") as "PREVIOUS_UPDATED_AT"

-- flights_with_derived CTE (WHERE RN=1):
CASE WHEN "PREVIOUS_UPDATED_AT" IS NULL THEN TRUE       -- new record
     WHEN "UPDATED_AT" != "PREVIOUS_UPDATED_AT" THEN TRUE  -- modified
     ELSE FALSE END as "HAS_CHANGED"
```

### TRY_CAST with Range Validation

Raw data is all VARCHAR. `TRY_CAST` returns NULL on failure; CASE guards enforce domain ranges:

```sql
CASE WHEN TRY_CAST(r."DEPDELAY" as NUMBER) BETWEEN -120 AND 1440
     THEN TRY_CAST(r."DEPDELAY" as NUMBER) ELSE 0
END as "DEPDELAY"
```

CASE-based categorisation (e.g. `DISTANCEGROUP` buckets, `DEPDELAYGT15` threshold flag) runs in the second CTE after dedup. See [[SQL Query Optimization]] for window function performance.

---

## 4. Staging View Pattern

Staging views are "delta detectors" -- they only return records not yet in the target dimension, making repeated `INSERT INTO dim SELECT FROM staging` idempotent.

### Airline (Simple Anti-Join)

```sql
CREATE OR REPLACE VIEW "VW_AIRLINE_STAGING" AS
SELECT DISTINCT s."AIRLINECODE", s."AIRLINENAME", s."AIRLINENAME_CLEAN"
FROM "TBL_FLIGHTS_SILVER" s
WHERE s."AIRLINECODE" IS NOT NULL AND s."DATA_QUALITY_FLAG" = TRUE
    AND NOT EXISTS (SELECT 1 FROM "DIM_AIRLINE" d WHERE d."AIRLINECODE" = s."AIRLINECODE");
```

### Airport (UNION for Origin + Destination)

```sql
CREATE OR REPLACE VIEW "VW_AIRPORT_STAGING" AS
SELECT DISTINCT s1."ORIGINAIRPORTCODE" as "AIRPORTCODE", s1."ORIGAIRPORTNAME" as "AIRPORTNAME",
    s1."ORIGINCITYNAME" as "CITYNAME", s1."ORIGINSTATE" as "STATE"
FROM "TBL_FLIGHTS_SILVER" s1
WHERE NOT EXISTS (SELECT 1 FROM "DIM_AIRPORT" d WHERE d."AIRPORTCODE" = s1."ORIGINAIRPORTCODE")
UNION  -- not UNION ALL: deduplicates airports appearing as both origin and destination
SELECT DISTINCT s2."DESTAIRPORTCODE", s2."DESTAIRPORTNAME", s2."DESTCITYNAME", s2."DESTSTATE"
FROM "TBL_FLIGHTS_SILVER" s2
WHERE NOT EXISTS (SELECT 1 FROM "DIM_AIRPORT" d WHERE d."AIRPORTCODE" = s2."DESTAIRPORTCODE");
```

### Date (Derived Parts from Single Column)

```sql
CREATE OR REPLACE VIEW "VW_DATE_STAGING" AS
WITH new_dates AS (
    SELECT DISTINCT s."FLIGHTDATE" FROM "TBL_FLIGHTS_SILVER" s
    WHERE NOT EXISTS (SELECT 1 FROM "DIM_DATE" d WHERE d."FULL_DATE" = s."FLIGHTDATE")
)
SELECT TO_NUMBER(TO_CHAR(nd."FLIGHTDATE", 'YYYYMMDD')) as "DATE_KEY",
    nd."FLIGHTDATE" as "FULL_DATE", YEAR(nd."FLIGHTDATE") as "YEAR",
    MONTHNAME(nd."FLIGHTDATE") as "MONTH_NAME",
    CASE WHEN DAYOFWEEK(nd."FLIGHTDATE") IN (1,7) THEN TRUE ELSE FALSE END as "IS_WEEKEND"
FROM new_dates nd;
```

### Fact Staging (Surrogate Key Resolution)

```sql
CREATE OR REPLACE VIEW "VW_FACT_STAGING" AS
SELECT s."TRANSACTIONID", a."AIRLINE_KEY",
    oa."AIRPORT_KEY" as "ORIGIN_AIRPORT_KEY",
    da."AIRPORT_KEY" as "DEST_AIRPORT_KEY", dt."DATE_KEY"
FROM "TBL_FLIGHTS_SILVER" s
JOIN "DIM_AIRLINE" a ON s."AIRLINECODE" = a."AIRLINECODE"
JOIN "DIM_AIRPORT" oa ON s."ORIGINAIRPORTCODE" = oa."AIRPORTCODE"
JOIN "DIM_AIRPORT" da ON s."DESTAIRPORTCODE" = da."AIRPORTCODE"
JOIN "DIM_DATE" dt ON s."FLIGHTDATE" = dt."FULL_DATE"
WHERE s."DATA_QUALITY_FLAG" = TRUE
    AND NOT EXISTS (SELECT 1 FROM "FACT_FLIGHTS" f WHERE f."TRANSACTIONID" = s."TRANSACTIONID");
```

INNER JOINs are intentional -- missing dimensions exclude flights until next run.

---

## 5. Star Schema DDL

```sql
-- Dimension: AUTOINCREMENT surrogate key + UNIQUE on natural key
CREATE OR REPLACE TABLE "DIM_AIRLINE" (
    "AIRLINE_KEY" NUMBER AUTOINCREMENT PRIMARY KEY,
    "AIRLINECODE" VARCHAR(10) NOT NULL, "AIRLINENAME" VARCHAR(255),
    "CREATED_AT" TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE ("AIRLINECODE")  -- prevents duplicate dimension entries
);
-- Date dim uses meaningful integer key (YYYYMMDD), not autoincrement
CREATE OR REPLACE TABLE "DIM_DATE" (
    "DATE_KEY" NUMBER PRIMARY KEY, "FULL_DATE" DATE NOT NULL,
    "YEAR" NUMBER(4), "MONTH_NAME" VARCHAR(20), "IS_WEEKEND" BOOLEAN,
    UNIQUE ("FULL_DATE")
);
-- Fact: natural PK, surrogate FKs, measures, audit columns
CREATE OR REPLACE TABLE "FACT_FLIGHTS" (
    "TRANSACTIONID" VARCHAR PRIMARY KEY,
    "AIRLINE_KEY" NUMBER, "ORIGIN_AIRPORT_KEY" NUMBER,
    "DEST_AIRPORT_KEY" NUMBER, "DATE_KEY" NUMBER,  -- role-playing dim
    "DEPDELAY" NUMBER, "ARRDELAY" NUMBER, "DISTANCE" NUMBER,
    "CANCELLED" BOOLEAN DEFAULT FALSE,
    "LOAD_BATCH_ID" NUMBER, "SOURCE_FILE" VARCHAR
);
```

`DIM_AIRPORT` is joined twice in `VW_FACT_STAGING` (origin + destination) -- a role-playing dimension pattern. See [[Dimensional Modelling (Kimball)]].

---

## 6. Data Quality in SQL

### Boolean Standardisation

```sql
CASE
    WHEN UPPER(TRIM(r."CANCELLED")) IN ('TRUE', 'T', '1', 'YES') THEN TRUE
    WHEN UPPER(TRIM(r."CANCELLED")) IN ('FALSE', 'F', '0', 'NO') THEN FALSE
    ELSE FALSE
END as "CANCELLED"
```

### NULL_IF at File Format Level

```sql
CREATE OR REPLACE FILE FORMAT csv_pipe_header
    TYPE = 'CSV', FIELD_DELIMITER = '|', SKIP_HEADER = 1,
    NULL_IF = ('NULL', 'null', ''), EMPTY_FIELD_AS_NULL = TRUE,
    ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE;
```

### Composite Data Quality Flag

```sql
CASE
    WHEN r."TRANSACTIONID" IS NULL THEN FALSE
    WHEN TRY_TO_DATE(r."FLIGHTDATE", 'YYYYMMDD') IS NULL THEN FALSE
    WHEN r."AIRLINECODE" IS NULL OR TRIM(r."AIRLINECODE") = '' THEN FALSE
    WHEN r."ORIGINAIRPORTCODE" = r."DESTAIRPORTCODE" THEN FALSE
    ELSE TRUE
END as "DATA_QUALITY_FLAG"
```

### Distance Parsing (Mixed Formats)

Handles `"450"` and `"450miles"`, rejects out-of-range with nested `TRY_CAST` + `BETWEEN 1 AND 5000` guard. Strip suffix first (`REPLACE(r."DISTANCE",'miles','')`) then cast.

---

## 7. File Format and External Stages

```sql
COPY INTO "TBL_FLIGHTS_RAW" ("TRANSACTIONID", "FLIGHTDATE", ..., "LOAD_BATCH_ID", "SOURCE_FILE")
FROM (
    SELECT $1, $2, ..., $31,           -- positional column references
        1 as "LOAD_BATCH_ID",
        METADATA$FILENAME as "SOURCE_FILE"  -- pseudo-column for lineage
    FROM @RECRUITMENT_DB.PUBLIC.S3_FOLDER/flights.gz
)
FILE_FORMAT = csv_pipe_header;
```

- `.gz` auto-decompressed; raw table uses all-VARCHAR (casting deferred to silver)

---

## 8. Snowflake Tasks

```sql
CREATE OR REPLACE TASK "TASK_REFRESH_FLIGHTS_DATA_ETL"
    WAREHOUSE = 'WH_CANDIDATE_00262'
    SCHEDULE = 'USING CRON 0 6 * * * UTC'  -- Daily at 06:00 UTC
AS
CALL "SP_LOAD_FLIGHTS_DATA_ETL"();

ALTER TASK "TASK_REFRESH_FLIGHTS_DATA_ETL" RESUME;   -- activate (created suspended)
ALTER TASK "TASK_REFRESH_FLIGHTS_DATA_ETL" SUSPEND;  -- pause
EXECUTE TASK "TASK_REFRESH_FLIGHTS_DATA_ETL";        -- manual run
```

CRON: `minute hour day-of-month month day-of-week timezone`. The `WAREHOUSE` clause assigns compute; auto-starts and auto-suspends.

---

## 9. ETL Audit Logging

### Log Table

```sql
CREATE OR REPLACE TABLE "TBL_ETL_LOG" (
    "LOG_ID" NUMBER AUTOINCREMENT PRIMARY KEY,
    "BATCH_ID" NUMBER,
    "PROCESS_NAME" VARCHAR(100),   -- 'RAW_DATA_LOAD', 'SILVER_LOAD', 'DIM_LOAD', etc.
    "STATUS" VARCHAR(20),          -- 'STARTED', 'COMPLETED', 'SKIPPED', 'ERROR', 'WARNING'
    "MESSAGE" VARCHAR(1000),
    "RECORDS_PROCESSED" NUMBER DEFAULT 0,
    "CREATED_AT" TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP
);
```

### Logging Pattern

Every step follows: execute work inside `try`, log `COMPLETED` with record count on success, log `ERROR` with `err.message` on failure. Errors don't halt downstream -- each step has its own `try/catch`.

```javascript
try {
    // ... do work ...
    snowflake.execute({
        sqlText: `INSERT INTO "TBL_ETL_LOG" ("BATCH_ID","PROCESS_NAME","STATUS","MESSAGE","RECORDS_PROCESSED")
                  VALUES (?, 'SILVER_LOAD', 'COMPLETED', ?, ?)`,
        binds: [current_batch_id, 'Silver loaded', silver_count]
    });
} catch (err) {
    snowflake.execute({
        sqlText: `INSERT INTO "TBL_ETL_LOG" ("BATCH_ID","PROCESS_NAME","STATUS","MESSAGE")
                  VALUES (?, 'SILVER_LOAD', 'ERROR', ?)`,
        binds: [current_batch_id, 'Failed: ' + err.message]
    });
}
```

Query a batch: `SELECT * FROM "TBL_ETL_LOG" WHERE "BATCH_ID" = 42 ORDER BY "CREATED_AT"`.

---

## Advanced Snowflake SQL Patterns

Patterns from audit views, monitoring procedures, and SCIM token management across the logistics analytics platform.

### LATERAL FLATTEN for Unnesting VARIANT Arrays

`LATERAL FLATTEN` converts a VARIANT array column into individual rows -- essential for querying Snowflake's semi-structured metadata views like `ACCESS_HISTORY`.

```sql
-- Unnest DIRECT_OBJECTS_ACCESSED (a VARIANT array of JSON objects)
-- into one row per accessed object per query.
select
    ah.query_id
    , ah.user_name
    , obj.value:objectName::varchar       as object_name
    , obj.value:objectDomain::varchar     as object_type
    , obj.value:operationType::varchar    as operation_type
    , obj.value:columns                   as columns_accessed_variant
from SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah
    , lateral flatten(input => ah.direct_objects_accessed) obj
where
    ah.query_start_time >= dateadd(day, -30, current_timestamp())
```

- The `obj` alias represents each element of the flattened array
- Comma-join syntax (`, lateral flatten(...)`) is equivalent to `CROSS JOIN LATERAL`
- `input =>` is required; the argument must be a VARIANT, OBJECT, or ARRAY column

### JSON Column Access Patterns

Extract typed values from VARIANT objects using colon notation with `::type` casting:

```sql
obj.value:objectName::varchar         -- extract string field
obj.value:objectDomain::varchar       -- another string field
obj.value:columns                     -- keep as raw VARIANT (nested array)
```

Key rules:

- Field names after `:` are **case-sensitive** (match the JSON key exactly)
- Cast with `::varchar`, `::number`, `::boolean`, `::timestamp` etc.
- Omitting the cast returns VARIANT, which is useful when the value is itself a nested structure
- For filtering, the cast must appear in the `WHERE` clause too: `obj.value:objectName::varchar ilike '%PATTERN%'`

### TO_JSON for Searching Within VARIANT Arrays

When a VARIANT column holds an array of objects (e.g. `[{columnName: 'X'}, ...]`), `TO_JSON` serialises it to a string so you can search with `ILIKE`:

```sql
-- Check whether any classified column was accessed
-- columns_accessed_variant is an array: [{columnName: '...'}]
(
    to_json(columns_accessed_variant) ilike '%CUSTOMER_NAME%'
    or to_json(columns_accessed_variant) ilike '%CONTACT_EMAIL%'
    or to_json(columns_accessed_variant) ilike '%CONTACT_PHONE%'
)                                               as accessed_classified_columns
```

This is a pragmatic alternative to a second `LATERAL FLATTEN` + filter when you only need a boolean "does it contain X?" check. Trade-off: `TO_JSON` serialises the entire array on every row, so it is less efficient than a targeted flatten for large arrays.

### QUALIFY Clause for Row Deduplication

`QUALIFY` filters on window function results without a wrapping CTE or subquery -- Snowflake-specific syntax (not ANSI SQL):

```sql
-- Keep only the most recent SCIM token per application
select
    SPLIT_PART(QUERY_TEXT, '''', 2)  as APP,
    ADD_MONTHS(START_TIME, 6)        as EXPIRES_ON,
    DATEDIFF('DAY', CURRENT_TIMESTAMP(), ADD_MONTHS(END_TIME, 6)) as EXPIRES_IN_DAYS
from SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
where
    EXECUTION_STATUS = 'SUCCESS'
    and QUERY_TEXT ilike 'SELECT%SYSTEM$GENERATE_SCIM_ACCESS_TOKEN%'
qualify ROW_NUMBER() over (partition by APP order by EXPIRES_ON desc) = 1
```

Contrast with the CTE approach already used in `VW_FLIGHTS_SILVER` (section 3 above), where dedup uses `WHERE RN = 1` in a second CTE. `QUALIFY` achieves the same in fewer lines:

```sql
-- Equivalent to: WITH ranked AS (... ROW_NUMBER() ... AS rn) SELECT ... WHERE rn = 1
select *
from source_table
qualify row_number() over (partition by natural_key order by updated_at desc) = 1
```

### OBJECT_CONSTRUCT for Building JSON Objects

`OBJECT_CONSTRUCT` creates a VARIANT object from key-value pairs -- useful for structured alert payloads and logging:

```sql
-- Build a structured alert payload for the monitoring system
OBJECT_CONSTRUCT(
    'check_time',        CURRENT_TIMESTAMP(),
    'stale_table_count', :stale_count,
    'sla_hours',         6
)

-- Cost monitoring variant
OBJECT_CONSTRUCT(
    'check_time',     CURRENT_TIMESTAMP(),
    'daily_credits',  :daily_credits,
    'threshold',      :threshold
)
```

- Accepts alternating key (string) / value (any type) arguments
- Output is VARIANT; store in VARIANT columns or pass to procedures
- Pairs with `TO_JSON()` when the object needs to be rendered as a string (e.g. in email bodies): `TO_JSON(ALERT_DATA)`

### PARSE_JSON for String-to-VARIANT Conversion

`PARSE_JSON` converts a JSON string literal into a VARIANT value -- the inverse of `TO_JSON`:

```sql
-- Convert a JSON string into a queryable VARIANT
select parse_json('{"name": "warehouse_01", "credits": 42.5}') as config;

-- Access fields from the result
select config:name::varchar, config:credits::number
from (select parse_json('{"name": "warehouse_01", "credits": 42.5}') as config);
```

Common use cases:

- Ingesting JSON payloads from external stages or APIs into VARIANT columns
- Building test fixtures in development queries
- Converting string parameters into structured objects inside stored procedures

### Execution Plan Reading Tips

Snowflake does not expose traditional `EXPLAIN` output. Use the **Query Profile** in the web UI instead.

| Technique | How |
|-----------|-----|
| Query Profile | Snowsight -> Query History -> select query -> Query Profile tab |
| `EXPLAIN` (limited) | `EXPLAIN USING TEXT SELECT ...` returns a simplified plan; no cost estimates |
| `SYSTEM$EXPLAIN_PLAN_JSON` | `SELECT SYSTEM$EXPLAIN_PLAN_JSON('SELECT ...')` returns the plan as JSON |
| Result scan | `SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))` to inspect metadata of the last query |

What to look for in the Query Profile:

- **Percentage of total time** per operator -- identifies the bottleneck
- **Bytes scanned vs. bytes sent** -- indicates partition pruning effectiveness
- **Spillage to local/remote storage** -- suggests the warehouse is undersized for that query
- **Exploding joins** -- row count increases dramatically at a join node

See [[SQL Query Optimization]] for general query tuning strategies.

---

## Related

- [[SQL Stored Procedures]]
- [[SQL Query Optimization]]
- [[Dimensional Modelling (Kimball)]]
- [[Data Quality Frameworks]]
