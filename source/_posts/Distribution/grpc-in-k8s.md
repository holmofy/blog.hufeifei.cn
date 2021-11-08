---
title: grpc在k8s中的负载均衡问题
date: 2021-10-11
categories: 分布式
tags: 
- Distributed
- gRPC
- Kubernetes
---

线上skywalking架构：

![skywalking](./skywalking.svg)


两台skywalking-oap接受并分析由agent采集的trace数据，但是问题是两台oap服务负载不均衡。

![grpc-in-k8s](https://p.pstatp.com/origin/pgc-image/3f7d24cab5424fdcb455ed043ad06337)

##k8s的service四层负载均衡

为了排除k8s的service负载均衡的问题，在线下环境还原了请求的过程。

skywalking提供了grpc(11800端口)和rest(12800端口)两种协议的服务。

从下图可以看到，skywalking提供了11800和12800的监听端口，以及连接ElasticSearch的9200端口

![skywalking](https://p.pstatp.com/origin/pgc-image/304a8e74bcab495aba183d0654daf135)

第一次请求连上了skywalking-oap1

![](https://p.pstatp.com/origin/pgc-image/dd123b9774c2484785a128610b1dd361)

第二次请求连上了skywalking-oap2

![](https://p.pstatp.com/origin/pgc-image/b013fc7590584e3d8102c09fb7e2ab3e)

多次请求负载均衡是没有问题的。但是请求会断开连接，实际上是两次**连接**连向了两台不同的server。这个概念很重要，请求和连接不是一个事物，多个请求可以复用一个连接。

##grpc长连接导致负载不均衡

观察线上的两台oap发现，两台server都维持了大量的长连接，其中负载高的一台明显连接数更多。

![](https://p.pstatp.com/origin/pgc-image/3d9e91705fc844ca97888f5955d079c1)

由于线上skywalking-agent和skywalking-oap使用的是grpc进行通信的，grpc基于http/2会维持一个长连接。k8s的service无法识别应用层的负载均衡。

##对比常见的负载均衡实现

**dubbo与SpringCloudRibbon的客户端负载均衡**

Dubbo因为有自己的注册中心可以直接获取服务ip，负载均衡直接由Dubbo客户端实现，两次调用会路由到不同的service，即使client持有多个service实例的连接，客户端也能根据连接个数进行负载均衡。这本质上和SpringCloud中的Ribbon原理一样。这种情况直接就不需要k8s的service来实现负载均衡。

![Dubbo Architecture](https://camo.githubusercontent.com/e11a2ff9575abc290657ba3fdbff5d36f1594e7add67a72e0eda32e449508eef/68747470733a2f2f647562626f2e6170616368652e6f72672f696d67732f6172636869746563747572652e706e67)

**基于反向代理的负载均衡**

Nginx，Apache，HAProxy等反向代理服务器都支持负载均衡的功能。

![](https://www.technicalhosts.com/wp-content/uploads/2020/08/Reverse-proxy.gif)



像Nginx是能识别HTTP消息的，即使是维持了长连接，也能截取出完整的http消息，从而实现应用层的负载均衡。

Nginx在1.9.0添加了[`ngx_stream_proxy_module`模块](http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html)支持TCP/UDP的反向代理，但是它只能处理TCP连接，但是处理不了应用层的请求负载均衡。

比如[使用nginx为mysql建立高可用的反向代理](https://www.nginx.com/blog/mysql-high-availability-with-nginx-plus-and-galera-cluster/)，它解析不了同一个mysql连接中的两条sql请求。比如想进行读写分离，nginx就实现不了，这些大多是在应用端实现的，或者使用专业的反向代理，比如[ProxySQL](https://proxysql.com/blog/configure-read-write-split/)。

k8s的service就有点类似于nginx的tcp反向代理。当然只是说很像，实际上区别还是很大的。

下面看一下k8s的service实现原理。

##k8s的service实现原理

[k8s的service是基于虚拟ip实现的](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)：使用iptables实现路由转发。k8s的service代理有三种运行模式：

## 1、userspace代理模式

这种模式，kube-proxy 会监控 Kubernetes control plane 对 Service 对象和 Endpoints 对象的添加和移除操作。 对每个 Service，它会在本地 Node 上打开一个端口（随机选择）。 任何连接到“代理端口”的请求，都会被代理到 Service 后端的某个Pod上（如 `Endpoints` 所报告的一样）。 使用哪个后端 Pod，是 kube-proxy 基于 `SessionAffinity` 来确定的。

最后，它配置 iptables 规则，将到达该 Service 的 `clusterIP`（是虚拟 IP） 和 `Port` 的请求重定向到代理的后端Pod的端口。

默认情况下，用户空间模式下的 kube-proxy 通过**轮询**算法选择后端服务。

![Services overview diagram for userspace proxy](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

## 2、iptables 代理模式

这种模式，`kube-proxy` 会监控 Kubernetes control plane 对 Service 对象和 Endpoints 对象的添加和移除。对每个 Service，它会配置 iptables 规则，从而捕获到达该 Service 的 `clusterIP` 和端口的请求，进而将请求重定向到 Service 的后端中的某个 Pod 上面。 对于每个 Endpoints 对象，它也会配置 iptables 规则，这个规则会选择一个后端Pod。

默认情况下，kube-proxy 在 iptables 模式下**随机**选择一个后端。

使用 iptables 处理流量具有较低的系统开销，因为流量由 Linux netfilter 处理， 而无需在用户空间和内核空间之间切换。 这种方法也可能更可靠。

如果 kube-proxy 在 iptables 模式下运行，并且所选的第一个 Pod 没有响应， 则连接失败。 这与用户空间模式不同：在这种情况下，kube-proxy 将检测到与第一个 Pod 的连接已失败， 并会自动使用其他后端 Pod 重试。

你可以使用 Pod [就绪探针](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) 验证后端 Pod 可以正常工作，以便 iptables 模式下的 kube-proxy 仅看到测试正常的Pod。 这样做意味着你避免将流量通过 kube-proxy 发送到已知已失败的 Pod。

![iptables代理模式下Service概览图](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

## 3、IPVS模式

在 `ipvs` 模式下，kube-proxy 监控 Kubernetes Services和Endpoints，调用 `netlink` 接口相应地创建 IPVS 规则， 并定期将 IPVS 规则与 Kubernetes 服务和端点同步。 该控制循环可确保IPVS 状态与所需状态匹配。访问Service时，IPVS 将流量定向到后端Pod之一。

IPVS代理模式基于类似于 iptables 模式的 netfilter hook函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。 与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

IPVS 提供了更多选项来平衡后端 Pod 的流量。 这些是：

- `rr`：轮替（Round-Robin）
- `lc`：最少链接（Least Connection），即打开链接数量最少者优先
- `dh`：目标地址哈希（Destination Hashing）
- `sh`：源地址哈希（Source Hashing）
- `sed`：最短预期延迟（Shortest Expected Delay）
- `nq`：从不排队（Never Queue）

> **说明：**
>
> 要在 IPVS 模式下运行 kube-proxy，必须在启动 kube-proxy 之前使 IPVS 在节点上可用。
>
> 当 kube-proxy 以 IPVS 代理模式启动时，它将验证 IPVS 内核模块是否可用。 如果未检测到 IPVS 内核模块，则 kube-proxy 将退回到以 iptables 代理模式运行。

![IPVS代理的 Services 概述图](https://d33wubrfki0l68.cloudfront.net/2d3d2b521cf7f9ff83238218dac1c019c270b1ed/9ac5c/images/docs/services-ipvs-overview.svg)

在这些代理模型中，绑定到服务 IP 的流量： 在客户端不了解 Kubernetes 或服务或 Pod 的任何信息的情况下，将 Port 代理到适当的后端。

如果要确保每次都将来自特定客户端的连接传递到同一 Pod， 则可以通过将 `service.spec.sessionAffinity` 设置为 "ClientIP" （默认值是 "None"），来基于客户端的 IP 地址选择会话关联。 你还可以通过适当设置 `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds` 来设置最大会话停留时间。 （默认值为 10800 秒，即 3 小时）。

##K8s中grpc负载均衡的解决方案

前面已经分析了k8s的service是无法实现应用层的负载均衡的。grpc基于http/2，因为http/2是长连接，负载均衡需要发生在每次调用，而非每次连接。K8s识别不了http/2的请求，就无法实现grpc的负载均衡。

## 1、使用Nginx进行反向代理

既然grpc基于http/2，那么可以使用Nginx进行grpc的反向代理，因为[Nginx在1.9.5开始支持Http/2协议](http://nginx.org/en/docs/http/ngx_http_v2_module.html)。这个方案能完全保证流量的均匀分配。

但是架构上就比较复杂，为了防止skywalking-oap的pod重启，ip改变后nginx需要重新修改配置。那么需要为每个skywalking-oap创建一个Service。另外为了防止Nginx重启后Pod的ip改变，Nginx也需要创建一个Service。

![](./skywalking-nginx.svg)

一个简单的方法是使用K8s的Ingress代替Nginx，[Ingress本身也有nginx的实现](https://kubernetes.github.io/ingress-nginx/)。

![](./skywalking-ingress.svg)

## 2、修改K8s的service运行模式

我们退而求其次，无法实现每次grpc调用的负载均衡，保证连接数的均衡，也算进一大步了。

前面分析K8s的Service运行原理的时候，Service有三种运行模式：

* userspace代理模式：过**轮询**算法选择后端服务
* iptables代理模式：**随机**选择后端服务
* IPVS模式：支持多种模式
  - `rr`：轮替（Round-Robin）
  - `lc`：最少链接（Least Connection），即打开链接数量最少者优先
  - `dh`：目标地址哈希（Destination Hashing）
  - `sh`：源地址哈希（Source Hashing）
  - `sed`：最短预期延迟（Shortest Expected Delay）
  - `nq`：从不排队（Never Queue）

使用轮询和随机的方式创建连接，问题是连接断了后重连，会出现连接不均衡的问题。

那么可以使用IPVS的`lc`模式，让连接优先分配给打开链接数少的server。这样能保证连接数的均衡。

## 3、为grpc添加LoaderBalancer组件

原生grpc就没有服务发现这个概念，而是使用[另外的LoaderBalancer组件实现](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)负载均衡。

![](https://p.pstatp.com/origin/pgc-image/10fbe36a93e9450ea628dbc3ccaf04f0)

这种方式类似于Dubbo的方案，让客户端实现负载均衡。但是实现起来就比较复杂了，除了要多部署一个LoadBalancer，还需要在skywalking-agent中配置grpc负载均衡的策略。skywalking-oap注册到LoadBalancer中也是一个大问题。

