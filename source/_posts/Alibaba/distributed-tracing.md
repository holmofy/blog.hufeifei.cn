---
title: 从谷歌Dapper到阿里EagleEye
date: 2020-07-23 12:26
categories: JAVA
mathjax: true
tags: 
- Dapper
- Alibaba
- Distributed
---

# 1、分布式链路追踪技术解决的问题

* 分布式系统服务非常多，很复杂
* 每个服务可能由不同项目组开发，没有一个人能详细地了解所有的系统。
* 每个服务都可能集群部署，有很多台机器，整个系统可能有成千上万台机器。
* 服务可能由不同语言开发的。
* 当需要了解系统的整体表现或系统瓶颈时，需要知道整个调用链路的每个部分的耗时情况。
* 当一次链路过程调用出错了，需要知道具体是哪个服务的哪一台机器出错，而不是到每一台机器上去看日志。

![image.png](http://tva1.sinaimg.cn/large/bda5cd74ly1gh03y8q74cj21aq0pykcr.jpg)

# 2、Google的Dapper

Google很早就已经为微服务化了，并在2010年发表了《[Dapper - a Large-Scale Distributed Systems Tracing Infrastructure](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf)》论文介绍他们的分布式系统跟踪技术。

Dapper论文中对实现一个分布式跟踪系统提出了如下几个需求：

- 性能低损耗：分布式跟踪系统对服务的性能损耗应尽可能做到可以忽略不计，尤其是对性能敏感的应用不能产生损耗
- 对应用透明：即要求尽可能用非侵入的方式来实现跟踪，尽可能做到业务代码的低侵入，对业务开发人员应该做到透明化
- 可伸缩性：高伸缩性是指不能随着微服务和集群规模的扩大而使分布式跟踪系统瘫痪
- 跟踪数据可视化和迅速反馈：即要有可视化的监控界面，从跟踪数据收集、处理到结果的展现尽量做到快速，就可以对系统的异常状况作出快速的反应
- 持续的监控：即要求分布式跟踪系统必须是7x24小时工作的，否则将难以定位到系统偶尔抖动的行为

在论文中举了个例子：

![Google Dapper](http://tva1.sinaimg.cn/large/bda5cd74ly1gh049xp5d1j20cl0auwf1.jpg)

A~E分别表示五个服务，用户发起一次请求到前端系统A，然后A分别发送RPC请求到中间层的B和C，B处理请求后返回，C还要发起两个RPC请求到两个后台系统D和E。

论文中使用***Trace***表示对一次请求完整调用链的跟踪，将两个服务（例如上面的服务A和服务B之间RPC）的请求/响应过程叫做一次***Span***。

整个链路就像一个RPC调用的树形结构。当然，核心的追踪数据模型不只局限于特定的RPC框架，Dapper还能追踪到外界发过来的HTTP请求，和末端对数据库等存储的读写。

而每一个这样的链路用定义一个全局唯一的TraceID进行标识。链路中的所有Span都将获取到这个TraceID，每个Span还有一个自己的SpanID以及ParentSpanID，ParentSpanID表示上一级Span。例子中A服务的ParentSpanID为空，SpanID为1；然后B服务的ParentSpanID为1，SpanID为2；C服务的ParentSpanID也为1，SpanID为3，以此类推。除此之外Span中还会记录自己调用其他服务的时间。

每个服务将自己直接关联的Span数据记录到一个日志文件中，会有一个`Dapper Collectors`集群实时去收集这些日志数据并进行处理，处理完成后每一个调用链路都会被作为一行Trace记录会写入到BigTable中。最后由监控平台展示这些数据。

![Dapper架构图](http://tva1.sinaimg.cn/large/bda5cd74ly1gh059qtux4j20jg0d10uy.jpg)

# 3、阿里的鹰眼

业界已有很多链路追踪技术的实现了，Twitter基于Google的Dapper论文开发了[Zipkin](https://github.com/openzipkin/zipkin)并提供了开源版本。Zipkin提供了Java版本的埋点库[Brave](https://github.com/openzipkin/brave)，[Spring Cloud Seluth](https://github.com/spring-cloud/spring-cloud-sleuth)通过自动化配置可以很方便地将Brave整合进Java应用以提供链路追踪功能。Uber公司的技术团队受[Dapper](https://research.google.com/pubs/pub36356.html) 和 [OpenZipkin](https://zipkin.io/)启发用Go语言也开发了一款链路追踪平台[Jaeger](https://github.com/jaegertracing/jaeger)。开源社区也启动了[OpenTracing计划](https://github.com/opentracing)，旨在将各种语言和平台间的链路追踪名词和概念进行[标准化](https://opentracing.io/specification/)。

EagleEye （鹰眼）是Google 的分布式调用跟踪系统 Dapper 在淘宝的Java实现，现在已经被作为中间件平台应用到阿里内部的各个业务线了。

在前端请求到达服务器时，应用容器在执行实际业务处理之前，会先执行EagleEye的埋点逻辑（基于Servlet的Filter的机制），埋点逻辑为这个前端请求分配一个全局唯一的调用链ID。这个ID在EagleEye 里面被称为 TraceId，埋点逻辑把TraceId 放在一个调用上下文对象里面，而调用上下文对象会存储在ThreadLocal里面。调用上下文里还有一个ID非常重要，在EagleEye里面被称作RpcId（等价于Dapper论文中的SpanID）。RpcId用于区分同一个调用链下的多个网络调用的发生顺序和嵌套层次关系。对于前端收到请求，生成的RpcId固定都是0。

![鹰眼](http://tva1.sinaimg.cn/large/bda5cd74ly1gh0qqmzynnj20fk0c7tan.jpg)

当这个前端执行业务处理需要发起RPC调用时，淘宝的RPC调用客户端HSF会首先从当前线程ThreadLocal上面获取之前EagleEye设置的调用上下文。然后，把RpcId递增一个序号。在EagleEye里使用多级序号来表示RpcId，比如前端刚接到请求之后的RpcId是0，那么它第一次调用RPC服务A时，会把RpcId改成0.1。之后，调用上下文会作为附件随这次请求一起发送到远程的HSF服务器。

HSF服务端收到这个请求之后，会从请求附件里取出调用上下文，并放到当前线程ThreadLocal上面。如果服务A在处理时，需要调用另一个服务，这个时候它会重复之前提到的操作，唯一的差别就是RpcId会先改成0.1.1再传过去。服务A的逻辑全部处理完毕之后，HSF在返回响应对象之前，会把这次调用情况以及TraceId、RpcId都打印到它的访问日志之中，同时，会从ThreadLocal清理掉调用上下文。

访问日志里面，一般会记录调用时间、远端IP地址、结果状态码、调用耗时之类，也会记录与这次调用类型相关的一些信息，如URL、服务名、消息topic等。很多调用场景会比上面说的完全同步的调用更为复杂，比如会遇到异步、单向、广播、并发、批处理等等，这时候需要妥善处理好ThreadLocal上的调用上下文，避免调用上下文混乱和无法正确释放。另外，采用多级序号的RpcId设计方案会比单级序号递增更容易准确还原当时的调用情况。

最后，EagleEye实时集群把调用链相关的所有访问日志都收集上来存储在HDFS和HBase中，EagleEye分析系统按TraceId汇总在一起，生成统计数据并生成报表，在鹰眼的控制台可以准确看到当时的调用情况了。

![Alibaba Eagleeye](http://tva1.sinaimg.cn/large/bda5cd74ly1gh0p1t1ioxj20ez09ajsr.jpg)

# 4、源码导读

首先大多数链路的入口是用户发起的HTTP请求。鹰眼提供了一个EagleEyeFilter用于处理入口处的请求，其中包括生成整个链路的TraceID等操作。

![](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuSfAB4kiq2jEBIhBIItHSylCAKajigdHrNLDJCz9TQrCXOXmEQJcfG2L0m00)

接着就是HSF的服务调用。服务调用方会负责将TraceID等链路上下文信息传递给下游，服务提供方则取出RPC调用中取出链路的上下文信息存入ThreadLocal以便继续传递给下游链路。

![](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuKhEpzKhISnFIipNooXEqylCAyjFJYp9pC_JIylCAKajqWiAS7O3isngT7MTSp9JyqeWV1Ar1gSMbQKMGRL2p458kYQcvwIwLgQYc0_HWQa8nII7rBmKe3y0)

除了同步的服务调用，还有异步的MQ消息。

![MetaQ](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuKhEpzLBpCbCIanAr2lAJyvEBSajr4lEoKpDAz7BoC_FrdFEpoikpKtrJIqkJanFzG0AsTJewlgcbYG6OafvvXOGOMHmQbuADlD0uhGmBAGeCHbXeWDG0kXp0000)

最后到TDDL、Tair等数据存储层。

![](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuKhEpzKhISnFIipNAqb9oT7BpS_BBCalqajDJCz9JQrCrGi1Yhf2EGesDRgw2cXQ84fTWKfTeOmGiAHdRa4EfWYNGsfU2j1Y0000)



参考链接：

Google Dapper：https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf

Google Dapper中文翻译：https://bigbully.github.io/Dapper-translation/

分布式追踪系统概述及主流开源系统对比: https://zhuanlan.zhihu.com/p/71024024

Distributed Systems Observability by Cindy Sridharan: https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html

https://spring.io/blog/2016/02/15/distributed-tracing-with-spring-cloud-sleuth-and-spring-cloud-zipkin

http://jm.taobao.org/2014/03/04/3465/

http://mw.alibaba-inc.com/products/eagleeye/_book/middle-insert-eagleEye-sunhua.html

