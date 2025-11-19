---
title: 如何解决微服务的数据一致性分发问题——使用 Outbox 模式实现可靠的微服务数据交换
date: 2025-11-10
categories: 数据库
tags: 
- DB
- Outbox
- Canal
---

在微服务架构中，服务除了要更新自身的本地数据存储，有时候还需要通知其他服务发生了数据变更。**Outbox（外写）模式** 就是一种能让服务以安全、一致的方式完成这两项任务的方法。它保证源服务（source service）具有“读你自己的写”（read-your-own-writes）语义，同时实现跨服务边界的可靠、最终一致的数据传播。 

> **更新（2019 年 9 月 13 日）**：为了简化 outbox 模式的使用，Debezium 现在提供了一个现成可用的 SMT（单消息转换器）用于路由 outbox 事件。本文中所讨论的自定义 SMT 已不再是必需的。

## 为什么要用异步消息，而不是同步调用

如果你实现了几个微服务，你可能会同意：**数据**是它们最难处理的部分。 微服务往往不能孤立工作，它们经常需要把一个服务中新写或变更的数据广播给其他服务。

举个例子：有一个负责 “采购订单（purchase order）” 的微服务。当一个新订单创建后，这个信息可能要传给“发货服务（shipment service）”去安排发货，也要传给“客户服务（customer service）”去更新客户的信用余额等。

一种比较直观的方法是，让订单服务在处理新订单的时候，通过 REST、gRPC 或其他同步 API 调用发货服务和客户服务。这样的问题是：

* 发送方（订单服务）必须知道目标服务是谁、在哪里；
* 目标服务可能暂时不可用，这会导致调用失败或很复杂的重试逻辑；
* 这种同步调用方式会耦合服务：一个服务的可用性影响另一个服务。

为了解决这些问题，可以采用异步数据交换：订单服务把事件写入一个 **持久化消息日志**（比如 Apache Kafka），其他服务订阅这些事件流，自己决定何时消费并处理。

这样做有几个好处：

1. **重新回放（re-playability）**：新的消费者可以随时加入，从头开始消费 Kafka topic，构建自己的数据视图（例如数据仓库、搜索索引等）。
2. **解耦**：服务间不直接调用，只通过事件通信，更灵活、更容错。
3. **可持久化**：Kafka 保证消息持久化，新消费者可以慢慢消费。

## 双写 (Dual Writes) 的问题

在微服务中，为了完成“变更自己的数据库 + 发布事件”这两个动作，常见的做法是：

1. 向本地数据库写入（比如订单服务插入 `PurchaseOrder` 表）；
2. 同时向 Kafka 发布一个事件（新的订单事件）。

