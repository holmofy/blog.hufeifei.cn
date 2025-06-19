---
title: K8S中探针请求与OTEL链路采集的问题
date: 2025-06-11
categories: 分布式
tags: 
- Distributed
- OpenTelemetry
- OTEL
keywords:
- 分布式
- 链路追踪
- Skywalking
---

最近给所有的[java应用加了就绪探针和存活探针](https://docs.spring.io/spring-boot/docs/3.1.0/reference/htmlsingle/#features.spring-application.application-availability)，并且通过`kubectl rollout status deploy`命令让Gitlab流水线能检测应用是否已经就绪。

```sh
kubectl rollout status deploy $DOCKER_APP_NAME --context=dev-admin@cluster.dev -n recircle-industry-platform-dev
```

![image](https://github.com/user-attachments/assets/074d682a-df64-4804-9b41-6a9a67e29e36)

但是又遇到一个新的问题，[OpenTelemetry的java-agent](https://opentelemetry.io/docs/zero-code/java/agent/getting-started/)默认也会把探针的请求上报到后端的监控服务。

![image](https://github.com/user-attachments/assets/fbff209b-e8b6-424d-9805-d3a0f6ca6cc7)

由于探针请求的频率有点高，而且每个应用都会有探针请求，结果导致OpenObserver处理不过来：

```sh
 2025-06-18T07:40:22.074188063+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:22.074317749+00:00 INFO actix_web::middleware::logger: 10.233.71.119 "POST /api/default/v1/traces HTTP/1.1" 503 74 "1706" "-" "OTel-OTLP-Exporter-Java/1.40.0" 0.000299    
 2025-06-18T07:40:22.449896048+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:22.450004619+00:00 INFO actix_web::middleware::logger: 10.233.80.211 "POST /api/default/v1/traces HTTP/1.1" 503 74 "2230" "-" "OTel-OTLP-Exporter-Java/1.40.0" 0.000270    
 2025-06-18T07:40:22.517553812+00:00 INFO actix_web::middleware::logger: 10.233.71.0 "POST /api/default/v1/metrics HTTP/1.0" 503 74 "3826" "-" "OpenTelemetry Collector Contrib/0.111.0 (linux/amd64)" 0.000867    
 2025-06-18T07:40:23.572270669+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:23.572416843+00:00 INFO actix_web::middleware::logger: 10.233.71.1 "POST /api/default/v1/traces HTTP/1.1" 503 74 "2278" "-" "OTel-OTLP-Exporter-Java/1.40.0" 0.000371    
 2025-06-18T07:40:23.657375098+00:00 INFO actix_web::middleware::logger: 10.233.71.247 "POST /api/default/v1/metrics HTTP/1.1" 503 74 "20634" "-" "OTel-OTLP-Exporter-Java/1.40.0" 0.000578    
 2025-06-18T07:40:23.803923763+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:24.635078516+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:24.635189325+00:00 INFO actix_web::middleware::logger: 10.233.71.223 "POST /api/default/v1/traces HTTP/1.1" 503 74 "2162" "-" "OTel-OTLP-Exporter-Java/1.40.0" 0.000268    
 2025-06-18T07:40:25.119413710+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError    
 2025-06-18T07:40:25.120251549+00:00 ERROR openobserve::service::traces: [TRACES:OTLP] ingestion error while checking memtable size: MemoryTableOverflowError
```

目前找到集中解决方法。

## java-agent上报时过滤掉actuator的请求

[opentelemetry-java-instrumentation#1060](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/1060)和[discussions#6605](https://github.com/open-telemetry/opentelemetry-java-instrumentation/discussions/6605)中提到了这个问题，但是agent并不打算实现Exclude URL的功能。

## 通过opentelemetry-spring-boot-starter过滤actuator请求

opentelemetry有一个三方提供的[samplers](https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/samplers)包，可以做到过滤[actuator](https://github.com/open-telemetry/opentelemetry-java-examples/blob/main/javaagent/sdk-config.yaml#L101)请求。

还有一种方式是通过[opentelemetry-spring-boot-starter](https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/sdk-configuration/#exclude-actuator-endpoints-from-tracing)过滤掉actuator请求。

但是这种方式需要侵入代码，比较好的方式是把这个功能和spring-boot-actuator一起封装成二方包。但是需要对业务开发人员灌输这个二方包的作用，仍然不够优雅。

## 通过opentelemetry-collector丢弃掉actuator请求

第三种方式就是在java应用和openobserver之间添加一个`opentelemetry-collector`的组件，这个好处是让`opentelemetry-collector`统一收集所有java应用上报的OTEL消息，然后批量发给openobserver可以缓解openobserver的压力（openobserver底层使用LSM数据结构，有后台merge的过程），其次就是有非常多的processor可以对OTEL消息进行处理。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  filter/skip_actuator:
    error_mode: ignore
    traces:
      span:
        - attributes["http.route"] != nil and IsMatch(attributes["http.route"], ".*actuator.*")
        - attributes["http.target"] != nil and IsMatch(attributes["http.target"], ".*/healthz")
        - IsMatch(attributes["url.path"], ".*actuator.*")

exporters:
  otlphttp/openobserve:
    endpoint: "http://openobserve.recircle-industry-platform-dev:5080/api/default"
    headers:
      Authorization: Basic cm9vdEBleGFtcGxlLmNvbTpjNEVxU1Jhb0F5SHNWTDVn
      stream-name: default

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [filter/skip_actuator, memory_limiter, batch]
      exporters: [otlphttp/openobserve]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/openobserve]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/openobserve]
```

这里推荐一个网站[`otelbin.io`](https://www.otelbin.io/)可以可视化opentelemetry-collector的配置。
