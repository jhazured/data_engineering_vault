
## About This Guide

This comprehensive study guide covers all topics for the Snowflake DEA-C02 certification exam. It includes:

- **Complete exam domain coverage** aligned with official exam guide
- **Hands-on examples** with real SQL code
- **Decision frameworks** for choosing between features
- **Exam tips** highlighting commonly tested concepts
- **Practice questions** for self-assessment

### Exam Overview

|Attribute|Details|
|---|---|
|Exam Code|DEA-C02|
|Duration|115 minutes|
|Questions|65-75 multiple choice/select|
|Passing Score|750/1000|
|Prerequisites|SnowPro Core Certification (recommended)|
|Cost|$375 USD|
|Validity|2 years|

### Exam Domains

|Domain|Weight|Topics|
|---|---|---|
|Data Movement|25%|COPY INTO, Snowpipe, Kafka, stages, file formats|
|Data Transformation|30%|Streams, Tasks, Snowpark, UDFs, stored procedures, Dynamic Tables|
|Performance Optimization|25%|Query Profile, clustering, warehouses, Search Optimization|
|Security & Governance|20%|RBAC, masking, row access, data sharing|

---

# WEEK 1: Data Movement & Ingestion

## 1.1 Snowflake Architecture Fundamentals

### Three-Layer Architecture

Understanding Snowflake's architecture is foundational for the exam.

```
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD SERVICES LAYER                      │
│  • Authentication & Access Control    • Query Optimization   │
│  • Metadata Management               • Transaction Management│
│  • Infrastructure Management         • Security              │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                 QUERY PROCESSING LAYER                       │
│         (Virtual Warehouses - Independent Compute)           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │   WH1   │  │   WH2   │  │   WH3   │  │   WH4   │        │
│  │ (ETL)   │  │  (BI)   │  │ (Adhoc) │  │  (Dev)  │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                   DATABASE STORAGE LAYER                     │
│              (Centralized, Columnar Storage)                 │
│     ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐          │
│     │  MP  │ │  MP  │ │  MP  │ │  MP  │ │  MP  │          │
│     └──────┘ └──────┘ └──────┘ └──────┘ └──────┘          │
│              (Micro-partitions: 50-500MB each)              │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Concepts

**Separation of Storage and Compute:**

- Storage and compute scale independently
- Pay separately for each
- Multiple warehouses can access same data simultaneously
- No resource contention between workloads

**Micro-partitions:**

- Immutable, compressed columnar storage units
- Size: 50-500 MB uncompressed (~16MB compressed)
- Automatic partitioning (no manual work)
- Store metadata: min/max values, null counts, distinct counts
- Enable partition pruning for query optimization

> **EXAM TIP:** Know that micro-partitions are immutable. Updates create new micro-partitions; old ones are retained for Time Travel.

### Virtual Warehouse Sizes

|Size|Servers|Credits/Hour|Relative Power|
|---|---|---|---|
|X-Small|1|1|1x|
|Small|2|2|2x|
|Medium|4|4|4x|
|Large|8|8|8x|
|X-Large|16|16|16x|
|2X-Large|32|32|32x|
|3X-Large|64|64|64x|
|4X-Large|128|128|128x|
|5X-Large|256|256|256x|
|6X-Large|512|512|512x|

> **EXAM TIP:** Each size increase doubles both compute power AND cost. Doubling size does NOT guarantee 2x performance improvement.

---

## 1.2 File Formats

### Supported File Formats

|Format|Type|Load|Unload|Best Use Case|
|---|---|---|---|---|
|CSV|Structured|✅|✅|Most common, structured data|
|JSON|Semi-structured|✅|✅|APIs, logs, IoT, nested data|
|Parquet|Columnar|✅|✅|Data lakes, analytics, schema evolution|
|Avro|Row-based|✅|❌|Streaming (Kafka), schema evolution|
|ORC|Columnar|✅|❌|Hadoop ecosystem, Hive|
|XML|Hierarchical|✅|❌|Legacy systems, enterprise data|

> **EXAM TIP:** Remember which formats support UNLOAD: Only CSV, JSON, and Parquet. Avro, ORC, and XML are load-only.

### Creating File Formats

**CSV File Format (Most Common):**

```sql
CREATE OR REPLACE FILE FORMAT my_csv_format
    TYPE = CSV
    FIELD_DELIMITER = ','
    RECORD_DELIMITER = '\n'
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    NULL_IF = ('NULL', 'null', '', 'NA', 'N/A')
    EMPTY_FIELD_AS_NULL = TRUE
    TRIM_SPACE = TRUE
    ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE
    COMPRESSION = AUTO
    DATE_FORMAT = 'YYYY-MM-DD'
    TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS';
```

**Key CSV Options:**

|Option|Description|Default|Exam Notes|
|---|---|---|---|
|COMPRESSION|AUTO, GZIP, BZ2, BROTLI, ZSTD, DEFLATE, RAW_DEFLATE, NONE|AUTO|AUTO detects automatically|
|SKIP_HEADER|Lines to skip|0|Use 1 for files with headers|
|FIELD_OPTIONALLY_ENCLOSED_BY|Quote character|NONE|Use '"' or "'"|
|NULL_IF|Strings treated as NULL|\N|Can specify multiple values|
|ERROR_ON_COLUMN_COUNT_MISMATCH|Fail if columns don't match|TRUE|Set FALSE for flexible loading|
|ESCAPE|Escape character for enclosed fields|NONE|Usually \|
|ESCAPE_UNENCLOSED_FIELD|Escape for unquoted fields|\|Different from ESCAPE|

**JSON File Format:**

```sql
CREATE OR REPLACE FILE FORMAT my_json_format
    TYPE = JSON
    COMPRESSION = AUTO
    STRIP_OUTER_ARRAY = TRUE      -- Remove [ ] wrapper
    STRIP_NULL_VALUES = TRUE      -- Remove null key-value pairs
    ALLOW_DUPLICATE = FALSE       -- Error on duplicate keys
    IGNORE_UTF8_ERRORS = FALSE;
```

> **EXAM TIP:** STRIP_OUTER_ARRAY = TRUE is commonly needed when JSON files contain arrays like `[{...}, {...}, {...}]`

**Parquet File Format:**

```sql
CREATE OR REPLACE FILE FORMAT my_parquet_format
    TYPE = PARQUET
    COMPRESSION = AUTO            -- AUTO, SNAPPY, LZO, NONE
    BINARY_AS_TEXT = TRUE         -- Convert binary to text
    USE_LOGICAL_TYPE = TRUE;      -- Use logical types (dates, decimals)
