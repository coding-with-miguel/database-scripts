## Postgres Scripts and Queries

### Show running queries (9.2+).
```text
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' 
AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start DESC;
```

### Kill running query.
```text
SELECT pg_cancel_backend(procpid);
```

### Kill idle query.
```text
SELECT pg_terminate_backend(procpid);
```

### Show all database users.
```text
SELECT * 
FROM pg_user;
```

### Show all database tables.
```text
SELECT * 
FROM pg_tables;
```

### Show queries running longer than 2 minutes (all users).
```text
SELECT pid, now() - query_start as "runtime", usename, datname, waiting, state, query
FROM pg_stat_activity
WHERE now() - query_start > '2 minutes'::interval 
AND state = 'active'
ORDER BY runtime DESC;
```

### Check storage used of all databases.
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

### Check storage used by table.
```text
SELECT nspname || '.' || relname AS "relation",
pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size"
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
AND C.relkind <> 'i'
AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size(C.oid) DESC;
```

### See all locks.
```text
SELECT t.relname, l.locktype, page, virtualtransaction, pid, mode, granted 
FROM pg_locks l, pg_stat_all_tables t 
WHERE l.relation = t.relid 
ORDER BY relation asc;
```

### Estimated row count of table when table becomes too large to execute count(*).
```text
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables 
WHERE relname = 'table_name'
```
