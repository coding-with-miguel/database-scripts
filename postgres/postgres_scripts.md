## Snowflake Scripts and Queries

### Administrative Queries

#### Show running queries
```text
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' 
AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start DESC;
```

#### Show all database users
```text
SELECT * 
FROM pg_user;
```

#### Show all database tables
```text
SELECT * 
FROM pg_tables;
```

#### Show queries running longer than 2 minutes (all users)
```text
SELECT pid, now() - query_start as "runtime", usename, datname, waiting, state, query
FROM pg_stat_activity
WHERE now() - query_start > '2 minutes'::interval 
    AND state = 'active'
ORDER BY runtime DESC;
```

#### Kill all running connections of a current database
```text
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE datname = current_database()  
  AND pid <> pg_backend_pid();
```

#### Kill running query
```text
SELECT pg_cancel_backend(procpid);
```

#### Kill idle query
```text
SELECT pg_terminate_backend(procpid);
```

#### Check storage used of all databases
```text
SELECT d.datname AS Name, pg_catalog.pg_get_userbyid(d.datdba) AS Owner,
CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT')
	THEN pg_catalog.pg_size_pretty(pg_catalog.pg_database_size(d.datname)) 
	ELSE 'No Access' 
END AS SIZE 
FROM pg_catalog.pg_database d 
ORDER BY 
CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT') 
	THEN pg_catalog.pg_database_size(d.datname)
	ELSE NULL 
END;
```

#### Check storage used by table
```text
SELECT nspname || '.' || relname AS "relation",
pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size"
FROM pg_class C
LEFT JOIN pg_namespace N 
    ON (N.oid = C.relnamespace)
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
    AND C.relkind <> 'i'
    AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size(C.oid) DESC;
```

#### See all locks
```text
SELECT t.relname, l.locktype, page, virtualtransaction, pid, mode, granted 
FROM pg_locks l, pg_stat_all_tables t 
WHERE l.relation = t.relid 
ORDER BY relation asc;
```

#### Estimated row count of table when table becomes too large to execute count(*).
```text
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables 
WHERE relname = 'table_name'
```

#### textcache hit rates (should not be less than 0.99)
```text
SELECT sum(heap_blks_read) as heap_read, sum(heap_blks_hit)  as heap_hit, (sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit) as ratio
FROM pg_statio_user_tables;
```

#### Find cardinality of index
```text
SELECT relname, relkind, reltuples as cardinality, relpages 
FROM pg_class 
WHERE relname LIKE 'tableprefix%';
```

#### See Connection type and count
```text
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

#### Restart all sequences
```text
SELECT 'ALTER SEQUENCE ' || relname || ' RESTART;' 
FROM pg_class 
WHERE relkind = 'S';
```

#### Querying Stats
```text
SELECT relname, relpages 
FROM pg_class 
ORDER BY relpages DESC 
LIMIT 1;
```


### Importing / Exporting Data

### Dump database on remote host to file
```commandline
$ pg_dump -U username -h hostname databasename > dump.sql
```

### Import dump into existing database
```commandline
$ psql -d newdb -f dump.sql
```


### Handling Duplicate Rows

### Find duplicate rows
```text
SELECT student_id, COUNT(student_id)
FROM tbl_scores
GROUP BY student_id
HAVING COUNT(student_id)> 1
ORDER BY student_id;
```

### Delete duplicate rows
```text
DELETE FROM tbl_scores
WHERE student_id IN (
    SELECT student_id
    FROM (
        SELECT student_id,
            ROW_NUMBER() OVER(
                PARTITION BY student_id
                ORDER BY student_id
            ) AS row_num
        FROM tbl_scores
    ) t
WHERE t.row_num > 1);
```


### Table Management

### Create table
```text
CREATE TABLE tbl_students (
    student_id serial PRIMARY KEY,
    full_name VARCHAR NOT NULL,
    teacher_id INT
    department VARCHAR NOT NULL,
);
```

### Insert records into table
```text
INSERT INTO tbl_students (
    student_id, full_name, teacher_id, department_id
)
VALUES
(1, ‘Elvis Wilson', NULL, 5),
(2, 'Cynthia Hilbert', 1, 7),
(3, 'Chadhuri Patel', 2, 5),
(4, 'Andre Kharlanov', 5, 9),
(5, 'Buster Chaplin', 2, 8);
```

### Use a PostgreSQL query result to create a table
```text
SELECT student_id, score
INTO tbl_top_students
FROM tbl_scores
WHERE score > AVG(score);
```


### Advanced Joining

### Advanced PostgreSQL self-join query and alias
```text
SELECT s1.full_name, s2.full_name, s1.score
FROM tbl_scores s1
INNER JOIN tbl_scores s2 
    ON s1.student_id <> s2.student_id
    AND s1.score = s2.score;
```

### FULL OUTER JOIN
```text
SELECT student_name, department_name
FROM tbl_departments e
FULL OUTER JOIN tbl_departments d 
    ON d.department_id = e.department_id;
```

### LEFT JOIN
```text
SELECT tbl_students.full_name, tbl_students.student_id,
    tbl_scores.student_id, tbl_scores.score
FROM tbl_students
LEFT JOIN tbl_scores 
    ON tbl_students.student_id = tbl_scores.student_id;
```

### CROSS JOIN
```text
CREATE TABLE Labels (label CHAR(1) PRIMARY KEY);
CREATE TABLE Scores (score INT PRIMARY KEY);

INSERT INTO Labels (label)
VALUES ('Fahr'), ('Cels');

INSERT INTO Scores (score)
VALUES (1), (2);

SELECT * FROM Labels CROSS JOIN Scores;
```

### NATURAL JOIN
```text
SELECT * 
FROM tbl_students 
NATURAL JOIN tbl_scores;
```

### UNION
```text
SELECT * 
FROM tbl_scores
UNION
SELECT *
FROM tbl_departments
ORDER BY tbl_departments.full_name ASC;
```

### UNION ALL
```text
SELECT * 
FROM tbl_scores
UNION ALL
SELECT *
FROM tbl_departments
ORDER BY tbl_departments.full_name ASC;
```


### Triggers

### Advanced PostgreSQL self-join query and alias
```text
CREATE OR REPLACE FUNCTION demo()
RETURNS trigger AS
$$
BEGIN
    INSERT INTO demo1 
    (col1, col2, col3)
    VALUES
    (NEW.col1, NEW.col2, current_date);
    RETURN NEW;
END;
$$
LANGUAGE 'plpgsql';
```