```

> **EXAM TIP:** Parquet is self-describing (schema embedded). USE_LOGICAL_TYPE = TRUE preserves original data types.

**Avro File Format:**

```sql
CREATE OR REPLACE FILE FORMAT my_avro_format
    TYPE = AVRO
    COMPRESSION = AUTO;           -- AUTO, DEFLATE, SNAPPY, ZSTD, NONE
```

> **EXAM TIP:** Avro has schema embedded in file. Cannot unload to Avro. Commonly used with Kafka.

---

## 1.3 Snowflake Stages

### Stage Types Overview

```
Snowflake Stages
├── Internal Stages (within Snowflake)
│   ├── User Stage (@~)           -- Personal, auto-created
│   ├── Table Stage (@%table)     -- Per-table, auto-created
│   └── Named Internal Stage      -- Created manually, most flexible
│
└── External Stages (cloud storage)
    ├── Amazon S3
    ├── Google Cloud Storage (GCS)
    └── Microsoft Azure Blob Storage
```

### Internal Stages Comparison

|Feature|User Stage (@~)|Table Stage (@%table)|Named Internal|
|---|---|---|---|
|Auto-created|✅ Yes|✅ Yes|❌ No|
|Can be dropped|❌ No|❌ No|✅ Yes|
|Multi-user access|❌ No (private)|✅ Yes|✅ Yes|
|Load to multiple tables|✅ Yes|❌ No (that table only)|✅ Yes|
|Set file format|❌ No|❌ No|✅ Yes|
|Reference|@~|@%table_name|@stage_name|

> **EXAM TIP:** User stage (@~) is private to each user. Table stage can only load to its associated table. Iceberg tables do NOT support table stages.

### External Stages

**Method 1: Storage Integration (Recommended for Production)**

```sql
-- Step 1: Create storage integration (requires ACCOUNTADMIN)
CREATE STORAGE INTEGRATION s3_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'S3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake_role'
    STORAGE_ALLOWED_LOCATIONS = ('s3://bucket/path1/', 's3://bucket/path2/')
    STORAGE_BLOCKED_LOCATIONS = ('s3://bucket/sensitive/');

-- Step 2: Get Snowflake's AWS IAM user ARN
DESC INTEGRATION s3_integration;
-- Note: STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID

-- Step 3: Update AWS IAM role trust policy (in AWS Console)

-- Step 4: Create external stage
CREATE STAGE my_s3_stage
    URL = 's3://my-bucket/data/'
    STORAGE_INTEGRATION = s3_integration
    FILE_FORMAT = my_csv_format;
```

> **EXAM TIP:** Always prefer Storage Integrations over direct credentials. They're more secure and don't expose keys.

### Querying Staged Data

```sql
-- Query CSV data in stage (without loading)
SELECT $1, $2, $3
FROM @my_stage/file.csv
(FILE_FORMAT => my_csv_format);

-- Query with metadata columns
SELECT 
    METADATA$FILENAME,           -- Source file name
    METADATA$FILE_ROW_NUMBER,    -- Row number in file
    METADATA$FILE_CONTENT_KEY,   -- Content hash
    METADATA$FILE_LAST_MODIFIED, -- Last modified timestamp
    $1, $2, $3
FROM @my_stage/
(FILE_FORMAT => my_csv_format);
```

> **EXAM TIP:** METADATA$ columns are available during COPY INTO and when querying staged files directly.

---

## 1.4 Data Loading with COPY INTO

### Basic Syntax

```sql
COPY INTO <table_name>
FROM { <stage> | <external_location> }
[ FILES = ('<file1>', '<file2>', ...) ]
[ PATTERN = '<regex>' ]
[ FILE_FORMAT = ( FORMAT_NAME = '<n>' | TYPE = <type> [ options ] ) ]
[ copyOptions ]
[ VALIDATION_MODE = RETURN_<n>_ROWS | RETURN_ERRORS | RETURN_ALL_ERRORS ];
```

### Copy Options

|Option|Values|Default|Description|
|---|---|---|---|
|ON_ERROR|CONTINUE, SKIP_FILE, SKIP_FILE_<n>, ABORT_STATEMENT|ABORT_STATEMENT|Error handling|
|SIZE_LIMIT|Number (bytes)|NULL|Max data size to load|
|PURGE|TRUE/FALSE|FALSE|Delete files after load|
|RETURN_FAILED_ONLY|TRUE/FALSE|FALSE|Only return failed files|
|MATCH_BY_COLUMN_NAME|CASE_SENSITIVE, CASE_INSENSITIVE, NONE|NONE|Match by name vs position|
|ENFORCE_LENGTH|TRUE/FALSE|TRUE|Enforce VARCHAR length|
|TRUNCATECOLUMNS|TRUE/FALSE|FALSE|Truncate strings exceeding length|
|FORCE|TRUE/FALSE|FALSE|Reload previously loaded files|

### ON_ERROR Options Explained

```sql
-- ABORT_STATEMENT (default): Stop on first error, rollback entire load
COPY INTO employees FROM @my_stage ON_ERROR = ABORT_STATEMENT;

-- CONTINUE: Skip error rows, continue loading
COPY INTO employees FROM @my_stage ON_ERROR = CONTINUE;

-- SKIP_FILE: Skip entire file if ANY error
COPY INTO employees FROM @my_stage ON_ERROR = SKIP_FILE;

-- SKIP_FILE_<n>: Skip file after n errors
COPY INTO employees FROM @my_stage ON_ERROR = SKIP_FILE_10;

