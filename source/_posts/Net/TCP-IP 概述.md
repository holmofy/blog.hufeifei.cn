---
title: TCP-IP  概述
date: 2017-07-21
categories: 网络
mathjax: true
---

TCP/IP起源于60年代末由美国政府资助的一个分组交换网络——ARPAnet(阿帕网)。到90年代TCP/IP就已成为事实上的工业标准了。

##网络分层

网络分层从ARPAnet开始就已经在使用，将网络协议分为不同层次开发，能简化设计的复杂性，各层既能相互独立又能高效地协调工作（这也是C语言这种面向过程语言解决问题的常用方法）。提到分层，我们通常会把TCP/IP层次模型与OSI参考模型进行对比。

![OSI模型与TCP/IP模型](http://img-blog.csdn.net/20170721144322946?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## OSI参考模型

OSI参考模型是ISO国际标准化组织的建议。虽然OSI并未成为事实上的标准，但仍然具有学习参考和工作指导的意义。

1. **物理层**：主要功能是**利用物理传输介质为链路层提供基于比特流的数据传输**；同时制定物理接口的相关标准(如接口的机械特性、电气特性、规程特性等)，这些标准一般由OSI或IEEE等协会以及各大硬件厂商(通常也是协会成员)协同制定。比如有线网卡接口与双绞线接口线序，蓝牙或WiFi信号的频段，甚至USB接口、Type-C接口，ThunderBolt等都是他们制定的。

   > 大二的时候自己还接过水晶头，依稀记得双绞线的两种线序：
   >
   > * 568A线序：
   >
   > 绿白—1，绿—2，橙白—3，蓝—4，蓝白—5， 橙—6，棕白—7，棕—8
   >
   > * 568B线序：
   >
   > 橙白—1，橙—2，绿白—3，蓝—4，蓝白—5， 绿—6，棕白—7，棕—8
   >
   > **两种线序仅仅把1和3对调，2和6对调。**
   >
   > * 直通线：单独使用一种线序，2头A序或2头B序。
   > * 交叉线：1头A序1头B序。
   >
   > **同层设备使用交叉线，不同层设备使用直通线。**
   >
   > 主机和路由器同属网络层设备所以使用交叉线。
   >
   > 而交换机属于链路层设备，如果与网络层设备相连就需要使用直通线，如果与交换机相连需要使用交叉线。

2. **数据链路层**：主要功能是**差错校验、流量控制、链路管理**等功能，**为网络层提供一个无错的透明的数据传输**。这一层会把网络层的数据包封装成数据帧(Frame)。常见的链路层封装协议有：PPP、HDLC、Frame-Relay、SDLC等。

   > 现在只依稀地记得差错校验用到了组成原理中学到的CRC、奇偶校验。

3. **网络层**：主要功能是**解决数据包的路径选择问题，为传输层提供网络连接**。这一层最核心的协议就是IP协议，它会把传输层的数据报封装成IP数据包(Packet，也称为分组包)，通过路由协议转发到指定的网络。另外，用于网络层管理的两个协议是ICMP和IGMP，以太网中ARP和RARP负责IP地址与Mac硬件地址之间的转换，ARP和RARP基于IP协议但工作在链路层，所以严格的说ARP和RARP应该数据数据链路层协议。

   > ICMP和IGMP的区别：
   >
   > * ICMP(Internet Control Message Protocol)：用于检测网络通断，主机是否可达，查看路由路径等。
   >
   >   ping命令和traceroute命令(windows上的pathping)就是实现了这个功能。
   >
   > * IGMP(Internet Group Management Protocol)：IGMP是一个组播(多播)管理协议。如网络电视直播就是组播(多播)的一种应用。后面也会讲到单播，多播(组播)，广播的区别。
   >
   > ARP和RARP的区别：
   >
   > * ARP(Address Resolution Protocol)：IP地址 -> Mac地址
   > * RARP(Reverse Address Resolution Protocol)：Mac地址 -> IP地址
   >
   > 可以使用arp命令查看或修改地址解析表。因为ARP是通过局域网广播的方式工作的，恶意虚假应答会造成ARP欺骗，所以在IPv6中ARP已被邻居发现协议(NDP)取代。

   **路由协议**分为两类：静态路由协议和动态路由协议。静态路由协议维护性不好，在网络中使用更广泛的是动态路由协议。**动态路由协议**分为两种：内部网关协议(IGP)和外部网关协议(EGP)。常见的IGP有：路由信息协议(RIP,有v1和v2两个版本)，内部网关路由协议(IGRP)，增强内部网关路由协议(EIGRP)，开放式最短路径优先(OSPF)。EGP主要是边界网关协议(BGP)。

   > IGP按实现原理也可以分为三类，下面这张图是当时在大三上学期学网络配置的时候总结的：
   >
   > ![路由协议](http://img-blog.csdn.net/20170721144406075?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4. **传输层**：主要功能是为上层协议提供端到端的可靠和透明的数据传输服务。这一层主要的两个协议就是TCP和UDP。

   > TCP与UDP的区别：
   >
   > * TCP(Transfer Controll Protocol)是面向连接的可靠的传输，TCP底层基于数据报实现并为上层提供面向数据流的传输。
   > * UDP(User Datagram Protocol)是面向数据报(Datagram)的不可靠的传输，UDP仅仅在IP协议的基础上增加了端口，报文长度，校验和等字段；相反TCP相对UDP就更复杂了。当然，你也可以基于UDP协议自己实现面向连接的传输，可靠性按照自己的需求来实现。
   >
   > | 区别     |  TCP   |  UDP   |
   > | :----- | :----: | :----: |
   > | 是否面向连接 |   是    |   否    |
   > | 传输可靠性  |   可靠   |  不可靠   |
   > | 应用场合   | 大数据量传输 | 少量数据传输 |
   > | 速度     |   慢    |   快    |

5. **会话层**：主要功能是管理和协调主机进程之间的会话，即建立、管理、终止进程间会话，简单的说就是**会话管理**。

6. **表示层**：主要功能是处理数据编解码问题。比如数据的压缩与加密就是在这一层实现。

7. **应用层**：主要功能是为用户提供接口。常见的如FTP协议实现文件传输，SMTP协议实现邮件的传输，HTTP协议实现超文本数据的传输。

> 网络工程师的工作范围主要就是下三层，而软件设计师的工作范围主要就是上三层。

## TCP/IP分层模型

OSI参考模型的缺点就是过于复杂，层次分的太细，对于那些基于UDP不需要会话的协议基本上就没有会话层这一概念。比如DNS协议查询域名对应的IP地址。

**OSI参考模型是理论上的模型，而TCP/IP事实上的工业标准**。TCP/IP是先有实现后有模型。

**TCP/IP中分为四层：链路层、网络层、传输层、应用层**。其中链路层对应着OSI模型中的物理层和链路层，应用层对应着OSI模型中的会话层、表示层和应用层。

来看一下《TCP/IP详解》中的两张图：

![TCP/IP分层模型](http://img-blog.csdn.net/20170721144450411?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![对等实体](http://img-blog.csdn.net/20170721144514924?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![协议栈封装过程](http://img-blog.csdn.net/20170721144535616?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> OSI模型与TCP/IP模型都不完美，但TCP/IP模型发展是因为在ISO制定OSI参考模型过程中总是着眼于一次制定达到完美，所以的制定过程中考虑的方面比较多，但去忽略了IP这一协议的重要性，但当ISO认识到时只好在网络层划出一个子层来完成类似的功能，在无连接服务一开始也不在考虑之列，还有就是网络管理功能的过度复杂等，造成了OSI迟迟没有成熟的产品推出的成因，进而影响了厂商对它的支持，而这时的TCP/IP通过实践得到到不断的完善，也得到了大厂商的支持，所以TCP/IP模型得到了发展。

##网络地址(IP地址)

IP地址用来标识网络中主机的唯一性的。IP地址分为两个版本：IPv4和IPv6。

## IPv4

IPv4地址长度为32bit，并划分为5类：

![IP地址分类](http://img-blog.csdn.net/20170721144600810?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

|  类型  | 范围                        | 网络数                                      | 主机数(去掉网络号和广播地址) |
| :--: | ------------------------- | ---------------------------------------- | --------------- |
|  A   | 0.0.0.0~127.255.255.255   | 0~127，共128=$2^{7}$个                      | $2^{24}-2$个     |
|  B   | 128.0.0.0~191.255.255.255 | 128.0~191.255，共64×256=$2^{14}$个          | $2^{16}-2$个     |
|  C   | 192.0.0.0~223.255.255.255 | 192.0.0~223.255.255，共32×256×256=$2^{21}$个 | $2^{8}-2$个      |
|  D   | 224.0.0.0~239.255.255.255 | /                                        | /               |
|  E   | 240.0.0.0~247.255.255.255 | /                                        | /               |

其中A、B、C分别有一个保留地址：

* A：10.0.0.0~10.255.255.255（相当于1个A类网络）
* B：172.16.0.0~172.31.255.255（相当于16个连续的B类网络）
* C：192.168.0.0~192.168.255.255（相当于256个连续的C类网络）

这些地址是不会被Internet分配的，因此在Internet上也不会被路由，它们常被称作“内网地址”或“私网地址”，虽然他们不能直接和Internet网连接，但仍旧可以被用来和Internet通讯，我们按需选择适当的类型，在内部网络中大胆地将这些地址当作公共IP一样使用，然后通过在边界路由上配置NAT转换映射成公网地址。

> IPv4由于只有32位，减掉D类和E类地址，最多只能分配四十亿个公网地址，IPv4地址早已枯竭，而通过NAT技术转换成私网地址使得IPv4能继续使用，同时由于私网地址不会在Internet上被路由，所以也保证了内网的安全性。

**特殊的IP地址**：

1. `0.0.0.0`通常用来表示缺省(默认)路由，就是所有不知道怎么路由的地址就用这个地址表示，而这条路由对应的网关就叫默认网关。

   在Linux中可以用`route`命令查看路由表

   ![Linux查看默认路由](http://img-blog.csdn.net/20170721144646956?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

   在Windows上可以用`route PRINT`命令查看路由表

   ![Windows默认路由](http://img-blog.csdn.net/20170721144712654?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2. `255.255.255.255`是限制广播地址。也就是对当前网段内(广播域)中的所有主机发送广播。

3. `127.0.0.1~127.255.255.254`用于表示本地主机`localhost`。这个地址数据包不会从网卡接口发送出去。

4. `169.254.0.1~169.254.255.254`如果你的Windows开启了DHCP功能来自动获取IP，系统没有找到任何DHCP服务器给它分配地址时，它会自动分配一个这样的地址作为IP，所以当一个机房都没有获取到DHCP的地址时，仍可以使用这个地址在机房内进行通信。

## IPv6

IPv6是下一代IP协议，用于解决IPv4地址枯竭的问题，目前IPv6的使用还在过渡阶段。IPv6地址长度为128bit，理论上有$3.4*10^{38}$地址，号称能为世界上每一粒沙分配ip地址(估计能用到人类灭亡)。

### 表示方法

128bit的IPv6长度是IPv4的4倍，明显IPv4的点分十进制表示方法不再适用。IPv6通常有一下几种表示方法：

1. 冒分十六进制表示法

   格式为`X:X:X:X:X:X:X:X`其中每个X表示16bit的16进制形式(一个16进制为4bit)。

   比如：`ABCD:EEFF:0011:2345:6789:ABCD:0011:EFEF`，这种表示方法每个冒号后的0可以省略，即`ABCD:EEFF:11:2345:6789:ABCD:11:EFEF`

2. 0位压缩表示法

   由于地址目前还比较多，所以中间可能会出现很多0，可以把连续的0写成`::`，但是`::`只能出现1次(不然就不知道你缩写了几个0)。比如：

   `EFEF:0:0:0:0:0:0:FEFE`可缩写成`EFEF::FEFE`

   `0:0:0:0:0:0:0:EF`可缩写成`::EF`

   `0:0:0:0:0:0:0:0`可缩写成`::`

3. 内嵌IPv4地址表示法

   为了方便IPv4地址转化为IPv6，后32bit可以使用点分十进制表示法。比如：

   `::FFFF:192.168.1.1`等价于`::FFFF:C0A8:0101`

> 目前IPv6技术还不成熟，所以也没有相关资料以供学习。IPv6的过渡应该仍有一段时间。

##IP地址+端口号=套接字

TCP和UDP均采用16bit的端口号来表示主机进程，而IP地址标识Internet上的主机，所以IP地址加上端口号能标识Internet主机上的任何一个网络进程。套接字英文“Socket”，直译过来就是“插座，插槽”，这个名字非常形象：“拿根USB线你就可以连接两个有USB插槽的设备了”。实现了TCP/IP协议的操作系统会提供一个socket API给程序开发人员，使用Socket API编程，操作系统会帮你将Socket套接字封装到TCP/UDP报文并发送出去。

**常见端口号**：

早期，有很多知名的程序使用Socket API已经帮你实现了很多常用的网络功能，逐渐地这些程序功能受到人们的认可，后来就成了规范，这些规范就是应用层协议，这些协议所使用的端口被记录在`/etc/services`文件中(Windows为`%windir%\System32\drivers\etc\services`)。

下面列举一些常用端口：

```powershell
##文件协议
ftp-data           20/tcp                           #FTP, data
ftp                21/tcp                           #FTP. control
##Secure Shell
ssh                22/tcp                           #SSH Remote Login Protocol
##Telnet
telnet             23/tcp
##邮件传输协议
smtp               25/tcp    mail                   #Simple Mail Transfer Protocol
##DNS域名服务
domain             53/tcp                           #Domain Name Server
domain             53/udp                           #Domain Name Server
##简单文件传输协议
tftp               69/udp                           #Trivial File Transfer
##HTTP超文本传输协议
http               80/tcp    www www-http           #World Wide Web
##HTTPS协议
https             443/tcp    MCom                   #HTTP over TLS/SSL
https             443/udp    MCom                   #HTTP over TLS/SSL
```

端口占16bit，所以范围在`0~65535`之间，其中`0~1024`是周知端口(众所周知的端口)，`1024~49151`是注册端口(大公司的应用使用，如果不冲突也可使用)，`49152~65535`是动态端口(随便使用，只要进程间不冲突就行)。

