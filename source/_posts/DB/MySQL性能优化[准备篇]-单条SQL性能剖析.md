---
title: MySQL性能优化[准备篇]-单条SQL性能剖析
date: 2018-03-08 00:17
categories: 数据库
tags: 
- DB
- MySQL
keywords:
- MySQL
- 性能优化
---


上面一篇文章已经将慢查询语句记录到日志中，接着我们就要对单条SQL查询进行性能分析，了解它慢在何处，才能对症下药进行性能优化。

# show profile

`show profile`命令是MySQL5.1之后引入的，由开源社区的[Jeremy Cole](https://blog.jcole.us/)贡献。

## 1. 开启profiling

[`profiling`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_profiling)系统变量是用来支持访问剖析信息的，`profiling`默认是关闭的，我们可以用`set profiling=1`命令开启profile。

```mysql
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           1 |
+-------------+
1 row in set, 1 warning (0.00 sec)
```

> 查看系统变量除了使用`show variables`命令外，还可以使用`select @@`语句，其中`@@`用来访问系统变量，一个`@`用来访问用户自定义变量。设置变量值可以使用`set`命令，命令语法如下：
>
> SET *variable_assignment* [, *variable_assignment*] ...
>
> *variable_assignment*:
> ​      *user_var_name* = *expr*
> ​    | *param_name* = *expr*
> ​    | *local_var_name* = *expr*
> ​    | [GLOBAL | SESSION]
> ​        *system_var_name* = *expr*
> ​    | [@@global. | @@session. | @@]
> ​        *system_var_name* = *expr*
>
> set命令更多详细内容可以参考[MySQL官方手册](https://dev.mysql.com/doc/refman/5.7/en/set-variable.html)

## 2. show profiles

[`show profiles`](https://dev.mysql.com/doc/refman/5.7/en/show-profiles.html)命令用来显示**当前会话最近执行语句**的耗时情况。

```mysql
mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show profiles;
Empty set, 1 warning (0.00 sec)

mysql> show databases;
+----------------------+
| Database             |
+----------------------+
| information_schema   |
| db_cnilink           |
| db_internation_trade |
| db_oa                |
| db_privilege         |
| db_spring_jpa        |
| db_tao_market        |
| db_zheng             |
| mysql                |
| performance_schema   |
| sys                  |
| test                 |
+----------------------+
12 rows in set (0.01 sec)

mysql> use test;
Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| dept           |
| emp            |
+----------------+
7 rows in set (0.00 sec)

mysql> select * from emp;
+-------+-------+------------+------+---------------------+------+------+--------+
| empno | ename | job        | mgr  | hireDate            | sal  | comm | deptno |
+-------+-------+------------+------+---------------------+------+------+--------+
|     1 | 张三  | 系统架构师 | NULL | 2015-01-07 18:30:25 |   20 |   20 |      1 |
|     2 | 李四  | 扫地员工   | NULL | 2015-01-07 18:30:25 |   20 |   20 |      1 |
|     3 | SMITH | 软件设计师 |    1 | 2017-11-15 16:12:00 |   10 |   10 |      2 |
|     4 | SMITH | 软件设计师 |    1 | 2017-11-15 16:24:25 |   10 |   10 |      2 |
+-------+-------+------------+------+---------------------+------+------+--------+
4 rows in set (0.01 sec)

mysql> show profiles;
+----------+------------+-------------------+
| Query_ID | Duration   | Query             |
+----------+------------+-------------------+
|        1 | 0.00629075 | show databases    |
|        2 | 0.00312050 | SELECT DATABASE() |
|        3 | 0.00216050 | show tables       |
|        4 | 0.01549250 | select * from emp |
+----------+------------+-------------------+
4 rows in set, 1 warning (0.00 sec)
```

可以看到show profiles展示了最近执行的四条查询语句。`show profiles`默认保存15条性能剖析信息，可以通过[**`profiling_history_size`**](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_profiling_history_size)系统变量来修改这个值，注意这个变量最大值为100。

## 3. show profile

我们要看到某一条查询的详细性能剖析信息，需要用到[`show profile`命令](https://dev.mysql.com/doc/refman/5.7/en/show-profile.html)。

```mysql
mysql> show profiles;
+----------+------------+-------------------+
| Query_ID | Duration   | Query             |
+----------+------------+-------------------+
|        1 | 0.00629075 | show databases    |
|        2 | 0.00312050 | SELECT DATABASE() |
|        3 | 0.00216050 | show tables       |
|        4 | 0.01549250 | select * from emp |
+----------+------------+-------------------+
4 rows in set, 1 warning (0.00 sec)

# show profile命令默认显示最近一条查询的剖析信息
mysql> show profile;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.002576 |
| checking permissions | 0.000011 |
| Opening tables       | 0.008276 |
| init                 | 0.000052 |
| System lock          | 0.000026 |
| optimizing           | 0.000008 |
| statistics           | 0.000026 |
| preparing            | 0.000021 |
| executing            | 0.000002 |
| Sending data         | 0.000816 |
| end                  | 0.000007 |
| query end            | 0.000007 |
| closing tables       | 0.000009 |
| freeing items        | 0.000079 |
| logging slow query   | 0.000007 |
| Opening tables       | 0.000163 |
| System lock          | 0.003389 |
| cleaning up          | 0.000019 |
+----------------------+----------+
18 rows in set, 1 warning (0.00 sec)

# 要获取更早的查询剖析信息要用for query字句
# for query字句后面跟QUERY_ID
mysql> show profile for query 3;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.001741 |
| checking permissions | 0.000010 |
| checking permissions | 0.000002 |
| Opening tables       | 0.000040 |
| init                 | 0.000009 |
| System lock          | 0.000005 |
| optimizing           | 0.000006 |
| statistics           | 0.000012 |
| preparing            | 0.000010 |
| executing            | 0.000005 |
| checking permissions | 0.000220 |
| Sending data         | 0.000016 |
| end                  | 0.000003 |
| query end            | 0.000004 |
| closing tables       | 0.000002 |
| removing tmp table   | 0.000004 |
| closing tables       | 0.000003 |
| freeing items        | 0.000055 |
| cleaning up          | 0.000016 |
+----------------------+----------+
19 rows in set, 1 warning (0.00 sec)
```

从`show profile`可以清楚地看到查询语句每个阶段的耗时情况。

> 关于show profile命令的更多语法查看官方手册：https://dev.mysql.com/doc/refman/5.7/en/show-profile.html

## 4. 直接查询INFORMATION_SCHEMA.PROFILING表

`show profile`命令列出来的信息的确很详细，但是我们无法快速的确定哪个步骤花费的时间最多，因为输出的顺序是按照语句执行的顺序排列的。但是我们实际更关心的是哪些部分开销较大，但是很不幸show profile命令暂时不支持order by这样的排序功能。实际上profile的性能剖析信息都存在information_schema数据库的profiling表中。

```mysql
mysql> set @query_id=4;
Query OK, 0 rows affected (0.00 sec)

mysql> select state,sum(duration) as Total_R,
    ->    round(
    ->         100 * sum(duration) /
    ->             (select sum(duration) from information_schema.profiling
    ->              where query_id=@query_id
    ->         ),2) as Pct_R,
    ->    count(*) as Calls,
    ->    sum(duration) / count(*) as 'R/Call'
    -> from information_schema.profiling
    -> where query_id=@query_id
    -> group by state
    -> order by Total_R desc;
+----------------------+----------+-------+-------+--------------+
| state                | Total_R  | Pct_R | Calls | R/Call       |
+----------------------+----------+-------+-------+--------------+
| Opening tables       | 0.008439 | 54.47 |     2 | 0.0042195000 |
| System lock          | 0.003415 | 22.04 |     2 | 0.0017075000 |
| starting             | 0.002576 | 16.63 |     1 | 0.0025760000 |
| Sending data         | 0.000816 |  5.27 |     1 | 0.0008160000 |
| freeing items        | 0.000079 |  0.51 |     1 | 0.0000790000 |
| init                 | 0.000052 |  0.34 |     1 | 0.0000520000 |
| statistics           | 0.000026 |  0.17 |     1 | 0.0000260000 |
| preparing            | 0.000021 |  0.14 |     1 | 0.0000210000 |
| cleaning up          | 0.000019 |  0.12 |     1 | 0.0000190000 |
| checking permissions | 0.000011 |  0.07 |     1 | 0.0000110000 |
| closing tables       | 0.000009 |  0.06 |     1 | 0.0000090000 |
| optimizing           | 0.000008 |  0.05 |     1 | 0.0000080000 |
| logging slow query   | 0.000007 |  0.05 |     1 | 0.0000070000 |
| query end            | 0.000007 |  0.05 |     1 | 0.0000070000 |
| end                  | 0.000007 |  0.05 |     1 | 0.0000070000 |
| executing            | 0.000002 |  0.01 |     1 | 0.0000020000 |
+----------------------+----------+-------+-------+--------------+
16 rows in set, 17 warnings (0.01 sec)
```

> 上面的查询语句参考自《高性能MySQL》第三版3.3节。

通过这个查询可以清楚地看到最耗时的部分。

通过看[官方手册对explain的解释](https://dev.mysql.com/doc/refman/5.7/en/explain.html)发现，`describe`(可简写成`desc`)和`explain`可以混着用(MySQL解析器将它们视为同义词)，只是大多数情况下更喜欢用`desc`来查看表结构(事实上是对`show columns`命令的简化，主要为了兼容Oracle数据库)，使用`explain`来查看执行计划(即解析MySQL是如何执行查询语句的)。

# 使用explain获取执行计划

我们可以使用`explain`来获取`select`,`delete`,`insert`,`replace`,`update`这几个语句经过MySQL优化器优化后的执行计划。

```mysql
mysql> explain select * from test.emp \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: emp
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 4
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

> mysql里面默认以`;`作为一条完整语句的终结符，但是可以通过`delimiter`(缩写`\d`)命令进行修改，可以使用`go`(缩写`\g`)命令将SQL语句发送到服务器，使用`ego`(缩写`\G`)命令可以让返回的查询结果垂直显示。

下表是上面explain命令返回结果各列的意义(其中json name是以`explain format=json`执行计划分析返回的结果)：

| column          | json name       | meaning                                  |
| --------------- | --------------- | ---------------------------------------- |
| `id`            | `select_id`     | 该`SELECT`查询的标识符                          |
| `select_type`   | 没有              | 该`SELECT`查询的类型，常见的取值有`SIMPLE`,`PRIMARY`,[`UNION`](https://dev.mysql.com/doc/refman/5.7/en/union.html),[`SUBQUERY`](https://dev.mysql.com/doc/refman/5.7/en/correlated-subqueries.html),`DERIVED`(from子句中的子查询)等 |
| `table`         | `table_name`    | 查询用到了哪些表                                 |
| `partitions`    | `partitions`    | 匹配的分区（[分区表](https://dev.mysql.com/doc/refman/5.7/en/partitioning-info.html)） |
| `type`          | `access_type`   | 连接join类型(从好到坏)：`system -> const -> eq_ref -> ref -> fulltext -> ref_or_null -> index_merge -> unique_subquery -> index_subquery -> range -> index -> ALL`。详细请参考[官方手册](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-join-types)。 |
| `possible_keys` | `possible_keys` | 可能选用的索引(可能有多个)，可以在查询中用 `FORCE INDEX`，`USE INDEX`或`IGNORE INDEX`来强制使用(或不适用)某个特定的索引。 |
| `key`           | `key`           | 实际选择选用的索引，大多数情况下和`possible_keys`值相同。如果为NULL，说明语句没有用到索引。 |
| `key_len`       | `key_length`    | 使用索引的长度，这个值可以确定MySQL实际使用了索引的哪些部分(复合索引经常出现只用到索引的前几列的情况)。 |
| `ref`           | `ref`           | 与索引进行比较的列(哪些列用到了索引)                      |
| `rows`          | `rows`          | 要扫描的行的估计值(对于InnoDB，这个数字是估计值)             |
| `filtered`      | `filtered`      | 表条件过滤的行的百分比(`rows * filtered%`与前一张表进行join连接的行数) |
| `Extra`         | 没有              | 附加信息，常见的附加信息如：`Using index`,`Using filesort`,`Using join buffer`,`Using temporary`,`Using where`。更多附加信息参考[官方手册](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-extra-information) |

关于explain这部分内容就不赘述了，官方手册给了详细的解释和例子。




>参考：
>
>《高性能MySQL》
>
>MySQL官方手册：https://dev.mysql.com/doc/refman/5.7/en/
>
>http://blog.csdn.net/littleboyandgirl/article/details/68486642
