---
title: 安装InnoDBCluster
date: 2025-05-08
categories: Linux运维
tags:
- Linux
- MySQL
- InnoDBCluster
- K8S
---

Oracle官方提供了MySQL集群方案——[InnoDBCluster](https://dev.mysql.com/doc/mysql-shell/9.3/en/mysql-innodb-cluster.html)

![InnoDB Cluster架构图](https://dev.mysql.com/doc/mysql-shell/9.3/en/images/innodb-cluster-overview.png)

如果自己部署的话，需要对着官方文档中的[InnoDBCluster](https://dev.mysql.com/doc/mysql-shell/9.3/en/mysql-innodb-cluster.html)和[MySQL Router](https://dev.mysql.com/doc/mysql-shell/9.3/en/admin-api-deploy-router.html)一步一步去配置了。

不过好在官方也提供了[K8S Operator](https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-installation.html)简化MySQL集群的部署。几个命令下来就能部署好，唯一需要注意的就是oracle官方的镜像国内拉取不到，所以我都是本地翻墙拉镜像后推到harbor私服。

按照官方文档的一个router，三个server的模式部署好：

![Router](https://github.com/user-attachments/assets/d69870e6-8734-42f1-9d16-313cc73ef1b8)

![InnoDB Cluster](https://github.com/user-attachments/assets/2eb40702-982e-4e9b-95d9-9ab9f3fa4149)

MySQL-Router会暴露读写、读写分离等多个端口，以及一个监控router状态的http探针端口：

![MySQL Router服务暴露端口](https://github.com/user-attachments/assets/5864bbd0-3a8d-44b8-96e6-58962867ba68)
