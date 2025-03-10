# MySQL 故障诊断：教你快速定位加锁的 SQL

### 为什么会加锁

锁是可以协调并发连接访问 MySQL 数据库资源的一种技术，可以保证数据的一致性。

有关 MySQL 锁的具体内容，可以详看我的另外一个 Chat，其中有一部分介绍的是“MySQL 锁机制（机智）”：

> \[MySQL 地基基础：事务和锁的面纱\](MySQL 地基基础：事务和锁的面纱.md)

### 数据库锁有什么威力

在实际应用中你肯定遇到过锁问题，这个锁的威力很大，那出现了数据库锁，会造成什么影响呢？

**锁等待** 一个连接申请了锁资源，其他连接要申请资源，无法获取，等待资源释放。**死锁**

你锁我，我也锁你，大家一起锁着吧。

### 快速定位加锁的 SQL

既然出现了锁，我们就要管理一下这个锁，找一找是什么 SQL 持有这个锁，早点发现锁，提前干预，下面我们看看如何快速定位加锁的 SQL。

首先我们创建一个测试表：

```sql
mysql> create table t1(id decimal,v_name varchar(10));
mysql> insert into t1 values(1,'a'),(2,'b'),(3,'c');
mysql> select * from t1;
+------+--------+
| id   | v_name |
+------+--------+
|    1 | a      |
|    2 | b      |
|    3 | c      |
+------+--------+
3 rows in set (0.00 sec)
```

会话 1，开启事务，更新 id=1 的数据：

```sql
mysql> begin;
mysql> update t1 set v_name='aa' where id=1;
```

会话 2，开启另一个事务，删除 id=1 的数据：

```sql
mysql> begin;
mysql> delete from t1 where id=1;
```

此时会话 2 被锁定，出现锁等待。

我们再开一个会话 3，查查当前的 processlist，看看是否能发现什么？

```sql
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+---------------------------+
| Id | User | Host      | db   | Command | Time | State    | Info                      |
+----+------+-----------+------+---------+------+----------+---------------------------+
| 38 | root | localhost | test | Sleep   |    5 |          | NULL                      |
| 41 | root | localhost | test | Query   |    2 | updating | delete from t1 where id=1 |
| 42 | root | localhost | NULL | Query   |    0 | starting | show processlist          |
+----+------+-----------+------+---------+------+----------+---------------------------+
```

我们可以看到 delete 这个 SQL 的进程在执行中，并没有发现其他有价值的内容，那锁在哪里了。接下来的步骤带你一步步的定位出加锁的 SQL。

**定位锁等待**