-- SKIP_FILE_<n>%: Skip file if error rate exceeds n%
COPY INTO employees FROM @my_stage ON_ERROR = 'SKIP_FILE_5%';
```

> **EXAM TIP:** ON_ERROR = CONTINUE is useful for data quality issues. Use SKIP_FILE for corrupt files.

### Validation Mode (Test Before Loading)

```sql
-- Preview first N rows (doesn't load)
COPY INTO employees FROM @my_stage
FILE_FORMAT = my_csv_format
VALIDATION_MODE = RETURN_10_ROWS;

-- Return only rows with errors
COPY INTO employees FROM @my_stage
FILE_FORMAT = my_csv_format
VALIDATION_MODE = RETURN_ERRORS;

-- Return ALL rows with errors
COPY INTO employees FROM @my_stage
FILE_FORMAT = my_csv_format
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

> **EXAM TIP:** Always use VALIDATION_MODE first to test data before actual load, especially with new file sources.

### Transformations During Load

```sql
-- Transform data while loading
COPY INTO employees (id, full_name, department, hire_date, salary_usd)
FROM (
    SELECT 
        $1::INT,                              -- Cast to INT
        UPPER(TRIM($2))::STRING,              -- Uppercase and trim
        $3::STRING,
        TO_DATE($4, 'MM/DD/YYYY'),            -- Parse date
        $5::DECIMAL(10,2) * 1.1               -- Calculate value
    FROM @my_stage
)
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);
```

### MATCH_BY_COLUMN_NAME

```sql
-- When file columns don't match table order
COPY INTO employees
FROM @my_stage
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 PARSE_HEADER = TRUE)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

> **EXAM TIP:** MATCH_BY_COLUMN_NAME requires PARSE_HEADER = TRUE to read column names from file.

### Load Metadata

**Preventing Duplicate Loads:**

- Snowflake tracks loaded files for **64 days**
- Stores: file path, name, size, last modified timestamp, ETag
- Same file won't reload unless FORCE = TRUE

```sql
-- Force reload (ignore load history)
COPY INTO employees FROM @my_stage
FILE_FORMAT = my_csv_format
FORCE = TRUE;
```

**Check Load History:**

```sql
-- Recent load history for table
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'EMPLOYEES',
    START_TIME => DATEADD('hours', -24, CURRENT_TIMESTAMP())
));
```

---

## 1.5 Snowpipe - Continuous Data Loading

### Overview

|Feature|COPY INTO|Snowpipe|
|---|---|---|
|Execution|Manual/scheduled|Automatic|
|Compute|User warehouse|Serverless|
|Latency|Minutes to hours|Seconds to minutes|
|Load size|Large batches|Micro-batches|
|Cost model|Warehouse credits|Per-file compute|
|Load metadata retention|**64 days**|**14 days**|

> **EXAM TIP:** Snowpipe metadata is retained for 14 days (vs 64 days for COPY INTO).

### Creating Pipes

**Pipe with AUTO_INGEST:**

```sql
CREATE PIPE my_auto_pipe
    AUTO_INGEST = TRUE
AS
COPY INTO employees
FROM @my_s3_stage
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1)
ON_ERROR = SKIP_FILE;
```

### Auto-Ingest Setup for AWS S3

```sql
-- Step 1: Create pipe with AUTO_INGEST
CREATE PIPE s3_pipe AUTO_INGEST = TRUE
AS COPY INTO my_table FROM @my_s3_stage FILE_FORMAT = my_format;

-- Step 2: Get notification channel (SQS ARN)
SHOW PIPES LIKE 's3_pipe';
-- Copy the NOTIFICATION_CHANNEL value

-- Step 3: Configure S3 event notification (in AWS Console)
-- Go to S3 bucket → Properties → Event notifications
-- Create notification → All object create events → SQS Queue
-- Enter the SQS ARN from step 2
```

### Managing Pipes

```sql
-- Pause pipe
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = TRUE;

-- Resume pipe
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = FALSE;

-- Refresh pipe (load files already in stage)
ALTER PIPE my_pipe REFRESH;

-- Refresh with prefix
ALTER PIPE my_pipe REFRESH PREFIX = 'path/to/files/';

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');
```

### Monitoring Snowpipe

```sql
-- Pipe usage history
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    DATE_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP()),
    PIPE_NAME => 'my_pipe'
));
```

---

## 1.6 Kafka Connector

### Architecture

```
Kafka Topic → Kafka Connect → Snowflake Connector → Internal Stage → Snowpipe → Table
```

### Table Schema for Kafka Data

```sql
CREATE TABLE kafka_events (
    record_metadata VARIANT,    -- Kafka metadata
    record_content VARIANT      -- Message payload
);
```

**Metadata in RECORD_METADATA:**

- `topic`: Kafka topic name
- `partition`: Partition number
- `offset`: Message offset
- `key`: Message key
- `timestamp`: Kafka timestamp
- `headers`: Kafka headers

### Buffer Settings

|Setting|Description|Default|
|---|---|---|
|buffer.count.records|Flush after N records|10000|
|buffer.flush.time|Flush after N seconds|120|
|buffer.size.bytes|Flush after N bytes|5000000|

> **EXAM TIP:** Buffer settings control the trade-off between latency and throughput. Lower values = lower latency but more Snowpipe overhead.

---

## 1.7 Week 1 Practice Questions

**Q1:** Which file formats can be used for both loading AND unloading data?

- A) CSV, JSON, Parquet, Avro
- B) CSV, JSON, Parquet ✓
- C) CSV, JSON, Parquet, ORC
- D) All formats support both

**Q2:** What is the default behavior of COPY INTO when an error occurs?

- A) Continue loading and skip error rows
- B) Skip the file with errors
- C) Abort the entire statement ✓
- D) Write errors to an error table

**Q3:** How long does Snowflake retain load metadata for COPY INTO vs Snowpipe?

- A) COPY INTO: 14 days, Snowpipe: 64 days
- B) COPY INTO: 64 days, Snowpipe: 14 days ✓
- C) Both: 64 days
- D) Both: 14 days

**Q4:** Which stage type can load data to multiple tables?

- A) User stage only
- B) Table stage only
- C) User stage and Named internal stage ✓
- D) All stage types

**Q5:** What must be configured for Snowpipe AUTO_INGEST with S3?

- A) AWS Lambda function
- B) S3 event notification pointing to Snowflake SQS queue ✓
- C) CloudWatch alarm
- D) AWS Step Functions

---

# WEEK 2: Data Transformation & Orchestration

## 2.1 Streams - Change Data Capture

### Stream Types

|Type|Tracks|Use Case|
|---|---|---|
|Standard (default)|INSERT, UPDATE, DELETE|Full CDC, data synchronization|
|Append-only|INSERT only|Log tables, event streams|
|Insert-only|INSERT only (explicit)|Strict insert-only scenarios|

> **EXAM TIP:** Append-only ignores UPDATE and DELETE. Insert-only only captures explicit INSERT statements (not MERGE inserts).

### Metadata Columns

|Column|Type|Description|
|---|---|---|
|METADATA$ACTION|VARCHAR|'INSERT' or 'DELETE'|
|METADATA$ISUPDATE|BOOLEAN|TRUE if row is part of UPDATE|
|METADATA$ROW_ID|VARCHAR|Unique row identifier|

**How UPDATE appears in stream:**

```
UPDATE → 1 DELETE row (old values) + 1 INSERT row (new values)
         Both have METADATA$ISUPDATE = TRUE
