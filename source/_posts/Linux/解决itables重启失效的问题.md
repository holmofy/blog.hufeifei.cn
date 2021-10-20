---
title: 解决itables重启失效的问题
date: 2017-09-16
tags:
categories: Linux运维
---

直接使用`iptables`命令修改防火墙配置的时候，防火墙规则只是保存在内存中，重启后就会失效。



一种最简单的方式是在修改防火墙陪之后，再使用`service iptables save`命令将防火墙配置保存起来；

使用该命令会将所有的防火墙规则保存在`/etc/sysconfig/iptables`文件中。



另一种方法是使用`iptables-save`命令，顾名思义，该命令用于保存当前的防火墙规则的。

直接使用该命令会直接将防火墙规则打印到控制台。

我们还需要配合IO流重定向，将防火墙规则保存在文件中。

```shell
iptables-save > /etc/sysconfig/iptables
```

> 保存在`/etc/sysconfig/iptables`中效果和`service iptables save`命令一样。
>
> 如果保存在其他路径，重启后可以使用 `iptables-restore`命令恢复防火墙规则。
>
> `iptables-restore < path`(path为保存的路径)
