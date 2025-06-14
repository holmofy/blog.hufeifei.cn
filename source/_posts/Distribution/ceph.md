---
title: 从Ceph俯瞰分布式文件系统
date: 2025-04-21
categories: 分布式
tags: 
- Distributed
- Ceph
- Kubernetes
---

## Ceph

Ceph 是一个Linux 的 PB 级分布式文件系统，提供块、文件和对象存储三个层次的抽象，并已部署在大规模生产集群中。

```lua
+------------------------+
| 应用层 / 客户端       |
+------------------------+
| 对象存储（API访问）   |  <-- 最高抽象
+------------------------+
| 文件存储（路径访问）   |  <-- 中等抽象
+------------------------+
| 块存储（原始块设备）   |  <-- 最底层
+------------------------+
| 物理磁盘、SSD 等       |
```

| 特性     | 块存储（Block）        | 文件存储（File）       | 对象存储（Object）       |
| ------ | ----------------- | ------------------ | ---------------- |
| 访问方式   | 以块为单位（如磁盘）        | 以路径访问（/dir/file） | 以对象+ID 访问（API）     |
| 协议/API | iSCSI、RBD、NVMe-oF | NFS、CIFS、CephFS  | S3、Swift、HTTP REST |
| 元数据支持  | 无                 | 基本文件属性           | 支持丰富自定义元数据         |
| 扩展性    | 中                 | 中到强（如 CephFS）    | 极强（对象存储优于其他）       |
| 并发能力   | 高                 | 中（锁机制有时成瓶颈）      | 高                  |
| 应用场景   | 数据库、VM、K8S 卷      | 文件共享、工程文件存储      | 云应用、日志、备份、AI       |
| 文件系统支持 | 客户端自己创建           | 有                | 无（扁平结构）            |

Ceph的历史比Hadoop更久远。Hadoop的HDFS是文件存储层的抽象，并且由于块粒度为128MB，所以只适合大数据应用。MinIO使用兼容Amazon S3 API的对象存储，抽象层次更高，MinIO也可以使用Ceph作为底层存储。

```plantuml
@startuml
skinparam monochrome true
skinparam defaultFontName "SansSerif"
skinparam ArrowColor Black
skinparam FontSize 14

start
:2004 - [[https://github.com/liewegas Sage Weil]] 开始 Ceph 项目;
:2006 - 博士论文发布，设计 CRUSH 数据分布算法;
:2010 - [[https://github.com/ceph/ceph Ceph]] 开源发布;
:2011 - 创建 Inktank 公司，商业化支持;
:2014 - [[https://ceph.io/en/news/blog/2014/red-hat-to-acquire-inktank/ Red Hat 收购 Inktank]];
:2015~2020 - 云原生集成（如 [[https://github.com/rook/rook Rook]]）、功能日趋成熟;
:2020~至今 - 聚焦云原生与多租户、Cephadm 成熟;
stop
@enduml
```

Ceph本身三个层次的抽象都支持，块存储（RBD）、文件存储（CephFS）、对象存储（RGW）

![Ceph架构图](https://docs.ceph.com/en/latest/_images/stack.png)

| 抽象层级     | Ceph 组件                 | 对应场景                       | 说明                        |
| -------- | ----------------------- | -------------------------- | ------------------------- |
| **块存储**  | RBD（RADOS Block Device） | VM磁盘、数据库持久卷、Kubernetes PVC | 提供类似于裸盘或云硬盘（如 AWS EBS）的能力 |
| **文件存储** | CephFS                  | 文件共享、POSIX 文件系统、高性能计算      | 提供类似 NFS 的 POSIX 文件系统语义   |
| **对象存储** | RGW（RADOS Gateway）      | 云原生应用、备份归档、大数据存储           | 提供兼容 S3、Swift 的对象接口       |

## 在K8S中使用Ceph

目前在K8S中使用Ceph有两种方式：

* 直接使用 Ceph（你自己搭建、管理 Ceph 集群，再通过 [Ceph-CSI](https://github.com/ceph/ceph-csi) 接入 K8S）
* 使用 Rook-Ceph（通过 Kubernetes Operator 部署和管理 Ceph，完全云原生）

![Rook架构图](https://rook.io/docs/rook/latest-release/Getting-Started/ceph-storage/Rook%20High-Level%20Architecture.png)

使用Ceph-CSI插件自己搭建的有几种模式，分别对应的ceph提供的几个抽象层次：
* [rbd](https://github.com/ceph/ceph-csi/blob/devel/docs/rbd/deploy.md)，官方仓库有相应的[K8S配置文件](https://github.com/ceph/ceph-csi/tree/devel/deploy/rbd/kubernetes)
* [cephfs](https://github.com/ceph/ceph-csi/blob/devel/docs/cephfs/deploy.md)，官方仓库有相应的[K8S配置文件](https://github.com/ceph/ceph-csi/tree/devel/deploy/cephfs/kubernetes)

另一种是通过Rook方式部署，Rook是社区实现的用来管理Ceph集群的Kubernetes Operator。Kubernetes Operator 是 Kubernetes 生态系统中的一个重要组件，它允许开发者将应用程序的部署、管理和运维任务自动化。Operator 基于 Kubernetes 的资源和控制器概念，通过自定义资源（CRD）和自定义控制器来实现对应用的管理。

![CRD自定义资源](https://github.com/user-attachments/assets/2cbc1a4f-1937-4159-be97-b51ad101fb45)

refs: 
* https://www.infoq.cn/article/B5VpI6e66leBUbIISap4
* https://www.xinfinite.net/t/topic/8743
* https://ceph.io/en/news/blog/2014/red-hat-to-acquire-inktank/
