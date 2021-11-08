---
title: OpenTelemetry：一统江湖的未来
date: 2021-10-11
categories: 分布式
tags: 
- Distributed
- OpenTelemetry
keywords:
- 分布式
- 链路追踪
- Skywalking
---

> 天下大势，分久必合，合久必分——《三国演义》

##乾坤未定，混沌初开

![Linux性能观测工具](https://www.brendangregg.com/Perf/linux_observability_tools.png)

分布式未开之际，软件世界里网站后台还是单体模式。有问题可以看日志，查性能有`top`以及[sysstat](https://github.com/sysstat/sysstat)下的一批工具包：`mpstat`、`pidstat`、`iostat`，要对日志做分析统计可以上文本三剑客`awk`、`sed`、`grep`。

随着互联网的突飞猛进，摩尔定律的失效，单体模式已经承接不了互联网下的滔天流量，应用的日志体量也在飙升，由于分布式集群环境日志也不再单独分布在一台机器上，怎么监控应用的性能、怎么基于日志做指标统计也变得异常复杂。于是衍生了[分布式系统的可观测性](https://blog.hufeifei.cn/2021/09/Distribution/grafana/)的三大基石：Logging、Metrics、Tracing。今天的主角就是Tracing。

![分布式观测性三大基石](https://pic3.zhimg.com/v2-246813d3962794604c4bc409a94693d6_r.jpg)

##Dapper启蒙，百花齐放

谷歌作为互联网的巨擘，手握独步天下的搜索引擎——Google。搜索引擎作为互联网的要隘，流量增长异常惊人，这也促使谷歌发展出自己的一套分布式系统。谷歌为了增加公司影响力便于吸引人才，将内部的系统设计以一篇篇论文的形式公布给业界。

> 谷歌就是这么自信，要知道谷歌的创始人也是基于自己发表的一篇论文创建的谷歌搜索引擎。

![谷歌论文与开源技术](http://duanple.com/wp-content/uploads/2019/08/paper.jpg)

这其中每一篇论文都曾掀起过层层巨浪，比如号称三驾马车的GFS、MapReduce、Bigtable开创了大数据时代，Spanner、F1给分布式数据库指明了方向，TensorFlow引领了人工智能机器学习的发展，Borg衍生出基于K8S云原生的大生态。而[Dapper](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36356.pdf)也只是谷歌浩瀚冰山下的一隅，却引领了分布式系统链路追踪的发展。

![Dapper链路追踪](http://ww1.sinaimg.cn/large/bda5cd74ly1gh049xp5d1j20cl0auwf1.jpg)

[Dapper中描述了谷歌内部系统的链路追踪技术](https://blog.hufeifei.cn/2020/07/Alibaba/distributed-tracing/)。论文在2010年一经发出，Twitter根据论文研发了[Zipkin](https://github.com/openzipkin/zipkin)，Uber根据论文研发了[Jaeper](https://github.com/jaegertracing/jaeger)。当然这只是最早开源出来的非常有名气的两个项目，还有像阿里的EagleEye等众多为开源出来的内部项目都是以谷歌的Dapper为原型设计的。

![链路追踪技术](https://p.pstatp.com/origin/pgc-image/d0f15af19cfe475f897c97a705aa4a2f)

##OpenTracing标准，号令天下; OpenCensus不出，谁与争锋

但是随着各大厂商对链路追踪系统的实现趋于碎片化，社区开始制定OpenTracing标准来统一链路追踪中涉及的相关概念和规范——这是一套平台无关、厂商无关的Trace协议，使得开发人员能够方便的添加或更换分布式追踪系统的实现。

在2016年11月的时候CNCF技术委员会投票接受OpenTracing作为Hosted项目，这是CNCF的第三个项目，第一个是Kubernetes，第二个是Prometheus，可见CNCF对OpenTracing背后可观察性的重视。大名鼎鼎的Zipkin、Jaeger也都开始遵循OpenTracing协议。

既然有了OpenTracing，OpenCensus又来凑什么热闹？

对不起，你要知道OpenCensus的发起者可是谷歌，也就是最早提出Tracing概念的公司（教练下场指导工作了:smirk:），而OpenCensus也就是Google Dapper的社区版。

OpenCensus和OpenTracing最大的不同在于除了Tracing外，它还把Metrics也包括进来，这样也可以在OpenCensus上做基础的指标监控；还有一点不同是OpenCensus并不是单纯的规范制定，他还把包括数据采集的Agent、Collector一股脑都实现了。OpenCensus也有众多的追随者，最大的新闻就是微软也宣布加入，OpenCensus可谓是如虎添翼。

OpenTracing这边有Elastic、Uber、Twitter、DataDog等互联网新秀作为拥趸已发展多年，而OpenCensus有谷歌、微软两大巨头撑腰。一时间也难分高低。

##横扫六合，一统天下

既然没办法分个高低，谁都有优劣势，咱们就别干了，统一吧。于是OpenTelemetry横空出世。

那么问题来了：统一可以，起一个新的项目从头搞吗？那之前追随我的弟兄们怎么办？不能丢了我的兄弟们啊。
放心，这种事情肯定不会发生的。要知道OpenTelemetry的发起者都是OpenTracing和OpenCensus的人，所以项目的第一宗旨就是：兼容OpenTracing和OpenCensus。对于使用OpenTracing或OpenCensus的应用不需要重新改动就可以接入OpenTelemetry。

OpenTelemetry可谓是一出生就带着无比炫目的光环：OpenTracing支持、OpenCensus支持、直接进入CNCF sanbox项目。但OpenTelemetry也不是为了解决可观察性上的所有问题，他的核心工作主要集中在3个部分：

1. 规范的制定，包括概念、协议、API，除了自身的协议外，还需要把这些规范和W3C、GRPC这些协议达成一致；
2. 相关SDK、Tool的实现和集成，包括各类语言的SDK、代码自动注入、其他三方库（Log4j、LogBack等）的集成；
3. 采集系统的实现，目前还是采用OpenCensus的采集架构，包括Agent和Collector。

可以看到OpenTelemetry只是做了数据规范、SDK、采集的事情，对于Backend、Visual、Alert等并不涉及，官方目前推荐的是用Prometheus去做Metrics的Backend、用Jaeger去做Tracing的Backend。

![img](https://pic3.zhimg.com/80/v2-f40112487219807fc5d66ad6cdce5d56_1440w.jpg)



看了上面的图大家可能会有疑问：Metrics、Tracing都有了，那Logging为什么也不加到里面呢？
其实Logging之所以没有进去，主要有两个原因：

1. 工作组目前主要的工作是在把OpenTracing和OpenCensus的概念尽早统一并开发相应的SDK，Logging是P2的优先级。
2. 他们还没有想好Logging该怎么集成到规范中，因为这里还需要和CNCF里面的Fluentd一起去做，大家都还没有想好。

## 终极目标

OpenTelemetry的终态就是实现Metrics、Tracing、Logging的融合，作为CNCF可观察性的终极解决方案。

Tracing：提供了一个请求从接收到处理完毕整个生命周期的跟踪路径，通常请求都是在分布式的系统中处理，所以也叫做分布式链路追踪。

Metrics：提供量化的系统内/外部各个维度的指标，一般包括Counter、Gauge、Histogram等。

Logging：提供系统/进程最精细化的信息，例如某个关键变量、事件、访问记录等。

这三者在可观测性上缺一不可：基于Metrics的告警发现异常，通过Tracing定位问题（可疑）模块，根据模块具体的日志详情定位到错误根源，最后再基于这次问题调查经验调整Metrics（增加或者调整报警阈值等）以便下次可以更早发现/预防此类问题。

## Metrics、Tracing、Logging融合的关键

实现Metrics、Tracing、Logging融合的关键是能够拿到这三者之间的关联关系.其中我们可以根据最基础的信息来聚焦，例如：时间、Hostname(IP)、APPName。这些最基础的信息只能定位到一个具体的时间和模块，但很难继续Digin，于是我们就把TraceID把打印到Log中，这样可以做到Tracing和Logging的关联。但这还是解决不了很多问题：

1. 如何把Metrics和其他两者关联起来
2. 如何提供更多维度的关联，例如请求的方法名、URL、用户类型、设备类型、地理位置等
3. 关联关系如何一致，且能够在分布式系统下传播

在OpenTelemetry中试图使用Context为Metrics、Logging、Tracing提供统一的上下文，三者均可以访问到这些信息，由OpenTelemetry本身负责提供Context的存储和传播：

- Context数据在Task/Request的执行周期中都可以被访问到
- 提供统一的存储层，用于保存Context信息，并保证在各种语言和处理模型下都可以工作（例如单线程模型、线程池模型、CallBack模型、Go Routine模型等）
- 多种维度的关联基于Tag（或者叫meta）信息实现，Tag内容由业务确定，例如：通过TrafficType来区别是生产流量还是压测流量、通过DeviceType来分析各个设备类型的数据...
- 提供分布式的Context传播方式，例如通过W3C的traceparent/tracestate头、GRPC协议等

## 当前状态以及后续路线

目前OpenTelemetry还处于策划和原型阶段，很多细节的点还在讨论当中，目前官方给的时间节奏是：

- 2019年9月，发布主要语言版本的SDK（Pre Release版）
- 2019年11月，OpenTracing和OpenCensus正式sunsetted（ReadOnly）
- 未来两年内，保证可以兼容OpenTracing和OpenCensus的SDK

## 总结

从Prometheus、OpenTracing、Fluentd到OpenTelemetry、Thanos这些项目的陆续进入就可以看出CNCF对于Cloud Native下可观察性的重视，而OpenTelemetry的出现标志着Metrics、Tracing、Logging有望全部统一。

但OpenTelemetry并不是为了解决客观性上的所有问题，后续还有很多工作需要进行，例如：

- 提供统一的后端存储，目前三类数据都是存储在不同系统中
- 提供计算、分析的方法和最佳实践，例如动态拓扑分析
- 统一的可视化方案
- AIOps相关能力，例如Anomaly Detection、Root Cause Analysis等



更多参考文章：

[A brief history of OpenTelemetry (So Far) | Cloud Native Computing Foundation (cncf.io)](https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/)

[OpenTracing joins the Cloud Native Computing Foundation | Cloud Native Computing Foundation (cncf.io)](https://www.cncf.io/blog/2016/10/11/opentracing-joins-the-cloud-native-computing-foundation/)

[open-telemetry/opentelemetry-specification: Specifications for OpenTelemetry (github.com)](https://github.com/open-telemetry/opentelemetry-specification)

