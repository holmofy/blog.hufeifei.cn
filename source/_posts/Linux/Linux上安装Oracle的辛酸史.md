---
title: Linux上安装Oracle的辛酸史
date: 2017-08-25 14:49:23
tags:
categories: Linux运维
---


下个礼拜就要开始学习Oracle了，得嘞先在我的CentOS7上装一个(貌似听说Oracle装在Oracle Linux能得到更好的性能，不过懒得下Oracle Linux镜像，在CentOS7上装个试试先)。

# 创建oracle用户与相关用户组

为什么要把这部分作为第一步呢，主要是为了避免后面创建文件以及解压缩等一系列步骤中，要将文件所有者修改为oracle才能在安装过程中有足够的权限创建文件或子目录(Linux的权限既带来了安全，也带来了各种不便，稍一走神就忘了赋权限)。

```shell
[root@Holmofy ~]# groupadd oinstall
[root@Holmofy ~]# groupadd dba
[root@Holmofy ~]# useradd -g oinstall -G dba oracle
```
> 创建用户之后，可以使用`passwd oracle`命令对oracle用户的密码进行设置或修改。

如果你之前有掉坑的经历，已经添加过用户了，可以使用`id oracle`命令核查oracle用户是否配置完善：看Oracle是否属于`oinstall`和`dba`用户组。

