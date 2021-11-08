---
title: SpringCloudConfig配置项刷新存在的问题
date: 2021-09-18
categories: JAVA
tags: 
- JAVA
- SpringCloud
- SpringCloudConfig
keywords:
- SpringCloudConfig配置项刷新
---

## pring-cloud-bus项目结构

![](./spring-cloud-monitor.svg)

spring-cloud-bus是用来实现服务间异步通信的服务总线，有基于kafka和rabbitmq的两个实现。

kafka和rabbitmq的消息处理逻辑本身也被抽象成了spring-cloud-stream，所以就有了上图中的依赖结构

spring-cloud-config-monitor是一个通过spring-cloud-bus实现配置实时更新的依赖库。

## pring-cloud-config-monitor架构

![](./config-server-refresh-single.svg)

monitor一般是作为config-server的一个依赖库放在config-server上，这个库里提供了一个PropertyPathEndpoint的Controller接口。流程大致如下：

1、使用者在gitlab上修改并提交配置

2、gitlab在收到新的push后，把调用预先配置的webhook接口。这个webhook接口一般就是monitor提供的http接口。通过这个接口把git的push事件发送给monitor。

3、monitor收到事件后，解析事件中修改的文件，广播一个RefreshRemoteApplicationEvent事件到eventbus。这个事件有三个字段。

```java
public abstract class RemoteApplicationEvent extends ApplicationEvent {
	private static final Object TRANSIENT_SOURCE = new Object();
    // 事件由哪个服务发送的
	private final String originService;
    // 事件发送给哪个服务，支持"**"的通配符，表示所有服务都接受这个事件
	private final String destinationService;
    // 事件ID
	private final String id;
    ...
}
```

4、所有依赖了eventbus的服务会收到这个Refresh事件，根据事件中的`destinationService`字段判断，这个事件是否需要处理，如果匹配成功，会由`RefreshListener`接受并处理这个事件。

5、`RefreshListener`中会调用`ContextRefresher`，拉取最新的配置，并更新Environment，重建所有加了@RefreshScope注解的Bean。

## pringBus的几个配置

```yaml
spring:
  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true
      env:
        enabled: true
      ack:
        enabled: true
      trace:
        enabled: false
```

1、`refresh`代表是否要接受远程EventBus发送过来的`RefreshRemoteApplicationEvent`事件。事件由`RefreshListener`处理。处理过程中，会从config-server拉取最新配置，并发送EnvironmentChangeEvent通知ConfigurationPropertiesRebinder刷新配置Bean

2、`env`是用来刷新部分配置的，事件里面要提供刷新的`name`和`value`值。

3、`ack`是收到EventBus远程发来的事件后，是否返回确认事件。

4、`trace`是用来记录这些事件的，目前SpringCloud没有实现，只是打了debug日志。但是SpringCloud提供了HttpTraceRepository接口，后期可能会扩展。

## 在的问题

![](./config-server-refresh-config.svg)

1、如果`config-server`存在多个实例，webhook没办法广播到多个`config-server`实例，`config-server`本地备份的git仓库如何更新。如果有实例没有更新，就会导致第5步，拉取不到最新的配置。

2、gitlab中通常只配置一个webhook，测试、预发、生产环境的`config-server`怎么能都收到这个配置进行仓库的更新。

## 解决方案一

1、首先mq必须全局共享，开发、测试、预发、生产都用同一套mq

2、gitlab中配置的webhook是生产环境的config-server，收到push事件后，根据修改的文件提取出来环境和应用，通知对应环境的`config-server`更新git仓库

3、对应环境的`config-server`更新git仓库完成后，需要返回响应，响应中需要带上该环境下其他的`config-server`实例的信息(这个信息可以从eureka中获取,eureka是环境隔离的)。

4、需要等其他的`config-server`都更新完成后，再推送Refresh事件到其他需要更新配置的服务，这个时候能保证所有的服务拉到的配置都是最新的

![](./config-server-refresh-multi-profile.svg)

## 解决方案二

1、gitlab中配置多个环境的webhook，这要求各环境的`config-server`域名区分开

2、webhook调用接口后，`config-server`检查更新的文件是否涉及当前环境，如果没有不需要做任何处理，相反如果涉及当前环境执行第3步

3、发送事件给当前环境的所有`config-server`实例，更新完成后返回确认

4、收到所有的确认后再由`config-server`发送Refresh事件到需要更新配置的服务

![](./config-server-refresh-isolation.svg)

## 解决方案三

SpringCloudBus的问题：

1、通过rabbitmq广播到每个环境，最后等所有环境确认，这段延时比较严重

2、通过rabbitmq推送刷新事件到应用，这段延时也比较严重

3、SpringCloudContext刷新时会重建Context，不可用的情况不能忍

抛弃SpringCloudBus，使用与Nacos、Consul一样的长轮询

1、首先mq必须全局共享，开发、测试、预发、生产都用同一套mq

2、gitlab中配置的webhook是生产环境的config-server，收到push事件后，根据修改的文件提取出来环境和应用，通知对应环境的`config-server`更新git仓库

3、`config-server`一旦收到更新仓库的通知后，更新完仓库，即刻通知自身hold住的长连接返回结果

![](./config-server-refresh-long-polling.svg)
