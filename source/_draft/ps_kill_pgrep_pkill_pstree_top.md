操作系统进程工具

* 查看进程状态：ps, pgrep, pstree
* 发送进程信号：kill, pkill, killall, nice
* 内存状态：vmstat
* 实时进程管理：top, htop, glances

# ps

[`ps`](https://en.wikipedia.org/wiki/Ps_(Unix))是**P**rogress **S**tatus的缩写，用来查看当前的进程状态。

在Linux下的`ps`命令支持三种不同风格的选项：

* UNIX/Posix options，以单划线`-`开头，可以组合多个选项
* BSD options，不用单划线开头，可以组合多个选项
* GNU options，双划线`--`开头，长参数

而Mac操作系统内核[Darwin](https://en.wikipedia.org/wiki/Darwin_%28operating_system%29)衍生自BSD系列，所以在MacOS上不支持`-`单划线开头的参数。

列出所有运行中的进程：

```sh
ps aux
```

其中各参数含义：

* `a`：显示你和其他用户的进程，跳过没有命令行终端的进程。
* `u`：显示这些信息`user`, `pid`, `%cpu`,`%mem`, `vsz`, `rss`,`tt`, `state`, `start`, `time`, `command`.
* `x`：显示没有命令行终端的进程，如守护进程。

当使用`ps ux`将会只显示当前用户的所有进程，而不会显示其他用户的进程。

输出格式参数除了`-u`还有`-l`、`-j`、`-v`，这些选项也可以进行组合。

`-u`输出格式如下：

```sh
USER               PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root               135   1.5  0.0  4330012   2552   ??  Ss    7 521    9:35.10 /usr/sbin/notifyd
holmofy          87921   1.1  0.1  4303028   8580 s001  Ss    3:57下午   0:04.43 -zsh
```

`-l`输出格式如下：

```sh
  UID   PID  PPID CPU PRI NI      VSZ    RSS WCHAN  STAT   TT       TIME COMMAND
    0     1     0   0  37  0  4341788  13280 -      Ss     ??   96:36.96 /sbin/launchd
    0    70     1   0   4  0  4330648    760 -      Ss     ??    1:48.57 /usr/sbin/syslogd
```

`-j`输出格式如下：

```sh
USER               PID  PPID  PGID   SESS JOBC STAT   TT       TIME COMMAND
root                 1     0     1      0    0 Ss     ??   96:42.55 /sbin/launchd
root                70     1    70      0    0 Ss     ??    1:48.58 /usr/sbin/syslogd
```

`-v`输出格式如下：

```sh
  PID STAT      TIME  SL  RE PAGEIN      VSZ    RSS   LIM     TSIZ  %CPU %MEM COMMAND
    1 Ss    96:43.05   0   0      0  4341788  13296     -        0   1.8  0.1 /sbin/launchd
   70 Ss     1:48.59   0   0      0  4330648    748     -        0   0.0  0.0 /usr/sbin/syslogd
```

