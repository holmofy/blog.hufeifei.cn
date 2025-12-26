---
title: K8S的CNI网络插件
date: 2024-10-11
categories: 分布式
tags: 
- Distributed
- K8S
keywords:
- 分布式
---

网络架构是 K8s 中较为复杂的方面之一。K8s 网络模型本身对某些特定的网络功能有着一定的要求，因此，业界已经有了不少的网络方案来满足特定的环境和要求。CNI 意为容器网络的 API 接口，为了让用户在容器创建或销毁时都能够更容易地配置容器网络。在本文中，作者将带领大家理解典型网络插件地工作原理、掌握 CNI 插件的使用。

## 一、CNI 是什么

首先我们介绍一下什么是 CNI，它的全称是 Container Network Interface，即容器网络的 API 接口。

<img width="1007" height="692" alt="image" src="https://github.com/user-attachments/assets/4cc35712-aaec-40a6-8087-fbbe01a59579" />

它是 K8s 中标准的一个调用网络实现的接口。Kubelet 通过这个标准的 API 来调用不同的网络插件以实现不同的网络配置方式，实现了这个接口的就是 CNI 插件，它实现了一系列的 CNI API 接口。常见的 CNI 插件包括 Calico、flannel，也是Kubesphere的KubeKey支持的两种CNI插件。

## 二、Kubernetes 中如何使用 CNI

K8s 通过 CNI 配置文件来决定使用什么 CNI。

基本的使用方法为：

* 首先在每个结点上配置 CNI 配置文件(`/etc/cni/net.d/xxnet.conf`)，其中 xxnet.conf 是某一个网络配置文件的名称(比如`calico-kubeconfig`)；
* 安装 CNI 配置文件中所对应的二进制插件；
* 在这个节点上创建 Pod 之后，Kubelet 就会根据 CNI 配置文件执行前两步所安装的 CNI 插件；
* 上步执行完之后，Pod 的网络就配置完成了。

![](https://pic3.zhimg.com/v2-6345d1836a098397291f26fe4afd38d4_1440w.jpg)

在集群里面创建一个 Pod 的时候，首先会通过 apiserver 将 Pod 的配置写入。apiserver 的一些管控组件（比如 Scheduler）会调度到某个具体的节点上去。Kubelet 监听到这个 Pod 的创建之后，会在本地进行一些创建的操作。当执行到创建网络这一步骤时，它首先会读取刚才我们所说的配置目录中的配置文件，配置文件里面会声明所使用的是哪一个插件，然后去执行具体的 CNI 插件的二进制文件，再由 CNI 插件进入 Pod 的网络空间去配置 Pod 的网络。配置完成之后，Kuberlet 也就完成了整个 Pod 的创建过程，这个 Pod 就在线了。

![](https://qingwave.github.io/img/blog/k8s-cni-arch.png)

大家可能会觉得上述流程有很多步（比如要对 CNI 配置文件进行配置、安装二进制插件等等），看起来比较复杂。

但如果我们只是作为一个用户去使用 CNI 插件的话就比较简单，因为很多 CNI 插件都已提供了一键安装的能力。以我们常用的 Flannel 为例，如下图所示：只需要我们使用 kubectl apply Flannel 的一个 Deploying 模板，它就能自动地将配置、二进制文件安装到每一个节点上去。

## 三、哪个 CNI 插件适合我

社区有很多的 CNI 插件，比如 [Calico](https://github.com/projectcalico/calico), [flannel](https://github.com/flannel-io/flannel), [Cilium](https://github.com/cilium/cilium), 阿里的[Terway](https://github.com/AliyunContainerService/terway) 等等。那么在一个真正具体的生产环境中，我们要选择哪一个 CNI 插件呢？

这就要从 CNI 的几种实现模式说起。我们需要根据不同的场景选择不同的实现模式，再去选择对应的具体某一个插件。

通常来说，CNI 插件可以分为三种：Overlay、路由及 Underlay。

![](https://pic2.zhimg.com/v2-b1999f128e6787dd47c1884984f7a707_r.jpg)

* Overlay 模式的典型特征是容器独立于主机的 IP 段，这个 IP 段进行跨主机网络通信时是通过在主机之间创建隧道的方式，将整个容器网段的包全都封装成底层的物理网络中主机之间的包。该方式的好处在于它不依赖于底层网络；
* 路由模式中主机和容器也分属不同的网段，它与 Overlay 模式的主要区别在于它的跨主机通信是通过路由打通，无需在不同主机之间做一个隧道封包。但路由打通就需要部分依赖于底层网络，比如说要求底层网络有二层可达的一个能力；
* Underlay 模式中容器和宿主机位于同一层网络，两者拥有相同的地位。容器之间网络的打通主要依靠于底层网络。因此该模式是强依赖于底层能力的。



refs: 
* https://zhuanlan.zhihu.com/p/466113622
* https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/work-with-terway
* https://qingwave.github.io/how-to-write-k8s-cni/
* https://ranchermanager.docs.rancher.com/zh/faq/container-network-interface-providers
