---
title: 一文搞懂Docker和K8S的历史
date: 2025-08-11
categories: 分布式
tags: 
- Docker
- K8S
- Containerd
---

> 安装部署K8S的时候是不是对Docker、Kubernetes、CRI、OCI、Containerd、CNI、CSI这些概念搞得一头雾水
>
> K8S集群我部署过两三次，遇到了很多问题，如果不了解Docker和K8S的发展历史，不搞懂这些组件之间的关系，很多问题都不知道如何解决

下面我给你讲一段 **清晰、时间顺序、故事线一样的历史**，让你彻底搞懂：Docker、Kubernetes、CRI、OCI、Containerd、CNI、CSI 等组件和标准的历史与关系

# 一、2013：Docker 横空出世（改变世界的起点）

[2013 年 Docker 公司发布 **Docker Engine**](https://docs.docker.com/engine/release-notes/prior-releases/#010-2013-03-23)（用 Go 重写的 LXC 封装）。

主要贡献：

* 第一次提出了“镜像（Image）”的标准格式（分层，基于 AUFS 作为容器文件系统层）
* 定义了“容器（Container）”的规范与命令
* 提供了 registry（镜像仓库）
* 让打包、分发、运行应用变得极其简单

当时所有容器技术基本 = Docker 全家桶。

> **启示**：Docker 是第一代，“一个CLI = 打包 + 分发 + 运行”。
> 
> ![Docker commands diagram](https://img-blog.csdnimg.cn/img_convert/06a539a30efc11ba47aa2767e15ce912.png)

但 Docker 本身太重、集成太杂，后面会看到问题如何出现。

# 二、2014–2015：Kubernetes 诞生（谷歌经验的开源化）

Google 把内部的 [Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/) / [Omega](https://research.google/pubs/omega-flexible-scalable-schedulers-for-large-compute-clusters/) 分布式集群管理的思路落地开源，也需要能用 Docker 做容器运行时，于是：

* Kubernetes 直接使用 Docker Engine 作为底层运行时
* Kubelet 直接调用 Docker 的 API（dockershim）
* Docker 不仅是一个打包工具，还变成了 K8s 的 runtime backend

但这时候隐患出现了：Docker 不是专为 Kubernetes 做的，被迫兼容Docker 内部有大量额外组件。例如：

* Swarm 集群功能
* network（docker bridge）
* volume（docker volume）
* builder（buildkit）
* registry client
* secret
* CLI
* Daemon

而 Kubernetes 只需要一个：**“帮我创建容器” 的接口**

所以 Docker 过于庞大，K8s 并不想依赖。

# 三、2015–2016：OCI 标准出现（Linux 基金会介入）

Docker 的强势让大家担心它垄断整个生态，所以 [2015年6月22日 Linux 基金会牵头成立**OCI（Open Container Initiative）**](https://opencontainers.org/about/overview/)

制定两个行业标准：

1. [**运行时标准 runtime-spec**](https://github.com/opencontainers/runtime-spec)（container 的 rootfs + process 如何运行）
2. [**镜像标准 image-spec**](https://github.com/opencontainers/image-spec)（镜像层的格式）

> 2018年又加入了一个[镜像分发规范](https://github.com/opencontainers/distribution-spec): 该规范用于标准化镜像的分发标准，使 OCI 的生态覆盖镜像的全生态链路，从而成为一种跨平台的容器镜像分发标准

Docker 将其容器格式和运行时 runC 捐赠给 OCI ：runC → OCI Runtime 的参考实现

这一步非常重要： **容器行业第一次从 Docker → 标准化。** 

> 目前OCI运行时也有[很多其他实现](https://github.com/topics/oci)

# 四、2016–2017：Containerd 独立于 Docker

Docker 把底层剥离出来：

* Docker 的核心运行时模块 → containerd
* containerd 再调用 runc（OCI runtime）

从[Docker 1.11版本](https://docs.docker.com/engine/release-notes/prior-releases/#1110-2016-04-13)开始，Docker 容器运行就不是简单通过 Docker Daemon 来启动了，而是通过集成 containerd、runc 等多个组件来完成的。虽然 Docker Daemon 守护进程模块在不停的重构，但是基本功能和定位没有太大的变化，一直都是 CS 架构，守护进程负责和 Docker Client 端交互，并管理 Docker 镜像和容器。现在的架构中组件 containerd 就会负责集群节点上容器的生命周期管理，并向上为 Docker Daemon 提供 gRPC 接口。

结构图变成这样：

![Docker架构](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114161214352-808124060.png)

当我们要创建一个容器的时候，现在 Docker Daemon 并不能直接帮我们创建了，而是请求 containerd 来创建一个容器，containerd 收到请求后，也并不会直接去操作容器，而是创建一个叫做 containerd-shim 的进程，让这个进程去操作容器，我们指定容器进程是需要一个父进程来做状态收集、维持 stdin 等 fd 打开等工作的，假如这个父进程就是 containerd，那如果 containerd 挂掉的话，整个宿主机上所有的容器都得退出了，而引入 containerd-shim 这个垫片就可以来规避这个问题了。

然后创建容器需要做一些 namespaces 和 cgroups 的配置，以及挂载 root 文件系统等操作，这些操作其实已经有了标准的规范，那就是 OCI（开放容器标准），runc 就是它的一个参考实现（Docker 被逼无耐将 libcontainer 捐献出来改名为 runc 的），这个标准其实就是一个文档，主要规定了容器镜像的结构、以及容器需要接收哪些操作指令，比如 create、start、stop、delete 等这些命令。runc 就可以按照这个 OCI 文档来创建一个符合规范的容器，既然是标准肯定就有其他 OCI 实现，比如 Kata、gVisor 这些容器运行时都是符合 OCI 标准的。

所以真正启动容器是通过 containerd-shim 去调用 runc 来启动容器的，runc 启动完容器后本身会直接退出，containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。

而 Docker 将容器操作都迁移到 containerd 中去是因为当前做 Swarm，想要进军 PaaS 市场，做了这个架构切分，让 Docker Daemon 专门去负责上层的封装编排，当然后面的结果我们知道 Swarm 在 Kubernetes 面前是惨败，然后 Docker 公司就把 containerd 项目捐献给了 CNCF 基金会，containerd 开始成为整个行业的基础设施，这个也是现在的 Docker 架构。

# 五、2016–2017：Kubernetes 不想被 Docker 束缚 → CRI 出现

Kubernetes 团队提出一个需求：

> 我希望用任意容器运行时，而不是强绑定 Docker。
> 
> 我们知道 Kubernetes 提供了一个 CRI 的容器运行时接口，那么这个 CRI 到底是什么呢？这个其实也和 Docker 的发展密切相关的。
> 
> 在 Kubernetes 早期的时候，当时 Docker 实在是太火了，Kubernetes 当然会先选择支持 Docker，而且是通过硬编码的方式直接调用 Docker API，后面随着 Docker 的不断发展以及 Google 的主导，出现了更多容器运行时，Kubernetes 为了支持更多更精简的容器运行时，Google 就和红帽主导推出了 CRI 标准，用于将 Kubernetes 平台和特定的容器运行时（当然主要是为了干掉 Docker）解耦。

CRI（Container Runtime Interface 容器运行时接口）本质上就是 Kubernetes 定义的一组与容器运行时进行交互的接口，所以只要实现了这套接口的容器运行时都可以对接到 Kubernetes 平台上来。不过 Kubernetes 推出 CRI 这套标准的时候还没有现在的统治地位，所以有一些容器运行时可能不会自身就去实现 CRI 接口，于是就有了 shim（垫片）， 一个 shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上，其中 dockershim 就是 Kubernetes 对接 Docker 到 CRI 接口上的一个垫片实现。

![cri shim](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114161636291-2100977762.png)

Kubelet 不再绑定 Docker，而是通过 CRI 调用 runtime。Kubelet 通过 gRPC 框架与容器运行时或 shim 进行通信，其中 kubelet 作为客户端，CRI shim（也可能是容器运行时本身）作为服务器。

Kubernetes 运行时被抽象为两部分，也就是 CRI 定义的 API(https://github.com/kubernetes/kubernetes/blob/release-1.5/pkg/kubelet/api/v1alpha1/runtime/api.proto) 主要包括两个 gRPC 服务，ImageService 和 RuntimeService：

* **CRI runtime**（容器生命周期）：RuntimeService 则是用来管理 Pod 和容器的生命周期，以及与容器交互的调用（exec/attach/port-forward）等操作
* **CRI image service**（拉镜像）：ImageService 服务主要是拉取镜像、查看和删除镜像等操作

> 可以通过 kubelet 中的标志 `--container-runtime-endpoint` 和 `--image-service-endpoint` 来配置这两个服务的套接字。

于是出现：

* `cri-o`（Red Hat 做的兼容 CRI 的 runtime）
* `cri-containerd`（containerd 的 CRI 插件）
* Docker → dockershim（后来被移除）

架构变成：

![kubelet cri](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114161716844-2117476393.png)

这时 Kubernetes 完全和 Docker 解耦。

> 由于 Docker 当时的江湖地位很高，Kubernetes 是直接内置了 dockershim 在 kubelet 中的，所以如果你使用的是 Docker 这种容器运行时的话是不需要单独去安装配置适配器之类的，当然这个举动似乎也麻痹了 Docker 公司。
>
> ![Dockershim](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114161731987-1250727730.png)
>
> 现在如果我们使用的是 Docker 的话，当我们在 Kubernetes 中创建一个 Pod 的时候，首先就是 kubelet 通过 CRI 接口调用 dockershim，请求创建一个容器，kubelet 可以视作一个简单的 CRI Client, 而 dockershim 就是接收请求的 Server，不过他们都是在 kubelet 内置的。
>
> dockershim 收到请求后, 转化成 Docker Daemon 能识别的请求, 发到 Docker Daemon 上请求创建一个容器，请求到了 Docker Daemon 后续就是 Docker 创建容器的流程了，去调用 containerd，然后创建 containerd-shim 进程，通过该进程去调用 runc 去真正创建容器。
>
> 其实我们仔细观察也不难发现使用 Docker 的话其实是调用链比较长的，真正容器相关的操作其实 containerd 就完全足够了，Docker 太过于复杂笨重了，当然 Docker 深受欢迎的很大一个原因就是提供了很多对用户操作比较友好的功能，但是对于 Kubernetes 来说压根不需要这些功能，因为都是通过接口去操作容器的，所以自然也就可以将容器运行时切换到 containerd 来。

# 六、2018–2021：Docker 被踢出 Kubernetes（dockershim removed）

Kubernetes 官方长期想摆脱 Docker，因为：

* Docker 不是专为 K8s 设计
* Docker 守护进程太重
* Docker 的默认网络/存储和 K8s 的 CNI/CSI 冲突
* 它不是 OCI runtime

最终：

* 2020：K8s 宣布弃用 dockershim
* 2021：K8s v1.24 移除 Docker runtime

![切换到containerd](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114161749700-734539112.png)

Kubernetes 推荐的 runtime 是：

* **containerd**
* **CRI-O**

之后 Kubernetes 世界已不再包含 Docker。

切换到 containerd 可以消除掉中间环节，操作体验也和以前一样，但是由于直接用容器运行时调度容器，所以它们对 Docker 来说是不可见的。 因此，你以前用来检查这些容器的 Docker 工具就不能使用了。

你不能再使用 `docker ps` 或 `docker inspect` 命令来获取容器信息。由于不能列出容器，因此也不能获取日志、停止容器，甚至不能通过 `docker exec` 在容器中执行命令。

当然我们仍然可以下载镜像，或者用 `docker build` 命令构建镜像，但用 Docker 构建、下载的镜像，对于容器运行时和 Kubernetes，均不可见。为了在 Kubernetes 中使用，需要把镜像推送到镜像仓库中去。

> 没有了docker的一系列命令，但是k8s和containerd分别提供了自己的命令来管理容器和镜像：
> 
> | 工具         | 层级                                  | 面向谁                   | 控制什么                          |
> | ---------- | ----------------------------------- | --------------------- | ----------------------------- |
> | **crictl** | **Kubernetes 层（调用 CRI 接口）**         | 面向 Kubelet / K8s 运维   | 通过 **CRI** 控制容器运行时            |
> | **ctr**    | **containerd 内部层（直接操作 containerd）** | 面向 containerd 开发/调试人员 | 直接操作 containerd 内部 API，不走 CRI |
>
> crictl = kubelet 的”手工版“（K8s 调试工具）
> 
> ctr = containerd 的”内部调试工具“（不推荐生产用）

从上图可以看出在 containerd 1.0 中，对 CRI 的适配是通过一个单独的 CRI-Containerd 进程来完成的，这是因为最开始 containerd 还会去适配其他的系统（比如 swarm），所以没有直接实现 CRI，所以这个对接工作就交给 CRI-Containerd 这个 shim 了。

然后到了 containerd 1.1 版本后就去掉了 CRI-Containerd 这个 shim，直接把适配逻辑作为插件的方式集成到了 containerd 主进程中，现在这样的调用就更加简洁了。

![container cri](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114162034000-1785646580.png)

与此同时 Kubernetes 社区也做了一个专门用于 Kubernetes 的 CRI 运行时 CRI-O，直接兼容 CRI 和 OCI 规范。

![cri-o](https://img2020.cnblogs.com/blog/794174/202201/794174-20220114162045509-544787859.png)

这个方案和 containerd 的方案显然比默认的 dockershim 简洁很多，不过由于大部分用户都比较习惯使用 Docker，所以大家还是更喜欢使用 dockershim 方案。

但是随着 CRI 方案的发展，以及其他容器运行时对 CRI 的支持越来越完善，Kubernetes 社区在2020年7月份就开始着手移除 dockershim 方案了：https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim，现在的移除计划是在 1.20 版本中将 kubelet 中内置的 dockershim 代码分离，将内置的 dockershim 标记为维护模式，当然这个时候仍然还可以使用 dockershim，目标是在 1.23⁄1.24 版本发布没有 dockershim 的版本（代码还在，但是要默认支持开箱即用的 docker 需要自己构建 kubelet，会在某个宽限期过后从 kubelet 中删除内置的 dockershim 代码）。 那么这是否就意味这 Kubernetes 不再支持 Docker 了呢？当然不是的，这只是废弃了内置的 dockershim 功能而已，Docker 和其他容器运行时将一视同仁，不会单独对待内置支持，如果我们还想直接使用 Docker 这种容器运行时应该怎么办呢？可以将 dockershim 的功能单独提取出来独立维护一个 cri-dockerd 即可，就类似于 containerd 1.0 版本中提供的 CRI-Containerd，当然还有一种办法就是 Docker 官方社区将 CRI 接口内置到 Dockerd 中去实现。

但是我们也清楚 Dockerd 也是去直接调用的 Containerd，而 containerd 1.1 版本后就内置实现了 CRI，所以 Docker 也没必要再去单独实现 CRI 了，当 Kubernetes 不再内置支持开箱即用的 Docker 的以后，最好的方式当然也就是直接使用 Containerd 这种容器运行时，而且该容器运行时也已经经过了生产环境实践的

# 七、CNI 和 CSI 的诞生（进一步模块化）

Kubernetes 的理念：所有东西都模块化，runtime 也模块化。

于是又有两个接口出现：

* **CNI（Container Network Interface）**: 负责 Pod 网络（Calico / Flannel / Cilium / Multus）
* **CSI（Container Storage Interface）**: 负责存储系统（Ceph / NFS / EBS / Longhorn）

对应的：

| 领域   | 抽象规范    | 实现                           |
| ---- | ------- | ---------------------------- |
| 容器运行 | **CRI** | containerd / CRI-O           |
| 容器网络 | **CNI** | Calico/Cilium/Flannel/Multus |
| 容器存储 | **CSI** | Ceph/NFS/Longhorn            |


# 八、2023–2024：Containerd 2.0 移除内置 CRI

containerd 2.x 把 CRI 从内置插件变为外部独立仓库：

* **让 containerd 更加纯粹（只负责容器生命周期）**
* CRI 成为外部项目（containerd/cri）

原因：

* 内置 CRI 太重，影响 containerd 的维护
* containerd 越来越通用（也服务 VM、WASM、Serverless）
* K8s 才是 CRI 的主要用户，不要绑死 containerd 核心

```
┌────────────── Kubernetes ───────────────┐
│                 Kubelet                 │
│                     │(CRI)              │
└─────────────────────┼───────────────────┘
                      ▼
            Container Runtime
      (containerd / CRI-O / others)
                      │(OCI)
                      ▼
                 runc / kata
──────────────────────────────────────────
CNI ← Pod 网络     |     CSI ← 存储插件
```

容器生态的演化路线：

* **Docker** → 第一代，一体化
* **OCI 标准化** → 第二代，解耦镜像格式和运行时
* **K8s + CRI** → 第三代，解耦集群管理与 runtime
* **containerd 主导** → 第四代，统一的云原生容器基础设施

refs：
* https://www.cnblogs.com/hahaha111122222/p/15802334.html
