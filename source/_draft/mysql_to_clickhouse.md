---
title: 基于Debezium与Kafka搭建MySQL到ClickHouse的同步平台
date: 2021-05-10 18:50
categories: JAVA
---

众所周知MySQL InnoDB是一个支持事务的OLTP数据库，但是对分析型业务支持并不友好，如果在同一个MySQL数据库上做OLAP业务，那就得加各种索引，反而会拖慢OLTP的业务。

所以我第一个想法是找MySQL面向OLAP的存储引擎，我在阿里那段时间，使用MySQL的时候是支持两个存储引擎的，一个是支持事务的OLTP引擎X-DB，也就是在阿里云上卖的商业数据库[PolarDB-X](https://www.aliyun.com/product/drds)，一个是支持列式存储的OLAP引擎[AnalyticDB](https://www.alibabacloud.com/zh/product/analyticdb-for-mysql)，这个也在阿里云上有的卖。

但是阿里这两款产品都卖的死贵，重点是不开源。鉴于只使用开源产品的策略，寻寻觅觅中，让我发现了一款[ClickHouse](https://github.com/clickhouse/clickhouse)。

clickhouse是战斗民族俄罗斯研发的，它的贡献者Yandex等价于俄罗斯的百度。这里[有篇文章](https://clickhouse.tech/blog/en/2016/evolution-of-data-structures-in-yandex-metrica/)讲述的是Yandex分析型数据库架构的演进。

从MySQL同步到ClickHouse，ClickHouse官方提供了[MaterializeMySQL](https://clickhouse.tech/docs/en/engines/database-engines/materialize-mysql/)引擎，支持ClickHouse作为MySQL的Replica读取binlog进行实时同步，但是目前(2021-5-11)还处于实验阶段。还有一个是基于[Altinity/clickhouse-mysql-data-reader](https://github.com/Altinity/clickhouse-mysql-data-reader)进行数据同步，但是由于项目是python写的，有问题不好解决。

这里我们使用Debezium+Kafka方式将MySQL的数据同步到ClickHouse中，[Debezium](https://blog.hufeifei.cn/2021/03/13/DB/mysql-binlog-parser/)在我上一篇文章中有介绍了，使用Binlog的方式进行数据同步，一方面能和MySQL上的业务解耦，另一方面ClickHouse适合大批量写入，需要引入Kafka这样的中间件暂存binlog消息。

Debezium官方提供了从mysql同步到postgresql和elasticsearch的[例子](https://github.com/debezium/debezium-examples/blob/master/unwrap-smt/README.md)。由于ClickHouse与PostgreSQL区别，所以不能使用kafka-connect-jdbc去写入Clickhouse，这里打算使用ClickHouse自带的Kafka引擎和物化视图来实现Kafka到ClickHouse的写入。

整体的架构图如下：

![](https://raw.githubusercontent.com/holmofy/drawio/42ad13fba9970745658c9fa434d930c32270698d/svg/mysql-to-clickhouse.svg)

# MySQL配置

开启binlog并设置`binlog_format`为ROW模式

```config
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html
[mysqld]
# ...
# ----------------------------------------------
# Enable the binlog for replication & CDC
# ----------------------------------------------
# 启用binlog，设置log过期时间，和日志格式
# 生产环境，日志保留时间设长一点。
# 日志必须设置为ROW模式
# serverId要保证集群中唯一
server-id         = 223344
log_bin           = mysql-bin
expire_logs_days  = 1
binlog_format     = row
```

# 搭建zookeeper+kafka集群

建议使用docker

# ClickHouse
