---
title: 传统关系型数据库的三宗罪
date: 2021-05-08 19:50
categories: DB
---

计算机说穿了就是存储/IO/CPU三大件，也就是冯诺依曼机中概括的存储器+运算器+控制器+输入设备+输出设备。而计算说白了就两个东西：数据与算法。常见的软件应用，除了机器学习模型训练、图片音视频加解码、游戏图形渲染等计算密集型的应用外，绝大多数应用都是数据密集型应用。从最抽象的意义上讲，，这些应用就是把数据拿进来，存进数据库，需要的时候再拿出来。

互联网应用大多都属于数据密集型应用，这就导致大多数应用只是基于数据库进行开发，数据表就是数据结构，索引和查询就是算法。而应用代码往往只是扮演胶水的角色，处理IO和业务逻辑，其他大部分工作都是在各种各样的数据系统之间搬运数据。

* 数据库：存储数据，以便自己或其他应用程序之后能再次找到（PostgreSQL，MySQL，Oracle）
* 缓存：记住开销昂贵操作的结果，加快读取速度（Redis，Memcached）
* 搜索索引：允许用户按关键字搜索数据，或以各种方式对数据进行过滤（ElasticSearch）
* 流处理：向其他进程发送消息，进行异步处理（Kafka，Flink，Storm）
* 批处理：定期处理累积的大批量数据（Hadoop）

传统的关系性数据库，作为一个已有[四五十年寿命](https://en.wikipedia.org/wiki/Relational_database#History)的系统，了解[它的工作原理](http://coding-geek.com/how-databases-work/)和它的局限性是非常有必要的。这篇文章会以世界上最流行的关系性数据库MySQL为对象，聊一聊它的局限性，以及相应的解决方案。

# 1、schema变更——大表DDL

关系型数据库需要预先定义数据模型，并且数据需要与模型匹配才能被存储在数据库中，这种严格的存储方式提供了一定的安全性，但也丧失了灵活性。

面对互联网的飞速变化，应用会经常加功能改需求，表结构也会经常修改，加字段，加索引。直接在生产环境的表中在线修改表结构，对用户使用网站是有影响。

由于MySQL修改表结构的过程如下：

1. 对表加锁(表此时只读)
2. 复制原表物理结构
3. 修改表的物理结构
4. 把原表数据导入中间表中，数据同步完后，锁定中间表，并删除原表
5. rename中间表为原表
6. 刷新数据字典，并释放锁

在这个过程中会锁表。造成当前操作的表无法写入数据，影响用户使用。由于需要复制原表的数据到中间表，所以表的数据量越大，等待的时候越长，卡死在那里(用户被拒绝执行update和insert操作,表现就是延迟了一直在等待)。

> 正因为关系型数据库的这个原因，催生了MongoDB这样的schemaless的[文档数据库](https://en.wikipedia.org/wiki/Document-oriented_database)。不过MySQL5.7.8引入了[JSON数据类型](https://dev.mysql.com/doc/refman/5.7/en/json.html)并提供了相应的[JSON函数](https://dev.mysql.com/doc/refman/5.7/en/json-functions.html)，结合5.7.5引入的[Generate Column](https://dev.mysql.com/doc/refman/5.7/en/create-table-generated-columns.html)为JSON内的字段创建索引，基本上可以实现MongoDB的schemeless。

解决方案：**这个问题的关键在于不能阻塞原表的数据修改。难点在于拷贝到新表的过程中，后续老表上的增删改操作也需要更新到新表中**。

### 1、[pt-osc](https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html)

percona推出的一个针对mysql在线ddl的工具pt-online-schema-change, 它[基于触发器](https://mydbops.wordpress.com/2018/03/12/online-schema-change-with-for-tables-with-triggers/)将后续的增删改操作更新到新表中。

采用触发器方式，需要解决增量回放与全量拷贝乱序问题。所以pt-osc经常出现死锁、在非唯一列上加唯一索引丢数据的问题，具体可以看[percona社区的讨论](https://forums.percona.com/search?q=pt-online-schema-change)

### 2、[gh-ost](https://github.com/github/gh-ost)

gh-ost是github推出的在线ddl工具，它使用binlog+回放线程来替换掉pt-osc的触发器方式。可以参考github团队的[这篇博客](https://github.blog/2016-08-01-gh-ost-github-s-online-migration-tool-for-mysql/)看他们是怎么解决这个问题的。

### 3、MySQL5.6之后的[Online DDL](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl.html)

Online DDL在MySQL 5.6才开始支持的，在5.5及之前版本，使用alter table/create index等命令进行表结构修改操作均会锁表，这在生产环境上是不可接受的。

在MySQL 5.7，Online DDL在性能和稳定性上不断得到优化，比如通过bulk load方式来去除表重建时的redo日志等。到了MySQL 8.0，Online DDL已经支持秒级加列特性，该特性来源于国内的腾讯互娱DBA团队。

与pt-osc/gh-ost不同，Online DDL的执行阶段又可以分为前后2个步骤，首先是拷贝全量数据，然后才回放增量DML日志。在全量拷贝期间，增量DML日志被保存在日志文件中，由innodb_online_alter_log_max_size 参数确定文件最大阈值，若积累的DML日志超过该阈值，则DDL操作返回失败。

refs: 

* mysql在线修改表结构大数据表的风险与解决办法归纳: https://www.cnblogs.com/wangtao_20/p/3504395.html

* X-Engine 如何实现 Fast DDL: https://developer.aliyun.com/article/766069

* MySQL 8.0 Online DDL和pt-osc、gh-ost深度对比分析: https://zhuanlan.zhihu.com/p/115277009

# 2、伸缩性

根据[CAP定理](https://en.wikipedia.org/wiki/CAP_theorem)中论述，系统最多只能提供CAP中的两项，系统必定要牺牲CAP中的某个特性：

* P([Partition tolerance](https://en.wikipedia.org/wiki/Network_partitioning)): 放弃分区容错性的话，则放弃了分布式，放弃了系统的可扩展性。
* A([Availability](https://en.wikipedia.org/wiki/Availability)): Reads and writes always succeed。放弃可用性的话，则在遇到网络分区或其他故障时，受影响的服务需要等待一定的时间，再此期间无法对外提供正常读写服务，即不可用。
* C([Consistency](https://en.wikipedia.org/wiki/Consistency_model)): All nodes see the **same data** at the **same time**。这里的一致性与ACID的一致性不是一个概念，而是由于分布式节点之间数据复制的replica要一致——所有的节点**相同时间**看到的**数据是一致**的，也即“**强一致性**”。放弃一致性的话（这里指**强一致**），则系统无法保证数据保持实时的一致性，在数据达到最终一致性前，有个时间窗口，在时间窗口内，数据是不一致的。

![CAP理论](https://p.pstatp.com/origin/pgc-image/b03323884ed74e6ab283243cfb355d45)

像MongoDB和ElasticSearch等NoSQL，原生就带了[replication](https://en.wikipedia.org/wiki/Replication_%28computing%29)和[sharding](https://en.wikipedia.org/wiki/Shard_%28database_architecture%29)，这也是它们伸缩性好的一个原因。



MySQL提供了基于binlog的Replication，但是



Refs: 

* 谈谈分布式系统的CAP理论： https://zhuanlan.zhihu.com/p/33999708

* A Primer on Database Replication：https://www.brianstorti.com/replication/

# Search

ES 多条件联合查询

Skip List和 Bitset

# Analysis







refs:

https://www.mongodb.com/compare/mongodb-mysql

https://www.zhihu.com/question/273489729/answer/377084748
