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

> 天下大势，分久必合，合久必分。

###### 乾坤未定，混沌初开

![Linux性能观测工具](https://www.brendangregg.com/Perf/linux_observability_tools.png)

分布式未开之际，软件世界里网站后台还是单体模式。有问题可以看日志，查性能有`top`以及[sysstat](https://github.com/sysstat/sysstat)下的一批工具包：`mpstat`、`pidstat`、`iostat`等一系列工具。

随着互联网的突飞猛进，摩尔定律的失效，单体模式已经承接不了互联网下的滔天流量。

于是分布式呼之欲出。

###### Dapper启蒙，百花齐放

谷歌作为互联网的巨擘，手握独步天下的搜索引擎——Google。搜索引擎作为互联网的要隘，流量增长是惊人的，这也促使谷歌发展出自己的一套分布式系统。

![谷歌论文与开源技术](http://duanple.com/wp-content/uploads/2019/08/paper.jpg)

这其中每一篇论文都曾掀起过层层巨浪，[Dapper](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36356.pdf)就是其中之一。

![Dapper链路追踪](http://tva1.sinaimg.cn/large/bda5cd74ly1gh049xp5d1j20cl0auwf1.jpg)

[Dapper中描述了谷歌内部系统的链路追踪技术](https://blog.hufeifei.cn/2020/07/Alibaba/distributed-tracing/)。论文在2010年一经发出，Twitter根据论文研发了[Zipkin](https://github.com/openzipkin/zipkin)，Uber根据论文研发了[Jaeper](https://github.com/jaegertracing/jaeger)。当然这只是最早开源出来的非常有名气的两个项目，还有像阿里的EagleEye等众多为开源出来的内部项目。

![链路追踪技术](https://p.pstatp.com/origin/pgc-image/d0f15af19cfe475f897c97a705aa4a2f)

###### OpenTracing标准，号令天下; OpenCensus不出，谁与争锋



###### 横扫六合，一统天下



https://blog.hufeifei.cn/2021/09/Distribution/grafana/

https://zhuanlan.zhihu.com/p/74930691





https://zhuanlan.zhihu.com/p/74930691