```sql
mysql> select * from information_schema.innodb_lock_waits;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 2207              | 2207:28:3:7       | 2206            | 2206:28:3:7      |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

结果显示有一个锁等待。

**定位锁**

```sql
mysql> select * from information_schema.innodb_locks;
+-------------+-------------+-----------+-----------+-------------+-----------------+------------+-----------+----------+----------------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table  | lock_index      | lock_space | lock_page | lock_rec | lock_data      |
+-------------+-------------+-----------+-----------+-------------+-----------------+------------+-----------+----------+----------------+
| 2207:28:3:7 | 2207        | X         | RECORD    | `test`.`t1` | GEN_CLUST_INDEX |         28 |         3 |        7 | 0x000000000211 |
| 2206:28:3:7 | 2206        | X         | RECORD    | `test`.`t1` | GEN_CLUST_INDEX |         28 |         3 |        7 | 0x000000000211 |
+-------------+-------------+-----------+-----------+-------------+-----------------+------------+-----------+----------+----------------+
2 rows in set, 1 warning (0.00 sec)
```

结果显示有两个锁相关内容。

**定位事务** 

```sql
mysql> select trx_id,trx_started,trx_requested_lock_id,trx_query,trx_mysql_thread_id from information_schema.innodb_trx;
+--------+---------------------+-----------------------+---------------------------+---------------------+
| trx_id | trx_started         | trx_requested_lock_id | trx_query                 | trx_mysql_thread_id |
+--------+---------------------+-----------------------+---------------------------+---------------------+
| 2207   | 2021-01-18 15:18:11 | 2207:28:3:7           | delete from t1 where id=1 |                  41 |
| 2206   | 2021-01-18 15:18:08 | NULL                  | NULL                      |                  38 |
+--------+---------------------+-----------------------+---------------------------+---------------------+
2 rows in set (0.01 sec)
```

结果有两个事务，MySQL 事务线程 id 为 38 和 41，很直观的看到 41 是我们的 delete 事务，被 38 锁定。**定位线程** 

```sql
mysql> select * from performance_schema.threads where processlist_id=38;
+-----------+---------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+
| THREAD_ID | NAME                      | TYPE       | PROCESSLIST_ID | PROCESSLIST_USER | PROCESSLIST_HOST | PROCESSLIST_DB | PROCESSLIST_COMMAND | PROCESSLIST_TIME | PROCESSLIST_STATE | PROCESSLIST_INFO | PARENT_THREAD_ID | ROLE | INSTRUMENTED | HISTORY | CONNECTION_TYPE | THREAD_OS_ID |
+-----------+---------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+
|        63 | thread/sql/one_connection | FOREGROUND |             38 | root             | localhost        | test           | Sleep               |               35 | NULL              | NULL             |             NULL | NULL | YES          | YES     | Socket          |        15070 |
+-----------+---------------------------+------------+----------------+------------------+------------------+----------------+---------------------+------------------+-------------------+------------------+------------------+------+--------------+---------+-----------------+--------------+
1 row in set (0.00 sec)
```

结果找到 MySQL 事务线程 38 对应的服务器线程 63。**定位加锁 SQL** 

```sql
mysql> select * from performance_schema.events_statements_current where thread_id=63;
+-----------+----------+--------------+----------------------+--------------------------+---------------------+---------------------+------------+-----------+--------------------------------------+----------------------------------+----------------------------------------------+----------------+-------------+---------------+-------------+-----------------------+-------------+-------------------+------------------------------------------+--------+----------+---------------+-----------+---------------+-------------------------+--------------------+------------------+------------------------+--------------+--------------------+-------------+-------------------+------------+-----------+-----------+---------------+--------------------+------------------+--------------------+---------------------+
| THREAD_ID | EVENT_ID | END_EVENT_ID | EVENT_NAME           | SOURCE                   | TIMER_START         | TIMER_END           | TIMER_WAIT | LOCK_TIME | SQL_TEXT                             | DIGEST                           | DIGEST_TEXT                                  | CURRENT_SCHEMA | OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | OBJECT_INSTANCE_BEGIN | MYSQL_ERRNO | RETURNED_SQLSTATE | MESSAGE_TEXT                             | ERRORS | WARNINGS | ROWS_AFFECTED | ROWS_SENT | ROWS_EXAMINED | CREATED_TMP_DISK_TABLES | CREATED_TMP_TABLES | SELECT_FULL_JOIN | SELECT_FULL_RANGE_JOIN | SELECT_RANGE | SELECT_RANGE_CHECK | SELECT_SCAN | SORT_MERGE_PASSES | SORT_RANGE | SORT_ROWS | SORT_SCAN | NO_INDEX_USED | NO_GOOD_INDEX_USED | NESTING_EVENT_ID | NESTING_EVENT_TYPE | NESTING_EVENT_LEVEL |
+-----------+----------+--------------+----------------------+--------------------------+---------------------+---------------------+------------+-----------+--------------------------------------+----------------------------------+----------------------------------------------+----------------+-------------+---------------+-------------+-----------------------+-------------+-------------------+------------------------------------------+--------+----------+---------------+-----------+---------------+-------------------------+--------------------+------------------+------------------------+--------------+--------------------+-------------+-------------------+------------+-----------+-----------+---------------+--------------------+------------------+--------------------+---------------------+
|        63 |       32 |           32 | statement/sql/update | socket_connection.cc:101 | 2757904906303653000 | 2757904906543381000 |  239728000 | 145000000 | update t1 set v_name='aa' where id=1 | 356a053ffb5eae2a35981b05090faa01 | UPDATE `t1` SET `v_name` = ? WHERE `id` = ?  | test           | NULL        | NULL          | NULL        |                  NULL |           0 | 00000             | Rows matched: 1  Changed: 0  Warnings: 0 |      0 |        0 |             0 |         0 |             3 |                       0 |                  0 |                0 |                      0 |            0 |                  0 |           0 |                 0 |          0 |         0 |         0 |             0 |                  0 |             NULL | NULL               |                   0 |
+-----------+----------+--------------+----------------------+--------------------------+---------------------+---------------------+------------+-----------+--------------------------------------+----------------------------------+----------------------------------------------+----------------+-------------+---------------+-------------+-----------------------+-------------+-------------------+------------------------------------------+--------+----------+---------------+-----------+---------------+-------------------------+--------------------+------------------+------------------------+--------------+--------------------+-------------+-------------------+------------+-----------+-----------+---------------+--------------------+------------------+--------------------+---------------------+
1 row in set (0.00 sec)
```

结果中我们找到了加锁的 update 的 SQL 语句。**总结** 在 MySQL 数据库中出现了锁，不要着急，我们通过这个方法可以快速定位加锁的 SQL，你学会了吗？
```
