---
title: 数据处理大观园——一个从无到有的大数据平台
date: 2020-08-16 18:50
categories: JAVA
tags: 
- BigData
- Alibaba
- JAVA
---

新环境的这几个月，自己接触了各种各样的新技术，极大的开阔了我的视野。其中最让我感到惊叹的就是听闻已久可是从未领会其妙用——大数据。

# 1、为什么需要大数据

互联网企业里需要采集用户的各种信息，对这些数据进行分析从而进行运营决策，甚至通过构建算法模型实现自动化决策。这类信息有很多，比如平时喜欢几点钟打开App、在App里做了什么操作、在某个页面停留了多少时间，此时此刻正看这篇文章时你的浏览数据已经悄悄地记录下来了。

近两年大火的各种短视频平台也就是通过不断地分析用户的行为数据，在你每天闲暇的时候精准推送一个你最爱看的视频内容，点开消息后欲罢不能，抖音就是这样侵占了你的闲暇时光。

再比如我们团队的用户增长业务。传统行业里通过电视等媒体进行广告投放，广告投放出去后，实际有多少人看到了这条广告，又有多少人因为这条广告下单，无从所知，而[央视一套黄金时段每30秒的广告可以卖到18W](http://www.cctv.com/profile/adver/guang5.html)，这样的价位让很多创业公司望而却步，也就有能力左右国家标准的乳制品企业才能通过这种广告赚得盆满钵满吧！而互联网广告有所不同，互联网中的流量可以通过PV, UV等指标进行计算的。比如百度的竞价排名，淘宝首页的店铺广告和搜索排名都是通过对页面曝光、链接点击，采集埋点数据后进行统计的。

这些数据有四大特征：

* 体量庞大。从传统的GB上升至了TB、PB、EB级的数据量；
* 来源丰富。可以是传统关系型数据库里的结构化数据，也可以是日志、视频、图片等各种半结构化或非结构化数据。
* 商业价值。采集用户的数据可以做各种分析，挖掘出蕴含的商业价值。在流量为王的时代里，[BAT等各大互联网企业很大一部分收入都来自于广告](https://www.36kr.com/p/724648414381952)。
* 处理时效性高。海量的数据不仅仅局限于离线分析，天猫双十一实时大屏上的几千亿的交易额都精确到秒。

# 2、Hadoop

2003年Google作为工业界的启蒙者发表了著名的三篇论文，不久后社区有了自己的开源实现——Hadoop。如今Hadoop已成为大数据的基石，提供了高可靠的大数据的存储与计算能力。随着2012年Hadoop技术框架的成熟和稳定，一线互联网公司纷纷使用Hadoop技术栈来构建企业大数据分析平台，随后两年基于大数据的应用如雨后春笋一样涌现，比如千人千面的推荐系统、精准定向程序化交易的广告系统、大数据风控系统。

**怎么存储**

很明显这么庞大的数据量，几台机器是无法存储的。谷歌的一篇论文[GFS](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/gfs-sosp2003.pdf)介绍了谷歌内部的分布式文件系统，Hadoop中的HDFS便是其开源实现。NameNode上负责存储所有的文件元信息，DataNode提供数据分片存储与备份功能。

**怎么计算**

另一篇论文[MapReduce](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/16cb30b4b92fd4989b8619a61752a2387c6dd474.pdf)介绍了大集群下的一种简单的数据处理方式。

**怎么分配计算资源**

Hadoop1.0时期使用JobTracker和TaskTracker的架构来实现MapReduce任务的调度，由于MasterNode上的JobTracker承担了接受任务、分配资源、与dataNode通信等任务，存在单点故障。所以Hadoop2.0将资源管理单独拎出来形成了Yarn。

> Hadoop1.0的JobTracker就像事必躬亲的诸葛孔明，Hadoop2.0的Yarn就像唐朝的三省六部制。
>
> Refs: http://blog.sina.com.cn/s/blog_829a682d0101lc9d.html

![image.png](http://tva1.sinaimg.cn/large/bda5cd74ly1ghstnrsrk1j216k0sgqr0.jpg)

![Yarn架构](http://tva1.sinaimg.cn/large/bda5cd74ly1gi6slp7p14j216o0o0mza.jpg)

> 分布式文件系统完美地解决了海量数据存储的问题，但是一个优秀的数据存储系统需要同时考虑数据存储和访问两方面的问题，比如你希望能够对数据进行随机访问，这是传统的关系型数据库所擅长的，但却不是分布式文件系统所擅长的，那么有没有一种存储方案能够同时兼具分布式文件系统和关系型数据库的优点，基于这种需求，就产生了 HBase。
>
> 谷歌的第三篇论文[BigTable](http://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)介绍了基于GFS实现的一个结构化数据存储系统。对应的开源实现就是基于Hadoop的[HBase](https://hbase.apache.org/)。数据存储结构上遵循[LSM-Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf)算法，由于Hadoop以128M大块为存储单位的，新数据都会先写到内存的memtable中并进行排序，当达到一定阈值后再flush到磁盘的SSTable中，SSTable数量会随着数据的增加而增加，LSM-Tree会有一个Compact机制对磁盘中的多个SSTable进行合并。值得一提的是LSM-Tree算法已被应用在了LevelDB、ElasticSearch、MongoDB等各种数据存储上。

# 3、Hive

MapReduce接口虽然对分布式计算进行了抽象，但是编程接口仍然不好用。

用MapReduce写一个WordCount可能需要上百行代码，如果用SQL的话：

```sql
SELECT word,COUNT(1) FROM wordcount GROUP BY word;
```

使用SQL处理分析Hadoop上的数据，方便、高效、易上手。

Hive则是SQL On Hadoop，Hive提供了SQL接口，开发人员只需要编写简单易上手的SQL语句，Hive负责把SQL翻译成MapReduce，提交运行。

> 在Hive之后还有[Pig](https://pig.apache.org/)也可以生成MapReduce任务，只是后面的发展都偏向于通用的SQL，Pig也就逐渐没落。
>
> Hive可以将SQL转成MapReduce任务，而[phoenix](https://phoenix.apache.org/)可以将SQL转成HBase scan。

目前认知中的大数据平台应该是这样的：

![Hive](http://tva1.sinaimg.cn/large/bda5cd74ly1ghszuc5a6tj211y096776.jpg)

那么问题来了，海量数据如何存到HDFS上的呢？

# 4、数据采集

HDFS提供了API可以对HDFS进行读写操作，但是一般我们很少编写这样的程序。

数据按不同来源有不同的工具进行处理。

## 4.1、Sqoop

[Sqoop](https://sqoop.apache.org/docs/1.4.2/SqoopUserGuide.html#_introduction)的功能就像它的名字一样（Sql+Hadoop）作为传统关系型数据库与Hadoop之间的管道。

就像Hive把SQL翻译成MapReduce一样，Sqoop把你指定的参数翻译成MapReduce，提交到Hadoop运行，完成Hadoop与其他数据库之间的数据交换。

Sqoop将Sql表同步到Hive表，涉及到存储格式与数据类型的兼容问题，增量同步还是全量同步的问题。

> Refs: Sqoop 使用指南(https://zhuanlan.zhihu.com/p/39113492 )

## 4.2、Flume

[Flume](https://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html)是一个处理分布式日志采集和传输的框架。Flume的使用不仅限于日志数据聚合，由于数据源是可定制的，因此Flume可用于传输大量事件数据，包括但不限于网络流量数据，社交媒体生成的数据，电子邮件消息以及几乎所有可能的数据源。

通常我们使用Flume监控一个不断追加数据的日志文件，并将数据传输到HDFS。

## 4.3、DataX

和Sqoop类似，[DataX](https://github.com/alibaba/datax)是阿里开源的一款离线数据同步工具，DataX有更[丰富的数据源支持](https://github.com/alibaba/DataX#support-data-channels)。

![Data Collector](http://tva1.sinaimg.cn/large/bda5cd74ly1ghszvbp25cj21fy0iadpk.jpg)

# 5、数据导出

大数据平台处理完数据后需要将数据展示出来，或者应用到各个应用系统中。

和数据采集一样，可以用HDFS API读取数据，但通常都是使用Sqoop或DataX将Hive的表导出到关系型数据库或HBase等数据库中以便业务系统查询。

至此，大数据平台已初具雏形。

![BigData](http://tva1.sinaimg.cn/large/bda5cd74ly1ghszwh3y59j21gk0ritmt.jpg)



# 6、算的更快一点

MapReduce编程模型非常通用，但是运算太慢——因为中间的计算结果需要落盘。

![Hive](http://tva1.sinaimg.cn/large/bda5cd74ly1ght0ysiqa9j20ad0ajdh0.jpg)

因此诞生了很多基于内存或半内存的计算引擎，大大加快了计算速度。

![Tez](http://tva1.sinaimg.cn/large/bda5cd74ly1ght102dtzbj20b50a6mxw.jpg)

其中就包括Tez，Spark。Spark号称比传统Hadoop的MapReduce快百倍以上。基于Spark的SQL项目[Shark](https://databricks.com/blog/2014/07/01/shark-spark-sql-hive-on-spark-and-the-future-of-sql-on-spark.html)也演变成了Spark下的子项目Spark SQL，为了兼容已有的Hive仓库，[SparkSQL也支持HiveQL](https://spark.apache.org/sql/)。Hive也吸收了Tez和Spark的思想，逐渐发展起[Hive On Spark](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)、[Hive On Tez](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Tez)等项目。与此同时，各种OLAP的数据仓库引擎百花齐放——[Impala](https://impala.apache.org/)、[Presto](https://prestodb.io/)、[Drill](https://drill.apache.org/)、[Hawq](http://hawq.apache.org/)、[Druid](https://druid.apache.org/)、[Pinot](https://pinot.apache.org/)、[Kylin](http://kylin.apache.org/)......

![BigData](http://tva1.sinaimg.cn/large/bda5cd74ly1ght14ok1twj21gg0rgas4.jpg)

# 7、日志数据的订阅

在实际业务场景下，特别是对于一些监控日志，想实时地从日志中了解一些指标，这时候，从HDFS上分析就太慢了，尽管是通过Flume采集的，但Flume也不能间隔很短就往HDFS上滚动文件，这样会导致小文件特别多(Hadoop块大小为128M，不适合存小文件)。所以一般日志都会先采集到消息队列中(如[Kafka](http://kafka.apache.org/))。

Flume只能采集应用日志，对于数据库的变更(增删改)操作也可能需要做一些处理，比如将数据库的变更同步到ElasticSearch搜索引擎中。阿里开源了一款MySQL的binlog订阅组件[Canal](https://github.com/alibaba/canal)。

![日志订阅](http://tva1.sinaimg.cn/large/bda5cd74ly1ght37lwu4oj217o0k4wsn.jpg)

# 8、Hadoop上的任务的调度

数据的采集、产出和交换等需要定时调度，数据平台上的任务一多，表与表之间的依赖关系也变的非常复杂。比如：表A的产出需要依赖表B，那么JobA就应该等待JobB执行。这个时候需要调度监控系统来完成这件事。调度监控系统是整个数据平台的中枢系统，负责分配调度、运维监控。

有两个现成的工作流引擎可以完成这件事儿：[oozie](https://oozie.apache.org/)、[Azkaban](https://azkaban.github.io/)

![实时监控](http://tva1.sinaimg.cn/large/bda5cd74ly1ghwdrpo58uj211q0ipgvp.jpg)

# 9、数据要实时--批处理vs.流处理

> 如果把数据比喻成水的话，批处理框架就像水桶，想喝水得一桶桶地从水井里打水喝，流处理就像自来水管道——不知出自哪位大牛的话

Hadoop中的Job属于“离线批处理任务”，根据数据量的不同它的执行时间往往需要几分钟甚至几个小时。通常我们按照一定的时间周期定时执行一次Job。比如每天凌晨定时处理前一天的订单汇总数据聚合生成报表，或者每个小时定时处理前一个小时内的商品降价数据生成消息给用户发送降价提醒。

但是很多时候，即使是海量数据，我们也希望及时查看一些数据指标。比如每年双十一的天猫交易大屏需要实时统计交易额，类似的还有股市实时交易数据，高德地图实时路况分析，未来自动驾驶技术和城市大脑都有非常高的实时性要求。

![双十一实时交易额](http://tva1.sinaimg.cn/large/bda5cd74ly1ghznlosp3hj20hs09yaap.jpg)

为了让数据更实时，很明显就不能再使用原来定时调度任务的方式了。所以市面上就出现了[Storm](https://storm.apache.org/index.html)、[Samza](http://samza.apache.org/)实时流计算框架和[Spark Streaming](https://spark.apache.org/streaming/)这样基于微批处理的准实时框架，正是Spark Streaming让Spark成为了流批一体的框架淘汰了原有的MapReduce。15年社区开源了新的流处理框架[Flink](https://flink.apache.org/)，并通过有界流的策略来实现批处理，18年年底Flink母公司被阿里收购，Flink便在中国开始了大规模的技术推广，国内已有[很多大厂在用Flink替代Storm和Spark了](https://zhuanlan.zhihu.com/p/71318235)。

阿里早期基于社区Storm自己实现了一套[JStorm](https://github.com/alibaba/jstorm)，后面又基于Flink定制了适用于企业内部的Blink，并作为16年后的双十一的技术支持，18年年底干脆就直接收购了Flink的母公司Data Artisans，次年便开源了Blink。

![实时计算](http://tva1.sinaimg.cn/large/bda5cd74ly1ghzte72nvdj21ks0u21kx.jpg)

# 10、数据查询与可视化

前面提到离线数据可以通过Sqoop或DataX导出到RDB或HBase中，这个一般也是通过设置定时任务将Hive表中的数据进行导出。

实时数据的导出要求延时非常低，我们可以根据数据生成的格式以及查询要求，选用其他更适合的存储方案：ElasticSearch、HBase、Redis、Kylin、Druid...

基于这些存储方案，我们可以通过各种手段将其透出，比如：提供HTTP数据接口给第三方查询；使用DataV、AntV、HighChart等各种前端可视化框架作出可视化报表（自行谷歌数据可视化:smirk_cat:）。

![BigData](http://tva1.sinaimg.cn/large/bda5cd74ly1ghzwqb0lsdj21l60xsb29.jpg)

# 11、高大上的大数据算法与机器学习

抖音的视频、知乎的文章越看越停不下来的秘密，是因为有推荐系统。推荐系统对浏览者平时浏览的信息进行统计生成用户画像，对发布者的视频和文本内容提取特征，依此进行推荐，所以你越喜欢看某类文章视频你的某个特征就越明显。

> 抖音作为娱乐性爆款产品，就像吸烟玩游戏一样容易上瘾，烟里的尼古丁让大脑分泌多巴胺产生短暂的愉悦感，为了找回消失的快感，大多数人会在再上一支，任何让人上瘾的产品原理都是一样的：吸烟->产生愉悦->愉悦感消失->继续吸烟，最后成瘾性依赖。
>
> 技术可以改变世界，移动支付、车辆导航、自动驾驶让人的生活变得越来越便捷，但如果技术被滥用，将人性不好的一面挖掘出来，也许能摧毁世界。

大数据算法和机器学习对微积分、线性代数、数理统计要求特别高，我大学里大部分时间都花在应用开发，这几门课学完就丢了，所以这块也没有深入研究。值得一提的是Spark有提供相应平台的机器学习框架[Spark ML](https://spark.apache.org/mllib/)，还有谷歌鼎鼎大名的[Tensorflow](https://www.tensorflow.org/)，打败世界第一围棋冠军柯洁的AlphaGo就是基于Tensorflow。

![BigData](http://tva1.sinaimg.cn/large/bda5cd74ly1gi7pfbc3iqj21ku0xob29.jpg)



**Refs：**

如何用形象的比喻描述大数据的技术生态？Hadoop、Hive、Spark 之间是什么关系？：https://www.zhihu.com/question/27974418/answer/38965760

一文读懂大数据平台——写给大数据开发初学者的话!：https://zhuanlan.zhihu.com/p/26545566

大数据最核心的价值：https://www.zhihu.com/question/23273263/answer/88182843

数据人必须了解的大数据中台技术架构: https://zhuanlan.zhihu.com/p/78705304

Apache Flink基础：https://www.cnblogs.com/maoxiangyi/p/11586125.html

《十年牧码记》第8篇：双11奇迹背后的大数据力量，首次探密十年发展的五部曲：https://www.atatech.org/articles/123418

阿里大数据计算技术领域大图详解| 阿里技术20年：https://www.atatech.org/articles/150041

他山之石(一)：Google论文、阿里、开源与云计算：https://www.atatech.org/articles/118481

Odps相比于Hadoop开源组件的优化：https://www.atatech.org/articles/114717

Google BigTable论文：http://blog.bizcloudsoft.com/wp-content/uploads/Google-Bigtable中文版_1.0.pdf

大数据实时和离线业务图分析: https://zhuanlan.zhihu.com/p/94429901

微量批处理: https://hazelcast.com/glossary/micro-batch-processing/

机器学习必知的15大框架：https://zhuanlan.zhihu.com/p/31714401

抖音的推荐算法是怎样的：https://www.zhihu.com/question/270224768/answer/1300640117

大数据入门指南：https://github.com/heibaiying/BigData-Notes

Awesome BigData：https://github.com/onurakpolat/awesome-bigdata

Awesome MachineLearning：https://github.com/josephmisiti/awesome-machine-learning
