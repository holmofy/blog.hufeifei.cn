---
title: 基于Binlog的实时同步功能——debezium、canel、databus技术选型
date: 2021-03-13 14:23
categories: JAVA
---

大厂这段时间学到的最有价值的两个技术除了大数据，另一个就是基于CDC和消息队列的实时同步技术。

[去年的一篇文章大致地讲了我对MQ的一些认识](https://blog.hufeifei.cn/2020/04/25/Alibaba/MetaQ&Notify/)，事实上Kafka在内的现代MQ，功能远不止这些。后面一定整理好自己的思路，肯定会在写一篇文章来讲讲。这篇文章的主角就是与MQ息息相关的CDC技术。

# 1、CDC技术

[CDC](https://en.wikipedia.org/wiki/Change_data_capture)全称叫：change data capture，是一种基于数据库数据变更的事件型软件设计模式。

比如有一张订单表trade，订单每一次变更录入到一张trade_change的队列表。然后另外一个调度线程可以消费trade_change这张队列表来做一些数据统计，如每日的付款用户统计、每日的下单用户统计等。

这就是我毕业入职的第一家公司的报表统计逻辑。这个设计在订单量小的时候是看不出问题的，而一旦某一时刻订单量增多。基于MySQL的队列表由于B+树的写入吞吐量不够，导致MySQL CPU经常飙升。比如双十一，618这样的大促，程序员就得在颤颤巍巍中度过。

> B+的写入性能肯定是不如直接顺序写文件的，所以大部分数据库都会设计Buffer来管理B+树脏页，来避免频繁的随机IO。(B+树的本质就是)

![database architecture](https://p.pstatp.com/origin/pgc-image/85c4945860ba4dd793acc42691226c8a)

# 2、基于Binlog的CDC

[Binlog](https://dev.mysql.com/doc/internals/en/binary-log.html)是MySQL 3.23.14引进的，[它包含所有的描述数据库修改的事件](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)——DML(增删改)、DDL(表结构定义与修改)操作。

![MySQL architecture](https://p.pstatp.com/origin/pgc-image/f20dc094cd6a4163829e616e02197acd)

与InnoDB中的[redo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)、[undo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)不同，binlog和[slow_query_log](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)一样是server层的日志，所以InnoDB和MyISAM等各种存储引擎的数据修改都会记录到这个日志中。

> MySQL拥有分层架构，支持可插拔的存储引擎，而像PostgreSQL这样的数据库，[WAL日志](https://www.postgresql.org/docs/8.1/wal-intro.html)用于保证事务的持久性，在Replica过程中也扮演着与MySQL的binlog相同的角色。

![CDC architecture](https://debezium.io/documentation/reference/1.4/_images/debezium-architecture.png)

对于CDC的架构设计，在大数据量的分布式场景下，我们都是使用binlog来做事件源。

一方面，将binlog复制到Kafka，再由Kafka下游的消费者处理这些事件不影响数据库的核心业务，可以降低系统的耦合度；

另一方面，binlog和Kafka都是基于日志的顺序写入，Kafka的吞吐量远比B+树高，系统的整体性能也能得到改善。

目前这项技术已经很成熟了，在github上也有很多实现，通过[Change Data Capture](https://github.com/search?q=change+data+capture&s=stars)、[replication](https://github.com/search?q=replication&s=stars)、[binlog](https://github.com/search?q=binlog&s=stars)等关键词可以搜索到相关项目。在此列举一下：

| Project            | Language                                               | Description                                       |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [alibaba/Canal](https://github.com/alibaba/canal)![stars](https://img.shields.io/github/stars/alibaba/canal) | Java                        | 阿里巴巴 MySQL binlog 增量订阅&消费组件 |
| [debezium/debezium](https://github.com/debezium/debezium)![stars](https://img.shields.io/github/stars/debezium/debezium) | Java | Debezium is an open source distributed platform for change data capture. Replicates from MySQL to Kafka. Uses mysql-binlog-connector-java. Kafka Connector. A funded project supported by Redhat with employees working on it full time. |
| [linkedin/databus](https://github.com/linkedin/databus)![stars](https://img.shields.io/github/stars/linkedin/databus) | Java | Precursor to Kafka. Reads from MySQL and Oracle, and replicates to its own log structure. In production use at LinkedIn. No Kafka integration. Uses Open Replicator. |
| [zendesk/Maxwell](https://github.com/zendesk/maxwell)![stars](https://img.shields.io/github/stars/zendesk/maxwell) | Java                       | Reads MySQL event stream, output events as JSON. Parses ALTER/CREATE TABLE/etc statements to keep schema in sync. Written in java. Well maintained. |
| [noplay/python-mysql-replication](https://github.com/noplay/python-mysql-replication)![stars](https://img.shields.io/github/stars/noplay/python-mysql-replication) | Python      | Pure python library that parses MySQL binary logs and lets you process the replication events. Basically, the python equivalent of mysql-binlog-connector-java |
| [shyiko/mysql-binlog-connector-java](https://github.com/shyiko/mysql-binlog-connector-java)![stars](https://img.shields.io/github/stars/shyiko/mysql-binlog-connector-java) | Java    | Library that parses MySQL binary logs and calls your code to process them. Fork/rewrite of Open Replicator. Has tests. |
| [confluentinc/bottledwater-pg](https://github.com/confluentinc/bottledwater-pg)![stars](https://img.shields.io/github/stars/confluentinc/bottledwater-pg) | C                         | Change data capture from PostgreSQL into Kafka |
| [moiot/gravity](https://github.com/moiot/gravity)![stars](https://img.shields.io/github/stars/moiot/gravity) | Go | A Data Replication Center |
| [whitesock/open-replicator](https://github.com/whitesock/open-replicator)![stars](https://img.shields.io/github/stars/whitesock/open-replicator) | Java                | Open Replicator is a high performance MySQL binlog parser written in Java. It unfolds the possibilities that you can parse, filter and broadcast the binlog events in a real time manner. |
| [mardambey/mypipe](https://github.com/mardambey/mypipe)![stars](https://img.shields.io/github/stars/mardambey/mypipe) | Scala                   | Reads MySQL event stream, and emits events corresponding to INSERTs, DELETEs, UPDATEs. Written in Scala. Emits Avro to Kafka. |
| [Yelp/mysql_streamer](https://github.com/Yelp/mysql_streamer)![stars](https://img.shields.io/github/stars/Yelp/mysql_streamer) | Python                  | MySQLStreamer is a database change data capture and publish system. It’s responsible for capturing each individual database change, enveloping them into messages and publishing to Kafka. |
| [actiontech/dtle](https://github.com/actiontech/dtle)![stars](https://img.shields.io/github/stars/actiontech/dtle) | Go | Distributed Data Transfer Service for MySQL |
| [krowinski/php-mysql-replication](https://github.com/krowinski/php-mysql-replication)![stars](https://img.shields.io/github/stars/krowinski/php-mysql-replication) | PHP        | Pure PHP Implementation of MySQL replication protocol. This allow you to receive event like insert, update, delete with their data and raw SQL queries. |
| [dianping/puma](https://github.com/dianping/puma)![stars](https://img.shields.io/github/stars/dianping/puma) | Java | 本系统还会实现数据库同步（同构和异构），以满足数据库冗余备份，数据迁移的需求。 |
| [JarvusInnovations/Lapidus](https://github.com/JarvusInnovations/lapidus)![stars](https://img.shields.io/github/stars/JarvusInnovations/Lapidus) | Javascript        | Streams data from MySQL, PostgreSQL and MongoDB as newline delimited JSON. Can be run as a daemon or included as a Node.js module. |

这里只讨论Java语言的几个实现。首先[whitesock/open-replicator](https://github.com/whitesock/open-replicator)和[shyiko/mysql-binlog-connector-java](https://github.com/shyiko/mysql-binlog-connector-java)是专门用来解析MySQL binlog的库，后者也是在前者的基础上重构的。[debezium/debezium](https://github.com/debezium/debezium)、[linkedin/databus](https://github.com/linkedin/databus)、[zendesk/Maxwell](https://github.com/zendesk/maxwell)三个中间件binlog解析都是基于这两个库。

# 3、Canal vs. Debezium vs. databus vs. MaxWell

1、[alibaba/Canal](https://github.com/alibaba/canal)![stars](https://img.shields.io/github/stars/alibaba/canal)

优点：

* 阿里开源，有大厂实践背书
* 资料大都是中文的，方便学习

缺点：

* 定位于MySQL binlog解析，所以只能支持MySQL数据库的CDC
* Github上项目活跃度很一般，issue堆积了太多，13、14年的问题还没解决。

2、[debezium/debezium](https://github.com/debezium/debezium) ![stars](https://img.shields.io/github/stars/debezium/debezium)

优点：

* Rethat开源，国际专干开源的大厂背书
* 支持MySQL、PostgreSQL、Oracle、SqlServer、MongoDB主流数据库
* [文档](https://debezium.io/documentation/)详细，资料齐全
* 社区完善，在[Gitter](https://gitter.im/debezium/user)上有专门的问题讨论。
* 与Kafka很好集成，可作为Kafka Connector插件使用，[embed模式](https://debezium.io/documentation/reference/1.4/development/engine.html)支持嵌入自己的程序方便控制，也支持[Server模式](https://debezium.io/documentation/reference/1.4/operations/debezium-server.html)单独运行。

缺点：

* 文档大多数是中文，得多花点耐心

> 有意思的是阿里开源的[Flink流处理](https://ci.apache.org/projects/flink/flink-docs-stable/dev/table/connectors/formats/debezium.html)系统也是使用Debezium来做CDC，当然它还支持Canel、Maxwell
>
> Kafka的提供商[confluentinc](https://github.com/confluentinc)刚开始开源了[bottledwater-pg](https://github.com/confluentinc/bottledwater-pg)，最后也投入了debezium的怀抱，有官方的支持。

3、[linkedin/databus](https://github.com/linkedin/databus) ![stars](https://img.shields.io/github/stars/linkedin/databus)

优点：

* 国际大厂领英开源
* 支持MySQL和Oracle

缺点：

* 项目已经很久没有人维护了
* [文档](https://github.com/linkedin/databus/wiki/_pages)也很一般
* 暂时不支持Kafka集成，只能用[Databus Client](https://github.com/linkedin/databus/wiki/Databus-2.0-Client)消费binlog。

> 我也很好奇kafka都是linkedin开源的，为啥databus没继承kafka呢:anguished:

4、[zendesk/maxwell](https://github.com/zendesk/maxwell)![stars](https://img.shields.io/github/stars/zendesk/maxwell)

优点：

* 相当简单，下载下来，简单进行配置就能运行
* [文档](http://maxwells-daemon.io/quickstart/)相对来说，还算齐全
* 支持Kafka、RabbitMQ、Redis等队列

缺点：

* 文档是英文的，不过好在maxwell简单，文档也没多少

* 没啥明显缺点，非要说个缺点就是和前三者比身份不够显赫，zendesk这家美国公司没怎么听过。

# 4、Debezimu-MySQL踩坑记录

综合下来，Debezium是最佳选择



