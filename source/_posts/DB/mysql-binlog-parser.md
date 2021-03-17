---
title: 基于Binlog的实时同步功能——debezium、canel、databus技术选型
date: 2021-03-13 14:23
categories: 数据库
---

大厂这段时间学到的最有价值的两个技术除了大数据，另一个就是基于CDC和消息队列的实时同步技术。

[去年的一篇文章大致地讲了我对MQ的一些认识](https://blog.hufeifei.cn/2020/04/25/Alibaba/MetaQ&Notify/)，事实上Kafka在内的现代MQ，功能远不止这些。后面整理好自己的思路，肯定会再写一篇文章来讲讲。这篇文章的主角就是与MQ息息相关的CDC技术。

# 1、CDC技术

[CDC](https://en.wikipedia.org/wiki/Change_data_capture)全称叫：change data capture，是一种基于数据库数据变更的事件型软件设计模式。

比如有一张订单表trade，订单每一次变更录入到一张trade_change的队列表。然后另外一个调度线程可以消费trade_change这张队列表来做一些数据统计，如每日的付款用户统计、每日的下单用户统计等。

这就是我毕业入职的第一家公司的报表统计逻辑。这个设计在订单量小的时候是看不出问题的，而一旦某一时刻订单量增多。基于MySQL的队列表由于B+树的写入吞吐量不够，导致MySQL CPU经常飙升。比如双十一，618这样的大促，程序员就得在颤颤巍巍中度过。

> B+的写入性能肯定是不如直接顺序写文件的，B+树的本质就是牺牲写性能，换取磁盘上的随机读的查找结构，所以大部分数据库都会设计[Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)来管理B+树脏页，以避免频繁的随机IO。
> 同时为了防止Buffer数据丢失同时为了保证事务的ACID，所以就有了[Redo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)来进行崩溃恢复，[Undo-log](https://dev.mysql.com/doc/refman/5.6/en/innodb-undo-logs.html)来做未提交事务的撤销。这些日志都是顺序写入，远比B+树的随机写性能高。

![database architecture](https://p.pstatp.com/origin/pgc-image/85c4945860ba4dd793acc42691226c8a)

# 2、基于Binlog的CDC

[Binlog](https://dev.mysql.com/doc/internals/en/binary-log.html)是MySQL 3.23.14引进的，[它包含所有的描述数据库修改的事件](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)——DML(增删改)、DDL(表结构定义与修改)操作。

![MySQL architecture](https://p.pstatp.com/origin/pgc-image/f20dc094cd6a4163829e616e02197acd)

与InnoDB中的[redo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)、[undo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-logs.html)不同，binlog和[slow_query_log](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)一样是server层的日志，所以InnoDB和MyISAM等各种存储引擎的数据修改都会记录到这个日志中。

> MySQL拥有分层架构，支持可插拔的存储引擎，所以服务层的binlog与[InnoDB引擎的redo-log](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design/)是不同的两个事物，这也是为什么MySQL支持以[`STATEMENT`格式](https://dev.mysql.com/doc/refman/5.7/en/binary-log-setting.html)直接将sql语句存入binlog。而像PostgreSQL这样的数据库，[WAL日志](https://www.postgresql.org/docs/8.1/wal-intro.html)除了作为redo-log用于保证事务的持久性外，WAL日志在Replica过程中也扮演着与MySQL的binlog相同的角色。

![CDC architecture](https://debezium.io/documentation/reference/1.4/_images/debezium-architecture.png)

对于CDC的架构设计，在大数据量的分布式场景下，我们都是使用binlog来做事件源。

一方面，将binlog复制到Kafka，再由Kafka下游的消费者处理这些事件不影响数据库的核心业务，可以降低系统的耦合度；

另一方面，binlog和Kafka都是基于日志的顺序写入，Kafka的吞吐量远比B+树高，系统的整体性能也能得到改善。

目前基于binlog的CDC技术已经很成熟了，在github上也有很多实现，通过[Change Data Capture](https://github.com/search?q=change+data+capture&s=stars)、[replication](https://github.com/search?q=replication&s=stars)、[binlog](https://github.com/search?q=binlog&s=stars)等关键词可以搜索到相关项目。在此列举一下：

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
* Github上项目活跃度很一般，issue堆积了太多，13、14年的问题都还没解决。

2、[debezium/debezium](https://github.com/debezium/debezium) ![stars](https://img.shields.io/github/stars/debezium/debezium)

优点：

* Rethat开源，专干开源的国际大厂背书
* 支持MySQL、PostgreSQL、Oracle、SqlServer、MongoDB主流数据库
* [文档](https://debezium.io/documentation/)详细，资料齐全
* 社区完善，在[Gitter](https://gitter.im/debezium/user)上有专门的问题讨论区。
* 与Kafka很好集成，可作为Kafka Connector插件使用，[embed模式](https://debezium.io/documentation/reference/1.4/development/engine.html)支持嵌入自己的程序方便控制，也支持[Server模式](https://debezium.io/documentation/reference/1.4/operations/debezium-server.html)单独运行。
* 支持SMT消息体转换，OpenTracing分布式链路追踪等集成功能

缺点：

* 文档大多数是中文，得多花点耐心

> 有意思的是阿里开源的[Flink流处理](https://ci.apache.org/projects/flink/flink-docs-stable/dev/table/connectors/formats/debezium.html)系统也是使用Debezium来做CDC，当然它还支持Canel、Maxwell
>
> Kafka创始人创办的[confluentinc](https://github.com/confluentinc)刚开始开源了[bottledwater-pg](https://github.com/confluentinc/bottledwater-pg)，最后也投入了debezium的怀抱，有官方的认可。

3、[linkedin/databus](https://github.com/linkedin/databus) ![stars](https://img.shields.io/github/stars/linkedin/databus)

优点：

* 国际大厂领英开源
* 支持MySQL和Oracle

缺点：

* 项目已经很久没有人维护了
* [文档](https://github.com/linkedin/databus/wiki/_pages)也很一般
* 暂时不支持Kafka集成，只能用[Databus Client](https://github.com/linkedin/databus/wiki/Databus-2.0-Client)消费binlog。

> Kafka最早是[Jay Kreps](https://www.linkedin.com/in/jaykreps)在领英创建并开源的，可能是Jay Kreps觉得Kafka在大数据领域大有可图，所以就带着Linkedin的几个工程师一起创立了[Confluent](https://www.confluent.io/about/)​专注于Kafka生态的开发与维护。
>
> 在Kafka文档[Log-Compact一节](https://kafka.apache.org/documentation/#compaction)可以看到这段话：
>
> This functionality is inspired by one of LinkedIn's oldest and most successful pieces of infrastructure—a database changelog caching service called [Databus](https://github.com/linkedin/databus). Unlike most log-structured storage systems Kafka is built for subscription and organizes data for fast linear reads and writes. Unlike Databus, Kafka acts as a source-of-truth store so it is useful even in situations where the upstream data source would not otherwise be replayable.
>
> 可以看出Databus是Linkedin非常老的一个基础服务，Kafka的Log Compact的一些设计也源自于Databus。

4、[zendesk/maxwell](https://github.com/zendesk/maxwell)![stars](https://img.shields.io/github/stars/zendesk/maxwell)

优点：

* 相当简单，下载下来，简单进行配置就能运行
* [文档](http://maxwells-daemon.io/quickstart/)相对来说，还算齐全
* 支持Kafka、RabbitMQ、Redis等队列

缺点：

* 文档是英文的，不过好在maxwell相对简单。

* 没啥明显缺点。非要说个缺点，就是和前三者比身份不够显赫，zendesk这家美国公司没怎么听过。

> 综合下来，Debezium是最佳选择。

# 4、Debezimu-MySQL的配置

要使用debezium需要[预先对mysql服务进行配置](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#setting-up-mysql)。

### 4.1、 MySQL配置

**1）**创建单独的用户，并授予debezium需要的权限

```sql
mysql> CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user' IDENTIFIED BY 'password';
mysql> FLUSH PRIVILEGES;
```

> MySQL提供的权限：https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html

debezium需要几个权限的作用：

| Keyword              | Description                                                  |
| :------------------- | :----------------------------------------------------------- |
| `SELECT`             | `SELECT`查询权限。只被用在初始化阶段。                       |
| `RELOAD`             | 执行 `FLUSH` 语句清除重新加载内部缓存。只被用在初始化阶段。  |
| `SHOW DATABASES`     | 执行 `SHOW DATABASE` 语句。只被用在初始化阶段。              |
| `REPLICATION SLAVE`  | 读取MySQL binlog。                                           |
| `REPLICATION CLIENT` | 执行`SHOW MASTER STATUS`、`SHOW SLAVE STATUS`、`SHOW BINARY LOGS`等语句。 |

**2）**开启MySQL服务的binlog功能

```properties
server-id         = 223344
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 10
```

各项配置的作用：

| Property           | Description                                                  |
| :----------------- | :----------------------------------------------------------- |
| `server-id`        | 在MySQL集群中每个server和replication的 `server-id` 必须是唯一的。Debezium是作为MySQL的replication，启动后也会分配一个`server-id`给debezium-connector。 |
| `log_bin`          | binlog文件的前缀                                             |
| `binlog_format`    | [`binlog-format`](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_binlog_format) 必须设置成`ROW`模式。因为`STATEMENT`模式只记录SQL语句，适合数据库间的数据备份，不适合往kafka这样的异构存储同步。 |
| `binlog_row_image` | [ `binlog_row_image` ](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_binlog_row_image)必须设置成`FULL`。`ROW`模式下binlog需要记录所有的列。 |
| `expire_logs_days` | binlog的过期时间。默认位 `0`, 意味着不会自动删除。这个值可根据自己的环境需求进行设置。 |

> 还有几项可选配置项：
>
> * [开启全局事务ID(GTIDs)](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#enable-mysql-gtids)方便确认主从备份之间的一致性。
> * [配置MySQL会话超时时间](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-session-timeouts)用于大表的快照读阶段。
> * [开启原始SQL语句的记录](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#enable-query-log-events)用于查看每条binlog记录的原始SQL。

### 4.2、准备Kafka环境，在Kafka-connect中安装Debezium

> [Kafka](http://kafka.apache.org/)需要依赖[Zookeeper](https://zookeeper.apache.org/)管理集群，所以还需要准备zookeeper环境。

1）下载Debezium：https://debezium.io/releases/

2）配置Kafka-connect插件路径，并将Debezimu插件解压到该目录

```properties
plugin.path=/kafka/connect
```

3）启动Kafka-connect进程：

Kafka-connect可以用[单机版(`standalone`)和分布式版(`distributed`)](https://docs.confluent.io/home/connect/userguide.html#standalone-vs-distributed-mode)两种启动方式：

* `standalone`模式下，启动时直接提供`properties`文件来创建Connector任务。
* `distributed`模式下，提供[REST接口](https://docs.confluent.io/platform/current/connect/references/restapi.html)对Connector任务进行增删改查。

### 4.3、Debezium的基础配置

在`distributed`模式下可以，调用[`POST /connectors`](https://docs.confluent.io/platform/current/connect/references/restapi.html#post--connectors)接口创建Debezium的Connector任务，任务的基本配置如下：

```json
{
  "name": "inventory-connector", 
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector", 
    "database.hostname": "192.168.99.100", 
    "database.port": "3306", 
    "database.user": "debezium-user", 
    "database.password": "debezium-user-pw", 
    "database.server.id": "184054", 
    "database.server.name": "fullfillment", 
    "database.include.list": "inventory", 
    "database.history.kafka.bootstrap.servers": "kafka:9092", 
    "database.history.kafka.topic": "dbhistory.fullfillment", 
    "include.schema.changes": "true" 
  }
}
```

> 这个配置主要是数据库的用户名密码，需要同步的数据库和相关数据表，以及kafka地址和数据库schema变更存储的topic。
>
> Debezium-Connector的所有配置：https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-connector-properties

# 5、Debezium踩坑记录

debezium配置起来还是比较简单的，但是这么复杂的项目，坑还是比较多的。

### 5.1、关闭快照初始化

Debezium的Connector第一次启动时，会给你的数据库执行一次[快照初始化](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-snapshots)。

因为对于老项目，早期的binlog肯定已经被删掉了，这个时候Debezium会帮你把数据库的所有数据都写到Kafka里，这次快照之后的增删改操作通过解析binlog写入kafka。这也是为什么Debezium需要获取数据库`SELECT`权限的原因。

但是快照读有这么几个问题：

* 在执行快照初始化过程中，Connector重启或者Kafka-connect Rebalance，重启后Debezium会重新初始化快照。因为Debezium的快照是通过[`SELECT * FROM table`](https://github.com/debezium/debezium/blob/1.4/debezium-connector-mysql/src/main/java/io/debezium/connector/mysql/SnapshotReader.java#L650)扫描全表实现的，没有记录进度，非常粗暴。
* 为了防止快照初始化过程中表的schema会变更，快照初始化前会获取全局读锁。

> 可以通过[`snapshot.locking.mode`](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-property-snapshot-locking-mode)属性配置是否获取全局读锁，`snapshot.locking.mode=none`即可关闭。

snapshot只适合在备份从库上执行，否则可能会影响正常用户的使用，通过[`snapshot.mode`](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-property-snapshot-mode)可以对初始化进行配置，这个选项支持以下几个配置值：

`initial` (default)- 只有当binlog的offset没有记录的时候才会执行一次快照初始化。

`when_needed` - 有需要时就会执行，比如第一次offset没有记录，或者Connector停了很久早期的binlog被删掉了，当前的offset已经不可用了，或者GTID对不上的时候。

`never` - 从不执行初始化。第一次启动Connector时就从binlog头部开始读取。需要注意，这种配置需要binlog包含所有的历史记录。

`schema_only` - Connector初始化时只读取表的`schame`而不读取数据。如果你只需要Connector启动后的数据库变更，那这个配置很有用。

`schema_only_recovery` - 这个配置是用来恢复丢失的历史数据。

### 5.2、修改topic

Debezium默认的行为是将一张表上的`INSERT`、`UPDATE`、`DELETE`操作记录到一个topic。Topic命名规则是`<serverName>.<databaseName>.<tableName>`

如果进行分库了，比如`server0`上有`db01`和`db02`两个逻辑库，`server1`上有`db11`和`db12`两个逻辑库，这四个逻辑库上都有一张`order`表。那此时就会有4个topic。

如果我们想把它们路由到同一个topic上，就需要用到[Kafka-Connect提供的SMT功能](https://docs.confluent.io/platform/current/connect/transforms/overview.html)了：

```properties
transforms=route
transforms.route.type=org.apache.kafka.connect.transforms.RegexRouter
transforms.route.regex=([^.]+)\\.([^.]+)\\.([^.]+)
transforms.route.replacement=$3
```

Kafka-Connect提供了一个[`RegexRouter`](https://docs.confluent.io/platform/current/connect/transforms/regexrouter.html)、[`TimestampRouter`](https://docs.confluent.io/platform/current/connect/transforms/timestamprouter.html)、[MessageTimestampRouter](https://docs.confluent.io/platform/current/connect/transforms/messagetimestamprouter.html)几个SMT让我们修改数据存入的topic。这里的RegexRouter，允许我们用正则表达式来对`Debezium`默认的topic进行修改。

### 5.3、Decimal数据的处理

对于MySQL中的[`decimal`](https://dev.mysql.com/doc/refman/8.0/en/fixed-point-types.html)类型的数据，Java里会转成`BigDecimal`，但是以json格式存入kafka的时候就会丢失精度。

> 毕竟json出自JS，[JS中只支持number数值类型](https://stackoverflow.com/questions/35709595/why-would-you-use-a-string-in-json-to-represent-a-decimal-number)，对应到Java就是double类型。

Debezium支持[`decimal.handling.mode`](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-property-decimal-handling-mode)选项可以将decimal配置成`string`类型。

### 5.4、时间类型数据的处理

Debezium底层的binlog解析用的是[shyiko/mysql-binlog-connector-java](https://github.com/shyiko/mysql-binlog-connector-java)。这中间做了很多转换：

| mysql                           | binlog-connector                     | debezium                      | debezium schema                 |
| ------------------------------- | ------------------------------------ | ----------------------------- | ------------------------------- |
| date (2021-01-28)               | LocalDate (2021-01-28)               | Integer (18655)               | io.debezium.time.Date           |
| time (17:29:04)                 | Duration (PT17H29M4S)                | Long (62944000000)            | io.debezium.time.Time           |
| timestamp (2021-01-28 17:29:04) | ZonedDateTime (2021-01-28T09:29:04Z) | String (2021-01-28T09:29:04Z) | io.debezium.time.ZonedTimestamp |
| datetime (2021-01-28 17:29:04)  | LocalDateTime (2021-01-28T17:29:04)  | Long (1611854944000)          | io.debezium.time.Timestamp      |

`date`类型，最后在Debezium中会调用`LocalDate.toEpochDay`转成了基于1970年的天数。

`time`类型，在binlog解析库中，被转成了Duration，在Debezium中最后被转成了毫秒值。

`timestamp`类型，最后在Debezium中被转成了一个ISO格式的字符串，但是时区默认是UTC时区。

`datetime`类型，最后在Debezium中被转成了一个long类型，时区是写死的UTC时区。

总之，Debezium时间的处理混乱不堪。所以我为Debezium写了一个[`datetime-converter`的补丁](https://github.com/holmofy/debezium-datetime-converter)可以将这四种类型转成字符串。配置如下：

```properties
converters=datetime
datetime.type=com.darcytech.debezium.converter.MySqlDateTimeConverter
datetime.format.date=YYYY-MM-dd
datetime.format.time=HH:mm:ss
datetime.format.datetime=YYYY-MM-dd HH:mm:ss
datetime.format.timestamp=YYYY-MM-dd HH:mm:ss
datetime.format.timestamp.zone=UTC+8
```

### 5.5、墓碑事件

Debezium会生成5种事件：

- [*create* events](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-create-events)：对应MySQL种的INSERT语句。

  ```json
  { 
      "op": "c", 
      "ts_ms": 1465491411815, 
      "before": null, 
      "after": { 
        "id": 1004,
        "first_name": "Anne",
        "last_name": "Kretchmar",
        "email": "annek@noanswer.org"
      },
      "source": { 
        "version": "1.4.2.Final",
        "connector": "mysql",
        "name": "mysql-server-1",
        "ts_ms": 0,
        "snapshot": false,
        "db": "inventory",
        "table": "customers",
        "server_id": 0,
        "gtid": null,
        "file": "mysql-bin.000003",
        "pos": 154,
        "row": 0,
        "thread": 7,
        "query": "INSERT INTO customers (first_name, last_name, email) VALUES ('Anne', 'Kretchmar', 'annek@noanswer.org')"
      }
    }
  ```

  此时payload种的before字段为null，after字段为新增的记录值。

- [*update* events](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-update-events)：对应MySQL种的UPDATE语句。

  ```json
  {
      "before": { 
        "id": 1004,
        "first_name": "Anne",
        "last_name": "Kretchmar",
        "email": "annek@noanswer.org"
      },
      "after": { 
        "id": 1004,
        "first_name": "Anne Marie",
        "last_name": "Kretchmar",
        "email": "annek@noanswer.org"
      },
      "source": { 
        "version": "1.4.2.Final",
        "name": "mysql-server-1",
        "connector": "mysql",
        "name": "mysql-server-1",
        "ts_ms": 1465581029100,
        "snapshot": false,
        "db": "inventory",
        "table": "customers",
        "server_id": 223344,
        "gtid": null,
        "file": "mysql-bin.000003",
        "pos": 484,
        "row": 0,
        "thread": 7,
        "query": "UPDATE customers SET first_name='Anne Marie' WHERE id=1004"
      },
      "op": "u", 
      "ts_ms": 1465581029523 
    }
  ```

  此时payload中，before为更新前的数据，after为更新后的数据。

- [Primary key updates](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-primary-key-updates)：修改主键的操作，会生成一个`DELETE`事件和`CREATE`事件：

  -  `DELETE` 事件会有 `__debezium.newkey` 的消息头。这个值是更新后的新主键。
  -  `CREATE` 事件会有 `__debezium.oldkey` 的消息头。这个值是更新前的老主键。

- [*delete* events](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-delete-events)：对应MySQL的DELTE语句。

  ```json
  {
    "schema": { ... },
    "payload": {
      "before": { 
        "id": 1004,
        "first_name": "Anne Marie",
        "last_name": "Kretchmar",
        "email": "annek@noanswer.org"
      },
      "after": null, 
      "source": { 
        "version": "1.5.0.Beta2",
        "connector": "mysql",
        "name": "mysql-server-1",
        "ts_ms": 1465581902300,
        "snapshot": false,
        "db": "inventory",
        "table": "customers",
        "server_id": 223344,
        "gtid": null,
        "file": "mysql-bin.000003",
        "pos": 805,
        "row": 0,
        "thread": 7,
        "query": "DELETE FROM customers WHERE id=1004"
      },
      "op": "d", 
      "ts_ms": 1465581902461 
    }
  }
  ```

  此时payload中，before为删除前的数据，after为null。

- [Tombstone events](https://debezium.io/documentation/reference/1.4/connectors/mysql.html#mysql-tombstone-events)：Debezium会为删除操作生成一条key与DELETE事件相同、value为null的空消息(墓碑事件)。

  > 墓碑事件主要用于[Kafka的compact](https://kafka.apache.org/documentation/#compaction)——Kafka会删除具有相同key的早期事件。但是要让Kafka删除所有具有相同key的消息，需要将消息指设置成null。

需要特别注意，墓碑事件的消息value为null，需要为这个事件做特殊处理。

### 5.6、禁用Kafka-Connect的Schema配置

Kafka-Connect为了保证每条消息是可以自我描述的，所以都会带schema。如果我们使用了[`JsonConverter`](https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)进行序列化，默认情况下，kafka中的消息格式是这样的：

```json
{
    "schema": { /* ... */ },
    "payload": {
    	"op": "u",
    	"source": {
    		...
    	},
    	"ts_ms" : "...",
    	"before" : {
    		"field1" : "oldvalue1",
    		"field2" : "oldvalue2"
    	},
    	"after" : {
    		"field1" : "newvalue1",
    		"field2" : "newvalue2"
    	}
	}
}
```

这里面的schema会包含下面payload里每个字段的类型解释，会导致Kafka中存储的消息非常臃肿。可以在Kafka-Connect中将Key和Value的schema禁用掉：

```properties
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
```

> 更好的解决方案是使用集中式的Schema Registry。Debezium也推荐使用这种方式。
>
> 在github搜索[`schema registry`](https://github.com/search?q=Schema+registry)关键词查找相关项目。[Debezium在文档中](https://debezium.io/documentation/faq/#avro-converter)推荐[Apicurio API and Schema Registry](https://github.com/Apicurio/apicurio-registry) 和 [Confluent Schema Registry](https://github.com/confluentinc/schema-registry)这两种SchemaRegistry。

### 5.5、对Debezium生成的消息进行处理

没有shema的时候，Debezium默认生成的数据格式是这样的：

```json
{
	"op": "u",
	"source": {
		...
	},
	"ts_ms" : "...",
	"before" : {
		"field1" : "oldvalue1",
		"field2" : "oldvalue2"
	},
	"after" : {
		"field1" : "newvalue1",
		"field2" : "newvalue2"
	}
}
```

消息体中`before`表示变更前的数据，`after`表示变更后的数据，`source`表示来源于哪个数据库、哪张表、哪个事务(GTID)。

为了方便与其他Connector集成，比如让[`kafka-connect-jdbc`](https://docs.confluent.io/kafka-connect-jdbc/current/index.html)把消息都写到另一个数据库中。那这个时候我们只想要`after`里面的数据了。

Debezium提供了一个[`Event-Flat`的SMT](https://debezium.io/documentation/reference/1.4/configuration/event-flattening.html)，我们只需要和上面的RegexRouter一样配置一下就可以了：

```properties
transforms=unwrap,...
transforms.unwrap.type=io.debezium.transforms.ExtractNewRecordState
```

那如果是删除操作呢，ExtractNewRecordState默认的行为是假删除(`tombstones`)，我们需要添加一下几个配置：

```properties
transforms=unwrap,...
transforms.unwrap.type=io.debezium.transforms.ExtractNewRecordState
transforms.unwrap.drop.tombstones=false
transforms.unwrap.delete.handling.mode=rewrite
transforms.unwrap.add.fields=table,lsn
```

这个时候ExtractNewRecordState会把重写delete事件的消息体，将删除的主键等信息存入`value`字段，以备后续处理。



Refs:

^ Debezium Document: https://debezium.io/documentation/reference/1.4/

^ Debezium FAQ: https://debezium.io/documentation/faq/

^ Confluent Document: https://docs.confluent.io/platform/current/overview.html