```

### Creating Streams

```sql
-- Standard stream
CREATE STREAM customer_stream ON TABLE customers;

-- Append-only stream
CREATE STREAM log_stream ON TABLE logs APPEND_ONLY = TRUE;

-- Stream with initial rows
CREATE STREAM product_stream ON TABLE products SHOW_INITIAL_ROWS = TRUE;

-- Stream on view
CREATE STREAM view_stream ON VIEW customer_summary;
```

### Consuming Streams

**CRITICAL:** SELECT alone does NOT consume the stream. Only DML operations (INSERT, UPDATE, DELETE, MERGE) within a transaction consume it.

```sql
-- Consume with MERGE (most common pattern)
MERGE INTO target_table t
USING source_stream s
ON t.id = s.id
WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' AND s.METADATA$ISUPDATE = FALSE 
    THEN DELETE
WHEN MATCHED AND s.METADATA$ACTION = 'INSERT' 
    THEN UPDATE SET t.name = s.name, t.value = s.value
WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' 
    THEN INSERT (id, name, value) VALUES (s.id, s.name, s.value);
```

### Stream Staleness

A stream becomes **stale** when:

- Source table's data retention period expires
- Table is dropped and recreated
- Significant table alterations occur

> **EXAM TIP:** Stream staleness is tied to table's DATA_RETENTION_TIME_IN_DAYS. Ensure retention period is longer than your processing interval.

---

## 2.2 Tasks - Workflow Orchestration

### Task Types

|Type|Compute|Use Case|
|---|---|---|
|User-managed|Specified warehouse|Predictable workloads|
|Serverless|Snowflake-managed|Variable workloads|

### Creating Tasks

```sql
-- User-managed task
CREATE TASK my_task
    WAREHOUSE = compute_wh
    SCHEDULE = '5 MINUTE'
AS
    INSERT INTO summary_table SELECT * FROM staging_table;

-- Serverless task
CREATE TASK serverless_task
    USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = 'SMALL'
    SCHEDULE = '5 MINUTE'
AS
    CALL process_data();
```

### Scheduling

**CRON expressions:**

```sql
SCHEDULE = 'USING CRON 0/5 * * * * UTC'   -- Every 5 minutes
SCHEDULE = 'USING CRON 0 * * * * UTC'     -- Every hour
SCHEDULE = 'USING CRON 0 2 * * * America/New_York'  -- Daily at 2 AM EST
SCHEDULE = 'USING CRON 0 9 * * MON UTC'   -- Every Monday at 9 AM
```

**CRON format:** `minute hour day_of_month month day_of_week timezone`

> **EXAM TIP:** Minimum schedule interval is 1 minute.

### Task Dependencies (DAGs)

```sql
-- Root task (has schedule)
CREATE TASK task_root
    WAREHOUSE = compute_wh
    SCHEDULE = 'USING CRON 0 * * * * UTC'
AS CALL ingest_data();

-- Child task (triggered by predecessor)
CREATE TASK task_child
    WAREHOUSE = compute_wh
    AFTER task_root
AS CALL transform_data();

-- Grandchild (waits for multiple predecessors)
CREATE TASK task_grandchild
    WAREHOUSE = compute_wh
    AFTER task_child1, task_child2
AS CALL aggregate_data();
```

**DAG Rules:**

- Maximum 1,000 tasks per DAG
- Maximum 100 predecessor tasks
- Only root task has SCHEDULE
- Child tasks use AFTER clause

### Conditional Execution (WHEN)

```sql
-- Only run if stream has data
CREATE TASK process_stream_task
    WAREHOUSE = compute_wh
    SCHEDULE = '1 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('my_stream')
AS
    MERGE INTO target USING my_stream source ON ...;
```

> **EXAM TIP:** WHEN clause is evaluated BEFORE task runs. If FALSE, task is SKIPPED (not failed). Child tasks still evaluate their own WHEN conditions.

### Managing Tasks

```sql
-- Tasks are created SUSPENDED by default!

-- Resume tasks (children first, then root)
ALTER TASK task_grandchild RESUME;
ALTER TASK task_child RESUME;
ALTER TASK task_root RESUME;

-- Or use system function
SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('task_root');

-- Execute immediately (for testing)
EXECUTE TASK my_task;
```

---

## 2.3 Stored Procedures

### Python Stored Procedures (Snowpark)

```sql
CREATE OR REPLACE PROCEDURE process_sales(start_date DATE, end_date DATE)
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas')
HANDLER = 'main'
AS
$$
from snowflake.snowpark.functions import col, sum as sum_

def main(session, start_date, end_date):
    sales_df = session.table("sales").filter(
        (col("sale_date") >= start_date) & (col("sale_date") <= end_date)
    )
    summary = sales_df.group_by("product_id").agg(
        sum_(col("amount")).alias("total_sales")
    )
    summary.write.mode("overwrite").save_as_table("sales_summary")
    return f"Processed {sales_df.count()} records"
$$;

CALL process_sales('2024-01-01', '2024-01-31');
```

### Execution Context

|Setting|Behavior|
|---|---|
|EXECUTE AS CALLER|Runs with caller's privileges|
|EXECUTE AS OWNER|Runs with owner's privileges (default)|

> **EXAM TIP:** EXECUTE AS OWNER allows users to perform operations they wouldn't normally have access to, through the procedure.

---

## 2.4 User-Defined Functions (UDFs)

### UDF vs Stored Procedure

|Feature|UDF|Stored Procedure|
|---|---|---|
|Return|Value or table|Any type|
|Execute SQL|❌ No|✅ Yes|
|Modify data|❌ No|✅ Yes|
|Session access|❌ No|✅ Yes|
|Use in SELECT|✅ Yes|❌ No|

### Scalar UDFs

```sql
CREATE OR REPLACE FUNCTION calculate_tax(amount FLOAT, rate FLOAT)
RETURNS FLOAT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
HANDLER = 'calc_tax'
AS
$$
def calc_tax(amount, rate):
    if amount is None or rate is None:
        return None
    return round(amount * rate / 100, 2)
