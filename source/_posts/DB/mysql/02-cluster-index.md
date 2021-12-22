---
title: MySQL性能优化[理论篇]-聚簇索引和非聚簇索引,InnoDB和MyISAM
date: 2018-04-21
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

## 聚簇索引

聚簇索引(Clustered Index)并不是一种新的数据结构，只是B树索引的一种存储方式。

聚簇索引的特点是完整的数据行就放在B树的叶子结点中，Clustered(聚簇,集群)就表示数据行与对应的键紧凑的存储在一起。

下图是《高性能MySQL》聚簇索引的截图，其中，叶子结点包含了数据行的完整数据，非叶子节点只包含索引列数据。

![聚簇索引](http://tva1.sinaimg.cn/large/bda5cd74gy1frp2q4d6i3j21200r0jx5.jpg)

数据行的逻辑顺序与聚簇索引的顺序一致。B+树中叶子结点以链表的形式串联的，叶子节点中数据行的逻辑顺序只有一种，所以**一张表只能有一个聚簇索引**。

相反非聚簇索引指的就是数据行和对应的键不存在一起，**非聚簇索引中叶子结点一般存的是数据行的引用**。

MySQL中索引由存储引擎实现，不同的存储引擎索引的存储结构是不一样的，**不是所有的存储引擎都支持聚簇索引**。

MySQL支持很[多种存储引擎](https://en.wikipedia.org/wiki/Comparison_of_MySQL_database_engines)(包括官方的和第三方的)，使用最多的就是[InnoDB(5.5之后默认的存储引擎)](https://en.wikipedia.org/wiki/InnoDB)和[MyISAM(5.5之前默认的存储引擎)](https://en.wikipedia.org/wiki/MyISAM)。

## [InnoDB存储引擎](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html)

**InnoDB中主键索引默认是聚簇的**。

如果表中没有定义主键，InnoDB会选择一个所有列非空的Unique索引作为聚簇索引。

如果表中既没有主键也没有非空的唯一索引，InnoDB内部会生成一个名为`GEN_CLUST_INDEX`的隐式聚簇索引。这个隐式的聚簇索引中包含每个数据行的RowID，并以RowID作为隐式主键进行排序。RowID是一个6字节的字段，并随着新行的插入而单调递增，隐式索引中的数据行在物理结构上是按插入顺序顺序存储的。

绝大多数情况下我们都会定义主键的，所以上面说的两种没有主键的情况下面都不考虑。

**InnoDB的二级索引会在叶子结点保存主键**。

当通过二级索引查找时，**InnoDB需要通过二级索引的叶子结点获得对应的主键，然后根据主键从主键索引中找对应的行**。也就是说**InnoDB查找数据行会经过两个B树**。这样做有个好处就是**减少了数据行移动或数据页分裂时二级索引的维护工作**，但是查找速度会略微变慢，有一种解决方法是实现**索引覆盖，让二级索引直接覆盖所需的查询字段**，这样就没必要查找数据行，也就没必要走主键索引了。

![InnoDB索引存储结构](http://tva1.sinaimg.cn/large/bda5cd74gy1frp7kvqfwgj20lr09y76c.jpg)



![B+树索引](http://tva1.sinaimg.cn/large/bda5cd74ly1fyrfmroao8j20ry0k379y.jpg)

#### 聚簇索引的优点

1. 聚簇索引将索引和数据行保存在同一个B-Tree中，查询通过聚簇索引可以直接获取数据，而非聚簇索引要进行多次I/O，所以聚簇索引通常比非聚簇索引查找更快。
2. 聚簇索引对主键范围查询的效率很高，因为其数据是按照主键排列的
3. 二级索引使用索引覆盖可以直接使用叶节点的主键值。

#### 聚簇索引的缺点

1. 聚簇索引最大限度地提高了I/O密集型应用的性能，但如果数据都存放在内存中，则访问顺序就不那么重要了，聚簇索引也没什么优势。
2. 插入速度严重依赖于插入顺序。按照主键顺序往InnoDB中进行数据导入是最快的。如果不是按照主键插入，最好在导入完成后使用[`OPTIMIZE TABLE`](https://dev.mysql.com/doc/refman/5.7/en/optimize-table.html)命令重新组织一下表。
3. 聚簇索引在插入新行和更新主键时，可能导致“页分裂”问题：当插入到某个已满的叶子结点时，B+树会分裂成两个页来容纳新插入的行数据。页分裂会导致表占用更多的磁盘空间（不要用UUID或随机数做主键，而应该使用单调递增的值做主键）。
4. 聚簇索引可能导致全表扫描速度变慢，因为可能需要加载物理上相隔较远的页到内存中（需要耗时的磁盘寻道操作）。
5. 二级索引访问数据行需要两次索引查找，由于二级索引保存了主键列，二级索引会占更大的空间(所以选用一个短主键是有利的)。

## [MyISAM存储引擎](https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html)

InnoDB通过主键聚集数据，整个聚簇索引就是一张完整的表。MyISAM存储引擎的数据相对简单。

MyISAM([ISAM，indexed sequential access method](https://en.wikipedia.org/wiki/ISAM))的数据行和索引是分开存储的：数据文件以`.MYD`为后缀，索引文件以`.MYI`为后缀(Innodb只有一个`.idb`文件)。

数据行按照数据插入顺序存储在数据文件中，数据行的存储支持多种[行存储格式](https://dev.mysql.com/doc/refman/5.7/en/myisam-table-formats.html)(ROW_FORMAT)。

**MyISAM没有聚簇索引，主键索引和二级索引工作方式是一样的：叶子结点存储数据行在数据文件中的物理偏移量(行指针)**

![MyISAM索引存储结构](http://tva1.sinaimg.cn/large/bda5cd74gy1frp7htdkc2j20fn0cxabx.jpg)

#### MyISAM优点

1. 读取数据行的速度快，特别是当数据行长度固定的时候。
2. 数据行插入容易，新行直接追加到数据文件末尾。

#### MyISAM缺点

1. 删除操作必须留出空白区域，否则后面的行数据偏移将会发生变化。正因如此当大量删除MyISAM表数据后，数据文件大小不会发生变化，所以要定期执行`OPTIMIZE TABLE`操作对MyISAM碎片空间进行整理。
2. 修改操作数据行长度如果变短，也会留白；数据行如果变长，数据将会分段存储。



MyISAM还有几个重要优缺点：

* 支持[全文(FULLTEXT)索引](https://en.wikipedia.org/wiki/Full-text_search)
* 不支持事务、不支持外键、不支持行级锁。

MyISAM更多具体内容参考https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html

InnoDB在5.6.4之后支持了全文索引，不过MySQL的全文索引都很鸡肋，都不支持中文。

InnoDB支持事务，外键，行级锁。这些都是MySQL使用InnoDB替代MyISAM作为默认存储引擎的理由。

InnoDB更多具体内容参考https://dev.mysql.com/doc/refman/5.7/en/innodb-introduction.html



参考：

《高性能MySQL》

https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html
