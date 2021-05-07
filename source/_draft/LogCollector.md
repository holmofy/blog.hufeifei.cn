Log Framework: slf4j,logback
Log collection : 
fluentd(fluent-bit)
logstash(beats)
facebook/scribe
[vector](https://github.com/timberio/vector)
[apache/flume](https://github.com/apache/flume)
[The Syslog Protocol](https://tools.ietf.org/html/rfc5424)
[syslog-ng](https://github.com/syslog-ng/syslog-ng)
[rsyslog](https://github.com/rsyslog/rsyslog)

Centralized log aggregation
Long-term log storage and retention: Kafka,
Log rotation
Log analysis (in real-time and in bulk after storage)
Log search and reporting.
elasticsearch(kibana), clickhouse(loghouse)
grafana

metrics:
从单机JMX监控到集群监控
Collectd,prometheus
[statsd](https://github.com/statsd/statsd)


函数调用栈
open tracing标准
https://opentracing.io/
https://opentracing.io/docs/overview/
https://opencensus.io/
https://opentelemetry.io/

https://openmetrics.io/

详解日志采集工具--Logstash、Filebeat、Fluentd、Logagent对比: https://zhuanlan.zhihu.com/p/63725444
阿里、facebook、Cloudera等巨头的数据收集框架：https://dbaplus.cn/news-73-657-1.html
日志采集中的关键技术分析: https://cloud.tencent.com/developer/article/1517898
https://en.wikipedia.org/wiki/Log_management
syslog协议介绍: https://developer.aliyun.com/article/17130
日志服务（原SLS）新功能发布(6)--使用logtail接入syslog数据: https://developer.aliyun.com/article/17129
知乎为什么要自己开发日志聚合系统「kids」而不用更简洁方便的「ELK」？: https://www.zhihu.com/question/27214433/answer/35813798