$$;

SELECT order_id, subtotal, calculate_tax(subtotal, 8.5) as tax FROM orders;
```

### Tabular UDFs (UDTFs)

```sql
CREATE OR REPLACE FUNCTION parse_tags(tag_string VARCHAR)
RETURNS TABLE (tag VARCHAR, position INT)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
HANDLER = 'TagParser'
AS
$$
class TagParser:
    def process(self, tag_string):
        if tag_string:
            for i, tag in enumerate(tag_string.split(','), 1):
                yield (tag.strip(), i)
$$;

SELECT product_id, t.tag FROM products, LATERAL TABLE(parse_tags(products.tags)) t;
```

> **EXAM TIP:** Vectorized UDFs process data in batches using Pandas Series, providing significantly better performance for large datasets.

---

## 2.5 Dynamic Tables

### Overview

Dynamic Tables automatically materialize query results and keep them updated based on source changes.

### Key Characteristics

- **Declarative:** Define WHAT, not HOW to update
- **Automatic refresh:** Snowflake manages scheduling
- **Incremental processing:** Only processes changes when possible
- **Dependency-aware:** Tracks upstream dependencies automatically

### Creating Dynamic Tables

```sql
CREATE DYNAMIC TABLE customer_summary
    TARGET_LAG = '1 hour'
    WAREHOUSE = analytics_wh
AS
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(amount) as total_spent
    FROM orders
    GROUP BY customer_id;
```

### TARGET_LAG Options

|Setting|Description|
|---|---|
|`'5 minutes'`|Refresh to keep data within 5 minutes of source|
|`'1 hour'`|Refresh to keep data within 1 hour|
|`DOWNSTREAM`|Inherit lag from downstream tables|

> **EXAM TIP:** Smaller TARGET_LAG = more frequent refreshes = higher cost. Minimum is 1 minute.

### Refresh Modes

|Mode|Description|
|---|---|
|AUTO (default)|Snowflake chooses incremental or full|
|INCREMENTAL|Only process changes|
|FULL|Complete recomputation each refresh|

### Dynamic Tables vs Alternatives

|Feature|Dynamic Table|Materialized View|Streams + Tasks|
|---|---|---|---|
|Setup|Low (declarative)|Low|High (imperative)|
|Joins|✅ Yes|❌ No|✅ Yes|
|Query rewrite|❌ No|✅ Yes|❌ No|
|Complex logic|Limited|Very limited|Unlimited|

> **EXAM TIP:** Use Dynamic Tables for multi-step transformations. Use Materialized Views for simple aggregations with query rewrite. Use Streams + Tasks for complex custom logic.

---

## 2.6 Week 2 Practice Questions

**Q1:** How does an UPDATE appear in a standard stream?

- A) One row with METADATA$ACTION = 'UPDATE'
- B) Two rows: DELETE + INSERT, both with METADATA$ISUPDATE = TRUE ✓
- C) Two rows: UPDATE (old) and UPDATE (new)
- D) One row with old and new values

**Q2:** What happens when a stream is queried with SELECT?

- A) Stream offset advances
- B) Stream is consumed
- C) Nothing - only DML consumes streams ✓
- D) Stream becomes stale

**Q3:** In a task DAG, what order should tasks be resumed?

- A) Root first, then children
- B) Children first, then root ✓
- C) Any order
- D) Alphabetical order

**Q4:** What is the minimum TARGET_LAG for a Dynamic Table?

- A) 1 second
- B) 1 minute ✓
- C) 5 minutes
- D) No minimum

**Q5:** What's the key difference between UDFs and stored procedures?

- A) UDFs can execute SQL
- B) Procedures can be used in SELECT
- C) UDFs cannot execute SQL or modify data ✓
- D) No difference

---

# WEEK 3: Performance Optimization

## 3.1 Query Profile Analysis

### Key Metrics

|Metric|Description|What to Look For|
|---|---|---|
|Partitions Scanned/Total|Partition pruning efficiency|Low ratio = good pruning|
|Bytes Scanned|Data read from storage|Lower = better|
|Spillage to Local|Data spilled to local SSD|Some OK|
|Spillage to Remote|Data spilled to remote storage|**Always bad - resize WH**|

### Reading Query Profile

**Step 1: Check partition pruning**

```
Partitions Scanned: 1,523 / 1,523  -- ❌ BAD: No pruning
Partitions Scanned: 45 / 1,523     -- ✅ GOOD: 97% pruned
```

**Step 2: Check for spillage**

```
Bytes Spilled to Local: 0 GB       -- ✅ Perfect
Bytes Spilled to Remote: > 0      -- ❌ CRITICAL: Warehouse too small
```

> **EXAM TIP:** Remote spillage is the clearest signal that warehouse is undersized.

### Common Patterns

**Poor Partition Pruning:**

```sql
-- ❌ BAD: Function on filter column
WHERE TO_CHAR(order_date, 'YYYY-MM-DD') = '2024-01-15'

-- ✅ GOOD: Direct comparison
WHERE order_date = '2024-01-15'::DATE
```

---

## 3.2 Micro-Partitions and Clustering

### Micro-Partition Basics

- **Size:** 50-500 MB uncompressed (~16 MB compressed)
- **Format:** Columnar storage
- **Immutable:** Cannot be modified, only replaced
- **Metadata:** Min/max values, null counts, distinct counts

### When to Use Clustering

✅ **Use when ALL true:**

1. Table has 1+ TB of data
2. Queries consistently filter on same columns
3. Queries are selective
4. Table changes infrequently

❌ **Don't use when:**

- Table < 1 TB
- Frequent UPDATE/DELETE
- Random access patterns

### Creating Clustering Keys

```sql
-- Single column
ALTER TABLE sales CLUSTER BY (sale_date);

-- Multi-column (order matters!)
ALTER TABLE sales CLUSTER BY (region, sale_date);

-- Expression-based
ALTER TABLE sales CLUSTER BY (DATE_TRUNC('MONTH', sale_date));

