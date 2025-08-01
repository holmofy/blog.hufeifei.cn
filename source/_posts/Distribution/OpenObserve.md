---
title: OpenObserve观测平台
date: 2025-07-11
categories: 分布式
tags: 
- Distributed
- OpenObserve
- OpenTelemetry
keywords:
- 分布式
- 链路追踪
- OpenObserve
---

OpenObserve是目前用过的性能最好的观测行平台，底层基于Rust写的，采用的是parquet列式存储，数据压缩效率极高，单机版的性能就已经超过很多市面上其他技术了。

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/78c979be-eceb-4577-95f4-45fe6d7a90ef" />

观测性数据属于时序数据范畴，所以OpenObserve和大多数时序数据库一样使用的是LSM-Tree的存储数据结构。这就导致数据采集量过大的时候，经常出现memtableOverflow的问题。

所以不得不迁移到[集群版的OpenObserve](https://openobserve.ai/docs/architecture/#high-availability-ha-mode)。

![OpenObserve高可用架构](https://openobserve.ai/docs/images/arch-ha.webp)

OpenObserve高可用架构中分以下几个角色：

* Router、Querier、Ingester、Compactor 和 AlertManager 节点都可以水平扩展以适应更高的流量。
* NATS 用作集群协调器并存储节点信息。它还用于集群事件。
* MySQL / PostgreSQL 用于存储元数据，如组织、用户、函数、警报规则、流模式和文件列表（parquet 文件的索引）。
* 对象存储（例如 s3、minio、gcs 等）存储 parquet 文件的所有数据。

官方提供了[基于helm charts的部署方式](https://openobserve.ai/docs/ha_deployment/)，我这边选用的是本地minio的存储方式。Router、Querier、Ingester、Compactor 和 AlertManager等节点是通过控制[`ZO_NODE_ROLE`](https://openobserve.ai/docs/environment-variables/#common)环境变量来实现不同角色的任务处理，[官方文档](https://openobserve.ai/docs/architecture/#components)有提到不同组件的作用。

* Ingester(数据采集器)：数据采集器用于接收数据采集请求，并将数据转换为 Parquet 格式并存储在对象存储中。在将数据传输到对象存储之前，它会将数据临时存储在 WAL 中。
  Ingester 包含三部分数据：
  - Memtable 中的数据
  - Immutable 中的数据
  - wal 中的 parquet 文件尚未上传到对象存储。
* Querier(查询器): 查询器用于查询数据。查询器节点完全无状态。
* Compactor(压缩器): 压缩器将小文件合并为大文件，以提高搜索效率。压缩器还强制执行数据保留策略、全流删除以及文件列表索引的更新。
* Router(路由器): 路由器将请求分发给Ingester或Querier。它还会通过浏览器中的 GUI 进行响应。路由器是一个超级简单的代理，用于在Ingester和Querier之间发送适当的请求。
* AlertManager: AlertManager 运行标准警报查询、报告作业并发送通知。

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/ed49746f-dd4f-4965-98ed-edb2aa31ec06" />

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/bd61829b-2744-42fb-aad1-6311751dd1e2" />


