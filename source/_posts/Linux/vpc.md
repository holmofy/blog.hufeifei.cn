---
title: 记一次VPC问题的排查
date: 2021-03-03
categories: Linux运维
tags: Linux
---

前几天新买了台机器想用来做日志分析，发现两台机器用内网地址死活连不上，腾讯云文档上都说同一VPC下不同子网默认是互通的。最后向腾讯云提了个工单才发现是自己服务器docker网段和内网网段冲突导致的。这里记录一下整个过程的始末。

# 1、 申请了台新机器

之前那台机器是大学里买的，穷逼学生机配置：1核CPU、1G内存、1M带宽，ElasticSearch起都起不来，漫天OOM。

所以新弄了台2G的机器，单独做日志分析。

![server](https://p.pstatp.com/origin/pgc-image/9038b52b19b64847a62f6312c6335a7f)

# 2、公网不行，用内网，可是内网也不通

穷逼一个，新机器也只买了1M带宽，所以两台机器肯定是不能通过公网来进行通信的。

那就用内网地址咯。

同一私有网络下，不同子网内网不通。公网ping没有问题，但是内网无法连接。

![ping](https://p.pstatp.com/origin/pgc-image/219f414e43da43ffa61094e25d75d13e)
![ping](https://p.pstatp.com/origin/pgc-image/128055bc6f9b4221beaee61ea6a76281)

# 3、安全组策略的问题？

因为刚开始安全组入站规则，我是把ICMP的ping给禁用了的，所以专门把ICMP开放了

![安全组策略](https://p.pstatp.com/origin/pgc-image/7ea5195936d4441c9892d4a388afb777)

# 4、原来是路由问题

我装了个telent尝试用别的协议连一下，这个报错让我发现了问题的关键: 路由配置有问题

![telnet](https://p.pstatp.com/origin/pgc-image/0be08b7079e34d68ad676a16e509a55f)

我看了下路由表

![路由表](https://p.pstatp.com/origin/pgc-image/8e7751c8623b4d2eaf7ff1cbf76f49ac)

可以看到docker网络与内网网段冲突了——docker的网段正好覆盖了内网网段

# 5、修改docker网卡地址

1）vim /etc/docker/daemon.json（这里没有这个文件的话，自行创建）
```json
{
    "bip":"192.168.0.1/24"
}
```

2）重启docker
```sh
systemctl restart docker
```

3) ifconfig 和 route检查网卡地址和路由表

![ping](https://ae04.alicdn.com/kf/Ub1ef8e3cb5cc4636bd7ebcbabfedfaaeM.jpg)

使用内网地址也能互相ping通了。😊

# 6、测试一下内网带宽

一端开启服务，监听流量
```sh
root@report:~ $ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```
一段连接测试
```sh
iperf3 -c 172.168.0.11
Connecting to host report, port 5201
[  4] local 172.17.240.11 port 42010 connected to 172.17.0.11 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   300 MBytes  2.52 Gbits/sec  1820    229 KBytes
[  4]   1.00-2.00   sec   182 MBytes  1.53 Gbits/sec  284    325 KBytes
[  4]   2.00-3.00   sec   181 MBytes  1.52 Gbits/sec  399    212 KBytes
[  4]   3.00-4.00   sec   176 MBytes  1.48 Gbits/sec  263    250 KBytes
[  4]   4.00-5.00   sec   186 MBytes  1.56 Gbits/sec  407    237 KBytes
[  4]   5.00-6.00   sec   190 MBytes  1.59 Gbits/sec  362    148 KBytes
[  4]   6.00-7.00   sec   182 MBytes  1.53 Gbits/sec  247    321 KBytes
[  4]   7.00-8.00   sec   185 MBytes  1.55 Gbits/sec  331    328 KBytes
[  4]   8.00-9.00   sec   181 MBytes  1.52 Gbits/sec  425    215 KBytes
[  4]   9.00-10.00  sec   170 MBytes  1.43 Gbits/sec  206    309 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.89 GBytes  1.62 Gbits/sec  4744             sender
[  4]   0.00-10.00  sec  1.89 GBytes  1.62 Gbits/sec                  receiver

iperf Done.
```

嚯，果然牛逼，1.6G带宽妥妥的。😄
