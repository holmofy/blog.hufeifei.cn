---
title: MySQL性能优化[实践篇]-索引合并与复合索引
date: 2018-05-20 15:02
categories: 数据库
tags: 
- DB
- MySQL
keywords:
- MySQL
- 性能优化
---

从[上一篇创建索引的实践中](https://blog.csdn.net/holmofy/article/details/80064823)，我们看到了索引给我们带来的性能提升是非常可观的。

我们上次创建的表结构非常简单，只有两三个字段，where子句查询条件只有一个字段。

实际应用场景中我们的表结构会更复杂，查询条件也会非常多。在多条件查询的情况下又如何才能用到索引呢，我们可以测试一下。

# 准备测试数据

创建表结构

```mysql
create table tb_test(id int primary key auto_increment,
                     c1 char(1),
                     c2 char(1),
                     c3 char(1),
                     c4 char(1));
```

生成随机字符的函数

```mysql
delimiter $$
# 生成随机字符串
create function rand_char() returns char(1)
begin
  declare CHARS char(52) default 'abcdefghijklmnopqrstuvwxyz';
  return substr(CHARS,floor(1+RAND()*52), 1);
end
$$
```

生成测试数据的存储过程

```mysql
create procedure insert_tb_test(c int)
begin
  declare i int default 0;
  set autocommit = 0;
  repeat
    set i = i + 1;
    insert into tb_test(c1,c2,c3,c4) values(rand_char(),rand_char(),rand_char(),rand_char());
  until i = c end repeat;
  set autocommit = 1;
end
$$

delimiter ;
```

调用存储过程生成一百万条测试数据：

```mysql
call insert_tb_test(1000000);
```



现在的查询场景是`select * from tb_test where c1=? and c2=? and c3=? and c4=?`

没有索引的情况下我们试着查询一次：

```mysql
mysql> select * from tb_test where c1='A' and c2='B' and c3='C' and c4='D';
Empty set (0.61 sec)  # 0.61秒，很慢
```

我们可以为c1、c2、c3、c4分别创建一个索引或者为他们创建一个复合索引。

我们先来试试第一种方案。

# 多个单列索引

创建多个索引

```mysql
alter table tb_test add index idx_tb_test_c1(c1);
alter table tb_test add index idx_tb_test_c2(c2);
alter table tb_test add index idx_tb_test_c3(c3);
alter table tb_test add index idx_tb_test_c4(c4);
```

创建索引后查询

```mysql
mysql> select * from tb_test where c1='A' and c2='B' and c3='C' and c4='D';
Empty set (0.05 sec)  # 花了 0.05秒

mysql> explain select * from tb_test where c1='A' and c2='B' and c3='C' and c4='D'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: index_merge
possible_keys: idx_tb_test_c1,idx_tb_test_c2,idx_tb_test_c3,idx_tb_test_c4
          key: idx_tb_test_c2,idx_tb_test_c4,idx_tb_test_c1,idx_tb_test_c3  # 用上了四个索引
      key_len: 4,4,4,4
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using intersect(idx_tb_test_c2,idx_tb_test_c4,idx_tb_test_c1,idx_tb_test_c3); Using where; Using index
1 row in set, 1 warning (0.00 sec)
```

我们在explain输出的内容上可以看到type为`index_merge`，并且附加信息上提示`Using intersect(idx_tb_test_c2,idx_tb_test_c4,idx_tb_test_c1,idx_tb_test_c3);`

这是MySQL5.0版后引入的的**“索引合并”**策略：**它将每个索引查询的结果进行合并**。

合并算法有三种`intersect`(交集)、`union`(并集)、`sort_union`(不常见)。

```mysql
mysql> explain select * from tb_test where c1='A' and c2='B' or c3='C' and c4='D'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: index_merge
possible_keys: idx_tb_test_c1,idx_tb_test_c2,idx_tb_test_c3,idx_tb_test_c4
          key: idx_tb_test_c2,idx_tb_test_c1,idx_tb_test_c4,idx_tb_test_c3
      key_len: 4,4,4,4
          ref: NULL
         rows: 2629
     filtered: 100.00
        Extra: Using union(intersect(idx_tb_test_c2,idx_tb_test_c1),intersect(idx_tb_test_c4,idx_tb_test_c3)); Using where
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from tb_test where c1='A' and c2='B' and c3>'C' and c4<'D'\G # 范围查询
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: index_merge
possible_keys: idx_tb_test_c1,idx_tb_test_c2,idx_tb_test_c3,idx_tb_test_c4
          key: idx_tb_test_c2,idx_tb_test_c1
      key_len: 4,4
          ref: NULL
         rows: 1300
     filtered: 25.00
        Extra: Using intersect(idx_tb_test_c2,idx_tb_test_c1); Using where # 范围查询的列不会进行索引合并
1 row in set, 1 warning (0.00 sec)
```

在某些时候索引合并是一种优化策略，但实际上更多时候它也说明表上的索引建的很糟糕：

* 当出现`using intersect`(多个AND条件)，更应该创建相关列的复合索引，而不是多个独立的单列索引。
* 当出现`using union`时，通常会耗费大量的CPU和内存资源用在缓存、排序、合并等操作上。特别是有的单列索引的选择性不高，需要合并扫描返回大量的数据。
* 更重要的是，MySQL优化器不会将这些操作算入“查询成本”，优化器只关心随机页面的读取。从而低估查询成本，可能导致执行计划还不如直接进行全表扫描。

如果在explain中看到`index_merge`，应该好好检查表结构和查询语句是否是最有的。

也可以通过[`optimizer_switch`](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_optimizer_switch)变量来关闭索引合并功能(具体参考[Switchable Optimizations](https://dev.mysql.com/doc/refman/5.6/en/switchable-optimizations.html))。

或者使用`force index`、`ignore index`提示优化器强制使用或忽略某些索引(具体参考[Index Hints](https://dev.mysql.com/doc/refman/5.6/en/index-hints.html))。

# 复合索引

删除之前的单列索引：

```mysql
 drop index idx_tb_test_c1 on tb_test;
 drop index idx_tb_test_c2 on tb_test;
 drop index idx_tb_test_c3 on tb_test;
 drop index idx_tb_test_c4 on tb_test;
```

创建复合索引：

```mysql
alter table tb_test add index idx_tb_test_c1_c2_c3_c4(c1,c2,c3,c4);
```

创建索引后查询：

```mysql
mysql> select * from tb_test where c1='A' and c2='B' and c3='C' and c4='D'\G
Empty set (0.00 sec)  # show profiles 显示只有0.00042275秒

mysql> explain select * from tb_test where c1='A' and c2='B' and c3='C' and c4='D'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4 # 用到的是复合索引
      key_len: 16  # 索引长度为16，用到了复合索引的所有列
          ref: const,const,const,const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
# 索引的使用与where条件的顺序没有任何关系，解析器将SQL解析成树后，这几个条件的层级是一样的
mysql> explain select * from tb_test where c3='A' and c1='B' and c4='C' and c2='D'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4
      key_len: 16
          ref: const,const,const,const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

很明显复合索引的速度和之前不是一个量级的。

# 选择合适的索引顺序

正确的索引顺序对查询性能也很重要。但是选择合适的索引顺序要从多方面考虑。

## 1. 范围查询放在索引最后面

```mysql
mysql> select * from tb_test where c1='A' and c2<'B' and c3='C' and c4='D'\G
Empty set (0.01 sec)  # show profiles显示消耗了0.01148575秒

mysql> explain select * from tb_test where c1='A' and c2<'B' and c3='C' and c4='D'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: range  # 范围查询
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4  # 用到了复合索引
      key_len: 8   # 但是只用到了索引的一半
          ref: NULL
         rows: 18572
     filtered: 1.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```

上面这个查询只用到了索引的一半，其中c1列精确匹配，c2列范围查询。

只针对这个SQL语句，我们需要把索引的顺序调整为`c1_c3_c4_c2`。

## 2. 索引列顺序尽量匹配`order by`子句的排序列顺序

之前说过B+树本身是有序的，使用B+树排序可以避免filesort。

```mysql
mysql> explain select * from tb_test where c1='A' and c2='B' order by c4,c3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref  # 精确匹配
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4  # 使用的复合索引
      key_len: 8    # 索引前两列精确匹配
          ref: const,const
         rows: 382
     filtered: 100.00
        Extra: Using where; Using index; Using filesort # 外排序
1 row in set, 1 warning (0.00 sec)
```

针对这个查询，索引的顺序应该设计为`c1_c2_c4_c3`。

```mysql
mysql> drop index idx_tb_test_c1_c2_c3_c4 on tb_test;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table tb_test add index idx_c1_c2_c4_c3(c1,c2,c4,c3);
Query OK, 0 rows affected (7.80 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from tb_test where c1='A' and c2='B' order by c4,c3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref  # 精确匹配
possible_keys: idx_c1_c2_c4_c3
          key: idx_c1_c2_c4_c3
      key_len: 8  # 索引前两列精确匹配
          ref: const,const
         rows: 382
     filtered: 100.00
        Extra: Using where; Using index  # 没有filesort
1 row in set, 1 warning (0.00 sec)

# 范围查询会导致后面的排序列用不上索引
mysql> explain select * from tb_test where c1='A' and c2<'B' order by c4,c3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: range
possible_keys: idx_c1_c2_c4_c3
          key: idx_c1_c2_c4_c3
      key_len: 8
          ref: NULL
         rows: 18572
     filtered: 100.00
        Extra: Using where; Using index; Using filesort  # 又变回filesort了
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from tb_test where c1='A' and c2<'B' order by c2,c4,c3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: range
possible_keys: idx_c1_c2_c4_c3
          key: idx_c1_c2_c4_c3
      key_len: 8
          ref: NULL
         rows: 18572
     filtered: 100.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```

## 3. 索引列顺序尽量匹配`group by`子句顺序

没有索引的情况下，`group by`子句执行`filesort`对数据进行排序，然后在进行分组。这个过程中**可能会导致全表扫描并且创建中间临时表**。

![Group By原理](http://tva1.sinaimg.cn/large/bda5cd74gy1frhxeblz75j20ud0gg3zp.jpg)

和`order by`子句相比，`group by`多了创建中间表和分组的操作，`group by`子句也是需要排序的。

```mysql
mysql> drop index idx_c1_c2_c4_c3 on tb_test;  # 删除索引
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from tb_test\G
*************************** 1. row ***************************
        Table: tb_test
   Non_unique: 0
     Key_name: PRIMARY  # 只剩下主键索引了
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 900858
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
1 row in set (0.00 sec)

mysql> explain select c1,c2 from tb_test group by c1,c2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ALL  # 全表扫描
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 998722  # 预估扫描99万多行数据
     filtered: 100.00
        Extra: Using temporary; Using filesort  # 临时表；文件排序
1 row in set, 1 warning (0.00 sec)
```

创建索引后查询：

```mysql
mysql> alter table tb_test add index idx_tb_test_c1_c2_c3_c4(c1,c2,c3,c4);
Query OK, 0 rows affected (7.55 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select c1,c2 from tb_test group by c1,c2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: range
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4
      key_len: 8  # 使用到索引的前两列
          ref: NULL
         rows: 727
     filtered: 100.00
        Extra: Using index for group-by # 使用索引执行group by子句
1 row in set, 1 warning (0.00 sec)

# 下面这个语句会用到c1,c2列进行精确匹配，c3,c4列用来分组
mysql> explain select count(c3) from tb_test where c1='A' and c2='B' group by c3,c4\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4
      key_len: 8
          ref: const,const
         rows: 382
     filtered: 100.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)

# 如果group by的顺序与索引顺序不一致,group by子句仍会filesort并创建中间表
mysql> explain select count(c3) from tb_test where c1='A' and c2='B' group by c4,c3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: ref
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4
      key_len: 8
          ref: const,const
         rows: 382
     filtered: 100.00
        Extra: Using where; Using index; Using temporary; Using filesort  # 中间表;文件排序
1 row in set, 1 warning (0.00 sec)

# 范围查询也会导致索引后面的列用不上索引进行group by
mysql> explain select count(c3) from tb_test where c1='A' and c2<'B' group by c3,c4\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_test
   partitions: NULL
         type: range
possible_keys: idx_tb_test_c1_c2_c3_c4
          key: idx_tb_test_c1_c2_c3_c4
      key_len: 8
          ref: NULL
         rows: 18572
     filtered: 100.00
        Extra: Using where; Using index; Using temporary; Using filesort # 中间表;文件排序
1 row in set, 1 warning (0.00 sec)
```

## 4. 选择性最高的列放在索引最前面

关于选择索引顺序有个经验：将选择性最高的列作为索引的最前列。在某些场景下确实很有帮助，但通常不如避免随机IO和排序那么重要。(很多时候需要全方面的考虑索引列的顺序)。

当不考虑排序和分组的时候，将选择性最高的列放在索引最前列通常是非常好的。索引的顺序将会优化where条件的查找：**这样设计的索引能最快的过滤出需要的行**。

```sql
create table tb_user(id int primary key auto_increment, age tinyint, gender tinyint, name varchar(16));
```

比如用户表中有性别、年龄、姓名这几列。很明显将性别作为索引最前列是不可取的，因为性别总共就两种。



下节待续：

假设某个表有一个联合索引(c1,c2,c3,c4)，问以下查询中只能使用该联合索引的c1,c2,c3部分的有那些

1. where c1=x and c2=x and c4>x and c3=x
2. where c1=x and c2=x and c4=x order by c3
3. where c1=x and c4=x group by c3,c2
4. where c1=? and c5=? order by c2,c3
5. where c1=? and c2=? and c5=? order by c2,c3



参考链接：

《高性能MySQL》

https://dev.mysql.com/doc/refman/5.7/en/index-merge-optimization.html

https://dev.mysql.com/doc/refman/5.7/en/multiple-column-indexes.html
