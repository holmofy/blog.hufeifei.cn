---
title: 在CentOS7上安装MySQL的心路历程
date: 2017-04-05
categories: Linux运维
tags:
	- Linux
description: 今天心血来潮，博客转移到云服务器上，首先得在CentOS 7上安装MySQL数据库。
---
今天突然心血来潮，想把博客转移到到自己的云服务器上，嗯，想法不错，正好能练练手（后来找到了别的解决方案Hexo，才发现自己有多幼稚，果然是脑子一热，啥事想的出来，但是在这个过程中也学到了一些东西）。CentOS上安装MySQL数据库，linux配mysql，哎哟，不错哦。（→_→在找到别的解决方案后，马上就被我给卸了）。

二话不说上yum安装大法
```shell
[root@VM_235_40_centos ~]# yum install mysql
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package mariadb.x86_64 1:5.5.52-1.el7 will be installed
--> Processing Dependency: mariadb-libs(x86-64) = 1:5.5.52-1.el7 for package: 1: mariadb-5.5.52-1.el7.x86_64
--> Running transaction check
---> Package mariadb-libs.x86_64 1:5.5.35-3.el7 will be updated
---> Package mariadb-libs.x86_64 1:5.5.52-1.el7 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package              Arch           Version                   Repository  Size
================================================================================
Installing:
 mariadb              x86_64         1:5.5.52-1.el7            os         8.7 M
Updating for dependencies:
 mariadb-libs         x86_64         1:5.5.52-1.el7            os         761 k

Transaction Summary
================================================================================
Install  1 Package
Upgrade             ( 1 Dependent package)

Total download size: 9.5 M
Is this ok [y/d/N]: y
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
(1/2): mariadb-libs-5.5.52-1.el7.x86_64.rpm                | 761 kB   00:00
(2/2): mariadb-5.5.52-1.el7.x86_64.rpm                     | 8.7 MB   00:00
--------------------------------------------------------------------------------
Total                                               16 MB/s | 9.5 MB  00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : 1:mariadb-libs-5.5.52-1.el7.x86_64                           1/3
  Installing : 1:mariadb-5.5.52-1.el7.x86_64                                2/3
  Cleanup    : 1:mariadb-libs-5.5.35-3.el7.x86_64                           3/3
  Verifying  : 1:mariadb-5.5.52-1.el7.x86_64                                1/3
  Verifying  : 1:mariadb-libs-5.5.52-1.el7.x86_64                           2/3
  Verifying  : 1:mariadb-libs-5.5.35-3.el7.x86_64                           3/3

Installed:
  mariadb.x86_64 1:5.5.52-1.el7

Dependency Updated:
  mariadb-libs.x86_64 1:5.5.52-1.el7

Complete!
```
不知道是不是当时太兴奋，也没看安装提示直接就填了yes，等安装完傻眼了，MariaDB是什么鬼。孤陋寡闻的我，赶忙去问度娘，原来是mysql的一个分支。
![MariaDB](http://img-blog.csdn.net/20170405220020021?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
我想要个“正房”，你却给我安了个“小三”；想养个海豚，你TM给我个海豹。
卧槽，我说这好歹也是IT三巨头之一啊，yum库竟然连个mysql都没有。

果断把“小三”给删了
```bash
[root@VM_235_40_centos ~]# yum erase mysql
Loaded plugins: fastestmirror, langpacks
Resolving Dependencies
--> Running transaction check
---> Package mariadb.x86_64 1:5.5.52-1.el7 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch            Version                    Repository    Size
================================================================================
Removing:
 mariadb          x86_64          1:5.5.52-1.el7             @os           48 M

Transaction Summary
================================================================================
Remove  1 Package

Installed size: 48 M
Is this ok [y/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Erasing    : 1:mariadb-5.5.52-1.el7.x86_64                                1/1
  Verifying  : 1:mariadb-5.5.52-1.el7.x86_64                                1/1

Removed:
  mariadb.x86_64 1:5.5.52-1.el7

Complete!
```
唉，自己动手丰衣足食，上官网逛逛去。
![mysql yum 安装](http://img-blog.csdn.net/20170405220219413?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)<br/>
![mysql yum安装向导](http://img-blog.csdn.net/20170405220318490?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
国外的东西就是好，东西给你了，连教程都准备好了，可惜了我这连四级都没过的英语水平啊，没办法硬着头皮也要看哪。

#看看官网给出的具体步骤：
## 1. 添加mysql yum 库
### a. 首先要到MySQL yum库的下载页面http://dev.mysql.com/downloads/repo/yum/
![mysql yum库下载](http://img-blog.csdn.net/20170405220446649?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### b. 找一个跟自己平台匹配的发行包，用``uname``命令看看自己的平台版本
![查看linux平台版本](http://img-blog.csdn.net/20170405220532165?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### c. 用``wget``命令把相应的rpm包下下来，这个包很小只有几k
```shell
	[root@VM_235_40_centos ~]# wget https://repo.mysql.com//mysql57-community-release-el7-9.ch.rpm
	--2017-04-05 16:32:26--  https://repo.mysql.com//mysql57-community-release-el7-9.noarch.rpm
	Resolving repo.mysql.com (repo.mysql.com)... 23.50.25.213
	Connecting to repo.mysql.com (repo.mysql.com)|23.50.25.213|:443... connected.
	HTTP request sent, awaiting response... 200 OK
	Length: 9224 (9.0K) [application/x-redhat-package-manager]
	Saving to: 'mysql57-community-release-el7-9.noarch.rpm'

	100%[============================================================>] 9,224       --.-K/s   in 0s

	2017-04-05 16:32:28 (75.1 MB/s) - 'mysql57-community-release-el7-9.noarch.rpm' saved [9224/9224]
```
### d. 把这个包安装上
```shell
	[root@VM_235_40_centos ~]# rpm -Uvh mysql57-community-release-el7-9.noarch.rpm
	warning: mysql57-community-release-el7-9.noarch.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5KEY
	Preparing...                          ################################# [100%]
	Updating / installing...
	   1:mysql57-community-release-el7-9  ################################# [100%]

	[root@VM_235_40_centos ~]cd /etc/yum.repos.d
	#安装完成后会发现/etc/yum.repos.d会多了两个mysql的repo文件
	[root@VM_235_40_centos yum.repos.d]# ls
	CentOS-Base.repo  mysql-community-source.repo
	CentOS-Epel.repo  mysql-community.repo
```
## 2. 看看文件里有啥东西
```shell
	[root@VM_235_40_centos yum.repos.d]# more mysql-community.repo
	[mysql-connectors-community]
	name=MySQL Connectors Community
	baseurl=http://repo.mysql.com/yum/mysql-connectors-community/el/7/$basearch/
	enabled=1
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	[mysql-tools-community]
	name=MySQL Tools Community
	baseurl=http://repo.mysql.com/yum/mysql-tools-community/el/7/$basearch/
	enabled=1
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	# Enable to use MySQL 5.5
	[mysql55-community]
	name=MySQL 5.5 Community Server
	baseurl=http://repo.mysql.com/yum/mysql-5.5-community/el/7/$basearch/
	enabled=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	# Enable to use MySQL 5.6
	[mysql56-community]
	name=MySQL 5.6 Community Server
	baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/7/$basearch/
	enabled=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	[mysql57-community]
	name=MySQL 5.7 Community Server
	baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
	enabled=1
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	[mysql80-community]
	name=MySQL 8.0 Community Server
	baseurl=http://repo.mysql.com/yum/mysql-8.0-community/el/7/$basearch/
	enabled=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

	[mysql-tools-preview]
	name=MySQL Tools Preview
	baseurl=http://repo.mysql.com/yum/mysql-tools-preview/el/7/$basearch/
	enabled=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```
** 按照官网的说法，在这个文件中你可以配置你想要安装的mysql版本，如果需要安装这个版本就设置``enabled=1``,不需要的版本就把它设置为0。这里我安装5.7的版本 **

## 3. 完事具备，接下来就可以来安装mysql了
```shell
[root@VM_235_40_centos yum.repos.d]# yum install mysql-community-server
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package mysql-community-server.x86_64 0:5.7.17-1.el6 will be installed
--> Processing Dependency: mysql-community-common(x86-64) = 5.7.17-1.el6 for package: mysql-communserver-5.7.17-1.el6.x86_64
......
---> Package mysql-community-libs.x86_64 0:5.7.17-1.el6 will be obsoleting
--> Finished Dependency Resolution
Error: Package: 2:postfix-2.10.1-6.el7.x86_64 (@anaconda)
           Requires: libmysqlclient.so.18()(64bit)
           Removing: 1:mariadb-libs-5.5.52-1.el7.x86_64 (@os)
               libmysqlclient.so.18()(64bit)
           Obsoleted By: mysql-community-libs-5.7.17-1.el6.x86_64 (mysql57-community)
              ~libmysqlclient.so.20()(64bit)
Error: Package: 2:postfix-2.10.1-6.el7.x86_64 (@anaconda)
           Requires: libmysqlclient.so.18(libmysqlclient_18)(64bit)
           Removing: 1:mariadb-libs-5.5.52-1.el7.x86_64 (@os)
               libmysqlclient.so.18(libmysqlclient_18)(64bit)
           Obsoleted By: mysql-community-libs-5.7.17-1.el6.x86_64 (mysql57-community)
               Not found
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```
纳尼，竟然安装出错，出什么鬼了。原来是mariadb没有删除干净，我的天，自己挖坑自己跳。
```shell
[root@VM_235_40_centos yum.repos.d]# rpm -qa|grep mariadb
mariadb-libs-5.5.52-1.el7.x86_64
[root@VM_235_40_centos yum.repos.d]# rpm -ev mariadb-libs-5.5.52-1.el7.x86_64 --nodeps
Preparing packages...
mariadb-libs-1:5.5.52-1.el7.x86_64
```
再来一遍
```shell
[root@VM_235_40_centos yum.repos.d]# yum install mysql-community-server
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package mysql-community-server.x86_64 0:5.7.17-1.el6 will be installed
......
---> Package mysql-community-libs.x86_64 0:5.7.17-1.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================================
 Package                       Arch          Version               Repository                Size
==================================================================================================
Installing:
 mysql-community-server        x86_64        5.7.17-1.el6          mysql57-community        151 M
Installing for dependencies:
 mysql-community-client        x86_64        5.7.17-1.el6          mysql57-community         23 M
 mysql-community-common        x86_64        5.7.17-1.el6          mysql57-community        328 k
 mysql-community-libs          x86_64        5.7.17-1.el6          mysql57-community        2.1 M
 numactl-libs                  x86_64        2.0.9-6.el7_2         os                        29 k

Transaction Summary
==================================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 177 M
Installed size: 879 M
Is this ok [y/d/N]: y
Downloading packages:
mysql-community-common-5.7.17- FAILED
http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-common-5.7.17-1.el6.x86_64.rpm: [Errno 14] HTTP Error 404 - Not Found
Trying other mirror.
mysql-community-client-5.7.17- FAILED
http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-client-5.7.17-1.el6.x86_64.rpm: [Errno 14] HTTP Error 404 - Not Found
Trying other mirror.
......
Error downloading packages:
  mysql-community-server-5.7.17-1.el6.x86_64: [Errno 256] No more mirrors to try.
  mysql-community-common-5.7.17-1.el6.x86_64: [Errno 256] No more mirrors to try.
  mysql-community-libs-5.7.17-1.el6.x86_64: [Errno 256] No more mirrors to try.
  mysql-community-client-5.7.17-1.el6.x86_64: [Errno 256] No more mirrors to try.
```
尼玛，又出错了，Linux上装软件就是蛋疼，404错误请求失败，这又是什么鬼，我前前后后有把安装过程捋了一遍，还是出错，不行，默默地问度娘去。
[这里](http://blog.csdn.net/zklth/article/details/6339662)有篇博客，里面说是yum缓存的问题，好吧清楚缓存。
```shell
[root@VM_235_40_centos yum.repos.d]# cd /var/cache
[root@VM_235_40_centos cache]# ls
httpd  ldconfig  man  yum
[root@VM_235_40_centos cache]# cd yum
[root@VM_235_40_centos yum]# ls
x86_64
[root@VM_235_40_centos yum]# rm -drf x86_64
[root@VM_235_40_centos yum]# ls
```
重新再来
```shell
[root@VM_235_40_centos yum]# yum install mysql-community-server
Loaded plugins: fastestmirror, langpacks
epel                                                                       | 4.3 kB  00:00:00
extras                                                                     | 3.4 kB  00:00:00
mysql-connectors-community                                                 | 2.5 kB  00:00:00
mysql-tools-community                                                      | 2.5 kB  00:00:00
mysql57-community                                                          | 2.5 kB  00:00:00
os                                                                         | 3.6 kB  00:00:00
updates                                                                    | 3.4 kB  00:00:00
(1/10): epel/7/x86_64/group_gz                                             | 170 kB  00:00:00
(2/10): epel/7/x86_64/updateinfo                                           | 762 kB  00:00:00
(3/10): epel/7/x86_64/primary_db                                           | 4.6 MB  00:00:00
(4/10): extras/7/x86_64/primary_db                                         | 139 kB  00:00:00
(5/10): os/7/x86_64/group_gz                                               | 155 kB  00:00:00
(6/10): updates/7/x86_64/primary_db                                        | 3.8 MB  00:00:00
(7/10): os/7/x86_64/primary_db                                             | 5.6 MB  00:00:00
(8/10): mysql-connectors-community/x86_64/primary_db                       |  13 kB  00:00:00
(9/10): mysql-tools-community/x86_64/primary_db                            |  32 kB  00:00:00
(10/10): mysql57-community/x86_64/primary_db                               |  96 kB  00:00:01
Determining fastest mirrors
Resolving Dependencies
--> Running transaction check
---> Package mysql-community-server.x86_64 0:5.7.17-1.el7 will be installed
......
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================================
 Package                       Arch          Version               Repository                Size
==================================================================================================
Installing:
 mysql-community-server        x86_64        5.7.17-1.el7          mysql57-community        162 M
Installing for dependencies:
 mysql-community-client        x86_64        5.7.17-1.el7          mysql57-community         24 M
 mysql-community-common        x86_64        5.7.17-1.el7          mysql57-community        271 k
 mysql-community-libs          x86_64        5.7.17-1.el7          mysql57-community        2.1 M
 numactl-libs                  x86_64        2.0.9-6.el7_2         os                        29 k

Transaction Summary
==================================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 188 M
Installed size: 847 M
Is this ok [y/d/N]: y
Downloading packages:
warning: /var/cache/yum/x86_64/7/mysql57-community/packages/mysql-community-common-5.7.17-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Public key for mysql-community-common-5.7.17-1.el7.x86_64.rpm is not installed
(1/5): mysql-community-common-5.7.17-1.el7.x86_64.rpm                      | 271 kB  00:00:03
(2/5): mysql-community-libs-5.7.17-1.el7.x86_64.rpm                        | 2.1 MB  00:00:52
(3/5): numactl-libs-2.0.9-6.el7_2.x86_64.rpm                               |  29 kB  00:00:00
(4/5): mysql-community-client-5.7. 6% [=-                       ] 101 kB/s |  12 MB  00:29:51 ETA
```
没问题了，只是我这服务器带宽不行，默默地等吧。
```shell
Installed:
  mysql-community-server.x86_64 0:5.7.17-1.el7

Dependency Installed:
  mysql-community-client.x86_64 0:5.7.17-1.el7
  mysql-community-common.x86_64 0:5.7.17-1.el7
  mysql-community-libs.x86_64 0:5.7.17-1.el7
  numactl-libs.x86_64 0:2.0.9-6.el7_2

Complete!
```
此处应有掌声
## 4. 最后一步了，启动服务
![启动服务](http://img-blog.csdn.net/20170405221046793?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
官网的安装指南使用``service mysqld start``，CentOS7改用systemctl命令启动服务了，使用service命令会默认转向调用systemctl命令的。
## 附加步骤，修改MySQL密码，修改MySQL默认端口
MySQL5.7之前的版本如果按照这种方式安装后，默认是没有密码的。对于MySQL5.7 有点特殊，下面是官网描述MySQL5.7的安装过程：
- 服务初始化
- 在data文件夹生成SSL证书和密钥
- 安装validate_password 插件并生效
- 创建数据库超级管理员'root@localhost'，并为他生成密码

也就是说MySQL5.7后生成了为root超级管理员生成了一个密码，这个密码在``/var/log/mysqld.log``文件中
```shell
shell> sudo grep 'temporary password' /var/log/mysqld.log
```
![mysql root 超级管理员密码](http://img-blog.csdn.net/20170405221202043?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
上面红圈部分就是生成的随即密码，然后进入mysql修改密码，当然修改密码的方式有很多种，这个可以自行百度。
![mysql root 密码修改](http://img-blog.csdn.net/20170405221234153?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

** 修改端口 **
mysql的配置文件在/etc/my.cnf
![修改MySQL默认端口](http://img-blog.csdn.net/20170406215317849?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
修改端口只需要添加一行``port=[修改的端口号]``，记得重启服务

搞了两节课终于TM折腾完了


## 令总结
** yum： ** Yellow dog Updater, Modified， 解决rpm包依赖的软件安装工具，可以在/etc/yum.config文件中进行yum的配置，yum数据源默认放在/etc/yum.repos.d/文件夹下。可以认为是rpm的加强版。
** rpm： ** RedHat Package Manager，软件安装包的管理器，可以用来安装或删除软件。
** wget： ** World Wide Web Get，wget是一个从网络上自动下载文件的自由工具，支持HTTP、HTTPS、FTP常见协议。
** uname： ** Unix Name，显示主机操作系统名称。