-- On VARIANT columns
ALTER TABLE events CLUSTER BY (event_data:user_id::INT);
```

### Choosing Clustering Keys

**Cardinality guidelines:**

- Low cardinality (10-1000 values): ✅ Good
- Medium cardinality (1K-100K): ✅ Often works
- High cardinality (millions): ❌ Usually not effective

> **EXAM TIP:** Clustering key cardinality should be ≤ number of micro-partitions.

### Monitoring Clustering

```sql
-- Check clustering depth (lower is better)
SELECT SYSTEM$CLUSTERING_DEPTH('sales', '(sale_date)');
-- Depth 1: Perfect | 2-4: Good | >10: Poor
```

---

## 3.3 Warehouse Sizing and Scaling

### Scaling Up vs Scaling Out

|Aspect|Scale Up (Resize)|Scale Out (Multi-cluster)|
|---|---|---|
|Purpose|Faster individual queries|Handle more concurrent queries|
|When|Spillage, slow complex queries|Query queueing|
|How|Increase size|Add clusters|

### When to Scale Up

**Indicators warehouse is too small:**

1. Remote spillage in Query Profile
2. Long execution for compute-heavy operations

### Multi-cluster Configuration

```sql
ALTER WAREHOUSE bi_wh SET
    MIN_CLUSTER_COUNT = 2
    MAX_CLUSTER_COUNT = 10
    SCALING_POLICY = 'STANDARD';
```

### Scaling Policies

|Policy|Behavior|Best For|
|---|---|---|
|STANDARD|Starts clusters immediately|User-facing, BI|
|ECONOMY|Waits ~6 min before starting|Batch, cost-sensitive|

### Auto-Suspend

```sql
-- ETL: Short suspension
ALTER WAREHOUSE etl_wh SET AUTO_SUSPEND = 60;

-- BI: Longer (keep cache warm)
ALTER WAREHOUSE bi_wh SET AUTO_SUSPEND = 600;
```

> **EXAM TIP:** Warehouse startup is < 10 seconds. Always enable AUTO_RESUME.

---

## 3.4 Search Optimization Service (SOS)

### When to Use SOS

✅ **Use when:**

- Selective lookups returning few rows
- Equality predicates: `WHERE col = value`
- Substring searches: `WHERE col LIKE '%pattern%'`
- VARIANT field access

❌ **Don't use when:**

- Small tables (< 1 TB)
- Full table scans
- Frequently updated tables

**Available in:** Enterprise Edition or higher

### Enabling Search Optimization

```sql
-- On specific columns (recommended)
ALTER TABLE customers ADD SEARCH OPTIMIZATION 
    ON EQUALITY(customer_id, email);

-- For substring searches
ALTER TABLE logs ADD SEARCH OPTIMIZATION ON SUBSTRING(message);

-- For VARIANT fields
ALTER TABLE events ADD SEARCH OPTIMIZATION ON EQUALITY(event_data:user_id);
```

### SOS vs Clustering

|Aspect|Clustering|Search Optimization|
|---|---|---|
|Best for|Range queries|Point lookups|
|Cost|Moderate|Can be high|

**Use both together:**

```sql
ALTER TABLE orders CLUSTER BY (order_date);
ALTER TABLE orders ADD SEARCH OPTIMIZATION ON EQUALITY(order_id);
```

---

## 3.5 Query Optimization Techniques

### Key Techniques

1. **Select only needed columns**
2. **Filter early** (before joins)
3. **Avoid functions on filter columns**
4. **Use semi-joins (EXISTS) instead of full joins**
5. **Replace OR with UNION ALL**
6. **Use QUALIFY for window function filtering**
7. **Use approximate functions** (APPROX_COUNT_DISTINCT)

### QUALIFY Example

```sql
-- Instead of subquery:
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY date DESC) as rn
    FROM orders
) WHERE rn = 1;

-- Use QUALIFY:
SELECT * FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY date DESC) = 1;
```

---

## 3.6 Caching

### Cache Types

|Cache|Duration|Scope|
|---|---|---|
|Result Cache|24 hours|Per-user|
|Warehouse Cache|While running|Per-warehouse|
|Metadata Cache|Always|Account-wide|

### Result Cache

- Invalidated when data changes or 24 hours elapse
- Zero compute cost for cached results

```sql
-- Disable for testing
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

---

## 3.7 Resource Monitors

```sql
CREATE RESOURCE MONITOR monthly_budget
    CREDIT_QUOTA = 5000
    FREQUENCY = MONTHLY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO SUSPEND
        ON 100 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE compute_wh SET RESOURCE_MONITOR = monthly_budget;
```

> **EXAM TIP:** SUSPEND stops new queries (existing finish). SUSPEND_IMMEDIATE cancels running queries.

---

## 3.8 Week 3 Practice Questions

**Q1:** What does remote spillage in Query Profile indicate?

- A) Query using too much memory
- B) Warehouse is too small ✓
- C) Network latency
- D) Storage issue

**Q2:** What is the recommended clustering key cardinality?

- A) As high as possible
- B) Lower than or equal to number of micro-partitions ✓
- C) Exactly 1000
- D) Doesn't matter

**Q3:** When should you use Search Optimization Service?

- A) All tables over 100 GB
- B) Point lookups on large tables ✓
- C) Range queries
- D) Frequently updated tables

**Q4:** Difference between STANDARD and ECONOMY scaling policies?

- A) STANDARD is faster
- B) STANDARD starts clusters immediately, ECONOMY waits ~6 min ✓
- C) STANDARD costs more per credit
- D) ECONOMY doesn't support auto-scaling

**Q5:** How long does result cache last?

- A) 1 hour
- B) 24 hours ✓
- C) 7 days
- D) Until warehouse suspends

---

# WEEK 4: Security, Governance & Data Sharing

## 4.1 Role-Based Access Control (RBAC)

### System-Defined Roles

|Role|Purpose|Key Capabilities|
|---|---|---|
|ACCOUNTADMIN|Top-level admin|Account settings, billing|
|SECURITYADMIN|Security management|Create/manage users, roles, grants|
|USERADMIN|User management|Create users and roles only|
|SYSADMIN|Object management|Create databases, warehouses|
|PUBLIC|Default for all|Minimal privileges|

```
Role Hierarchy:
    ACCOUNTADMIN
         │
    SECURITYADMIN ──── USERADMIN
         │
      SYSADMIN
         │
    Custom Roles
         │
       PUBLIC
```

