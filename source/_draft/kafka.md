# 1、ack


# 2、日志清除策略

墓碑标记

# 3、partition的设计

https://www.zhihu.com/question/28925721

# 消费者

customer group

超时，kafka会认为这个消费者死了，重新rebalance，即使你后面提交了offset，commit也会失败。下次poll还会poll到这个消息。所以要保证处理好消息的幂等性。

# rebalance的问题

rebalance协议

# 4、客户端

https://github.com/birdayz/kaf

https://github.com/fgeller/kt

https://github.com/adevinta/zoe

https://github.com/lensesio/kafka-connect-tools