![双写问题](https://wiki.hugogu.cn/dual-write.svg)

但这样做会有一致性问题，因为它们不能放在一个分布式事务里（例如 Postgres + Kafka 无法共享同一个分布式事务）。

结果可能是：

* 订单写进了数据库，但事件没发布出去（网络问题、Kafka 不可用等）；
* 或者事件发布成功，但数据库插入失败（网络问题、异常回滚）。

这些都很糟糕：可能发货服务没收到订单消息，也可能有发货但订单服务自己数据库里根本找不到订单。

## Outbox 模式

为了解决这种问题，我们可以在订单服务的数据库中加一个 **outbox 表**。

具体做法：

* 当订单服务接收到“创建订单”请求时，它在 **同一个数据库事务**内：

  * 向 `PurchaseOrder` 表插入订单；
  * 向 `outbox` 表插入一个记录，表示一个 “订单已创建” 的事件。

* 这个 outbox 表的记录里包含事件内容（例如用 JSON 存储订单详情、订单行、上下文信息等）。

* 然后，有一个异步进程（或服务）不断监视 outbox 表的新条目，把它们取出来，发布到 Kafka。

![outbox表模式](https://wiki.hugogu.cn/outbox.svg)

这样做的好处：

1. **原子性**：插订单 + 写 outbox 是一个事务，要么都成功，要么都失败。
2. **读你自己的写语义**：因为订单写入数据库是同步完成的，用户如果紧接着查询订单服务，很快就能看到新订单（事务提交后）。
3. **异步传播**：事件通过 Kafka 异步被广播给其他服务，实现最终一致性。

典型的 outbox 表结构如下：

| 列名              | 类型             | 说明                                                                        |
| --------------- | -------------- | ------------------------------------------------------------------------- |
| `id`            | `uuid`         | 唯一 ID，用于消费者做重复检测。                                          |
| `aggregatetype` | `varchar(255)` | 聚合根类型，比如 “Order” 或 “Customer”。用来路由到不同的 Kafka topic。        |
| `aggregateid`   | `varchar(255)` | 聚合根 ID（例如订单 ID），用于作为 Kafka 消息 key，这样关联的事件会落在同一个 partition。 |
| `type`          | `varchar(255)` | 事件类型，比如 “OrderCreated” 或 “OrderLineCanceled”。              |
| `payload`       | `jsonb`        | 事件具体内容（订单详情、行项目等）。                                         |

这种模式的缺点也很明显：Outbox的主要问题是，它有**额外的数据库负担**，而且非常容易成为瓶颈。

尤其是当Outbox被设计成了一个通用事件存储器，用来存储所有事件的时候。

![通用Outbox模式](https://wiki.hugogu.cn/dual-wirte/all-in-one-box.svg)

在做好数据Partition的情况下，至少可以确保Outbox本身不会成为性能瓶颈。最极端的情况如下图：

![Outbox](https://wiki.hugogu.cn/outboxes.svg)

## 用 Change Data Capture (CDC) 实现

**Log-based Change Data Capture（CDC）** 是捕获 outbox 表新增内容的很好的方式。它比轮询高效，延时低。

![CDC模式](https://wiki.hugogu.cn/cdc.svg)

目前有很多个开源的CDC实现：

| Project            | Language                                               | Description                                       |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [alibaba/Canal](https://github.com/alibaba/canal)![](https://img.shields.io/github/stars/alibaba/canal) | Java                        | 阿里巴巴 MySQL binlog 增量订阅&消费组件 |
| [debezium/debezium](https://github.com/debezium/debezium)![](https://img.shields.io/github/stars/debezium/debezium) | Java | Debezium is an open source distributed platform for change data capture. Replicates from MySQL to Kafka. Uses mysql-binlog-connector-java. Kafka Connector. A funded project supported by Redhat with employees working on it full time. |
| [linkedin/databus](https://github.com/linkedin/databus)![](https://img.shields.io/github/stars/linkedin/databus) | Java | Precursor to Kafka. Reads from MySQL and Oracle, and replicates to its own log structure. In production use at LinkedIn. No Kafka integration. Uses Open Replicator. |
| [zendesk/Maxwell](https://github.com/zendesk/maxwell)![](https://img.shields.io/github/stars/zendesk/maxwell) | Java                       | Reads MySQL event stream, output events as JSON. Parses ALTER/CREATE TABLE/etc statements to keep schema in sync. Written in java. Well maintained. |
| [noplay/python-mysql-replication](https://github.com/noplay/python-mysql-replication)![](https://img.shields.io/github/stars/noplay/python-mysql-replication) | Python      | Pure python library that parses MySQL binary logs and lets you process the replication events. Basically, the python equivalent of mysql-binlog-connector-java |
| [shyiko/mysql-binlog-connector-java](https://github.com/shyiko/mysql-binlog-connector-java)![](https://img.shields.io/github/stars/shyiko/mysql-binlog-connector-java) | Java    | Library that parses MySQL binary logs and calls your code to process them. Fork/rewrite of Open Replicator. Has tests. |
| [confluentinc/bottledwater-pg](https://github.com/confluentinc/bottledwater-pg)![](https://img.shields.io/github/stars/confluentinc/bottledwater-pg) | C                         | Change data capture from PostgreSQL into Kafka |
| [uber/storagetapper](https://github.com/uber/storagetapper)![](https://img.shields.io/github/stars/uber/storagetapper) | Go                        | StorageTapper is a scalable realtime MySQL change data streaming, logical backup and logical replication service |
| [moiot/gravity](https://github.com/moiot/gravity)![](https://img.shields.io/github/stars/moiot/gravity) | Go | A Data Replication Center |
| [whitesock/open-replicator](https://github.com/whitesock/open-replicator)![](https://img.shields.io/github/stars/whitesock/open-replicator) | Java                | Open Replicator is a high performance MySQL binlog parser written in Java. It unfolds the possibilities that you can parse, filter and broadcast the binlog events in a real time manner. |
| [mardambey/mypipe](https://github.com/mardambey/mypipe)![](https://img.shields.io/github/stars/mardambey/mypipe) | Scala                   | Reads MySQL event stream, and emits events corresponding to INSERTs, DELETEs, UPDATEs. Written in Scala. Emits Avro to Kafka. |
| [Yelp/mysql_streamer](https://github.com/Yelp/mysql_streamer)![](https://img.shields.io/github/stars/Yelp/mysql_streamer) | Python                  | MySQLStreamer is a database change data capture and publish system. It’s responsible for capturing each individual database change, enveloping them into messages and publishing to Kafka. |
| [actiontech/dtle](https://github.com/actiontech/dtle)![](https://img.shields.io/github/stars/actiontech/dtle) | Go | Distributed Data Transfer Service for MySQL |
| [krowinski/php-mysql-replication](https://github.com/krowinski/php-mysql-replication)![](https://img.shields.io/github/stars/krowinski/php-mysql-replication) | PHP        | Pure PHP Implementation of MySQL replication protocol. This allow you to receive event like insert, update, delete with their data and raw SQL queries. |
| [dianping/puma](https://github.com/dianping/puma)![](https://img.shields.io/github/stars/dianping/puma) | Java | 本系统还会实现数据库同步（同构和异构），以满足数据库冗余备份，数据迁移的需求。 |
| [JarvusInnovations/Lapidus](https://github.com/JarvusInnovations/lapidus)![](https://img.shields.io/github/stars/JarvusInnovations/Lapidus) | Javascript        | Streams data from MySQL, PostgreSQL and MongoDB as newline delimited JSON. Can be run as a daemon or included as a Node.js module. |

我之前写过[Debezium](https://blog.hufeifei.cn/2021/03/DB/mysql-binlog-parser/)和[Canal](https://blog.hufeifei.cn/2025/10/DB/canal-in-k8s/)相关的文章。

Debezium 提供了多个数据库（MySQL、PostgreSQL、SQL Server 等）的 CDC 连接器。而Canal只专注于MySQL binlog的解析。

CDC(Change Data Capture)本质上是对Outbox模式的泛化实现，能在不侵入业务逻辑的前提下，达成和Outbox同样的效果。但是其主要问题在于：

* 泄露了Order服务的实体结构。当然，我们可以在发布消息前，进行格式转换，但是这样，其整体复杂度是更高的。
* 并不是所有的事件，都会有数据变更。每个数据变更，也并不一定由一个事件引起。本质上，领域事件是业务对象，而CDC采集的是存储层数据。想要让其发布的事件真正符合领域模型，本质上是要做一次ORM的逆运算。
* CDC可以屏蔽下游依赖。但是并不是所有的依赖都应该被屏蔽掉。比如针对，Order和Payment，这是一个业务强依赖，我们并不一定希望要用如此松的模式，把本来可以存在的依赖强行消解掉。

CDC会依赖于数据库本身的能力，所以可以处理的场景会受到限制。比如，PostgreSQL的Logical Replication可以被用来实现CDC，但是会受制于PostgreSQL本身的约束[6]。如：

* 只支持普通表生效，不支持序列、视图、物化视图、外部表、分区表和大对象
* 只支持普通表的DML(INSERT、UPDATE、DELETE)操作,不支持truncate、DDL操作
* 需要同步的表必须设置REPLICA IDENTITY 不能为noting(默认值是default)，同时表中必须包含主键

## 结合Outbox与CDC

Outbox模式的缺点和CDC的优点正好互补。所以不难得出一个集合二者优点的方案。即：用Outbox存放对外的领域事件，然后利用CDC将Outbox中的数据发送到消息系统中。

这样，使用方就只需要定义领域事件的结构，同时避免对外暴露内部数据对象的存储模型。同时，又不必麻烦编写额外的代码去负责把Outbox中的新增数据发送到消息系统。

其基本设计如下图所示：

![](https://wiki.hugogu.cn/outbox_with_cdc.svg)

关于这个实现方案的更多细节，可以参考[这篇文章](https://medium.com/codex/outbox-pattern-for-reliable-data-exchange-between-microservices-9c938e8158d9)。

## 支持事务的消息系统

如果消息中间件，把自己模拟成数据库，并支持了数据库的XA分布式事务协议。便可以让消息与数据库变更事务化。但是并不是所有的消息中间件都支持消息事务。已知支持某种XA协议的消息中间件有：

* [RocketMQ](https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage/)
* endurox

更常见的消息中间件，如RabbitMQ, ActiveMQ及Kafka，均不支持事务。原因也很简单：影响性能。

[1]: https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/ "Reliable Microservices Data Exchange With the Outbox Pattern"
[2]: https://docs.aws.amazon.com/zh_cn/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html "AWS 事务发件箱模式"
[3]: https://wiki.hugogu.cn/tech/patterns/outbox
[4]: https://wiki.hugogu.cn/tech/design/dual-writes