> **EXAM TIP:** ACCOUNTADMIN is NOT a superuser - it must still be granted privileges. Never use as default role.

### Granting Privileges

```sql
-- Future grants (auto-apply to new objects)
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.public TO ROLE analyst;
```

> **EXAM TIP:** FUTURE GRANTS automatically apply to objects created after the grant.

### Managed Access Schemas

```sql
CREATE SCHEMA finance.restricted WITH MANAGED ACCESS;
-- Only schema owner can grant privileges
-- Object owners CANNOT grant access
```

---

## 4.2 Dynamic Data Masking

### Creating Masking Policies

```sql
-- Full masking based on role
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('HR_ADMIN') THEN val
        ELSE '********'
    END;

-- Partial masking
CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('HR_ADMIN') THEN val
        WHEN CURRENT_ROLE() IN ('HR_VIEWER') THEN 'XXX-XX-' || RIGHT(val, 4)
        ELSE 'XXX-XX-XXXX'
    END;
```

### Applying Masking Policies

```sql
ALTER TABLE employees MODIFY COLUMN email SET MASKING POLICY email_mask;
```

### Tag-Based Masking

```sql
-- Create tag and associate policy
CREATE TAG pii_type ALLOWED_VALUES 'email', 'ssn';
ALTER TAG pii_type SET MASKING POLICY email_tag_mask;

-- Tag column (policy auto-applies)
ALTER TABLE employees MODIFY COLUMN email SET TAG pii_type = 'email';
```

> **EXAM TIP:** Tag-based masking allows centralized policy management.

---

## 4.3 Row Access Policies

```sql
CREATE ROW ACCESS POLICY region_filter AS (region STRING) RETURNS BOOLEAN ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN') THEN TRUE
        WHEN CURRENT_ROLE() = 'US_MANAGER' AND region = 'US' THEN TRUE
        ELSE FALSE
    END;

ALTER TABLE employees ADD ROW ACCESS POLICY region_filter ON (region);
```

> **EXAM TIP:** Same column CANNOT have both masking AND row access policy. Use different columns.

---

## 4.4 Secure Views and Data Sharing

### Secure Views

```sql
CREATE SECURE VIEW customer_public_view AS
    SELECT customer_id, company_name FROM customers WHERE status = 'ACTIVE';
```

|Feature|Standard View|Secure View|
|---|---|---|
|Definition visible|✅ Yes|❌ No|
|Use in sharing|❌ No|✅ Yes|

### Creating Shares

```sql
CREATE SHARE sales_share;
GRANT USAGE ON DATABASE sales_db TO SHARE sales_share;
GRANT SELECT ON VIEW sales_db.public.partner_view TO SHARE sales_share;
ALTER SHARE sales_share ADD ACCOUNTS = partner_account;
```

### Secure View for Account Filtering

```sql
CREATE SECURE VIEW shared_data AS
    SELECT * FROM base_table
    WHERE region = (
        SELECT region FROM account_mapping 
        WHERE account_locator = CURRENT_ACCOUNT()
    );
```

> **EXAM TIP:** CURRENT_ACCOUNT() returns the consumer's account identifier in shared views.

### Consuming Shared Data

```sql
CREATE DATABASE partner_data FROM SHARE provider_account.sales_share;
GRANT IMPORTED PRIVILEGES ON DATABASE partner_data TO ROLE analyst;
```

---

## 4.5 Data Governance Features

### Object Tagging

```sql
CREATE TAG data_sensitivity ALLOWED_VALUES 'PUBLIC', 'CONFIDENTIAL', 'RESTRICTED';
ALTER TABLE customers SET TAG data_sensitivity = 'CONFIDENTIAL';
```

### Access History

```sql
SELECT user_name, query_text, objects_accessed
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE query_start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP());
```

---

## 4.6 Security Best Practices

1. **Principle of Least Privilege** - Grant minimum necessary access
2. **Use Future Grants** - Auto-grant on new objects
3. **Network Policies** - Restrict access by IP
4. **MFA** - Require for sensitive users
5. **Separate Environments** - Different databases for prod/dev
6. **Regular Access Reviews**

---

## 4.7 Week 4 Practice Questions

**Q1:** Which system role should be used for creating databases?

- A) ACCOUNTADMIN
- B) SECURITYADMIN
- C) SYSADMIN ✓
- D) USERADMIN

**Q2:** Which function returns the consumer's account in a shared view?

- A) CURRENT_USER()
- B) CURRENT_ROLE()
- C) CURRENT_ACCOUNT() ✓
- D) CURRENT_REGION()

**Q3:** Can the same column have both masking AND row access policy?

- A) Yes
- B) No - use different columns ✓

**Q4:** What does FUTURE GRANTS do?

- A) Grants on existing objects
- B) Auto-grants on objects created after the grant ✓
- C) Schedules grants
- D) Temporary grants

---

# Additional Exam Topics

## Time Travel

```sql
-- Query historical data
SELECT * FROM my_table AT (TIMESTAMP => '2024-01-15 10:00:00');
SELECT * FROM my_table AT (OFFSET => -3600);  -- 1 hour ago

-- Restore table
CREATE TABLE restored CLONE my_table AT (TIMESTAMP => '2024-01-15');

-- Undrop
UNDROP TABLE my_table;
```

### Retention Periods

|Edition|Default|Maximum|
|---|---|---|
|Standard|1 day|1 day|
|Enterprise|1 day|90 days|
|Business Critical|1 day|90 days|

### Fail-safe

- **Duration:** 7 days after Time Travel
- **Access:** Snowflake support only
- **Purpose:** Disaster recovery

> **EXAM TIP:** Time Travel is user-accessible. Fail-safe requires Snowflake support.

---

## Snowflake Editions

|Feature|Standard|Enterprise|Business Critical|
|---|---|---|---|
|Time Travel|1 day|90 days|90 days|
|Multi-cluster WH|❌|✅|✅|
|Materialized Views|❌|✅|✅|
|Search Optimization|❌|✅|✅|
|Column-level Security|❌|✅|✅|
|Row Access Policies|❌|✅|✅|
|Failover/Failback|❌|❌|✅|

> **EXAM TIP:** Know which features require Enterprise or Business Critical.

---

# Quick Reference

## Key Numbers

