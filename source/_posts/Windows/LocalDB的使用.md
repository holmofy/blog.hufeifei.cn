---
title: LocalDB使用详解
date: 2017-08-14
tags:
categories: Windows
---

# LocalDB是什么

我们知道微软有一个SQL Server的免费版本[SQL Server Express](https://en.wikipedia.org/wiki/SQL_Server_Express)，它是作为学习以及构建桌面或小型服务器应用的入门级的免费数据库。但是作为编程人员，还是觉得体积过大。所以微软为开发者量身定制了一款专门用于编程开发的小数据库[SQL Server Express LocalDB](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-2016-express-localdb)(实际上就是从SQL Server Express中抽离出来的)。

# 下载和安装LocalDB

如果你使用Visual Studio开发，那么恭喜你Visual Studio从2012版本开始就自带了LocalDB。

比如我的VS2017中就带了LocalDB。

![Visual Studio](http://img-blog.csdn.net/20170909210158785?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如果你觉得Visual Studio体积太大了(吃了我好几个G)，不想安装这个巨无霸，那么可以从https://www.microsoft.com/en-us/sql-server/sql-server-downloads下载一个Express版本的下载器：!

![SQL Server Express LocalDB](http://img-blog.csdn.net/20170909211030541?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

选择LocalDB单独下载

![LocalDB](http://img-blog.csdn.net/20170909211129625?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

下载完成后只有一个44M的安装包(很小(⊙o⊙)哦)。

![SQLServer LocalDB](http://img-blog.csdn.net/20170909211230879?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

安装过程很简单这里就不演示了。

**注意：**

安装完成后，LocalDB就可以创建并打开数据库了，系统默认的数据库文件会存储在用户的AppData下，比如`C:\Users\Administrator\AppData\Local\Microsoft\Microsoft SQL Server Local DB\Instances`，该目录下是数据库实例，我们可以用SqlLocalDB命令行工具类管理数据库实例。

> sqllocaldb工具的参考文档：https://docs.microsoft.com/en-us/sql/tools/sqllocaldb-utility

# 使用sqllocaldb命令行工具管理数据库实例

命令行工具很好用，所有的文档都是中文的：

```shell
C:\>sqllocaldb /?
Microsoft (R) SQL Server Express LocalDB 命令行工具
版本 13.0.1601.5
版权所有(C) Microsoft Corporation。保留所有权利。

用法: SqlLocalDB 操作 [parameters...]

操作:

  -?
    打印此信息

  create|c ["instance name" [version-number] [-s]]
    使用指定的名称和版本创建新的 LocalDB 实例
    如果忽略 [version-number] 参数，则它默认为
    系统中安装的最新 LocalDB 版本。
    -s 创建后启动新的 LocalDB 实例

  delete|d ["instance name"]
    删除具有指定名称的 LocalDB 实例

  start|s ["instance name"]
    启动具有指定名称的 LocalDB 实例

  stop|p ["instance name" [-i|-k]]
    当前查询完成后，停止具有指定
    名称的 LocalDB 实例
    -i 使用 NOWAIT 选项请求关闭 LocalDB 实例
    -k 在不与之联系的情况下终止 LocalDB 实例进程

  share|h ["owner SID or account"] "专用名称" "共享名称"
    使用指定的共享名称共享指定的专用实例。
    如果省略了用户 SID 或帐户名称，它将默认为当前用户。

  unshare|u ["shared name"]
    停止共享指定的共享 LocalDB 实例。

  info|i
    列出当前用户所拥有的所有现有 LocalDB 实例
    以及所有共享的 LocalDB 实例。

  info|i "实例名称"
    打印有关指定的 LocalDB 实例的信息。

  versions|v
    列出在计算机上安装的所有 LocalDB 版本。

  trace|t on|off
    打开或关闭跟踪

SqlLocalDB 将空格作为分隔符处理。需要用引号将
包含空格和特殊字符的实例名称引起来。
例如:
   SqlLocalDB create "My LocalDB Instance"

如上所述，有时可以省略实例名称，或者
将其指定为 ""。在这种情况下，引用的是默认的 LocalDB
实例 "MSSQLLocalDB"。
```

## 创建数据库实例

```shell
C:\>sqllocaldb create MyLocalDB
已使用版本 13.1.4001.0 创建 LocalDB 实例“MyLocalDB”。
```

> 如果你电脑里装了多个版本的LocalDB那你还需要指定LocalDB的版本

命令执行完成后就会在`C:\Users\Administrator\AppData\Local\Microsoft\Microsoft SQL Server Local DB\Instances`创建一个文件夹保存该实例的数据库文件。

![MyLocalDB](http://img-blog.csdn.net/20170909211346669?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**启动并查看数据库实例信息**

```shell
C:\>sqllocaldb create MyLocalDB
已使用版本 13.1.4001.0 创建 LocalDB 实例“MyLocalDB”。

C:\>sqllocaldb start MyLocalDB
LocalDB 实例“MyLocalDB”已启动。

C:\>sqllocaldb info MyLocalDB
名称:               MyLocalDB
版本:            13.1.4001.0
共享名称:
所有者:              DESKTOP-DQJB4BI\Holmofy
自动创建:        否
状态:              正在运行
上次启动时间:    2017/9/9 17:23:40
实例管道名称: np:\\.\pipe\LOCALDB#BC6488BE\tsql\query
C:\>tasklist | find "sql"
sqlwriter.exe                15488 Services                 0      7,364 K
sqlservr.exe                 13680 Console                  1    170,684 K
```

> 这里的`sqlservr`进程就是LocalDB的后台进程，这个exe文件就在LocalDB的安装目录下：`C:\Program Files\Microsoft SQL Server\130\LocalDB\Binn`。

**共享数据库实例**

为了支持多个计算机用户连接到单个LocalDB实例，可以使用`sqllocaldb share`命令选择允许计算机上的其他用户连接到该实例，注意不是将实例共享给其他计算机(跟网络没啥关系)，所以这个功能很鸡肋。

## 连接LocalDB

**1、 客户端应用连接**

使用sqlcmd命令行(前身是osql)工具连接。

1、 安装：如果你使用VS，这个工具又可以不用安装了，VS自带(SQLServer Manager Studio中也自带了)。

如果你追求简洁，这里也有sqlcmd的下载地址：https://www.microsoft.com/en-us/download/details.aspx?id=53591，只有3M不到放心的下吧

不过这个软件还依赖于odbc驱动，下载链接：https://www.microsoft.com/en-us/download/details.aspx?id=53339，这个也不超过4M。

安装完成后可以使用了：

```
C:\>sqlcmd /?
Microsoft (R) SQL Server Command Line Tool
Version 13.1.811.168 NT
Copyright (c) 2015 Microsoft. All rights reserved.

usage: Sqlcmd            [-U login id]          [-P password]
  [-S server]            [-H hostname]          [-E trusted connection]
  [-N Encrypt Connection][-C Trust Server Certificate]
  [-d use database name] [-l login timeout]     [-t query timeout]
  [-h headers]           [-s colseparator]      [-w screen width]
  [-a packetsize]        [-e echo input]        [-I Enable Quoted Identifiers]
  [-c cmdend]            [-L[c] list servers[clean output]]
  [-q "cmdline query"]   [-Q "cmdline query" and exit]
  [-m errorlevel]        [-V severitylevel]     [-W remove trailing spaces]
  [-u unicode output]    [-r[0|1] msgs to stderr]
  [-i inputfile]         [-o outputfile]        [-z new password]
  [-f <codepage> | i:<codepage>[,o:<codepage>]] [-Z new password and exit]
  [-k[1|2] remove[replace] control characters]
  [-y variable length type display width]
  [-Y fixed length type display width]
  [-p[1] print statistics[colon format]]
  [-R use client regional setting]
  [-K application intent]
  [-M multisubnet failover]
  [-b On error batch abort]
  [-v var = "value"...]  [-A dedicated admin connection]
  [-X[1] disable commands, startup script, environment variables [and exit]]
  [-x disable variable substitution]
  [-j Print raw error messages]
  [-g enable column encryption]
  [-G use Azure Active Directory for authentication]
  [-? show syntax summary]
```

使用sqlcmd连接MyLocalDB:

```shell
C:\>sqllocaldb start MyLocalDB
LocalDB 实例“MyLocalDB”已启动。

C:\>sqlcmd -S (localdb)\MyLocalDB
1> :help
:!! [<command>]
  - Executes a command in the Windows command shell.
:connect server[\instance] [-l timeout] [-U user [-P password]]
  - Connects to a SQL Server instance.
:ed
  - Edits the current or last executed statement cache.
:error <dest>
  - Redirects error output to a file, stderr, or stdout.
:exit
  - Quits sqlcmd immediately.
:exit()
  - Execute statement cache; quit with no return value.
:exit(<query>)
  - Execute the specified query; returns numeric result.
go [<n>]
  - Executes the statement cache (n times).
:help
  - Shows this list of commands.
:list
  - Prints the content of the statement cache.
:listvar
  - Lists the set sqlcmd scripting variables.
:on error [exit|ignore]
  - Action for batch or sqlcmd command errors.
:out <filename>|stderr|stdout
  - Redirects query output to a file, stderr, or stdout.
:perftrace <filename>|stderr|stdout
  - Redirects timing output to a file, stderr, or stdout.
:quit
  - Quits sqlcmd immediately.
:r <filename>
  - Append file contents to the statement cache.
:reset
  - Discards the statement cache.
:serverlist
  - Lists local and SQL Servers on the network.
:setvar {variable}
  - Removes a sqlcmd scripting variable.
:setvar <variable> <value>
  - Sets a sqlcmd scripting variable.
```

sqlcmd中除了可以输入T-SQL语句，还有很多内建命令：

| **GO** [*count*]  | **:List**                    |
| ----------------- | ---------------------------- |
| [**:**] **RESET** | **:Error**                   |
| [**:**] **ED**    | **:Out**                     |
| [**:**] **!!**    | **:Perftrace**               |
| [**:**] **QUIT**  | **:Connect**                 |
| [**:**] **EXIT**  | **:On Error**                |
| **:r**            | **:Help**                    |
| **:ServerList**   | **:XML** [**ON** \| **OFF**] |
| **:Setvar**       | **:Listvar**                 |

其中除 GO 以外，所有 **sqlcmd** 命令必须以冒号 (:) 为前缀。

不过为了保持现有 **osql** 脚本的向后兼容性，有些命令会被视为不带冒号。 这由 [**:**] 指示。

>  更多命令的详细参考可以参考官方文档：https://docs.microsoft.com/zh-cn/sql/tools/sqlcmd-utility

**使用navicat连接**

如果你喜欢图形界面，可能会首先想到微软自己的推出的SQL Server Management Studio，SSMS集成了一大堆的工具，包括配置、监控、管理SQLServer实例等等(上面说的sqlcmd也包括在内)，但是八百多兆的安装包，你实际用到的功能少得可怜。个人推荐使用navicat premium，首先就是因为它可以连接各种数据库，不单单局限于SQLServer，navicat premium下载百度一下随处都有。不过SSMS下载链接我也扔这了，你自己选择，https://docs.microsoft.com/zh-cn/sql/ssms/download-sql-server-management-studio-ssms

不过使用navicat等图形界面客户端需要Native Client依赖库，不过在官网找了一下，发现[SQLServer 2016](https://www.microsoft.com/en-us/download/details.aspx?id=52676)等版本都是使用SQLServer2012的Native Client依赖库，所以下载时候不要因为版本不一样而有什么疑虑。sqlncli.msi依赖：https://www.microsoft.com/zh-cn/download/details.aspx?id=50402。

![Navicat](http://img-blog.csdn.net/20170909211440846?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 创建数据库

这部分就属于T-SQL语言的部分了，忘记了赶快翻大学课本回忆一下：

```sql
CREATE DATABASE [MyFirstDB]
ON PRIMARY
(
	NAME = 'MyFirstDB',
	FILENAME = 'C:\Users\Administrator\AppData\Local\Microsoft\Microsoft SQL Server Local DB\Instances\MyLocalDB\MyFirstDB.ndf',
	SIZE = 8192KB,
	MAXSIZE = UNLIMITED,
	FILEGROWTH = 65536KB
)
LOG ON
(
	NAME = 'MyFirstDB_log',
	FILENAME = 'C:\Users\Administrator\AppData\Local\Microsoft\Microsoft SQL Server Local DB\Instances\MyLocalDB\MyFirstDB_log.ldf',
	SIZE = 8192KB,
	MAXSIZE = UNLIMITED,
	FILEGROWTH = 65536KB
)
```

> 需要注意的是SQLServer在创建数据库的时候不能和model database有任何连接，不然就会创建数据库失败。model database是其他数据库创建的模板，这里有对model database的详细介绍：https://docs.microsoft.com/en-us/sql/relational-databases/databases/model-database



最后贴一个微软官方在Github上提供的各语言的SQLServer连接的Demo：

https://github.com/Azure/azure-sql-database-samples



> **参考链接 :**
>
> MSDN：
>
> * https://docs.microsoft.com/zh-cn/sql/sql-server/editions-and-components-of-sql-server-2016
> * https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-2016-express-localdb