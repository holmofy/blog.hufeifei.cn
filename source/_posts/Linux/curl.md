---
title: 一个有意思的命令行工具,速查表tldr vs. cht
date: 2021-03-7 18:50
categories: Linux运维
tags: 
- Linux
- curl
- cheat sheet
---


查公网ip可以用`curl ifconfig.me`，这个我们也可以在nginx上通过简短的指令实现：
```nginx
server {
    listen 80;
    server_name ip.hufeifei.cn;
    location / {
        default_type text/plain;
        return 200 "$remote_addr\n";
    }
}
```
`ifconfig.me`还提供了很多其他接口：
![ifconfig.me](https://p.pstatp.com/origin/pgc-image/a6ef7269003442f98680e514ab41ced1)

当然今天的主角不是这位，而是一个与之类似的工具`cht.sh`。

[cht.sh](https://github.com/chubin/cheat.sh)是[chubin](https://github.com/chubin)开发的一个"cheat sheet"服务。和[tldr](https://github.com/tldr-pages/tldr)类似提供了一个便捷的命令行速查表。除此之外它还支持各种语言的代码片段的搜索。

> [chubin](https://github.com/chubin)这位大佬还写了很多类似的服务，[wttr.in](https://github.com/chubin/wttr.in)用来查天气的，[rate.sx](https://github.com/chubin/rate.sx)用来查汇率的。除此之外还有人开发了[qrenco.de](https://github.com/fukuchi/libqrencode)进行二维码编码的，[transfer.sh](https://transfer.sh/)进行云存储的。大佬们真是让命令行工具大放异彩。

# 1、文档

任何命令行工具，第一个看的就是文档在哪儿，方便自己学习。

```sh
curl cht.sh/:help
```

# 2、查询命令行速查表

man手册内容很详细，但是对于一个新的命令行工具，我们就想尽快上手，所以`tldr`项目提供了一个简单的命令行速查表，tldr项目本意就是`too long don't read`，摘出一些常用的命令。

cht.sh也提供了这个功能，而且它把`tldr`的速查表页面也包含进来了。

使用方法是这样的`curl cht.sh/~keyword`，这个`~keyword`你可以替换成任何常用的命令，你见过的没见过的基本上大多数命令都能在这里找到。

```sh
$ curl cht.sh/ps
 cheat:ps
# To list every process on the system:
ps aux

# To list a process tree:
ps axjf

# To list every process owned by foouser:
ps -aufoouser

# To list every process with a user-defined format:
ps -eo pid,user,command

# Exclude grep from your grepped output of ps.
# Add [] to the first letter. Ex: sshd -> [s]shd
ps aux | grep '[h]ttpd'

 tldr:ps
# ps
# Information about running processes.

# List all running processes:
ps aux

# List all running processes including the full command string:
ps auxww

# Search for a process that matches a string:
ps aux | grep string

# List all processes of the current user in extra full format:
ps --user $(id -u) -F

# List all processes of the current user as a tree:
ps --user $(id -u) f

# Get the parent pid of a process:
ps -o ppid= -p pid

# Sort processes by memory consumption:
ps --sort size
```

# 3、语言代码片段

`cht.sh`相比于`tldr`，优势的一点是支持各种语言的代码片段搜索

比如你遇到项目里有一门新语言，想要快速学习一下它的语法和各种基础类库:
```sh
curl cht.sh/java/:learn | less
```
上面的一串命令，你就能拿到java的基础教程，从入门到精通。:stuck_out_tongue_winking_eye:

又比如你想看一下python里怎么查找文件。

```sh
curl cht.sh/python/find+file
```
它会把你能想到的所有方式都列出来。

比如你想知道怎么用ruby下载文件：
```sh
curl cht.sh/ruby/download+file | less
```
怎么用bash拷贝目录到另一台机器
```sh
curl cht.sh/bash/transfer+directory+remote | less
```
反正最常用的一些语言的代码片段都有，只要关键词明确都能找到对应的代码片段，如果输入一个不明确的关键词，或者想可以刁难`cht.sh`呢，它也会给一个相应的解决方案。
比如你想问怎么用c语言写网站，怎么用java写硬件驱动，怎么用ruby写游戏，怎么用js写多线程应用。它都会给你指引方向告诉你应该怎么去做，或者告诉你为什么这个语言不能做这件事儿。
```sh
curl cht.sh/c/website | less
curl cht.sh/java/hardware+driver | less
curl cht.sh/ruby/game | less
curl cht.sh/js/thread | less
```
> 我已经竭尽全力刁难它了，它都能给我很满意的答案。
> 这么“智能”的工具，让我一度以为后台是不是有“人工”，原来它只是将stackoverflow等几个平台的功能聚合到了一起 :satisfied:
> 也难怪这个项目在[github](https://github.com/chubin/cheat.sh)上有23k的star。
> 名副其实地实现了它在github上说的几个特点。

![cht.sh](https://p.pstatp.com/origin/pgc-image/82987ec453dc49009289c0e005192402)

# 4、安装客户端和命令行自动补全

`cht.sh`除了web接口，还支持命令行客户端

```
    curl https://cht.sh/:cht.sh | sudo tee /usr/local/bin/cht.sh
    chmod +x /usr/local/bin/cht.sh
```
使用客户端就可以用空格替代关键词之间的"+"了。
```sh
$ cht.sh go reverse a list
$ cht.sh python random list elements
$ cht.sh js parse json
```
[安装shell补全功能](https://github.com/chubin/cheat.sh#tab-completion)：
我用的是mac上的zsh：
```sh
$ curl https://cheat.sh/:zsh > ~/.zsh.d/_cht
$ echo 'fpath=(~/.zsh.d/ $fpath)' >> ~/.zshrc
$ # Open a new shell to load the plugin
```
