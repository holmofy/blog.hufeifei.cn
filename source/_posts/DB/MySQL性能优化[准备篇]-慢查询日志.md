---
title: MySQL性能优化[准备篇]-慢查询日志
date: 2018-03-08 00:17
categories: 数据库
tags: 
- DB
- MySQL
keywords:
- MySQL
- 性能优化
---

在MySQL5.0及之前的版本中，慢查询日志的响应时间单位是秒。显然对于互联网与电子商务如此发达的现在，“秒”级别的查询实在太慢了。在MySQL5.1及更新的版本中，慢查询日志的功能得到了增强，甚至可以通过设置`long_query_time`为0来捕获所有的查询。

在MySQL的当前版本中，慢查询日志是开销最低、精度最高的测量查询时间的工具。我们完全不用担心开启慢查询日志会带来额外的I/O开销，这方面已经经过了测试，慢查询日志带来的开销可以忽略不计。更需要担心的是日志可能消耗大量的磁盘空间。如果长期开启慢查询日志，注意部署日志轮转(log rotation)工具，或者不要长期启用慢查询气质，只需在收集负载样本期间开启即可。

MySQL还有另外一种查询日志，被称为“通用日志”，但极少用于分析服务器性能。通用日志在查询请求达到服务器时记录，所以不包括响应时间和执行计划等重要的性能信息。MySQL5.1之后支持将日志记录到数据库的表中，但大多数情况没必要记录到表中。这对性能有较大的影响，MySQL5.1在将慢查询记录到文件中时已经支持微妙级的信息，然而将慢查询记录到表中会导致时间退化到秒级别。秒级别的慢查询日志没有太大意义。

# 开启慢查询日志

默认情况下，慢查询日志被禁用；默认慢查询日志放在数据目录，默认文件名为*host_name*-slow.log。

```mysql
mysql> show variables like 'slow_query%';
+---------------------+------------------------------------------------------+
| Variable_name       | Value                                                |
+---------------------+------------------------------------------------------+
| slow_query_log      | OFF                                                  |
| slow_query_log_file | E:\mysql-5.7.15-winx64\data\DESKTOP-B76J065-slow.log |
+---------------------+------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

-- 开启慢查询日志
mysql> set global slow_query_log=1;
Query OK, 0 rows affected (0.01 sec)

-- 设置慢查询日志路径
mysql> set global slow_query_log_file='E:\\slow_query.log';
Query OK, 0 rows affected (0.01 sec)

mysql> show variables like 'slow_query%';
+---------------------+-------------------+
| Variable_name       | Value             |
+---------------------+-------------------+
| slow_query_log      | ON                |
| slow_query_log_file | E:\slow_query.log |
+---------------------+-------------------+
2 rows in set, 1 warning (0.00 sec)
```

> **注意**：使用`set global slow_query_log=1`开启慢查询日志只对当前数据库生效，MySQL重启后则会失效。如果要永久生效，就必须修改配置文件my.cnf(windows下为my.ini)，或者在使用命令行启动mysql的时候，在[`--slow-query-log`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_slow_query_log)参数中指定，其它系统变量也是如此。

# 设置慢查询时间阈值

开启了慢查询日志后，什么样的SQL才会记录到慢查询日志里面呢？ 这个是由参数[`long_query_time`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_long_query_time)控制，默认`long_query_time=10`，也就是10秒。

```mysql
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set, 1 warning (0.01 sec)

mysql> set global long_query_time=1;
Query OK, 0 rows affected (0.00 sec)

-- 注意show variables查看的是当前会话的变量值
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set, 1 warning (0.00 sec)

mysql> show global variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
1 row in set, 1 warning (0.00 sec)
```

