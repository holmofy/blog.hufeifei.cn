---
title: 安装redis并配置服务自启动
date: 2017-04-15
tags:
categories: Linux运维
---

# 安装Redis

安装过程很简单，[官网](https://redis.io/download)也有安装教程，这里贴一下redis-3.2.11版的安装过程：

**1.**下载源码包

```shell
wget http://download.redis.io/releases/redis-3.2.11.tar.gz
```

**2.**解压

```shell
tar -xzvf redis-3.2.11.tar.gz -C /usr/local/
```

**3.**编译redis源码并安装

> 因为redis是C语言编写的，所以编译redis时需要gcc-c++编译器。安装gcc-c++的命令：
>
> `yum -y install gcc-c++`

```shell
# 切换到解压的源码目录
cd /usr/local/redis-3.2.11
# 编译源码
make
# 安装redis
make PREFIX=/usr/local/redis-3.2.11 install
```

> 如果不添加`PREFIX`选项，则默认安装在`/usr/local/bin`目录。
>
> 安装过程只是把生成的几个可执行文件拷贝到安装目录：
>
> * redis-benchmark：性能测试工具
> * redis-check-aof：用于修复出问题的aof文件
> * redis-check-rdb：用于修复出问题的dump.rdb文件
> * redis-cli：命令行形式的客户端工具
> * redis-sentinel：集群管理工具
> * redis-server：redis核心服务程序

**4.**启动redis服务

安装完成后即可使用redis-server启动redis。但是直接使用该命令redis会运行在前台。

![Redis服务](http://img-blog.csdn.net/20171207221009579?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们可以通过指定配置文件的方式启动，redis源码包中已经提供了原始配置文件。

启动之前我们需要对配置文件进行一些修改：

![Redis服务启动配置文件](http://img-blog.csdn.net/20171207221030041?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 安装redis服务并配置开机自启

通常我们要安装服务，需要在`/etc/init.d/`目录下添加符合一定规则的启动脚本。

redis源码包中已经提供了这些脚本，而且提供了安装脚本。

![安装服务](http://img-blog.csdn.net/20171207221051501?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

服务安装完成后我们看到在`/etc/init.d/`目录下安装了一个名为`redis_6379`的服务。

![安装后测试](http://img-blog.csdn.net/20171207221119020?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 安装完成后即可像管理其他服务一样使用service命令进行管理。





>参考链接
>
>* redis官方下载：https://redis.io/download