|Item|Value|
|---|---|
|Micro-partition size|50-500 MB (uncompressed)|
|COPY load metadata retention|**64 days**|
|Snowpipe metadata retention|**14 days**|
|Result cache duration|**24 hours**|
|Time Travel (Standard)|1 day max|
|Time Travel (Enterprise+)|90 days max|
|Fail-safe duration|7 days|
|Minimum task schedule|1 minute|
|Minimum TARGET_LAG|1 minute|
|Max tasks per DAG|1,000|

## Key SQL Commands

```sql
-- Stages
CREATE STAGE name [URL=...] [STORAGE_INTEGRATION=...];
LIST @stage;

-- Loading
COPY INTO table FROM @stage FILE_FORMAT = format;
CREATE PIPE name AUTO_INGEST = TRUE AS COPY INTO...;

-- Streams
CREATE STREAM name ON TABLE table [APPEND_ONLY = TRUE];
SELECT SYSTEM$STREAM_HAS_DATA('stream');

-- Tasks
CREATE TASK name WAREHOUSE = wh SCHEDULE = '...' [WHEN ...] AS sql;
ALTER TASK name RESUME;
SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('root_task');

-- Dynamic Tables
CREATE DYNAMIC TABLE name TARGET_LAG = '...' WAREHOUSE = wh AS query;

-- Clustering
ALTER TABLE name CLUSTER BY (columns);
SELECT SYSTEM$CLUSTERING_DEPTH('table');

-- Search Optimization
ALTER TABLE name ADD SEARCH OPTIMIZATION ON EQUALITY(columns);

-- Warehouses
ALTER WAREHOUSE name SET WAREHOUSE_SIZE = 'SIZE';
ALTER WAREHOUSE name SET MIN_CLUSTER_COUNT = n MAX_CLUSTER_COUNT = n;

-- Security
CREATE MASKING POLICY name AS (val TYPE) RETURNS TYPE -> expression;
CREATE ROW ACCESS POLICY name AS (args) RETURNS BOOLEAN -> expression;
CREATE SHARE name;
GRANT USAGE ON DATABASE db TO SHARE share;
```

## Important Functions

```sql
-- Stream metadata
METADATA$ACTION, METADATA$ISUPDATE, METADATA$ROW_ID

-- Stage metadata
METADATA$FILENAME, METADATA$FILE_ROW_NUMBER

-- Context functions
CURRENT_ROLE(), CURRENT_USER(), CURRENT_ACCOUNT()

-- System functions
SYSTEM$STREAM_HAS_DATA('stream')
SYSTEM$CLUSTERING_DEPTH('table', '(columns)')
SYSTEM$PIPE_STATUS('pipe')
SYSTEM$TASK_DEPENDENTS_ENABLE('task')
```

---

# Exam Day Checklist

## Key Concepts to Review

- [ ] Query Profile metrics (spillage = undersized warehouse)
- [ ] When to use clustering vs SOS vs materialized views
- [ ] Stream metadata columns (UPDATE = DELETE + INSERT)
- [ ] Task DAG rules (resume children first)
- [ ] COPY INTO options (ON_ERROR, VALIDATION_MODE)
- [ ] Snowpipe AUTO_INGEST setup
- [ ] Masking policy syntax
- [ ] Row access policy limitations
- [ ] Data sharing with CURRENT_ACCOUNT()
- [ ] System-defined roles
- [ ] Edition requirements for features
- [ ] Retention periods (64d, 14d, 24h)

## Exam Tips

1. Read questions carefully - look for "best," "most efficient"
2. Eliminate wrong answers first
3. Think about scale - many questions involve large data
4. Consider cost implications
5. Watch for edition requirements
6. ~1.5 minutes per question
7. Flag and return to difficult questions

---

**Good luck with your SnowPro Advanced Data Engineer certification!**

---

## Real-World Supplement: Production Patterns

The following patterns come from a production Snowflake + dbt + Fivetran platform and illustrate exam concepts in practice.

### Resource Monitor Setup

```sql
CREATE OR REPLACE RESOURCE MONITOR RM_ACCOUNT
  WITH CREDIT_QUOTA = 435
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO NOTIFY
    ON 110 PERCENT DO SUSPEND;

ALTER ACCOUNT SET RESOURCE_MONITOR = RM_ACCOUNT;
```

### Warehouse Sizing Pattern

Production uses X-Small warehouses with aggressive auto-suspend, separated by workload:

```sql
-- Ingestion warehouse (Fivetran)
CREATE WAREHOUSE IN_PROD_CENTRAL_WH
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 120 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE;

-- Transformation warehouse (dbt)
CREATE WAREHOUSE TRN_PROD_CENTRAL_WH
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 120 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE;

-- Consumption warehouse (BI tools)
CREATE WAREHOUSE OUT_PROD_CENTRAL_WH
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE;
```

### ACCOUNT_USAGE Monitoring Views

Real monitoring views query `SNOWFLAKE.ACCOUNT_USAGE` for cost tracking, query analysis, and data freshness — see [[Snowflake Cost Monitoring]] for full examples.

**Key exam insight:** ACCOUNT_USAGE views have ingestion lag (45 min for QUERY_HISTORY, 3 hours for TABLES/ACCESS_HISTORY). Use INFORMATION_SCHEMA for near-real-time needs.

### LATERAL FLATTEN for VARIANT Data

ACCESS_HISTORY stores accessed objects as a VARIANT array — LATERAL FLATTEN unnests it:

```sql
SELECT
  ah.query_id,
  ah.user_name,
  obj.value:objectName::VARCHAR AS object_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
  LATERAL FLATTEN(input => ah.direct_objects_accessed) obj
WHERE ah.query_start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
```

### Data Classification with ACCOUNT_USAGE.COLUMNS

Join all columns across tier databases against a classification seed to produce a complete catalogue — classified columns show their tier, unclassified appear as `UNCLASSIFIED`:

```sql
SELECT
  c.table_catalog, c.table_name, c.column_name, c.data_type,
  COALESCE(t.classification, 'UNCLASSIFIED') AS classification
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS c
LEFT JOIN data_classification_tags t
  ON UPPER(c.table_name) = UPPER(t.table_name)
  AND UPPER(c.column_name) = UPPER(t.column_name)
WHERE c.deleted IS NULL
  AND c.table_catalog LIKE '%_T%_STAGING' OR c.table_catalog LIKE '%_T%_INTEGRATION'
```