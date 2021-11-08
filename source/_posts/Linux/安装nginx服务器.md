---
title: 源码安装nginx服务器并配置服务自启动
date: 2017-09-16
tags:
categories: Linux运维
---
## 下载

可以到官网找下载地址：http://nginx.org/en/download.html

这里使用1.8.1版本的源码安装

```shell
wget http://nginx.org/download/nginx-1.8.1.tar.gz
```

> 下载方式多种多样，你也可以用ftp传上去

## 安装前环境准备

**1.**准备gcc-c++编译器

```shell
yum install gcc-c++
```

**2.**准备依赖库

nginx依赖于PCRE、Zlib、OpenSSL。

* PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库。
* zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip压缩，所以需要在linux上安装zlib库。
* OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。

```shell
yum install -y pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

##解压编译并安装

**解压**

```shell
##解压gz压缩包
tar -zxvf nginx-1.8.1.tar.gz
```

**配置**

```shell
##切换到nginx源码目录
cd nginx-1.8.1
##使用默认配置即可
./configure
```

> 如果要自行配置安装文件或配置文件所在目录，可添加相应的配置选项。除了配置安装目录，还可以对官方提供的一些额外模块的编译进行配置，详细的配置选项可参考官方文档http://nginx.org/en/docs/configure.html
>
> 常用的模块如：[ngx_http_ssl_module](http://nginx.org/en/docs/http/ngx_http_ssl_module.html)、[ngx_http_gzip_module](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)等
>
> 安装时如果需要附加这些功能模块，使用如下命令即可：
> `./configure --prefix=/usr/local/nginx --with-http_ssl_module`
> 配置了安装目录在`/usr/local/nginx`，同时编译安装了`http_ssl_module`模块。

**编译安装**

```shell
##编译
make
##安装
make install
```

## 启动与停止nginx

默认配置下，Nginx的安装目录为`/usr/local/nginx`

所以可执行文件路径为`/usr/local/nginx/sbin/nginx`

**启动nginx**

```shell
##启动nginx
/usr/local/nginx/sbin/nginx
```

> 默认情况下`./nginx`执行时自动加载`/usr/local/nginx/conf/nginx.conf`配置，可以使用`-c`选项自定义配置文件的路径。默认配置文件的路径可在安装前`./configure`时配置。

**查看nginx进程状况**

```shell
[root@localhost ~]# ps -aux | grep nginx
Warning: bad syntax, perhaps a bogus '-'? See /usr/share/doc/procps-3.2.8/FAQ
##主进程
root     13405  0.0  0.1   5344   640 ?        Ss   03:46   0:00 nginx: master process ./nginx
##工作进程
nobody   13406  0.0  0.1   5544   984 ?        S    03:46   0:00 nginx: worker process
root     13439  0.0  0.1   4360   756 pts/0    S+   04:10   0:00 grep nginx
```

> 进程启动的id号会被写入`/usr/local/nginx/logs/nginx.pid`日志文件中。

**Fast Shutdown**

```shell
/usr/local/nginx/sbin/nginx -s stop
```

> 这个命令会从`/usr/local/nginx/logs/nginx.pid`文件中查出进程id，然后强制杀死。

**graceful shutdown**

```shell
/usr/local/nginx/sbin/nginx -s quit
```

> 此方式停止步骤是待nginx进程处理任务完毕进行停止。

**reloading the configuration file**

```shell
/usr/local/nginx/sbin/nginx -s reload
```

> 该命令用于重新加载配置文件

## 开放80端口

nginx默认使用80端口通信，所以需要打开80端口

```
iptables -I INPUT -s 0.0.0.0 -p tcp --dport 80 -j ACCEPT
```

## 测试

![测试](http://img-blog.csdn.net/20171126204627503?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 开启自启动

将以下脚本写入到`/etc/init.d/nginx`文件中。

> 我不会shell脚本
> 脚本参考：https://www.nginx.com/resources/wiki/start/topics/examples/redhatnginxinit/

```shell
#!/bin/bash
# nginx Startup script for the Nginx HTTP Server
# it is v.0.0.2 version.
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
#              It has a lot of features, but it's not for everyone.
# processname: nginx
# pidfile: /var/run/nginx.pid
# config: /usr/local/nginx/conf/nginx.conf
nginxd=/usr/local/nginx/sbin/nginx
nginx_config=/usr/local/nginx/conf/nginx.conf
nginx_pid=/var/run/nginx.pid
RETVAL=0
prog="nginx"
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0
[ -x $nginxd ] || exit 0
# Start nginx daemons functions.
start() {
if [ -e $nginx_pid ];then
   echo "nginx already running...."
   exit 1
fi
   echo -n $"Starting $prog: "
   daemon $nginxd -c ${nginx_config}
   RETVAL=$?
   echo
   [ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
   return $RETVAL
}
# Stop nginx daemons functions.
stop() {
        echo -n $"Stopping $prog: "
        killproc $nginxd
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /var/run/nginx.pid
}
# reload nginx service functions.
reload() {
    echo -n $"Reloading $prog: "
    #kill -HUP `cat ${nginx_pid}`
    killproc $nginxd -HUP
    RETVAL=$?
    echo
}
# See how we were called.
case "$1" in
start)
        start
        ;;
stop)
        stop
        ;;
reload)
        reload
        ;;
restart)
        stop
        start
        ;;
status)
        status $prog
        RETVAL=$?
        ;;
*)
        echo $"Usage: $prog {start|stop|restart|reload|status|help}"
        exit 1
esac
exit $RETVAL
```

修改该文件的权限（所有用户可执行）

```shell
chmod a+x /etc/init.d/nginx
```

在`/etc/rc.local`文件中添加启动命令：

```shell
/etc/init.d/nginx start
```

> 保存并退出，下次重启会生效。





**参考链接：**

nginx官方文档：http://nginx.org/en/docs/

http://blog.csdn.net/u013870094/article/details/52463026
