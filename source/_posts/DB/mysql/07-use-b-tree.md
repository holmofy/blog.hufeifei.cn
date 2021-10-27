---
title: MySQL性能优化[实践篇]-使用B树索引
date: 2018-04-26
categories: 数据库
tags: 
- DB
- MySQL
keywords:
- MySQL
- 性能优化
---

**系列文章：**

* [MySQL性能优化[理论篇]-B树索引与hash索引](https://blog.hufeifei.cn/2018/04/DB/mysql/01-b-tree-hash-index/)
* [MySQL性能优化[理论篇]-聚簇索引和非聚簇索引,InnoDB和MyISAM](https://blog.hufeifei.cn/2018/04/DB/mysql/02-cluster-index/)
* [MySQL性能优化[准备篇]-慢查询日志](https://blog.hufeifei.cn/2018/04/DB/mysql/03-slow-log/)
* [MySQL性能优化[准备篇]-单条SQL性能剖析](https://blog.hufeifei.cn/2018/04/DB/mysql/04-profiling)
* [MySQL性能优化[实践篇]-索引合并与复合索引](https://blog.hufeifei.cn/2018/04/DB/mysql/05-index-merge-composite-index/)
* [MySQL性能优化[实践篇]-复合索引实例](https://blog.hufeifei.cn/2018/04/DB/mysql/06-composite-index-instance/)
* [MySQL性能优化[实践篇]-使用B树索引](https://blog.hufeifei.cn/2018/04/DB/mysql/07-use-b-tree/)
* [分库分表的一些思考](https://blog.hufeifei.cn/2020/04/DB/Alibaba/TDDL/)

---

# 准备测试数据

```mysql
create table tb_user(
  id int auto_increment primary key,
  name varchar(10),
  birth date
);
```

使用存储过程创建测试数据，不过在这之前我们先创建两个工具函数。

> mysql里面默认以`;`作为一条完整语句的终结符，为了不与函数和存储过程中的`;`冲突，我们要提前使用`delimiter`命令将命令结束符修改成其他符号。
>
> 比如用`delimiter $$`可以将结束符修改为`$$`。
>
> 在定义完函数后用`$$`将缓冲区中的命令发送到服务器处理，然后有必要的话建议把终结符设置回默认的`;`

```mysql
delimiter $$
# 生成随机字符串
create function rand_str(n int) returns varchar(10)
begin
  declare CHARS char(52) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  declare result varchar(255) default '';
  declare i int default 1;
  while i < n do
    set result = concat(result, substr(CHARS,floor(1+RAND()*52), 1));
    set i = i + 1;
  end while;
  return result;
end
$$

# 生成随机数(i <= R < j)
create function rand_num(i int, j int) returns int
begin
  return floor(i + rand() * (j - i));
end
$$

delimiter ;
```

测试数据存储过程：

```mysql
delimiter $$
create procedure insert_tb_user(c int)
begin
  declare start_date int default TO_DAYS(STR_TO_DATE('1970-01-1','%Y-%m-%e'));
  declare end_date int default TO_DAYS(CURDATE());
  declare i int default 0;
  set autocommit = 0;
  repeat
    set i = i + 1;
    insert into tb_user(name,birth) values(rand_str(10),FROM_DAYS(rand_num(start_date,end_date)));
  until i = c end repeat;
  set autocommit = 1;
end
$$
delimiter ;
```

调用存储过程生成一百万条随机数据（可能需要两三分钟）：

```mysql
call insert_tb_user(1000000);
```

# 未创建索引

为了能更细致的看到每个查询的耗时情况，我们把`profiling`打开。

```mysql
mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

## 使用主键`id`查询

```mysql
mysql> select * from tb_user where id=100000;
+--------+-----------+------------+
| id     | name      | birth      |
+--------+-----------+------------+
| 100000 | UZlihlHEY | 1975-10-09 |
+--------+-----------+------------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+---------------------------------------+
| Query_ID | Duration   | Query                                 |
+----------+------------+---------------------------------------+
|        1 | 0.00048275 | select * from tb_user where id=100000 |
+----------+------------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

使用`id`查询耗时不到**0.5毫秒**(不同机器肯定会有所区别)。

> MySQL默认创建的主键索引就是B树索引，所以主键查询速度会很快。

```mysql
# tb_user目前只有一个主键索引
mysql> show index from tb_user\G
*************************** 1. row ***************************
        Table: tb_user
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id     # id列
    Collation: A
  Cardinality: 977600
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE  # 主键索引也是B树索引
      Comment:
Index_comment:
1 row in set (0.00 sec)

mysql> explain select * from tb_user where id=100000\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY  # 用到了主键索引
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

> 关于主键还有很多内容，会在后续文章中介绍。

## 使用`name`字段查询

```mysql
mysql> select * from tb_user where name='UZlihlHEY';
+--------+-----------+------------+
| id     | name      | birth      |
+--------+-----------+------------+
| 100000 | UZlihlHEY | 1975-10-09 |
+--------+-----------+------------+
1 row in set (0.42 sec)

mysql> show profiles;
+----------+------------+-----------------------------------------------+
| Query_ID | Duration   | Query                                         |
+----------+------------+-----------------------------------------------+
|        1 | 0.00048275 | select * from tb_user where id=100000         |
|        2 | 0.00061350 | show index from tb_user                       |
|        3 | 0.00052125 | explain select * from tb_user where id=100000 |
|        4 | 0.42458000 | select * from tb_user where name='UZlihlHEY'  |
+----------+------------+-----------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```

使用`id`查询耗时达到**0.42秒**，与主键查询相比速度**慢了近千倍**。

我们可以用`show profile`比较两个查询，看看`name`字段查询慢在哪里。

```mysql
mysql> show profile for query 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000106 |
| checking permissions | 0.000008 |
| Opening tables       | 0.000025 |
| init                 | 0.000031 |
| System lock          | 0.000016 |
| optimizing           | 0.000013 |
| statistics           | 0.000076 |
| preparing            | 0.000013 |
| executing            | 0.000002 |
| Sending data         | 0.000016 |
| end                  | 0.000004 |
| query end            | 0.000007 |
| closing tables       | 0.000010 |
| freeing items        | 0.000118 |
| cleaning up          | 0.000040 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)

mysql> show profile for query 4;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000073 |
| checking permissions | 0.000008 |
| Opening tables       | 0.000018 |
| init                 | 0.000030 |
| System lock          | 0.000011 |
| optimizing           | 0.000010 |
| statistics           | 0.000018 |
| preparing            | 0.000013 |
| executing            | 0.000002 |
| Sending data         | 0.424281 |  # 这个阶段最耗时
| end                  | 0.000013 |
| query end            | 0.000009 |
| closing tables       | 0.000009 |
| freeing items        | 0.000071 |
| cleaning up          | 0.000016 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)
```

不要被`Sending data`的字面意思给误导了，它并不是指网络传输的速度慢，`executing`状态的时间也并不是整个SQL语句执行的时间。

[MySQL官方手册](https://dev.mysql.com/doc/refman/8.0/en/general-thread-states.html)对这两种状态的解释如下：

```
executing
The thread has begun executing a statement.
线程已经开始执行语句

Sending data
The thread is reading and processing rows for a SELECT statement, and sending data to the client. 
Because operations occurring during this state tend to perform large amounts of disk access (reads), 
it is often the longest-running state over the lifetime of a given query.
线程正在读取并处理select语句选择的行数据，然后将数据发送给客户端。
因为这个状态期间的操作偏重执行大量的磁盘访问(读取磁盘)，
它通常是整个查询生命周期中运行时间最长的状态。
```

事实证明**磁盘IO才是罪魁祸首**。

# 对`name`字段创建索引

```mysql
# 不加“using btree”子句，mysql也会默认创建B树索引
mysql> create index idx_tb_user_name on tb_user(name);
Query OK, 0 rows affected (5.32 sec)
Records: 0  Duplicates: 0  Warnings: 0

# 现在有两条索引了
mysql> show index from tb_user \G
*************************** 1. row ***************************
        Table: tb_user
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 977600
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
*************************** 2. row ***************************
        Table: tb_user
   Non_unique: 1
     Key_name: idx_tb_user_name
 Seq_in_index: 1
  Column_name: name      # name列创建的索引
    Collation: A
  Cardinality: 987287
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE     # B树索引
      Comment:
Index_comment:
2 rows in set (0.00 sec)
```

>  创建索引的详细语法可以参考官方文档[`create index`](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)和 [`alter table`](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html)

# 创建索引后查询

```mysql
mysql> select * from tb_user where name='UZlihlHEY';
+--------+-----------+------+
| id     | name      | age  |
+--------+-----------+------+
| 100000 | DnGrPZAGD |    2 |
+--------+-----------+------+
1 row in set (0.00 sec)

# 创建索引后耗时从原来的0.5秒缩短到现在的0.67毫秒
mysql> show profiles;
+----------+------------+------------------------------------------------+
| Query_ID | Duration   | Query                                          |
+----------+------------+------------------------------------------------+
|        1 | 0.00048275 | select * from tb_user where id=100000          |
|        2 | 0.00061350 | show index from tb_user                        |
|        3 | 0.00052125 | explain select * from tb_user where id=100000  |
|        4 | 0.42458000 | select * from tb_user where name='UZlihlHEY'   |
|        5 | 5.31390800 | create index idx_tb_user_name on tb_user(name) |
|        6 | 0.00033175 | show index from tb_user                        |
|        7 | 0.00046250 | select * from tb_user where name='UZlihlHEY'   |
+----------+------------+------------------------------------------------+
7 rows in set, 1 warning (0.00 sec)

mysql> explain select * from tb_user where name='UZlihlHEY'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: ref
possible_keys: idx_tb_user_name
          key: idx_tb_user_name     # 使用了刚刚创建的idx_tb_user_name索引
      key_len: 33
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

> 即使创建了索引，使用`name`字段查询仍然没有主键查询的速度快，这是因为`name`字段创建的索引是二级索引，InnoDB的二级索引叶子节点并不保存实际的数据行或数据行的引用，而是保存了主键id。二级索引走了一遍得到主键id，还要拿着主键id在主键索引再走一遍，才能查到数据行。关于这部分内容会在后续的文章中详细介绍。

# 正确使用索引

索引虽然能加快我们的查询速度，但是如果姿势不对还是不能达到预期的速度。

## 1. 避免索引列在表达式中出现

```mysql
# 索引列在表达式中
mysql> select * from tb_user where id+30000=130000;
+--------+-----------+------+
| id     | name      | age  |
+--------+-----------+------+
| 100000 | DnGrPZAGD |    2 |
+--------+-----------+------+
1 row in set (0.38 sec)

# SQL解析器无法优化这样的等式
mysql> explain select * from tb_user where id+30000=130000\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: ALL   # 全表扫描
possible_keys: NULL
          key: NULL  # 没有用到主键索引
      key_len: NULL
          ref: NULL
         rows: 998430
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

# 索引列单独在等式的一端，仍可以使用索引
mysql> select * from tb_user where id=30000+70000;
+--------+-----------+------+
| id     | name      | age  |
+--------+-----------+------+
| 100000 | DnGrPZAGD |    2 |
+--------+-----------+------+
1 row in set (0.00 sec)

mysql> explain select * from tb_user where id=30000+70000\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

## 2. 避免索引列作为函数参数 

```mysql
# 如果birth列创建了索引，但查询条件中这一列作为函数参数，birth这个索引就不会被用到
select count(*) from tb_user where YEAR(current_date) - YEAR(birth) >= 18;
```

## 3. 前缀索引

进行**字符串模糊匹配**的时候，经常会用`like`操作符，通配符开头的字符串匹配无法使用索引，

```mysql
mysql> select * from tb_user where name like '%UZlih%';
+--------+-----------+------------+
| id     | name      | birth      |
+--------+-----------+------------+
|  33090 | ZjUZlihkF | 1988-08-03 |
|  37993 | iUZlihlKQ | 1970-08-07 |
...// 省略
| 903589 | mgTauZLIh | 2004-12-12 |
| 999434 | IUzLiHNSa | 1994-03-06 |
+--------+-----------+------------+
35 rows in set (0.51 sec)  # 耗时0.50994975秒

mysql> explain select * from tb_user where name like '%UZlih%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: ALL    # 全表扫描
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 998244
     filtered: 11.11
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

mysql> select * from tb_user where name like 'UZlih%';
+--------+-----------+------------+
| id     | name      | birth      |
+--------+-----------+------------+
| 309324 | uZLIhKgBm | 2001-03-18 |
| 100000 | UZlihlHEY | 1975-10-09 |
| 891682 | uZLIhLlUQ | 1988-06-24 |
| 568787 | uzlIHmnFl | 1982-07-30 |
| 621614 | uZLIhMnfM | 1987-08-30 |
|  41360 | uZLIhMpmq | 2011-08-25 |
|  42195 | UzLiHNSYo | 1993-04-18 |
| 600167 | uZLIhNtFU | 2002-05-30 |
+--------+-----------+------------+
8 rows in set (0.00 sec)   # 耗时0.00046500秒

mysql> explain select * from tb_user where name like 'UZlih%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_user
   partitions: NULL
         type: range             # 范围查询
possible_keys: idx_tb_user_name
          key: idx_tb_user_name  # 用到了name索引
      key_len: 33
          ref: NULL
         rows: 8
     filtered: 100.00
        Extra: Using index condition  # 使用索引进行条件查询
1 row in set, 1 warning (0.00 sec)
```



参考：

《高性能MySQL》

https://dev.mysql.com/doc/refman/5.7/en/index-btree-hash.html

https://dev.mysql.com/doc/refman/5.7/en/index-condition-pushdown-optimization.html