![oracle用户](http://img.blog.csdn.net/20170827181607857?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

创建用户完成后，后面的工作大部分就用oracle用户去完成了，需要用到root权限再切换或者使用sudo命令(sudoers需要配置，这个不是本文的内容)。

# 下载安装包

软件包官网下载链接如下：

http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html

官方提供的文档链接如下：

http://www.oracle.com/technetwork/database/enterprise-edition/documentation/index.html

![官方链接](http://img.blog.csdn.net/20170827181903347?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**注意一定要选择平台相同的安装包**。如果你操作系统是64位的，下载的安装包是32位的，安装时会报交叉编译的错误信息：`/lib/ld-linux.so.2: bad ELF interpreter`。虽然有方法有解决方法，但是为了省去不必要的麻烦也为了程序的执行效率最好还是选择平台一致的安装包(走过的坑，你就不要再往下跳了＞︿＜)。

我这里选用的是x64的安装包：

![x86安装包](http://img.blog.csdn.net/20170827181834079?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用`unzip`命令将其解压(直接解压就行)，解压完成后会生成一个database文件夹：

![解压后](http://img.blog.csdn.net/20170827182208644?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

解压完成后有个doc目录，该目录下有Oracle安装以及管理的各种文档(不过是英文的,而且安装文档中没有CentOS的技术支持,不过有RHEL的也一样可以照着操作)：

![文档](http://img.blog.csdn.net/20170827182724654?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 检查硬件需求

毕竟Oracle是个大型软件，如果你的硬件给的不够，我劝你还是终止后面的步骤吧(铁定的安装失败)

## 1. 内存要求

* 至少1GB的RAM(小于1G的机器你还要它干啥)

  可以使用以下命令查看内存大小：

  ```shell
  # grep MemTotal /proc/meminfo
  ```

* 检查RAM与交换分区之间的要求：

| RAM          | 需要交换空间的大小  |
| ------------ | ---------- |
| 1 GB和2 GB之间  | RAM大小的1.5倍 |
| 2 GB和16 GB之间 | 等于RAM的大小   |
| 超过16 GB      | 16 GB      |

  使用一下命令查看交换分区大小：

  ```shell
  # grep SwapTotal /proc/meminfo
  ```

## 2. 硬盘要求

* 保证`/tmp`目录只要有1GB可用空间

  ```shell
  # df -h /tmp
  ```

* 确定可用硬盘空间满足以下要求：

| 安装类型 | 软件文件要求（GB） |
| ---- | ---------- |
| 企业版  | 3.95       |
| 标准版  | 3.88       |

| 安装类型 | 数据文件要求（GB） |
| ---- | ---------- |
| 企业版  | 1.7        |
| 标准版  | 1.5        |

  可使用以下命令你给查看你系统可用硬盘空间

  ```shell
  # df -h
  ```

# 检查软件需求

## 1. 操作系统要求

官方文档中说11g版本Oracle安装包支持以下操作Linux发行版：

- Asianux 2.0
- Asianux 3.0
- Oracle Enterprise Linux 4.0 Update 7 或更新版本
- Oracle Enterprise Linux 5.0
- Red Hat Enterprise Linux 4.0 Update 7 或更新版本
- Red Hat Enterprise Linux 5.0
- SUSE Linux Enterprise Server 10.0
- SUSE Linux Enterprise Server 11.0

CentOS应该和RHEL一样对待，所以说这里要求并没有那么严格

## 2. 软件包依赖

注意这一步是重点了，安装失败很大一部分原因是包依赖的问题没有解决。

官方文档中对于RHEL5及以上版本的Linux发行版，要求需要以下的软件包(更高版本也行)

```shell
binutils-2.17.50.0.6
compat-libstdc++-33-3.2.3
elfutils-libelf-0.125
elfutils-libelf-devel-0.125
elfutils-libelf-devel-static-0.125
gcc-4.1.2
gcc-c++-4.1.2
glibc-2.5-24
glibc-common-2.5
glibc-devel-2.5
glibc-headers-2.5
kernel-headers-2.6.18
ksh-20060214
libaio-0.3.106
libaio-devel-0.3.106
libgcc-4.1.2
libgomp-4.1.2
libstdc++-4.1.2
libstdc++-devel-4.1.2
make-3.81
sysstat-7.0.2
unixODBC-2.2.11
unixODBC-devel-2.2.11
```
你可以使用以下命令查看上面这些软件包的版本是否大于等于上面的要求：

```shell
# rpm -q binutils compat-libstdc++ elfutils-libelf elfutils-libelf-devel elfutils-libelf-devel-static gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers kernel-headers ksh libaio libaio-devel libgcc libgomp libstdc++ libstdc++-devel make sysstat unixODBC unixODBC-devel
```

如果符合都符合要求就没啥问题了，如果出现有未安装的软件包，比如我出现的这种情况：

```shell
[root@localhost ~]# rpm -q binutils compat-libstdc++ elfutils-libelf elfutils-libelf-devel elfutils-libelf-devel-static gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers kernel-headers ksh libaio libaio-devel libgcc libgomp libstdc++ libstdc++-devel make sysstat unixODBC unixODBC-devel
binutils-2.23.52.0.1-55.el7.x86_64
未安装软件包 compat-libstdc++
elfutils-libelf-0.163-3.el7.x86_64
未安装软件包 elfutils-libelf-devel
未安装软件包 elfutils-libelf-devel-static
gcc-4.8.5-4.el7.x86_64
gcc-c++-4.8.5-4.el7.x86_64
glibc-2.17-105.el7.x86_64
glibc-common-2.17-105.el7.x86_64
glibc-devel-2.17-105.el7.x86_64
glibc-headers-2.17-105.el7.x86_64
kernel-headers-3.10.0-327.el7.x86_64
未安装软件包 ksh
libaio-0.3.109-13.el7.x86_64
未安装软件包 libaio-devel
libgcc-4.8.5-4.el7.x86_64
libgomp-4.8.5-4.el7.x86_64
libstdc++-4.8.5-4.el7.x86_64
libstdc++-devel-4.8.5-4.el7.x86_64
make-3.82-21.el7.x86_64
sysstat-10.1.5-7.el7.x86_64
未安装软件包 unixODBC
未安装软件包 unixODBC-devel
```

我的建议是使用yum把这些软件包都更新一遍：

```shell
# yum install binutils compat-libstdc++ elfutils-libelf elfutils-libelf-devel elfutils-libelf-devel-static gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers kernel-headers ksh libaio libaio-devel libgcc libgomp libstdc++ libstdc++-devel make sysstat unixODBC unixODBC-devel
```

# 配置内核参数

> 以下命令都需要root用户权限执行

如果安装Oracle用于生产的话，内核参数是一个很重要的优化系统性能的配置项，比如配置信号量，I/O，共享内存等参数配置，这个建议参考官方文档进行详细配置，官方文档对这方面有很详细的说明。如果你和我一样只是安装个Oracle用来学习那只需要使用官方文档中建议的最低配置就行。具体可以在`/etc/sysctl.conf`文件中，添加以下内核参数：

```shell
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 536870912
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586
```

为使上述配置生效而不重启系统，执行如下命令

```shell
# /sbin/sysctl -p
```

# 为oracle用户添加shell配置

为了提高Oracle软件性能，需要为Oracle用户添加以下shell配置：

| Shell Limit  | 在limits.conf中的项 | 硬限制   |
| ------------ | --------------- | ----- |
| 打开文件描述符的最大数量 | `nofile`        | 65536 |
| 单个用户可用的最大进程数 | `nproc`         | 16384 |
| 进程堆栈段的最大大小   | `stack`         | 10240 |

步骤如下：

1. 在`/etc/security/limits.conf`文件，添加以下参数：

```shell
oracle           soft    nproc   2047
oracle           hard    nproc   16384
oracle           soft    nofile  1024
oracle           hard    nofile  65536
```

2. 在`/etc/pam.d/login`文件中添加一行：

```shell
session    required     pam_limits.so
```
3. 在`/etc/profile`文件添加以下脚本：

  ```shell
  if [ $USER = "oracle" ]; then
          if [ $SHELL = "/bin/ksh" ]; then
                ulimit -p 16384
                ulimit -n 65536
          else
                ulimit -u 16384 -n 65536
          fi
  fi
  ```
# 创建并配置环境变量

安装路径可以自选，我这里直接在根路径下创建了一个oracle目录，如果用于生产建议不要这么干，不方便以后的扩展。

> 注意权限

```shell
[root@localhost /]# mkdir /oracle/app
[root@localhost /]# chown -R oracle:oinstall /oracle
[root@localhost /]# chmod -R 775 /oracle
```
配置oracle用户环境变量：

1、使用`su - oracle`命令切换为oracle用户登录

2、使用任意文本编辑器打开Shell启动脚本，如：

```shell
vi .bash_profile
```

3、添加如下环境变量：

```shell
export ORACLE_BASE=/oracle
export ORACLE_HOME=/oracle/app
export ORACLE_SID=oracleSID
export PATH=$ORACLE_HOME/bin:$PATH
```
# 开始正式安装

进入到之前解压的安装包，运行`runInstaller`脚本开始安装

注意：一定要使用oracle用户登录图形界面，否则运行`runInstaller`会报错显示错误(因为后面使用图形化界面安装的，需要权限去运行图形界面程序)。

![开始安装](http://img.blog.csdn.net/20170827182808590?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如果你弹出了以下界面，那么恭喜你离成功只有一半的距离了。

1、第一步用来配置更新以及技术支持的，把勾去掉直接下一步就行

![第一步](http://img.blog.csdn.net/20170827182837997?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2、 配置安装选项，这三个选项分别表示：①创建并配置一个新数据库，适用于新安装数据库的用户；②只安装数据库软件，适用于已有Oracle数据库数据用于数据迁移的；③升级已有数据库，适用于将老数据库升级成新数据库的用户。毫无疑问这里选择第一个选项。

![第二步](http://img.blog.csdn.net/20170827182907151?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3、第三步配置桌面版还是服务器版，桌面版是最小化配置，这里为了练习选择服务器版的配置。

![第三步](http://img.blog.csdn.net/20170827182951095?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4、第四步分布式网格配置，这里选择单实例服务器配置，如果要配置分布式服务的话可以参考前面说的安装文档，里面有详细的分布式服务安装过程。

![第四步](http://img.blog.csdn.net/20170827183118931?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

5、这个我们为了达到练手的目的选择高级安装(典型安装基本都已经帮我们配置好了,有啥挑战性)

![第五步](http://img.blog.csdn.net/20170827183308431?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

6、选择语言，这里我选择英文和简体中文。

![第六步](http://img.blog.csdn.net/20170827183426003?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

7、第七步选择数据库版本，这里选择企业版

![第七步](http://img.blog.csdn.net/20170827183440676?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

8、选择安装路径，这个已经在环境变量里配置过了

![第八步](http://img.blog.csdn.net/20170827183524516?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

9、这个Inventory Directory目录用于记录Oracle的清单信息的，清单信息中包括Oracle的安装路径等信息。这里我选择在oracle的家目录下建一个目录存放这些安装信息。

![第九步](http://img.blog.csdn.net/20170827183706335?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

10、第十步用于创建数据库的类型：

* 通用/事务处理：专为一般用途或交互较多的应用程序而设计。
* 数据仓库：对数据存储应用程序进行优化。

这里选择通用的就行。

![第十步](http://img.blog.csdn.net/20170827183735559?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

11、配置数据库名，和Oracle服务ID号

> 注意数据库名一定要记住，以后进行程序开发会用到这个数据库名

![第十一步](http://img.blog.csdn.net/20170827183833837?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

12、十二步，这里需要配置一下字符集，将字符集设置成UTF-8，其他的不用修改(如果有特殊需求可以参考文档来配置)。

![第十二步](http://img.blog.csdn.net/20170827184038712?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

13、这一步用来配置系统信息邮件通知的，可以跳过

![第十三步](http://img.blog.csdn.net/20170827184417360?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

14、这一步用于配置数据存储的（数据文件存储位置），这里我们把数据存储在`/oracle/oradata`目录下。

![第十四步](http://img.blog.csdn.net/20170827184559723?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

15、十五步用于配置数据备份，这里我们只是用来学习不需要自动备份，实际生产肯定是要做备份的。

![第十五步](http://img.blog.csdn.net/20170827184751430?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

16、十六步配置账号密码，这里我将所有的默认用户统一使用相同的密码(如果密码太简单可能会报错，需要大小写数字都包含，需要精心设计一个密码)。

> 注意密码不能忘了，不管是数据库管理还是软件开发都会用到这个密码。

![第十六步](http://img.blog.csdn.net/20170827185113179?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

17、这一步用于配置数据库用户组的，只要安装前的配置工作完成了，这一步可以直接使用默认的。

![第十七步](http://img.blog.csdn.net/20170827185259468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

18、这一步会检测交换分区大小、内核参数以及依赖包是否安装。只要前面准备工作都完成了，下面的错误可以直接忽略(比如它要求的软件包，我们的版本实际上比它要求的还高，所以这里的报错没必要理会它)。

![第十八步](http://img.blog.csdn.net/20170827185343308?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

19、这一步是对前面所有配置的一个总结，我们可以直接点击完成

![第十九步](http://img.blog.csdn.net/20170827185707450?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

20、只要前面的配置没问题，我们就可以安心的等待安装成功了

![第二十步](http://img.blog.csdn.net/20170827185751058?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

21、安装完成后，弹出下面这个界面，我们点击password management对数据库用户的密码进行一些配置

![第二十一步](http://img.blog.csdn.net/20170827190115444?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

22、配置数据库用户密码

![第二十二步](http://img.blog.csdn.net/20170827190148663?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里叫你运行使用root用户运行两个脚本，运行一下就可以了。

![第二十二步](http://img.blog.csdn.net/20170827190302440?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![第二十二步](http://img.blog.csdn.net/20170827190332358?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

安装成功，此处应有掌声。

# 使用SQLplus查询scott表进行测试

![SQLplus](http://img.blog.csdn.net/20170827190402574?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 如果SQLplus命令找不到，注意看一下环境变量是否配置正确，然后将oracle注销后再重新登录

大功告成
