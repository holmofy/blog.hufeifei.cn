---
title: 在K8S中管理多个Canal集群
date: 2025-10-10
categories: 数据库
tags: 
- DB
- Canal
---

公司涉及到业务团队与IM团队的数据同步，为了解耦避免修改我们这边的代码，同时为了性能考虑，打算用canal来处理。

![canal-server部署架构图](https://img-blog.csdnimg.cn/2542dfe00519416589877afcf42a89e7.png)

具体步骤都是参考了[canal官方的文档](https://github.com/alibaba/canal/blob/master/charts/README.md)

## 1、安装canal-admin

一个canal-admin是可以管理多个canal集群的。所以我把canal-admin安装在了`base`命名空间下。

配置好[canal-manager](https://github.com/alibaba/canal/blob/master/admin/admin-web/src/main/resources/canal_manager.sql)数据库后，安装admin到base命名空间下。

```shell
## 切换至base命名空间
kubens recircle-industry-base-system
## 安装admin
helm install canal-admin -f ./admin-values.yaml ./canal-admin
```

<img width="1431" height="814" alt="image" src="https://github.com/user-attachments/assets/7f975a29-2b7c-4867-a202-9639eb93137d" />

我们现在是每个命名空间一个canal集群，dev、test、pre分别对应了三个环境。

## 2、安装zookeeper集群

```shell
## 切换到preview命名空间
kubens recircle-industry-platform-preview
## 创建zk集群
helm install canal-zookeeper oci://registry-1.docker.io/bitnamicharts/zookeeper
```

<img width="1432" height="774" alt="image" src="https://github.com/user-attachments/assets/6e1879ba-6570-4b27-826a-c7eab39b8774" />

安装完成后，把zookeeper的k8s service地址填到canal-admin中。

<img width="1432" height="774" alt="image" src="https://github.com/user-attachments/assets/c7c08f5b-398a-4137-ab60-ae73a6200d97" />

然后可以针对这个集群添加主配置，这个配置就是集群下所有canal-server的instance默认的配置，在这里可以配置canal-admin地址、记录表结构发生变化的tsdb的地址、消息队列的地址：

```properties
#################################################
######### 		common argument		#############
#################################################
# tcp bind ip
canal.ip =
# register ip to zookeeper
canal.register.ip =
canal.port = 11111
canal.metrics.pull.port = 11112
# canal instance user/passwd
canal.user = admin
canal.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

# canal admin config
canal.admin.manager = canal-admin.recircle-industry-base-system:8089
canal.admin.port = 11110
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
# admin auto register
#canal.admin.register.auto = true
#canal.admin.register.cluster =
#canal.admin.register.name =


canal.zkServers = canal-zookeeper.recircle-industry-platform-preview:2181
# flush data to zk
canal.zookeeper.flush.period = 1000
canal.withoutNetty = false
# tcp, kafka, rocketMQ, rabbitMQ, pulsarMQ
canal.serverMode = rocketMQ
# flush meta cursor/parse position to file
canal.file.data.dir = ${canal.conf.dir}
canal.file.flush.period = 1000
## memory store RingBuffer size, should be Math.pow(2,n)
canal.instance.memory.buffer.size = 16384
## memory store RingBuffer used memory unit size , default 1kb
canal.instance.memory.buffer.memunit = 1024 
## meory store gets mode used MEMSIZE or ITEMSIZE
canal.instance.memory.batch.mode = MEMSIZE
canal.instance.memory.rawEntry = true

## detecing config
canal.instance.detecting.enable = false
#canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
canal.instance.detecting.sql = select 1
canal.instance.detecting.interval.time = 3
canal.instance.detecting.retry.threshold = 3
canal.instance.detecting.heartbeatHaEnable = false

# support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
canal.instance.transaction.size =  1024
# mysql fallback connected to new master should fallback times
canal.instance.fallbackIntervalInSeconds = 60

# network config
canal.instance.network.receiveBufferSize = 16384
canal.instance.network.sendBufferSize = 16384
canal.instance.network.soTimeout = 30

# binlog filter config
canal.instance.filter.druid.ddl = true
canal.instance.filter.query.dcl = false
canal.instance.filter.query.dml = false
canal.instance.filter.query.ddl = false
canal.instance.filter.table.error = false
canal.instance.filter.rows = false
canal.instance.filter.transaction.entry = false
canal.instance.filter.dml.insert = false
canal.instance.filter.dml.update = false
canal.instance.filter.dml.delete = false

# binlog format/image check
canal.instance.binlog.format = ROW,STATEMENT,MIXED 
canal.instance.binlog.image = FULL,MINIMAL,NOBLOB

# binlog ddl isolation
canal.instance.get.ddl.isolation = false

# parallel parser config
canal.instance.parser.parallel = true
## concurrent thread number, default 60% available processors, suggest not to exceed Runtime.getRuntime().availableProcessors()
#canal.instance.parser.parallelThreadSize = 16
## disruptor ringbuffer size, must be power of 2
canal.instance.parser.parallelBufferSize = 256

# table meta tsdb info
canal.instance.tsdb.enable = true
#canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
canal.instance.tsdb.url = jdbc:mysql://db.irecircle.com:3306/canal_tsdb?useUnicode=true&characterEncoding=UTF-8&useSSL=false
canal.instance.tsdb.dbUsername = root
canal.instance.tsdb.dbPassword = 123456
# dump snapshot interval, default 24 hour
canal.instance.tsdb.snapshot.interval = 24
# purge snapshot expire , default 360 hour(15 days)
canal.instance.tsdb.snapshot.expire = 360



#################################################
######### 		destinations		#############
#################################################
canal.destinations =
# conf root dir
canal.conf.dir = ../conf
# auto scan instance dir add/remove and start/stop instance
canal.auto.scan = true
canal.auto.scan.interval = 5
# set this value to 'true' means that when binlog pos not found, skip to latest.
# WARN: pls keep 'false' in production env, or if you know what you want.
canal.auto.reset.latest.pos.mode = false

#canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml

canal.instance.global.mode = manager
canal.instance.global.lazy = false
canal.instance.global.manager.address = ${canal.admin.manager}
#canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
#canal.instance.global.spring.xml = classpath:spring/file-instance.xml
canal.instance.global.spring.xml = classpath:spring/default-instance.xml

##################################################
######### 	      MQ Properties      #############
##################################################
# aliyun ak/sk , support rds/mq
canal.aliyun.accessKey =
canal.aliyun.secretKey =
canal.aliyun.uid=

canal.mq.flatMessage = true
canal.mq.canalBatchSize = 50
canal.mq.canalGetTimeout = 100
# Set this value to "cloud", if you want open message trace feature in aliyun.
canal.mq.accessChannel = local

canal.mq.database.hash = true
canal.mq.send.thread.size = 30
canal.mq.build.thread.size = 8

##################################################
######### 		     Kafka 		     #############
##################################################
kafka.bootstrap.servers = 127.0.0.1:6667
kafka.acks = all
kafka.compression.type = none
kafka.batch.size = 16384
kafka.linger.ms = 1
kafka.max.request.size = 1048576
kafka.buffer.memory = 33554432
kafka.max.in.flight.requests.per.connection = 1
kafka.retries = 0

kafka.kerberos.enable = false
kafka.kerberos.krb5.file = ../conf/kerberos/krb5.conf
kafka.kerberos.jaas.file = ../conf/kerberos/jaas.conf

# sasl demo
# kafka.sasl.jaas.config = org.apache.kafka.common.security.scram.ScramLoginModule required \\n username=\"alice\" \\npassword="alice-secret\";
# kafka.sasl.mechanism = SCRAM-SHA-512
# kafka.security.protocol = SASL_PLAINTEXT

##################################################
######### 		    RocketMQ	     #############
##################################################
rocketmq.producer.group = preview-canal
rocketmq.enable.message.trace = false
rocketmq.customized.trace.topic =
rocketmq.namespace =
rocketmq.namesrv.addr = 192.168.110.46:9876
rocketmq.retry.times.when.send.failed = 0
rocketmq.vip.channel.enabled = false
rocketmq.tag =

##################################################
######### 		    RabbitMQ	     #############
##################################################
rabbitmq.host =
rabbitmq.virtual.host =
rabbitmq.exchange =
rabbitmq.username =
rabbitmq.password =
rabbitmq.deliveryMode =


##################################################
######### 		      Pulsar         #############
##################################################
pulsarmq.serverUrl =
pulsarmq.roleToken =
pulsarmq.topicTenantPrefix =
```

## 3、部署canal-server

```properties
# 主要配置
server:
  config: |
    canal.port = 11111
    canal.metrics.pull.port = 11112

    # register ip
    canal.register.ip =

    # 配置canal-admin地址
    canal.admin.manager = canal-admin.recircle-industry-base-system:8089
    canal.admin.port = 11110
    canal.admin.user = admin
    canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
    # admin auto register
    canal.admin.register.auto = true
    canal.admin.register.cluster = pre # 配置集群
```

部署`canal-server`

```shell
helm install canal-server -f ./server-values.yaml ./canal-server
```

<img width="1432" height="774" alt="image" src="https://github.com/user-attachments/assets/b957dc1a-b882-4cbf-8116-5ae2414415cf" />

这个canal-server是可以部署多个replica的

## instance 实例创建

```properties
# 配置你要监听的数据库
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=****
canal.instance.dbPassword=****

# 过滤监听哪些表
canal.instance.filter.regex=.*\\..*

# 如果是推送到rabbitmq，需要配置 Routing Key
canal.mq.topic=你的Routing Key
```
