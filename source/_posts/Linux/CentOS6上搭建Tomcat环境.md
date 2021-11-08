---
title: CentOS6上搭建Tomcat环境
date: 2017-08-15
tags:
categories: Linux运维
---

## 载并安装JDK

**卸载原装的OpenJDK(如果有)**

```shell
## 看是否安装Java
java -version
## 看Java的安装包信息
rpm -qa | grep java
## 载原装Java,<java_package>为查找到的安装包信息
rpm -e --nodeps <java_package>
```

> OpenJDK是JDK的开源版本，Linux使用yum源安装的JDK都是这个版本，建议使用OracleJDK代替OpenJDK。
> 我这里使用的是最小化安装，所以就没有自带JDK了。

**下载OracleJDK，官网下载地址：**

http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-i586.tar.gz

```shell
## 建安装目录
mkdir -p /usr/local/java
## 压
tar -xzvf jdk-8u151-linux-i586.tar.gz -C /usr/local/java
```

![解压后的JDK目录](http://img-blog.csdn.net/20171128143021657?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**配置JAVA环境变量：**

```
vi /etc/profile
```

在/etc/profile文件末尾添加以下几行配置，注意第二行的最前面的“.”指的是当前路径，不是手误。还有`JAVA_HOME`目录的路径尽量靠过来，避免手残，敲错了找半天。

```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_151
export CLASSPATH=.:$JAVA_HOME/lib/tool.jar:$JAVA_HOME/lib/dt.jar
export PATH=$PATH:$JAVA_HOME/bin
```

![JAVA环境设置](http://img-blog.csdn.net/20171128143120386?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用source命令让配置生效

```shell
source /etc/profile
```

![测试JDK环境是否安装成功](http://img-blog.csdn.net/20171128143347441?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 载并安装Tomcat

从清华大学的镜像站下载会快一点：

https://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.23/bin/apache-tomcat-8.5.23.tar.gz

> 因为Tomcat是Java写的，所以只要有了JRE就可以“一次编译到处运行”。so，Tomcat解压即可使用。

**解压**

```shell
tar -xzvf apache-tomcat-8.5.23.tar.gz -C /usr/local/java
```

![解压Tomcat](http://img-blog.csdn.net/20171128143448805?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**配置Tomcat的环境变量**

在`/etc/profile`文件后再追加一条TOMCAT的环境变量

```shell
## /etc/profile文件末尾追加TOMCAT的环境变量
export CATALINA_HOME=/usr/local/java/apache-tomcat-8.5.23
```

> [CATALINA](http://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/startup/Catalina.html)是Tomcat的启动程序，Tomcat的启动脚本都是使用`CATALINA_HOME`作为变量，所以这里我们要设置`CATALINA_HOME`

![配置Tomcat环境变量](http://img-blog.csdn.net/20171128143533299?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用`source`命令完成是配置生效

```shell
source /etc/profile
```

**将Tomcat配置为服务**

> 将Tomcat配置为系统服务后，就方便使用`service`命令来启动或关闭Tomcat服务
>
> 省的每次启动后还要到tomcat的bin目录下找startup脚本

```shell
## tomcat的脚本文件拷一份到/etc/init.d目录
cp /usr/local/java/apache-tomcat-8.5.23/bin/catalina.sh /etc/init.d/tomcat8

## 把改脚本授权给所有用户执行
chmod 755 /etc/init.d/tomcat8
```

拷贝的脚本并不能直接使用，还需要修改添加一些配置。

```shell
vi /etc/init.d/tomcat8
```

添加`chkconfig`和`description`两行注释。有这两行注释才能支持chkconfig命令配置服务；

同时加上`JAVA_HOME`和`CATALINA_HOME`两个变量的声明。

```shell
#chkconfig: 2345 10 90
#description: tomcat8 service

export JAVA_HOME=/usr/local/java/jdk1.8.0_151
export CATALINA_HOME=/usr/local/java/apache-tomcat-8.5.23
```

> 这里配置的2345指的是2345这4个运行级别会开机自启动，10是启动优先级，90是关闭优先级，优先级的值为0-99，越小优先级越高。
>
> 前面在`/etc/profile`文件配置中的环境变量只会在shell登录后执行，开机的过程中并不会加载`/etc/profile`，但是tomcat的启动脚本中需要这两个变量，所以需要在启动脚本中加入这两个变量。

![配置Tomcat服务](http://img-blog.csdn.net/20171128143715873?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用`chkconfig --add`命令添加服务

```shell
[root@localhost ~]# chkconfig --add tomcat8
```

> 配置完成后Tomcat服务即可开机自启动
>
> 同时还可以使用`service tomcat8 start`和`service tomcat8 stop`命令来启动和停止tomcat服务。

![检查服务是否安装成功](http://img-blog.csdn.net/20171128143801312?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 置防火墙打开8080端口并访问测试

```shell
## 内网网段，打开8080端口
iptables -I INPUT -s 192.168.10.0/24 -p tcp --dport 8080 -j ACCEPT
```

> 网络的配置由实际的环境决定

物理机访问测试：

![物理机访问测试](http://img-blog.csdn.net/20171128143833074?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
