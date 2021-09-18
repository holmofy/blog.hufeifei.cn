# 从单体应用到分布式应用的观测

![分布式系统观测性的三大基石](https://engineering.zenduty.com/assets/images/tracinglogging.png)

## Logging

大多数语言、应用程序框架或库都支持日志，**日志记录本质上是一个事件**，表现形式可以是字符串这样的非结构化数据，也可以是JSON等半结构化数据。可以通过日志来分析应用的执行状况，报错信息，性能分析...... 正因为日志极其灵活，生成非常容易，没有一个统一的结构，所以它的体量也是最大的。

对于单体应用，查看日志我们可以直接登上服务器。但是分布式应用，面对部署在数十数百台机器的应用，亟需一个日志收集、存储、查询的系统。

开源社区最早流行的是Elastic体系的ELK。Logstash负责收集，ElasticSearch负责索引与存储，Kibana负责查询与展示。ElasticSearch支持全文索引可以进行全文搜索，而且支持DocValue可以用于结构化数据的聚合分析。再加上[MetricBeats](https://www.elastic.co/cn/beats/)提供了监控指标的收集，[APM](https://www.elastic.co/cn/apm/)提供的链路收集，Elastic俨然已是一个集Logging、Metrics、Trace的大一统技术体系。

Elastic野心很大，但是这也导致ElasticSearch并不专注在其中的一个领域。使用全文索引受限于分词器，对于日志查询非常鸡肋(两个单词能搜索到，三个单词就搜索不到的现象也不少)。而且索引阶段特别耗时，很多用户都无法忍受ElasticSearch索引不过来时抛出的EsReject。基于JVM的Logstash极其笨重，经常因为[GC无响应](https://discuss.elastic.co/t/logstash-with-long-gc/42159)，为此Elastic专门用Go语言开发了轻量级的FileBeat日志采集工具。

目前K8S生态下以Fluentd和C语言编写的fluent-bit为主作为日志收集工具。

![](https://p.pstatp.com/origin/pgc-image/0c15041f721449ab9d993bcdf7b89c6c)

## Metrics

单体应用中应用的性能可以通过[Linux自带的各种工具](https://www.brendangregg.com/linuxperf.html)进行观测。

比如最常用的top命令可以看到每个进程的CPU、内存等使用情况，`mpstat`、`vmstat`、`iostat`可以看到系统的CPU、内存、磁盘读写等情况。

![Linux Performance](https://www.brendangregg.com/Perf/linux_observability_tools.png)

对于不同语言编写的应用也有针对应用内部的监控工具，如Java体系有`jmap`、`jconsole`、[Eclipse Memory Analyzer](http://www.eclipse.org/mat/)等工具，JDK也提供了JMX对应用指标接口进行标准化。

在分布式系统中，由于应用部署在多台机器上，应用性能的观测面临着巨大的挑战。

指标数据和日志数据的区别在于它更加结构化，而且这些metrics的结构化数据与传统的OLTP数据库中的数据，区别在于：

* 数据不可变，只有插入，没有修改
* 按时间依次生成，顺序追加
* 数据量远比传统的OLTP数据库大
* 主要以时间戳和单独的主键(serverId、deviceId、accountId......)做索引
* 数据需要聚合。需要看同一秒内应用所有机器的整体性能，就需要把这些机器的数据聚合起来

这类数据也被称为[时序数据](https://en.wikipedia.org/wiki/Time_series_database)，[时序数据库的发展](https://zhuanlan.zhihu.com/p/29367404)就是专门用来解决这类数据的存储问题。

正因为时序数据是按照时间依次生成顺序追加的，所以除了早期的[VividCortex（基于MySQL）](https://orangematter.solarwinds.com/2014/12/16/in-case-you-missed-it-building-a-time-series-database-in-mysql/)和[TimeScaleDB（基于PostgreSQL）](https://blog.timescale.com/blog/building-a-distributed-time-series-database-on-postgresql/)使用就地更新的B+树来存储，其他大多数时序数据库都是用LSM-Tree(Log-Structured Merge Tree)作为底层的索引结构（在InfluxDB中被叫做Time-Structured Merge Tree）。

![Time Series DB Rank](https://p.pstatp.com/origin/pgc-image/c86afeeb70f749c7a8526ca0be61117d)

时序数据并不局限于系统与应用的性能指标，还包括了各种业务指标，如互联网应用的流量分析、接口性能、消息发送量......物联网传感器的功率、风速、温度等各种信号......金融领域的股票交易数据......

目前prometheus+grafana的组合几乎成了分布式系统指标观测的事实标准。

![](https://p.pstatp.com/origin/pgc-image/46dc82602df94297b2f11c4beb3886a7)

## Trace

单体应用的调用只局限于内存的堆栈，可以通过**stack trace**进行调用链追踪，调用的性能分析可以通过这些堆栈打印出火炬图，[github](https://github.com/search?q=Flame+Graph)上也有大多数语言生成火炬图的工具。

![FlameGraph](https://p.pstatp.com/origin/pgc-image/2d6f3a7eb8244c75b3fb5889356ebfe7)

但是分布式应用，不再是内存内的堆栈调用了，而是穿透网络的RPC调用。用户一个请求过来，从A服务到B服务再到C服务……这个调用链可能很长。再也不能像单机版应用一样直接看程序堆栈，直接用火炬图就能分析应用的性能了。而且一个应用部署了多台机器，具体调用了集群中哪台机器也不知道了。

所以催生了**调用链收集**的工具，将分布式应用的调用链整成一个跟程序堆栈类似的东西，最好还能告诉我每个服务调用过程中的耗时，这就是**Distributed Tracing**。

[谷歌的Dapper论文](https://zhuanlan.zhihu.com/p/163806366)就介绍了谷歌是怎么实现这个功能的。然后开源社区便产生了Zipkin、Jaeger、Spring Cloud也有一个Sleuth，这种组件很多，每种实现可能有所差别。

在谷歌论文里服务之间的调用被称为Span，整个链路被叫做Trace，但是在一些实现里调用被称为Rpc或者其他的名字，为了规范链路追踪的技术，有了[OpenTracing](https://opentracing.io/)标准和[OpenCensus](https://opencensus.io/)标准。

OpenTracing和OpenCensus的出发点不一样，OpenTracing专注于链路追踪数据的采集，OpenCensus包括了Metrics和Trace两者的数据收集。随着K8s催生的云原生的发展，这两者合并成了[OpenTelemetry](https://opentelemetry.io/)。

![](https://p.pstatp.com/origin/pgc-image/d0f15af19cfe475f897c97a705aa4a2f)

https://openapm.io/landscape

# 基于Grafana的一站式分布式观测系统



# 与ElasticSearch体系的观测系统的对比



参考链接：

^ 1. 分布式系统观测性的三大基石：https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html

^ 2. 云原生生态远景图：https://landscape.cncf.io/

^ 3. 面向列的数据库：http://www.timestored.com/time-series-data/what-is-a-column-oriented-database

https://openapm.io/landscape

