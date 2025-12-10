---
title: Kubernetes Dashboard & Kubesphere & Rancher
date: 2025-08-11
categories: 分布式
tags: 
- Kubernetes
- Kubesphere
- Rancher
---

K8S已经是分布式容器调度的实施标准了，目前已知的K8S面板有多个

| 工具                       | 定位                                       | 适用人群                                     |
| ------------------------ | ---------------------------------------- | ---------------------------------------- |
| **Kubernetes Dashboard** | 官方轻量级 Web UI，只做基础资源查看和简单操作               | 想要一个简单、开箱即用的图形界面                         |
| **Rancher**              | 企业级多集群管理平台，主打“管理多个 Kubernetes 集群”        | 多集群、混合云、需要权限/认证/约束体系                     |
| **KubeSphere**           | 以 Kubernetes 为底座的 **全家桶**，面向 DevOps/应用管理 | 想要“像 OpenShift 的中国版”：CI/CD、应用商店、日志、监控全都要 |

## 功能对比

官方提供的[Kubernetes Dashboard](https://github.com/kubernetes/dashboard)是最简单的，[使用helm charts安装也极为简单](https://github.com/kubernetes/dashboard/blob/master/charts/kubernetes-dashboard/README.md)

<img width="1928" height="1040" alt="image" src="https://github.com/user-attachments/assets/a028e179-6a6f-4d16-bafc-c90970445e87" />

[Rancher](https://github.com/rancher/rancher)相对来说功能更强一点，提供了RBAC权限控制等功能，官方也提供了[helm charts](https://ranchermanager.docs.rancher.com/zh/getting-started/installation-and-upgrade/resources/choose-a-rancher-version)的安装方式。值得一提的是[k3s也是这家公司开源的](https://docs.rancher.cn/docs/k3s/)

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/0a02a3e8-9694-4664-9220-4db4851d63d1" />

[KubeSphere](https://github.com/kubesphere/kubesphere)是国内青云科技开源的产品，功能更全面，非常适合国内的用户使用体验。

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/cd69bb5f-736d-4f2d-be8d-8898e02db7ac" />

不过[7月31日](https://github.com/kubesphere/kubesphere/issues/6550)官方把项目闭源了，连文档也下掉了，[有人提前fork了前端的console项目](https://github.com/openksc/console)

| 功能                | Kubernetes Dashboard | Rancher                 | KubeSphere                        |
| ----------------- | -------------------- | ----------------------- | --------------------------------- |
| **基础资源管理**        | ✔️ 基础够用              | ✔️ 全套                   | ✔️ 全套 & 更细                        |
| **多集群管理**         | ❌ 无                  | ⭐ 强                     | ✔️ 支持，但侧重应用层                      |
| **用户与权限 (RBAC)**  | 简单                   | ⭐ 企业级（AD/LDAP/OIDC）     | ⭐ 企业级（内置工作空间结构）                   |
| **DevOps（流水线）**   | ❌                    | ❌（弱，仅整合 GitOps）         | ⭐ 内置 CI/CD（Jenkins / Tekton）      |
| **应用商店 / APP 模板** | ❌                    | ✔️ 但有限                  | ⭐ 很强（Helm、模板、应用部署）                |
| **日志系统**          | ❌                    | ❌                       | ⭐ Fluent-bit + ElasticStack（开箱即用） |
| **监控系统**          | ❌                    | 需安装                     | ⭐ 内置 Prometheus + Grafana         |
| **网络管理、CNI**      | ❌                    | ✔️ 集成 Longhorn、Calico 等 | ✔️ 支持，但可选                         |
| **多租户体系**         | ❌                    | ✔️ 项目/集群级               | ⭐ 组织 / 工作空间 / 项目三级                |
| **架构复杂度**         | ⭐ 极简                 | 中等                      | ❗ 偏重                              |
| **资源消耗**          | ⭐ 最小                 | 中等                      | ❗ 最重（ES/Prometheus 等组件）           |

KubeSphere官方把社区版阉割了重新发布出来，和之前不同的是，[KubeSphere](https://docs.kubesphere.com.cn/v4.2.0/03-installation-and-upgrade/02-install-kubesphere/01-online-install-kubernetes-and-kubesphere/#_%E5%AE%89%E8%A3%85_kubesphere)是分开安装的，使用时需要[官方的授权码进行激活](https://docs.kubesphere.com.cn/v4.2.0/03-installation-and-upgrade/02-install-kubesphere/03-activate-ks/)，最长一年续期一次。

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/a5736f3e-2eb6-4473-9286-b9a7248a49c5" />

像多集群管理等之前3.0版全都能使用的一些高级功能现在需要企业版授权码才能使用。

<img width="1912" height="954" alt="image" src="https://github.com/user-attachments/assets/7e8bcd6d-3024-4132-8140-7239a8089bc8" />

今年社区又开源了一款K8S的管理界面[Kite](https://github.com/zxh326/kite)，不仅支持多集群而且极其轻量：

<img width="1912" height="948" alt="image" src="https://github.com/user-attachments/assets/4ae0c22e-b1ac-4ebe-9718-cba2d5b35e21" />

这款的亮点在于可以自定义侧边栏，把K8S的自定义资源按照自己的需求组织成菜单组

<img width="1912" height="948" alt="image" src="https://github.com/user-attachments/assets/da72143e-7329-4f26-93b2-10fe7fff67d6" />
