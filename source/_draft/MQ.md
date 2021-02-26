之前写过[一篇文章](https://blog.hufeifei.cn/2020/04/25/Alibaba/MetaQ&Notify/)，大致地讲了一下mq的解决的问题，以及从传统的ActiveMQ到现在的Kafka和RocketMQ的演进过程。
这次项目中引进Kafka来解决数据同步的问题，自己也搭了kafka，期间碰到了一些问题以及自己的一些思考，这篇文章再记录一下。

![MQ Architech](http://www.plantuml.com/plantuml/svg/FSqz2i904CNnVaynfCyL96Yzu05ibimpBYOpE1_tTr6qUloAFs_nQ1Pvx6N7FIYKh6-F8Ew6DRfA4MNGrPHpXNrrKV4yXbw914qLxct3JSwcJzX4pQcMNqFpV1gid_sd2uJ7xHi0)

MQ的架构中有三个角色：Producer、Broker、Consumer

在消息的发送与消费过程中，这三者都可能出现问题：

生产者：
1. 生产者重复生产
2. 生产者事务，分布式事务

broker：
1. 存储问题，基于数据库，基于文件，或其他存储
2. 高可用问题，replica

消费者：
1. pull vs. push
2. 多个消费组，其中一组消费者挂了怎么办
3. 重复消费，消费者的幂等性


kafka consumer
https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html



https://spring.io/blog/2010/06/14/understanding-amqp-the-protocol-used-by-rabbitmq
https://www.zhihu.com/question/54152397
https://www.zhihu.com/question/34243607
https://tech.meituan.com/2016/07/01/mq-design.html
