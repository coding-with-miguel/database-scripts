## MySQL Scripts and Queries

## Index

* [Administrative Queries](#administrative-queries)
* [Flush Commands](#flush-commands)
* [Kill Commands](#kill-commands)
* [Connect](#connect)


### Administrative Queries

#### Set MySQL root password
```text
# mysqladmin -u root password YOURNEWPASSWORD
```

#### Change MySQL root password
```text
# mysqladmin -u root -p123456 password 'xyz123'
```

#### Check status of MySQL server
```text
# mysqladmin -u root -p ping

Enter password:
mysqld is alive
```

#### Check MySQL version
```text
# mysqladmin -u root -p version

Enter password: 
mysqladmin  Ver 9.1 Distrib 10.3.32-MariaDB, for Linux on x86_64
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab, and others.
```

#### MySQL server variables and values
```text
# mysqladmin -u root -p variables

Enter password: 
+--------------------------------------------------------------+
| Variable_name                          | Value               |
+--------------------------------------------------------------+
| auto_increment_increment                   | 1                           |
| auto_increment_offset                      | 1                           |
| autocommit                                 | ON                          |
| automatic_sp_privileges                    | ON                          |
| back_log                                   | 50                          |
| basedir                                    | /usr                        |
| big_tables                                 | OFF                         |
| binlog_cache_size                          | 32768                       |
| binlog_direct_non_transactional_updates    | OFF                         |
| binlog_format                              | STATEMENT                   |
| binlog_stmt_cache_size                     | 32768                       |
| bulk_insert_buffer_size                    | 8388608                     |
| character_set_client                       | latin1                      |
| character_set_connection                   | latin1                      |
| character_set_database                     | latin1                      |
| character_set_filesystem                   | binary                      |
| character_set_results                      | latin1                      |
| character_set_server                       | latin1                      |
| character_set_system                       | utf8                        |
| character_sets_dir                         | /usr/share/mysql/charsets/  |
| collation_connection                       | latin1_swedish_ci           |
+---------------------------------------------------+----------------------+
...
```

#### Check active threads of MySQL server
```text
# mysqladmin -u root -p processlist

Enter password: 
+----+-------------+-----------+----+---------+------+--------------------------+------------------+----------+
| Id | User        | Host      | db | Command | Time | State                    | Info             | Progress |
+----+-------------+-----------+----+---------+------+--------------------------+------------------+----------+
| 2  | system user |           |    | Daemon  |      | InnoDB purge coordinator |                  | 0.000    |
| 1  | system user |           |    | Daemon  |      | InnoDB purge worker      |                  | 0.000    |
| 4  | system user |           |    | Daemon  |      | InnoDB purge worker      |                  | 0.000    |
| 3  | system user |           |    | Daemon  |      | InnoDB purge worker      |                  | 0.000    |
| 5  | system user |           |    | Daemon  |      | InnoDB shutdown handler  |                  | 0.000    |
| 20 | root        | localhost |    | Query   | 0    | Init                     | show processlist | 0.000    |
+----+-------------+-----------+----+---------+------+--------------------------+------------------+----------+
```

**[⬆ Back to Index](#index)**


### Flush Commands

#### Useful MySQL flush commands
Following are some useful flush commands with their description.

**flush-hosts**: Flush all host information from the host cache.
**flush-tables**: Flush all tables.
**flush-threads**: Flush all threads cache.
**flush-logs**: Flush all information logs.
**flush-privileges**: Reload the grant tables (same as reload).
**flush-status**: Clear status variables.

```text
# mysqladmin -u root -p flush-hosts
# mysqladmin -u root -p flush-tables
# mysqladmin -u root -p flush-threads
# mysqladmin -u root -p flush-logs
# mysqladmin -u root -p flush-privileges
# mysqladmin -u root -p flush-status
```

**[⬆ Back to Index](#index)**


### Kill Commands

#### Kill sleeping MySQL client process

```text
# mysqladmin -u root -p processlist

Enter password:
+----+------+-----------+----+---------+------+-------+------------------+
| Id | User | Host      | db | Command | Time | State | Info             |
+----+------+-----------+----+---------+------+-------+------------------+
| 5  | root | localhost |    | Sleep   | 14   |       |			 |
| 8  | root | localhost |    | Query   | 0    |       | show processlist |
+----+------+-----------+----+---------+------+-------+------------------+
```

Now, run the following command with kill and process ID as shown below.

```text
# mysqladmin -u root -p kill 5

Enter password:
+----+------+-----------+----+---------+------+-------+------------------+
| Id | User | Host      | db | Command | Time | State | Info             |
+----+------+-----------+----+---------+------+-------+------------------+
| 12 | root | localhost |    | Query   | 0    |       | show processlist |
+----+------+-----------+----+---------+------+-------+------------------+
```

#### Kill multiple MySQL processes

```text
# mysqladmin -u root -p kill 5,10
```

**[⬆ Back to Index](#index)**


### Connect

#### Connect to remote Mysql server

```text
# mysqladmin -h 172.16.25.126 -u root -p
```

**[⬆ Back to Index](#index)**

