---
title: Windows上使用压缩包安装MYSQL
date: 2017-05-14
tags:
categories: Windows
description: Windows上使用压缩包安装MySQL
---

上个月在学校，好几个人过来叫我帮他们装MySQL，我说“你们可以到官网下个傻瓜式的安装向导，很简单，我这只有压缩包版的”，他们懒得下，我就用压缩包方式帮他们装。上次写过一篇在CentOS 7上安装MySQL的文章，索性把Windows上安装过程也写下来，对比一下。

## 下载Zip压缩包

傻瓜式安装方式我就不讲了，可以从[官网](https://dev.mysql.com/downloads/windows/installer/5.7.html)下载msi安装器。

MySQL压缩包可以从[https://dev.mysql.com/downloads/mysql/](https://dev.mysql.com/downloads/mysql/)下载，我下载的版本是5.7.15。

下载完安装包后把它解压到你想安装的目录，传统的MySQL服务器都会安装在`C:\mysql`目录下，如果你不安装在这个目录下，后续的配置步骤中需要安装目录。

## MySQL压缩包目录简单介绍

Windows上的安装唯一的好处就是软件的所有文件都比较集中，不像Linux上那么分散。

| 目录                                      | 目录内容                                     |
| :-------------------------------------- | ---------------------------------------- |
| `bin`                                   | [**mysqld**](https://dev.mysql.com/doc/refman/5.7/en/mysqld.html)服务器，客户端和实用工具程序(一些命令) |
| `%PROGRAMDATA%\MySQL\MySQL Server 5.7\` | 日志文件，数据库，Windows系统默认%PROGRAMDATA%变量为`C:\ProgramData` |
| `examples`                              | 示例程序和脚本，压缩版可能没有                          |
| `include`                               | MySQL提供的C语言头文件                           |
| `lib`                                   | MySQL提供的C语言库文件                           |
| `share`                                 | 其他支持文件，包括错误消息，字符集文件，示例配置文件，用于数据库安装的SQL脚本(安装过程中会创建系统数据库) |

##  配置MySQL启动信息

解压的文件夹下有一个`my-default.ini`文件，该文件是MySQL默认配置，我们把该文件拷贝一份并重命名为`my.ini`，修改`basedir`和`datadir`的配置

```ini
[mysqld]
# MySQL安装主目录
basedir=E:/mysql
# 数据库文件目录
datadir=E:/mydata/data
```

## 数据库初始化

从`5.7.6`版开始，Zip压缩包不再提供`data`目录，所以也不会有MySQL系统数据库的数据文件，我们需要使用`mysqld --initialize`或者`--initialize-insecure`对数据库进行初始化

* `--initialize`：初始化时会随机生成一个`root`用户密码，改密码会以日志的方式打印出来，如果控制台没有密码可能记录在日志文件下，在Windows上可以使用`--console`选项把信息打印到控制台。
* `--initialize-insecure`：不会生成随机密码，但是在登录服务器是可以使用`--skip-password`跳过密码输入：`mysql -u root --skip-password`。

## 修改数据库root用户密码

使用`mysql -u root -p`命令登录MySQL服务器后即可修改数据库密码

```shell
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password'
```

或

```
UPDATE USER SET password="******" where user="root"
```



**更多详细内容参考[MySQL参考文档](https://dev.mysql.com/doc/)**