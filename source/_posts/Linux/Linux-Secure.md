---
title: Linux服务器基础安全防范
date: 2017-04-15
tags:
categories: Linux运维
---

每次登录服务器的时候总有提示说有人通过ssh尝试n次登录失败。
![Linux安全防范](http://img-blog.csdn.net/20170415164214748?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
查一查``/var/log/secure``日志文件

```shell
 grep "Failed password for invalid user" /var/log/secure | awk '{print $13}' | uniq -c | sort -nr | more
```
![日志文件](http://img-blog.csdn.net/20170415164324913?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
卧槽，简直就是国内外云集啊，什么乌克兰、保加利亚、俄罗斯、巴西、阿根廷.....，我听过的没听过的国家都齐了，这尼玛都可以举办个奥运会了。
虽然我自认我密码设置的已经相当复杂了，而且服务器上也没什么重要的东西，但是心里还是疙得慌。怎么说还是得有个防范好，害人之心不可有，防人之心不可无啊。

##修改SSH端口并禁止root登录
ssh的配置文件全路径是：``/etc/ssh/sshd_config``
![SSH配置](http://img-blog.csdn.net/20170415164349764?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
```shell
Port 2222	#这个端口默认是22，改成不容易猜的
PermitRootLogin no
```
> 如果使用的是云服务器，修改端口后还需要配置安全组

然后我们需要重启ssh服务
```
systemctl restart sshd
```
限制root用户登录，攻击者就没办法尝试root密码了，我们可以用普通用户登录，然后用sudo命令来执行权限操作，当然登录普通用户后，也可以通过su命令来切换到root用户。
修改端口，个人觉得作用并不明显，人家一个端口扫描立马能找到你服务器开启了哪些端口，我们还需要做更多的限制。

##安装denyhosts
看看百度百科怎么说的
> DenyHosts是Python语言写的一个程序，它会分析sshd的日志文件（/var/log/secure），当发现重 复的攻击时就会记录IP到/etc/hosts.deny文件，从而达到自动屏蔽IP的功能。

这方法确实省时省力，废话不多说先把软件装上
```shell
 yum install denyhosts
```
基本上默认的配置就可以使用了，如果想要更多的配置信息，可以查看``/etc/denyhosts.conf``文件。
denyhosts这个毕竟是软件分析/var/log/secure文件，你某次不小心输错了密码，它可能把你也给墙了(自己有亲身经验，自己挖坑自己填::>_<::)，所以还需要配置白名单，白名单文件全路径为``/var/lib/denyhosts/allowed-hosts``。在里面写上自己的公网地址就行了。
配置完成后，我们就可以启动该服务了
```shell
systemctl start denyhosts
```

##命令总结
**grep：** globally search a regular expression and print， 正则表达式匹配，类似于Windows上的findstr命令。[命令参考文章](http://www.cnblogs.com/end/archive/2012/02/21/2360965.html)
**awk：** 该命令是AWK语言创始人的缩写，AWK是文本文件处理语言，也是一个强大的文本分析工具。[命令参考文章](http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2858470.html)
**sort：** 对文件中的行进行排序
**uniq**  unique，去除文件重复行
[sort,uniq,cut,wc命令参考文章](http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2858385.html)
**systemctl：** system control ，Systemd进程的系统全局控制面板。[参考文章](http://hufeifei.cn/2017/04/08/CentOS-7-Systemd/index.html)
