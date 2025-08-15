---
title: vSphere & ESXi虚拟化
date: 2024-09-08
categories: Linux运维
tags: 
- Linux
- vSphere
- ESXi
---

我之前的云服务器都是用[cockpit](https://github.com/cockpit-project/cockpit)去管理的，这是RedHat开发的一个极其轻量的服务器管理Web界面。

但是如果是自己买的硬件资源，现在的服务器硬件都非常便宜，几万块就能买一台配置相当高的刀片机，但是这么大的资源不可能直接装操作系统去管理，出了问题回滚、销毁的成本太高了，怎么做虚拟化，把硬件虚拟成多台服务器就非常重要了。

目前主流的就是用VMWare家的套件。VMWare有ESXi的单机管理工具，也有高可用集群vSphere的管理工具。

![VMware & vSphere](https://i.ibb.co/21jQMzCK/image.png)

简单来说：

* **ESXi** 是 VMware 的虚拟化**操作系统（Hypervisor）**。它直接安装在物理服务器上，用来运行和管理虚拟机。你可以把它理解为“虚拟化底座”，单台服务器级别的虚拟化靠它就能完成。

* **vSphere** 不是一个单独的软件，而是 VMware 虚拟化的**产品套件**，它包含了 ESXi 和 vCenter 等多个组件。

  * ESXi：运行虚拟机的 Hypervisor
  * vCenter Server：集中式管理工具，可以统一管理多台 ESXi 主机、做集群、高可用、分布式调度等
  * 其他功能：如 vMotion（虚拟机热迁移）、HA（高可用）、DRS（资源调度）、vSAN（分布式存储）等

### 区别总结

* **层级不同**：ESXi 是底层 Hypervisor，vSphere 是产品套件的总称。
* **功能范围不同**：单个 ESXi 只能管自己；要管理多台 ESXi 并启用高级功能，就需要 vSphere（主要是 vCenter）。
* **关系**：你买 vSphere，其实就是买了 ESXi + vCenter 等一整套能力。

👉 举个比喻：

* ESXi = 单台服务器上的“虚拟机工厂”
* vCenter（属于 vSphere）= 工厂群的“管理中心”
* vSphere = 这套“工厂+管理中心”的整体解决方案
