## Snowflake Scripts and Queries

## Index

* [Warehouses](#warehouses)
* [Stages](#stages)
* [File Formats](#file-formats)
* [Parameters](#parameters)
* [Tables](#tables)
* [Users](#users)

### Warehouses

#### See Warehouses with `auto_resume` turned off
```text
SHOW WAREHOUSES;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "auto_resume" = FALSE;
```

#### See Warehouses with `auto_suspend` turned off
```text
SHOW WAREHOUSES;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "auto_suspend" IS NULL;
```

#### See Warehouses with long `auto_suspend` values
```text
SHOW WAREHOUSES;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "auto_suspend" > 600;  --Your Threshold in Seconds
```

#### Set account statement timeout per Warehouse
```text
ALTER WAREHOUSE LOAD_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
SHOW PARAMETERS IN WAREHOUSE LOAD_WH;
```

#### Warehouse credit usage greater than 7-Day average
```text
SELECT WAREHOUSE_NAME, DATE(START_TIME) AS DATE, 
SUM(CREDITS_USED) AS CREDITS_USED,
AVG(SUM(CREDITS_USED)) OVER (PARTITION BY WAREHOUSE_NAME ORDER BY DATE ROWS 7 PRECEDING) AS CREDITS_USED_7_DAY_AVG,
(TO_NUMERIC(SUM(CREDITS_USED)/CREDITS_USED_7_DAY_AVG*100,10,2)-100)::STRING || '%' AS VARIANCE_TO_7_DAY_AVERAGE
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
GROUP BY DATE, WAREHOUSE_NAME
ORDER BY DATE DESC;
```

#### Warehouses approaching cloud service billing threshold
```text
WITH
cloudServices AS (SELECT
     WAREHOUSE_NAME, MONTH(START_TIME) AS MONTH,
     SUM(CREDITS_USED_CLOUD_SERVICES) AS CLOUD_SERVICES_CREDITS, 
     COUNT(*) AS NO_QUERYS 
     FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
     GROUP BY WAREHOUSE_NAME,MONTH
     ORDER BY WAREHOUSE_NAME,NO_QUERYS DESC),
warehouseMetering AS (SELECT
     WAREHOUSE_NAME, MONTH(START_TIME) AS MONTH,
     SUM(CREDITS_USED) AS CREDITS_FOR_MONTH
     FROM "SNOWFLAKE"."ACCOUNT_USAGE"."WAREHOUSE_METERING_HISTORY"
     GROUP BY WAREHOUSE_NAME,MONTH
     ORDER BY WAREHOUSE_NAME,CREDITS_FOR_MONTH DESC)
SELECT *, TO_NUMERIC(CLOUD_SERVICES_CREDITS/NULLIF(CREDITS_FOR_MONTH,0)*100,10,2) AS PERCT_CLOUD 
FROM cloudServices
JOIN warehouseMetering USING(WAREHOUSE_NAME,MONTH)
ORDER BY PERCT_CLOUD DESC;
```

#### Find Warehouses without Resource Monitors
```text
SHOW WAREHOUSES;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "resource_monitor" = 'null';
```

#### Create a Warehouses Resource Monitor
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


### Tables

#### See unused tables
```text
--DML from the Information Schema to identify Table sizes and Last Updated Timestamps
SELECT TABLE_CATALOG || '.' || TABLE_SCHEMA || '.' || TABLE_NAME AS TABLE_PATH, 
    TABLE_NAME, TABLE_SCHEMA AS SCHEMA,
    TABLE_CATALOG AS DATABASE, BYTES,
    TO_NUMBER(BYTES / POWER(1024,3),10,2) AS GB, 
    LAST_ALTERED AS LAST_USE,
    DATEDIFF('Day',LAST_USE,CURRENT_DATE) AS DAYS_SINCE_LAST_USE
FROM INFORMATION_SCHEMA.TABLES
WHERE DAYS_SINCE_LAST_USE > 90 --Use your Days Threshold
ORDER BY BYTES DESC;
 
-- Last DML on Object
SELECT (SYSTEM$LAST_CHANGE_COMMIT_TIME(
'DATABASE.SCHEMA.TABLE_NAME')/1000)::TIMESTAMP_NTZ;
 
-- Queries on Object in Last NN Days
SELECT COUNT(*) FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
WHERE CONTAINS(UPPER(QUERY_TEXT),'TABLE_NAME')
AND DATEDIFF('Day',START_TIME,CURRENT_DATE) < 90;
 
-- Last Query on Object in Last NN Days
SET TABLE_NAME = 'MY_TABLE';
 
SELECT START_TIME, QUERY_ID, QUERY_TEXT
FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
WHERE CONTAINS(UPPER(QUERY_TEXT),$TABLE_NAME) 
AND QUERY_ID = (
     SELECT TOP 1 QUERY_ID 
     FROM "SNOWFLAKE"."ACCOUNT_USAGE"."QUERY_HISTORY"
     WHERE CONTAINS(UPPER(QUERY_TEXT),$TABLE_NAME)
     ORDER BY START_TIME DESC)
AND DATEDIFF('Day',START_TIME,CURRENT_DATE) < 90;
 
-- Object Used in a View Definition in Last NN Days
SELECT * FROM INFORMATION_SCHEMA.VIEWS
WHERE CONTAINS(VIEW_DEFINITION,'TABLE_NAME')
AND DATEDIFF('Day',LAST_ALTERED,CURRENT_DATE) < 90;
```
**[⬆ Back to Index](#index)**



### Users

#### See dormant users
```text
--Never Logged In Users
SHOW USERS;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "last_success_login" IS NULL
AND DATEDIFF('Day',"created_on",CURRENT_DATE) > 30;
 
--Stale Users
SHOW USERS;
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE DATEDIFF('Day',"last_success_login",CURRENT_DATE) > 30;
```
**[⬆ Back to Index](#index)**

