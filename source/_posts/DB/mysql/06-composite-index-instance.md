---
title: MySQL性能优化[实践篇]-复合索引实例
date: 2018-04-25
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

[上篇文章](https://blog.csdn.net/holmofy/article/details/80384637)最后提了个问题

假设某个表有一个**复合索引(c1,c2,c3,c4)**，问以下查询中只能使用该复合索引的c1,c2,c3部分的有那些

**1.** where c1=x and c2=x and c4>x and c3=x

**2.** where c1=x and c2=x and c4=x order by c3

**3.** where c1=x and c4=x group by c3,c2

**4.** where c1=? and c5=? order by c2,c3

**5.** where c1=? and c2=? and c5=? order by c2,c3

**建表测试**

![create table](http://tva1.sinaimg.cn/large/bda5cd74gy1fro2c2mibzj209s03xa9y.jpg)

测试表中有五个列(c1,c2,c3,c4,c5)，均为char(1)类型且不为空。字符集为utf8(**索引长度以字节数计算**)

**创建复合索引**

![add index](http://tva1.sinaimg.cn/large/bda5cd74gy1fro2im49q2j20fp01j3yd.jpg)

**插入几条测试数据**

![insert into](http://tva1.sinaimg.cn/large/bda5cd74gy1fro2mn7j5gj20i406r3yj.jpg)

> 这里插几条数据，主要是为了防止空表对SQL优化器的影响

# where c1=x and c2=x and c4>x and c3=x

用到了索引的所有部分，其中c1,c2,c3精确匹配，c4范围查询：

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1fro32xo8iij20k106z3yq.jpg)

这里**key_len=12**，因为每个utf8字符占3个字节（BMP平面字符）。

虽然utf8对`A`、`B`、`C`这几个英文字符的编码方式与ASCII是兼容的(也就是一个字节)，但char(1)为了保证能有足够的空间存储完整的utf8字符(比如中文)，它会尽量申请一个最大的单字符空间，不然将来修改字符比较麻烦。也就是说**MySQL会将原本utf8变长编码使用定长存储**。

而utf8四字节以上的字符都属于补充平面，几乎不可能用到，所以MySQL就取了3个字节一个字符，这三字节utf8在mysql中叫做[**utf8mb3**](https://dev.mysql.com/doc/refman/5.6/en/charset-unicode-utf8mb3.html)，mysql也支持4字节的utf8编码——[**utf8mb4**](https://dev.mysql.com/doc/refman/5.6/en/charset-unicode-utf8mb4.html)。MySQL中utf8指的就是utf8mb3。另外我们建表时对每个字段都指定了`not null`的约束，如果使用默认的`default null`会[多出一个字节](https://dev.mysql.com/doc/refman/5.7/en/innodb-physical-record.html)。

> 关于utf-8编码原理可以参考[《从ASCII、ISO-8859、GB2312、GBK到Unicode的UCS-2、UCS-4、UTF-8、UTF-16、UTF-32》](https://blog.csdn.net/holmofy/article/details/72846118)
>
> 关于MySQL对Unicode编码的支持可以参考[《Unicode Support》](https://dev.mysql.com/doc/refman/5.6/en/charset-unicode.html)

**Using index condition**

出现`Using index condition`意味着**没有达到索引覆盖**。

查询语句通过索引过滤出几条记录，但是查询的内容超出索引范围，需要读取完整的数据行(这个过程也被叫做[ICP，Index Condition Pushdown](https://dev.mysql.com/doc/refman/5.7/en/index-condition-pushdown-optimization.html))。

出现ICP主要是因为我们用了`select *`。我们把SQL稍微改动一下，让它能达到**索引覆盖**

![索引覆盖](http://tva1.sinaimg.cn/large/bda5cd74gy1fro3m5i6a9j20lh071wet.jpg)

# where c1=x and c2=x and c4=x order by c3

用到了索引的c1,c2,c3列，其中c1、c2列用于查询，c3用于排序。由于c3列没有精确匹配，导致c4列无法用到索引。

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1fro40gcgebj20jl0760t2.jpg)

[**type: ref**](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_ref)

ref指的是从表中读取匹配索引值的所有行。type=ref说明使用了索引的左前缀，或者完整地使用了索列但是索引不是primary key或unique key。

换句话说type=ref表明，查询语句不能通过索引查找到单独一行数据。

相反[**type: eq_ref**](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_eq_ref)就是使用了primary key或unique key的查询，这种查询能从表中唯一一条记录。

#  where c1=x and c4=x group by c3,c2

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1froty4ypchj20kl07amxh.jpg)

group by子句执行时会先排序，再分组。这条语句由于group by的顺序为c3,c2与索引顺序不匹配，所以没用到索引。

![group by](http://tva1.sinaimg.cn/large/bda5cd74gy1frhxeblz75j20ud0gg3zp.jpg)

我们把group by的顺序调换一下就能然c2和c3列能用上索引进行排序分组。

![优化](http://tva1.sinaimg.cn/large/bda5cd74gy1frou412zw6j20jy075aae.jpg)

# where c1=? and c5=? order by c2,c3

因为group by本质上也会执行order by操作，所以这条语句原理上和上面的差不多。

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1frou877wxfj20jk0723yr.jpg)

# where c1=? and c2=? and c5=? order by c2,c3

这条查询和上条略有不同c1列和c2列已经使用索引精确匹配了，而order by再对c2进行排序已经没有意义了，因为过滤后的数据c2都是相等的，所以实际上只有c3列才用到排序。

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1frouaplzopj20kg06zt8y.jpg)

这个时候的修改order by中c2、c3列的顺序没有任何关系，因为c2列已经精确匹配了。

![explain](http://tva1.sinaimg.cn/large/bda5cd74gy1frouf4qnxlj20kh076q3d.jpg)



**1.** where c1=x and c2=x and c4>x and c3=x

​	用到(c1,c2,c3,c4)列进行数据查找

**2.** where c1=x and c2=x and c4=x order by c3

​	用到(c1,c3)列进行数据查找,c3列索引排序

**3.** where c1=x and c4=x group by c3,c2

​	只是用了(c1)列进行数据查找

**4.** where c1=? and c5=? order by c2,c3

​	使用(c1)列进行数据查找,c2,c3列索引排序

**5.** where c1=? and c2=? and c5=? order by c2,c3

​	使用(c1,c2)列进行数据查找,c3列索引排序



> 这个问题原出自一个[论坛](http://www.zixue.it/thread-9218-1-1.html)，这里重新测试并对结果稍作整理

参考：

* https://dev.mysql.com/doc/refman/5.7/en/explain-output.html