> **注意**：使用`show variables like 'long_query_time'`查看是当前连接会话的变量值。因为`long_query_time`变量在GLOBAL和SESSION中都有，如果不加访问域则使用默认的SESSION访问域，所以要想看到全局的`long_query_time`变量需要用如下语句`show global variables like 'long_query_time'`。
>
> [`show variables`完整语法](https://dev.mysql.com/doc/refman/5.7/en/show-variables.html)为：show [global|session] variables  [like '*pattern*' | where *expr*]

# 将慢查询日志记录到数据表中

尽管将慢查询日志记录到数据表中会影响性能，但谁也无法保证不会出现这样的需求。

[`log_output`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_log_output)变量用于指定日志的存储方式，这个变量有两个取值：`FILE`,`TABLE`，默认取值为`FILE`。MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。

```mysql
mysql> show variables like 'log_output';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> set global log_output='TABLE';
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'log_output';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | TABLE |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

# 将未使用到索引的查询当做慢查询记录到日志中

系统变量[log-queries-not-using-indexes](https://dev.mysql.com/doc/refman/5.7/en/server-options.html#option_mysqld_log-queries-not-using-indexes)：未使用索引的查询也被记录到慢查询日志中（可选项）。如果调优的话，建议开启这个选项。另外，开启了这个参数，其实使用full index scan(索引全扫描)的sql也会被记录到慢查询日志。

```mysql
mysql> show variables like 'log_queries_not_using_indexes';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | OFF   |
+-------------------------------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> set global log_queries_not_using_indexes=1;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'log_queries_not_using_indexes';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | ON    |
+-------------------------------+-------+
1 row in set, 1 warning (0.00 sec)
```

# 将管理性质的SQL语句记录慢查询

[`log-slow-admin-statements`](https://dev.mysql.com/doc/refman/5.7/en/server-options.html#option_mysqld_log-slow-admin-statements)变量会将管理性质的慢SQL记录到慢查询日志中。管理性质的SQL语句包括：[`ALTER TABLE`](https://dev.mysql.com/doc/refman/5.7/en/alter-table.html), [`ANALYZE TABLE`](https://dev.mysql.com/doc/refman/5.7/en/analyze-table.html), [`CHECK TABLE`](https://dev.mysql.com/doc/refman/5.7/en/check-table.html), [`CREATE INDEX`](https://dev.mysql.com/doc/refman/5.7/en/create-index.html), [`DROP INDEX`](https://dev.mysql.com/doc/refman/5.7/en/drop-index.html), [`OPTIMIZE TABLE`](https://dev.mysql.com/doc/refman/5.7/en/optimize-table.html), 和[`REPAIR TABLE`](https://dev.mysql.com/doc/refman/5.7/en/repair-table.html)。

```mysql
mysql> show variables like 'log_slow_admin_statements';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| log_slow_admin_statements | OFF   |
+---------------------------+-------+
1 row in set (0.00 sec)
```

# 查看慢查询记录的条数

如果你想查询有多少条慢查询记录，可以使用这个[系统状态变量](https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html)——[Slow_queries](https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html#statvar_Slow_queries)。

```mysql
mysql> show global status like 'slow_queries';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 0     |
+---------------+-------+
1 row in set (0.00 sec)
```

# 日志分析工具

在实际生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活，MySQL提供了日志分析工具**mysqldumpslow**

注意：mysqldumpslow是一个perl语言写的脚本，最好在*nix系统上使用。

```shell
[root@localhost~]# mysqldumpslow --help
Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]

Parse and summarize the MySQL slow query log. Options are

  --verbose    verbose
  --debug      debug
  --help       write this text to standard output

  -v           verbose
  -d           debug
  # s 是表示按照何种方式排序
  -s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
                al: average lock time 平均锁定时间
                ar: average rows sent 平均返回记录数
                at: average query time 平均查询时间
                 c: count 访问计数
                 l: lock time 锁定时间
                 r: rows sent 返回记录
                 t: query time 查询时间
  -r           reverse the sort order (largest last instead of first)
  # t top n的意思，即为返回前面多少条的数据；
  -t NUM       just show the top n queries
  -a           don't abstract all numbers to N and strings to 'S'
  -n NUM       abstract numbers with at least n digits within names
  # g 后边可以写一个正则匹配模式，大小写不敏感的
  -g PATTERN   grep: only consider stmts that include this string
  -h HOSTNAME  hostname of db server for *-slow.log filename (can be wildcard),
               default is '*', i.e. match all
  -i NAME      name of server instance (if using mysql.server startup script)
  -l           don't subtract lock time from total time

mysqldumpslow --help
```

使用示例：

```shell
# 得到返回记录集最多的10个SQL。
mysqldumpslow -s r -t 10 /database/mysql/mysql06_slow.log

# 得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /database/mysql/mysql06_slow.log

# 得到按照时间排序的前10条里面含有左连接的查询语句。
mysqldumpslow -s t -t 10 -g “left join” /database/mysql/mysql06_slow.log

# 另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现刷屏的情况。
mysqldumpslow -s r -t 20 /mysqldata/mysql/mysql06-slow.log | more
```



参考：

《高性能MySQL》

MySQL官方文档--慢查询日志：https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html

https://www.cnblogs.com/saneri/p/6656161.html
