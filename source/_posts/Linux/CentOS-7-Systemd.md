---
title: CentOS 7 Systemd取代init进程
date: 2017-04-08
categories: Linux运维
tags: Linux
description: Systemd进程取代init进程(很多书籍资料讲的都是init作为启动进程)，在CentOS7中使用``systemctl``代替``service``和``chkconfig``两个命令。
---

由于这个学期学校有Linux课程，我也一直期待着这门课，为了练习在Linux上搭建一些应用，so 我把原来的Windows2012的云服务器换成了CentOS7（其实有很多其他原因，比如mstsc传输速度慢的可怕，而且服务器带宽本来就不行）。在学习过程中遇到了很多问题，主要在于很多Linux命令在CentOS7中有了替代品，也就是所谓的新特性，其中最“坑”的就是Systemd进程取代init进程(很多书籍资料讲的都是init作为启动进程)。这个主要体现在``service``和``chkconfig``命令上，在CentOS7中使用``systemctl``代替``service``和``chkconfig``两个命令。

## ervice命令
service命令本质上是``/sbin``目录下的一个shell脚本，
![service命令本质](http://img-blog.csdn.net/20170408134832232?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
是用来管理系统服务的，这个类似于windows上的sc命令和services.msc，准确的来说Linux上的服务应该叫守护进程(Daemon)，这也是为什么Linux的服务程序后面都会加一个字母d(如httpd，sshd)。service命令用法如下：
```shell
Usage: service  	<option> |
				--status-all |
				[ service_name [ command | --full-restart ] ]
```
1. option参数主要是：-h ，--help，--version
2. --status-all 参数是列出当前所有服务的状态
3. 最后这种用法也是最常用的。
 * service_name顾名思义是服务名，它主要指的是``/etc/init.d``目录下的服务脚本，事实上service脚本就是间接的去调用了该目录下的服务脚本，所以你也可以直接使用这些服务脚本进行服务的启动或关闭，比如``/etc/init.d/sshd start``可以启动ssh服务
 * command是对服务进行的一些操作，比如：start、stop、restart、status
  下面这张图是Oracle Linux5中/etc/init.d目录下的服务脚本
  ![init.d目录下的脚本](http://img-blog.csdn.net/20170408135107033?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## hkconfig命令
chkconfig修改或查询系统服务在各种运行级别中系统开机时的开启|关闭状态，简单的说就是“开机自启动”的配置。
Linux开机启动的第一个进程就是init进程(init程序全路径为``/sbin/init``)，关于Linux“开机启动项”又有一大堆名堂了。首先要说的就是运行级别，运行级别在``/etc/inittab``文件中有详细的描述(标准的Linux运行级别为3或5)：
```shell
0 : 系统停机状态，系统默认运行级别不能设置为0，否则不能正常启动，机器关闭。
1 : 单用户工作状态，root权限，用于系统维护，禁止远程登陆，就像Windows下的安全模式登录。
2 : 多用户状态，没有NFS支持。
3 : 完整的多用户模式，有NFS，登陆后进入控制台命令行模式。
4 : 系统未使用，保留一般不用，在一些特殊情况下可以用它来做一些事情。例如在笔记本电脑的电池用尽时，可以切换到这个模式来做一些设置。
5 : X11控制台，登陆后进入图形GUI模式，XWindow系统。
6 : 系统正常关闭并重启，默认运行级别不能设为6，否则不能正常启动。运行init6机器就会重启。
```
而``/etc/rc.d``下有七个运行级别的启动项配置，也就是七个rc*.d目录
![七个运行级别](http://img-blog.csdn.net/20170408135239077?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这7个目录下记录都是链接文件，这些链接文件以"K"或"S"打头，也就分别对应者这个运行级别下相应服务的“关闭”或“启动”。这些链接文件指向的是``/etc/rc.d/init.d``目录下的shell脚本。事实上前面service命令中说道``/etc/init.d``目录其实是一个链接文件，指向的就是``/etc/rc.d/init.d``目录。
![运行级别对应的连接文件](http://img-blog.csdn.net/20170408135338459?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
在网上找到这个比较容易理解的图，需要注意的是``/etc/rc*.d``连接到了``/etc/rc.d/rc*.d``;
![Linux系统启动过程](http://img-blog.csdn.net/20170408135420023?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

chkconfig具体用法如下(“#”号后面是注释)
```shell
usage:
		① chkconfig --list [name]    	#①列出所有的系统服务
		② chkconfig --add <name>    	#②添加一个系统服务
		③ chkconfig --del <name>    	#③删除一个系统服务
		④ chkconfig [--level <levels>] <name> <on|off|reset|resetpriorities>
				#④修改指定服务在指定级别中的开闭状态
```
## ystemctl命令
说道systemctl命令就要说道Systemd这个守护进程了，Systemd是用来取代传统的开机进程init进程的，主要原因是init是串行启动，前一个服务进程启动完成后下一个服务进程才能启动，这也就导致了init进程启动耗时比较长。关于Systemd体系与init的比较。在维基百科中有这样一段描述：

与System V风格init相比，systemd采用了以下新技术：
- 将service（服务）、target（运行模式，类似于运行档次）、mount、timer、snapshot、path、socket、swap等称为Unit。比如，一个auditd服务（就是auditd.service）就是一个Unit，一个multi-user.target运行模式也是一个Unit。
- 采用Socket激活式与D-Bus激活式服务，以提高相互依赖的各服务的并行运行性能；
  用cgroups代替进程ID来追踪进程，以此即使是两次fork之后生成的守护进程也不会脱离systemd的控制。
- 用target代替System V的运行级别（Runlevel），比如，SystemD的graphical.target相当于System V的init 5，multi-user.target相当于System V的init 3。
- 内置新的journald 日志管理系统。
- 引入localectl、timedatectl等新命令，系统配置更方便。

而且还配了一张神图：
![Systemd体系结构](http://img-blog.csdn.net/20170408135533712?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
上图最顶层“系统工具”中最常用的就是systemctl命令了(其实部分命令有些发行版系统默认都没有安装)，总之呢，你可以把它理解为前面讲到的service和chkconfig命令的结合体。systemctl命令的使用还是比较复杂的，而且systemd体系也比较复杂，我怕我理解的还不够透彻讲不好，所以推荐两篇关于systemd的文章：
- [Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)
- [Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)
