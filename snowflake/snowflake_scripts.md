## Snowflake Scripts and Queries

## Index

* [Warehouses](#warehouses)
* [Stages](#stages)
* [File Formats](#file-formats)
* [Parameters](#parameters)
* [Databases](#databases)
* [Schemas](#schemas)
* [Tables](#tables)
* [Users](#users)
* [Tasks](#tasks)
* [Qualify](#qualify)
* [Pipes](#pipes)
* [Sequences](#sequences)
* [Cancelling Statements](#cancelling-statements)

### Warehouses

#### Create a virtual Warehouse
```text
CREATE OR REPLACE WAREHOUSE demo_wh
WAREHOUSE_SIZE = 'X-SMALL'
AUTO_SUSPEND = 180 -- time in seconds
AUTO_RESUME = TRUE
INITIALLY_SUSPENDED = TRUE;
```

#### See Warehouses with `auto_resume` turned off
```text
SHOW WAREHOUSES;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    "auto_resume" = FALSE;
```

#### See Warehouses with `auto_suspend` turned off
```text
SHOW WAREHOUSES;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    "auto_suspend" IS NULL;
```

#### See Warehouses with long `auto_suspend` values
```text
SHOW WAREHOUSES;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    "auto_suspend" > 600;  --Your Threshold in Seconds
```

#### Set account statement timeout per Warehouse
```text
ALTER WAREHOUSE LOAD_WH SET 
    STATEMENT_TIMEOUT_IN_SECONDS = 3600;
SHOW PARAMETERS IN WAREHOUSE LOAD_WH;
```

#### Warehouse credit usage greater than 7-Day average
```text
SELECT 
    WAREHOUSE_NAME, 
    DATE(START_TIME) AS DATE, 
    SUM(CREDITS_USED) AS CREDITS_USED,
    AVG(SUM(CREDITS_USED)) OVER (PARTITION BY WAREHOUSE_NAME ORDER BY DATE ROWS 7 PRECEDING) AS CREDITS_USED_7_DAY_AVG,
    (TO_NUMERIC(SUM(CREDITS_USED)/CREDITS_USED_7_DAY_AVG*100,10,2)-100)::STRING || '%' AS VARIANCE_TO_7_DAY_AVERAGE
FROM 
    "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
GROUP BY 
    DATE, 
    WAREHOUSE_NAME
ORDER BY 
    DATE DESC;
```

#### Warehouses approaching cloud service billing threshold
```text
WITH
cloudServices AS (
    SELECT
        WAREHOUSE_NAME, 
        MONTH(START_TIME) AS MONTH,
        SUM(CREDITS_USED_CLOUD_SERVICES) AS CLOUD_SERVICES_CREDITS, 
        COUNT(*) AS NO_QUERYS 
    FROM 
        "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
    GROUP BY 
        WAREHOUSE_NAME, MONTH
    ORDER BY 
        WAREHOUSE_NAME, NO_QUERYS DESC
),
warehouseMetering AS (
    SELECT
        WAREHOUSE_NAME, 
        MONTH(START_TIME) AS MONTH,
        SUM(CREDITS_USED) AS CREDITS_FOR_MONTH
    FROM 
        "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
    GROUP BY 
        WAREHOUSE_NAME, MONTH
    ORDER BY 
        WAREHOUSE_NAME, CREDITS_FOR_MONTH DESC
)
SELECT 
    *, 
    TO_NUMERIC(CLOUD_SERVICES_CREDITS/NULLIF(CREDITS_FOR_MONTH,0)*100,10,2) AS PERCT_CLOUD 
FROM 
    cloudServices
JOIN 
    warehouseMetering USING(WAREHOUSE_NAME, MONTH)
ORDER BY 
    PERCT_CLOUD DESC;
```

#### Find Warehouses without Resource Monitors
```text
SHOW WAREHOUSES;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    "resource_monitor" = 'null';
```

#### Create a Warehouse Resource Monitor
```text
CREATE RESOURCE MONITOR "CURTLOVESCUBES_RM" WITH CREDIT_QUOTA = 150 
 TRIGGERS 
 ON 75 PERCENT DO NOTIFY 
 ON 90 PERCENT DO NOTIFY;
ALTER WAREHOUSE "CURT_WH" SET RESOURCE_MONITOR = "CURTLOVESCUBES_RM";
```

**[⬆ Back to Index](#index)**


### Stages

#### See Stages created
```text
SHOW STAGES;
```

**[⬆ Back to Index](#index)**


### File Formats

#### See File Formats created
```text
SHOW FILE FORMATS;
```

**[⬆ Back to Index](#index)**


### Parameters

#### See Parameters created
```text
SHOW PARAMETERS;
```

**[⬆ Back to Index](#index)**


### Databases

#### Create a database
```text
CREATE DATABASE demo;
```

#### Create (or replace) a database - this will replace an existing database
```text
CREATE OR REPLACE DATABASE demo;
```

#### Create a database unless one with that name already exists
```text
CREATE DATABASE demo IF NOT EXISTS;
```

**[⬆ Back to Index](#index)**


### Schemas

#### Create a schema
```text
CREATE SCHEMA demo;
```

#### Create (or replace) a schema - this will replace an existing database
```text
CREATE OR REPLACE SCHEMA demo;
```

#### Create a schema unless one with that name already exists
```text
CREATE SCHEMA demo IF NOT EXISTS;
```

**[⬆ Back to Index](#index)**


### Tables

#### Create a table - this will error if the table exists.
```text
CREATE TABLE emp_details (
    first_name STRING,
    last_name STRING,
    email STRING,
    address STRING,
    city STRING,
    start_date DATE
);
```

#### Create or replace a table - this will replace a table if it exists
```text
CREATE OR REPLACE TABLE emp_details (
    first_name STRING,
    last_name STRING,
    email STRING,
    address STRING,
    city STRING,
    start_date DATE
);
```

#### See unused tables
```text
--DML from the Information Schema to identify Table sizes and Last Updated Timestamps
SELECT 
    TABLE_CATALOG || '.' || TABLE_SCHEMA || '.' || TABLE_NAME AS TABLE_PATH, 
    TABLE_NAME, 
    TABLE_SCHEMA AS SCHEMA,
    TABLE_CATALOG AS DATABASE, 
    BYTES,
    TO_NUMBER(BYTES / POWER(1024,3),10,2) AS GB, 
    LAST_ALTERED AS LAST_USE,
    DATEDIFF('Day',LAST_USE,CURRENT_DATE) AS DAYS_SINCE_LAST_USE
FROM 
    INFORMATION_SCHEMA.TABLES
WHERE 
    DAYS_SINCE_LAST_USE > 90 -- Use your Days Threshold
ORDER BY 
    BYTES DESC;
 
-- Last DML on Object
SELECT (SYSTEM$LAST_CHANGE_COMMIT_TIME(
'DATABASE.SCHEMA.TABLE_NAME')/1000)::TIMESTAMP_NTZ;
 
-- Queries on Object in Last NN Days
SELECT 
    COUNT(*) 
FROM 
    "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
WHERE 
    CONTAINS(UPPER(QUERY_TEXT),'TABLE_NAME')
AND 
    DATEDIFF('Day',START_TIME,CURRENT_DATE) < 90;
 
-- Last Query on Object in Last NN Days
SET TABLE_NAME = 'MY_TABLE';
 
SELECT 
    START_TIME, 
    QUERY_ID, 
    QUERY_TEXT
FROM 
    "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
WHERE 
    CONTAINS(UPPER(QUERY_TEXT),$TABLE_NAME) 
AND QUERY_ID = (
     SELECT 
        TOP 1 QUERY_ID 
     FROM 
        "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
     WHERE 
        CONTAINS(UPPER(QUERY_TEXT),$TABLE_NAME)
     ORDER BY 
        START_TIME DESC
)
AND 
    DATEDIFF('Day', START_TIME, CURRENT_DATE) < 90;
 
-- Object Used in a View Definition in Last NN Days
SELECT 
    * 
FROM 
    INFORMATION_SCHEMA.VIEWS
WHERE 
    CONTAINS(VIEW_DEFINITION,'TABLE_NAME')
AND 
    DATEDIFF('Day', LAST_ALTERED, CURRENT_DATE) < 90;
```

**[⬆ Back to Index](#index)**


### Users

#### See dormant users
```text
--Never Logged In Users
SHOW USERS;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    "last_success_login" IS NULL
AND 
    DATEDIFF('Day',"created_on",CURRENT_DATE) > 30;
 
--Stale Users
SHOW USERS;
SELECT 
    * 
FROM 
    TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE 
    DATEDIFF('Day', "last_success_login", CURRENT_DATE) > 30;
```

**[⬆ Back to Index](#index)**


### Qualify

#### Qualify clause
The `qualify` clause allows us filter directly on the results of window functions, rather than first creating the result in a CTE and then filtering on it later. A very common approach for window functions is to use a subquery to first get the row_number().
```text
row_number() over (partition by email order by created_at desc) as date_ranking
```

And then later filter on this in another CTE to get the first row in a group.
```text
where date_ranking = 1
```

**[⬆ Back to Index](#index)**


### Tasks

#### Check for failed Tasks
```text
SELECT 
    * 
FROM 
    SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE 
    completed_time > DATEADD(hours, -24, current_timestamp())
AND 
    scheduled_time::date > DATEADD(hours, -24, current_timestamp())::date
AND 
    error_message IS NOT NULL
ORDER BY 
    database_name, 
    scheduled_time desc;
```

**[⬆ Back to Index](#index)**


### Pipes

#### Check for failed Pipes
```text
SELECT CH.table_name AS table_name, 
  CH.table_schema_name AS schema_name, 
  CH.table_catalog_name AS database_name, 
  CONCAT(CH.stage_location, CH.file_name) AS file_name,
  CH.status AS load_status, 
  CH.row_count AS rows_loaded, 
  CH.row_parsed AS rows_parsed,
  CH.first_error_message AS error_message,
  CH.first_error_line_number AS error_line,
  CH.first_error_character_pos AS error_POSITION,
  CH.first_error_column_name AS error_column,
  CH.error_count AS error_count,
  CH.error_limit AS error_limit,
  CH.stage_location AS load_stage,
  CH.file_name AS load_file,
  RIGHT(CONCAT(CH.stage_location, CH.file_name), LEN(CONCAT(CH.stage_location, CH.file_name)) - POSITION('/', CH.stage_location, POSITION('://', CH.stage_location) + 3)) AS full_load_file,
  CH.last_load_time AS load_time,
  L.type AS load_type,
  L.name AS load_type_name,
  L.scheduled_time AS next_load_time,
  CH2.final_loaded AS final_load_status,
  CH2.last_load_time AS final_load_time
FROM 
    snowflake.account_usage.copy_history CH
LEFT JOIN (
    SELECT 
        'SNOWPIPE' AS type, 
        pipe_name AS name, 
        RTRIM(pipe_name, '_PIPE') AS table_name, 
        pipe_schema AS schema_name, 
        pipe_catalog AS database_name, 
        definition,
        null AS scheduled_time 
    FROM 
        snowflake.account_usage.pipes 
    WHERE 
        deleted is null
    AND 
        scheduled_time > dateadd(day, -1, current_timestamp())
    UNION (
        SELECT 
            'TASK' AS type, 
            name, 
            RTRIM(name, '_TASK') AS table_name, 
            schema_name, 
            database_name, 
            query_text AS definition, 
            scheduled_time 
        FROM 
            table([your_database_name].information_schema.task_history())
        WHERE 
            scheduled_time > dateadd(day, -1, current_timestamp())
        )
    ) L ON CH.table_name = L.table_name 
        AND CH.table_schema_name = L.schema_name 
        AND CH.table_catalog_name = L.database_name
LEFT JOIN (
    SELECT 
        status, 
        stage_location, 
        file_name, 
        'File Successfully Loaded' AS final_loaded, 
        last_load_time 
    FROM 
        snowflake.account_usage.copy_history 
    WHERE 
        status = 'Loaded'
    AND 
        last_load_time > dateadd(day, -1, current_timestamp())
) CH2 ON CH.stage_location = CH2.stage_location 
    AND 
        CH.file_name = CH2.file_name
WHERE 
    ch.last_load_time > dateadd(day, -1, current_timestamp())
AND 
    lower(load_status) != 'loaded'
AND 
    lower(load_status) != 'load in progress'
ORDER BY 
    CH.last_load_time DESC; 
```

**[⬆ Back to Index](#index)**


### Sequences

#### Create a Sequence
```text
CREATE OR REPLACE SEQUENCE seq1;
```

#### Use an existing Sequence in a table 
```text
CREATE OR REPLACE TABLE foo (
    k NUMBER DEFAULT seq1.NEXTVAL, 
    v NUMBER
);
```

**[⬆ Back to Index](#index)**


### Cancelling Statements

#### Cancels all active/running queries in the specified session
```text
SYSTEM$CANCEL_ALL_QUERIES( <session_id> );
```

#### Cancels the specified query (or statement) if it is currently active/running
```text
SYSTEM$CANCEL_QUERY( <query_id> );
```

**[⬆ Back to Index](#index)**
