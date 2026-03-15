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

## Related

- [[SQL Stored Procedures]]
- [[SQL Query Optimization]]
- [[Dimensional Modelling (Kimball)]]
- [[Data Quality Frameworks]]
