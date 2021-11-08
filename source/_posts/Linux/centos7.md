---
title: 解决CentOS7种yum源版本过低的问题
date: 2021-02-13
categories: Linux运维
tags: Linux
---


最近准备在自己服务器上玩玩docker，把网站都用docker部署下，用ELK技术栈分析一下服务器上的日志。服务器是大学里搭的，版本是CentOS6，这次重装了系统升级到CentOS7，遇到的最头疼的问题就是装软件。因为自己在Mac上有homebrew，装啥软件都很爽，基本上软件都是最新的，但是CentOS非常保守，官方的软件库里软件版本都非常低，yum装了发现很多东西都用不了。比如tmux，官方仓库版本仍然是1.8，已经不支持[tmux-plugin](https://github.com/tmux-plugins/tpm)的功能了。所以这篇文章记录一下自己解决这个问题的过程。

##CentOS7配置官方软件仓库(yum软件源)

yum安装软件时，下载速度都非常慢，因为CentOS-Base.repo文件的baseurl连接的都是`mirror.centos.org`国外服务器。这个我不用担心了，因为云服务器(我用的是腾讯云)在重装系统时已经修改了yum软件源的连接。

```sh
[root@VM-235-40-centos ~]# cd /etc/yum.repos.d/
[root@VM-235-40-centos yum.repos.d]# ls
CentOS-Base.repo  CentOS-Epel.repo  CentOS-x86_64-kernel.repo
[root@VM-235-40-centos yum.repos.d]# head CentOS-Base.repo 
[extras]
gpgcheck=1
gpgkey=http://mirrors.tencentyun.com/centos/RPM-GPG-KEY-CentOS-7
enabled=1
baseurl=http://mirrors.tencentyun.com/centos/$releasever/extras/$basearch/
name=Qcloud centos extras - $basearch
[os]
gpgcheck=1
gpgkey=http://mirrors.tencentyun.com/centos/RPM-GPG-KEY-CentOS-7
enabled=1
[root@VM-235-40-centos yum.repos.d]# head CentOS-Epel.repo 
[epel]
name=EPEL for redhat/centos $releasever - $basearch
failovermethod=priority
gpgcheck=1
gpgkey=http://mirrors.tencentyun.com/epel/RPM-GPG-KEY-EPEL-7
enabled=1
baseurl=http://mirrors.tencentyun.com/epel/$releasever/$basearch/
[root@VM-235-40-centos yum.repos.d]# head CentOS-x86_64-kernel.repo
[centos-kernel]
name=CentOS LTS Kernels for $basearch
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=kernel&infra=$infra
#baseurl=http://mirror.centos.org/altarch/7/kernel/$basearch/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[centos-kernel-experimental]
name=CentOS Experimental Kernels for $basearch
```
但是软件版本非常低
```sh
[root@VM-235-40-centos yum.repos.d]# yum search tmux
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
=================================================== N/S matched: tmux ================================================================
tmux-top.x86_64 : Monitoring information for your tmux status line.
tmux-top-devel.x86_64 : Monitoring information for your tmux status line.
tmux.x86_64 : A terminal multiplexer
xpanes.noarch : Awesome tmux-based terminal divider

  名称和简介匹配 only，使用“search all”试试。
[root@VM-235-40-centos yum.repos.d]# yum info tmux.x86_64
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
可安装的软件包
名称    ：tmux
架构    ：x86_64
版本    ：1.8
发布    ：4.el7
大小    ：243 k
源    ：os/7/x86_64
简介    ： A terminal multiplexer
网址    ：http://sourceforge.net/projects/tmux
协议    ： ISC and BSD
描述    ： tmux is a "terminal multiplexer."  It enables a number of terminals (or
         : windows) to be accessed and controlled from a single terminal.  tmux is
         : intended to be a simple, modern, BSD-licensed alternative to programs such
         : as GNU Screen.
```
国内的[清华软件源](https://mirrors.tuna.tsinghua.edu.cn/)、[阿里软件源](https://developer.aliyun.com/mirror/)、[腾讯软件源](https://mirrors.cloud.tencent.com/)中也提供了[`centos`](https://mirrors.tuna.tsinghua.edu.cn/centos/)、[`epel`](https://mirrors.tuna.tsinghua.edu.cn/epel/)，都只是加快了国内的访问速度，并没有解决软件包少的问题。

##第三方软件仓库

最后在CentOS官网终于找到一个解决方案：https://wiki.centos.org/AdditionalResources/Repositories

CentOS列出了可用的软件源以及第三方软件仓库，除了[epel](https://fedoraproject.org/wiki/EPEL)外还有[`ELRepo`](http://elrepo.org/tiki/HomePage)、[`ius`](https://ius.io/)等十来个社区软件仓库。看了一下ius是最活跃的(官网颜值最高)，而且清华、阿里、腾讯都有ius的镜像同步。

根据ius提供的[安装文档](https://ius.io/setup)，将rpm包替换成腾讯镜像
```sh
yum install https://mirrors.cloud.tencent.com/ius/ius-release-el7.rpm 
```
然后再重新搜索一下tmux的软件
```sh
[root@VM-235-40-centos yum.repos.d]# yum search tmux
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
ius                                                                                                                                                        | 1.3 kB  00:00:00
ius/x86_64/primary                                                                                                                                         | 104 kB  00:00:01
ius                                                                                                                                                                       460/460
============================================================= N/S matched: tmux =================================================================
tmux-top.x86_64 : Monitoring information for your tmux status line.
tmux-top-devel.x86_64 : Monitoring information for your tmux status line.
tmux.x86_64 : A terminal multiplexer
tmux2.x86_64 : A terminal multiplexer
xpanes.noarch : Awesome tmux-based terminal divider

  名称和简介匹配 only，使用“search all”试试。
[root@VM-235-40-centos yum.repos.d]# yum info tmux2.x86_64
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
可安装的软件包
名称    ：tmux2
架构    ：x86_64
版本    ：2.9a
发布    ：2.el7.ius
大小    ：329 k
源    ：ius/x86_64
简介    ： A terminal multiplexer
网址    ：https://tmux.github.io/
协议    ： ISC and BSD
描述    ： tmux is a "terminal multiplexer."  It enables a number of terminals (or
         : windows) to be accessed and controlled from a single terminal.  tmux is
         : intended to be a simple, modern, BSD-licensed alternative to programs such
         : as GNU Screen.
```
可以看到这里有ius发布的tmux2.9的版本，还是相对比较新的一个版本

##将ius改为腾讯镜像

上面安装ius软件仓库后，可以看到`yum.repos.d`目录下已经有几个仓库了
```sh
[root@VM-235-40-centos yum.repos.d]# yum repolist
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
源标识                                                                      源名称                                                                                          状态
epel/7/x86_64                                                               EPEL for redhat/centos 7 - x86_64                                                               13,519
extras/7/x86_64                                                             Qcloud centos extras - x86_64                                                                      451
ius/x86_64                                                                  IUS for Enterprise Linux 7 - x86_64                                                                460
os/7/x86_64                                                                 Qcloud centos os - x86_64                                                                       10,072
updates/7/x86_64                                                            Qcloud centos updates - x86_64                                                                   1,640
repolist: 26,142
```
这时候ius.repo还是官方的baseurl，因为我的服务器是腾讯云上的，所以把ius.repo中的baseurl改为腾讯内网
```sh
vim /etc/yum.repos.d/ius.repo
```
修改后文件如下：
```conf
[ius]
name = IUS for Enterprise Linux 7 - $basearch
baseurl = http://mirrors.tencentyun.com/ius/7/$basearch/
enabled = 1
repo_gpgcheck = 0
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-IUS-7

[ius-debuginfo]
name = IUS for Enterprise Linux 7 - $basearch - Debug
baseurl = http://mirrors.tencentyun.com/ius/7/$basearch/debug/
enabled = 0
repo_gpgcheck = 0
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-IUS-7

[ius-source]
name = IUS for Enterprise Linux 7 - Source
baseurl = http://mirrors.tencentyun.com/ius/7/src/
enabled = 0
repo_gpgcheck = 0
gpgcheck = 1
gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-IUS-7
```
这里`http://mirrors.tencentyun.com`是内网访问地址

##同样的道理安装加速docker的安装

[docker官方文档](https://docs.docker.com/engine/install/centos/)中提到的需要安装docker-ce.repo仓库

```sh
$ sudo yum install -y yum-utils
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

同样的道理，为了快速安装docker(我腾讯云服务器的带宽只有1G，所以只能用腾讯云内网镜像来加速了)，需要修改`docker-ce.repo`的baseurl

```sh
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=http://mirrors.tencentyun.com/docker-ce/linux/centos/$releasever/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
```

将baseurl替换为腾讯云内网地址`http://mirrors.tencentyun.com/docker-ce`

接着是安装docker-compose，文档在此: https://docs.docker.com/compose/install/ ，这个简单的多，但是有了上面配置的腾讯云镜像，只需一句命令即可：

```sh
sudo yum install docker-compose.noarch
```
