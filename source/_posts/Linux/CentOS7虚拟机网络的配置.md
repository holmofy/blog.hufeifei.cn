---
title: CentOS7虚拟机网络的配置
date: 2017-7-08 13:56
categories: Linux运维
tags: Linux
---

这几天做一个项目，要搭建图片服务器，所以在虚拟机上先模拟一下，因为项目后续可能需要使用集群进行测试，我这渣渣电脑，带不起，所以我把虚拟机的内存限制在512M，硬盘存储限制在10G。考虑到图形界面太耗内存，也占空间，所以全用命令行的方式进行各种配置。Linux命令这东西几个月不碰，果然忘得很快，这里我把虚拟机网络配置的过程写下来，方便日后再用。

**虚拟机安装过程省略**

网上一大把。

# 物理机配置

为了方便测试虚拟机是否能ping通物理机，物理机需要先打开ICMP回显功能。Win7以上的操作系统需要在高级安全防火墙中配置：

高级安全Windows防火墙==> 入站规则 ==>ICMPv4回显

![高级防火墙](http://img.blog.csdn.net/20171122145106264?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 配置VMware虚拟网络

我这里都是使用NAT的方式对虚拟机进行配置，所以需要将VMware的NAT服务打开。

![开启NAT服务](http://img.blog.csdn.net/20171122145644387?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 一般安装完VMware，这个几个相关服务都是默认开启的，因为我有一段时间没用VMware，所以我把这几个服务都关了。如果关了这个服务记住这个时候应该把它打开。

**配置NAT**

![NAT配置](http://img.blog.csdn.net/20171122145728736?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里为了方便以后的测试，所有的虚拟机都使用静态IP地址，所以先把DHCP关了。

![NAT](http://img.blog.csdn.net/20171122145954837?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**配置网关：**

![默认网关](http://img.blog.csdn.net/20171122150204631?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 这些配置最终都会保存在`C:\ProgramData\VMware\vmnetnat.conf`文件中。



另外**建议**把物理机的虚拟IP地址也设置成静态的：

![VM8](http://img.blog.csdn.net/20171122150333269?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![IP](http://img.blog.csdn.net/20171122150446734?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![IPv4](http://img.blog.csdn.net/20171122150828270?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# Linux虚拟机配置

**IP地址等配置**

为了防止每次Linux重启，都要使用`ifconfig`进行配置，所以需要在`etc/sysconfig/network-scripts/`目录中找到网卡接口的配置文件，并把地址设置成静态地址：

> `ifconfig`属于`net-tools`中的一个命令，由于年久失修无人维护，在CentOS/RHEL7等发行版中已被`iproute2`工具包取代。所以最小化安装时找不到`ifconfig`命令。
>
> ![Net-Tools与IpRoute2](https://dn-linuxcn.qbox.me/data/attachment/album/201406/04/003404uy9l1t5zayzllylm.png)

![设置IP](http://img.blog.csdn.net/20171122150927555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 这里配置的IP地址需要在前面配置的子网范围内。

配置完成后重启网络服务：

```shell
service network restart
```

# 测试

![物理机IP信息](http://img.blog.csdn.net/20171122151026692?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![测试](http://img.blog.csdn.net/20171122151054662?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

