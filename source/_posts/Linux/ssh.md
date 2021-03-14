---
title: ssh的各种玩法
date: 2021-02-21 12:54
categories: Linux运维
---

记得n年前刚接触Linux，用ssh远程登陆的时候有[配置过sshd进程的几个参数](https://blog.hufeifei.cn/2017/04/15/Linux/Linux-Secure/)

其实ssh的功能还是很强大的，可以玩出各种花儿来。

# 1. 使用公钥登陆服务器

使用公钥登陆，服务器端`sshd_config`需要配置如下：
```sh
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
```

客户端使用`ssh-keygen`生成一对公私钥，然后将公钥存到登陆用户家目录的`.ssh/authorized_keys`

这样客户端就能用公私钥登陆了，使用`ssh`客户端的`-i`选项可以指定私钥路径。
```sh
ssh -i /path/to/file.pem user@example.com
```
如果没有指定文件路径，默认会找客户端用户目录下的`~/.ssh/id_dsa`或`~/.ssh/id_rsa`。

> 需要注意的是，私钥的权限应该是`600`（当前用户可读可写）

使用`ssh-keygen`创建私钥时可能设置了密码，可以使用`ssh-agent`管理这些密钥

```sh
# 将密钥交给ssh-agent管理
ssh-add ~/.ssh/id_rsa
```
如果设置了密码，这个命令会让你输入密钥的密码，之后使用ssh登陆就不用输入密码了。

# 2. 换服务器端口

由于默认的22号端口众所周知，很容易收到网上的黑客流量的暴力攻击，可以在`sshd_config`中换掉端口：
```sh
Port 2222   #这个端口默认是22，改成不容易猜的
```
这时候客户端需要加上`-p`选项
```sh
ssh -p 2222 user@example.com
```
也可以使用ssh URI方式来指定端口
```sh
ssh ssh://user@example.com:2222
```
> 和`http://xxx`的80端口类似，`ssh://xxx`不指定端口默认就是22号端口

# 3. 远程执行命令

有时候，我们并不想进入远程服务器的终端中执行命令(比如在shell脚本中)。这时候我们可以在ssh最后跟上需要执行的命令，让远程服务器执行

```sh
ssh user@example.com "ps aux"
```

除此之外，我们还可以将命令的输出与客户端的命令进行交互

```sh
ssh user@example.com "ps aux" | less
```

甚至可以从远端拷贝文件夹到客户端
```sh
ssh user@example.com 'tar cz /usr/share/nginx/html/' | tar xzv
```
这里压缩是为了加快传输速度
```sh
ssh user@example.com 'tar -C /usr/share/nginx/html/ zcf - 404.html 50x.html' | tar zxf -
```
同样，我们可以使用ssh自己实现一个带压缩的scp的功能
```sh
tar czv src | ssh user@example.com 'tar xz'
```

所以前面提到在`.ssh/authorized_keys`中添加公钥，也可以用ssh远程执行
```sh
ssh user@example.com 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```
> 也可以用`ssh-copy-id <user>@<host>`实现上面的功能，当然如果是用其他端口或其他公钥，可以另外用参数指定：
> `ssh-copy-id -i ~/.ssh/otherkey -p 2222 username@host`

还可以远程执行一段脚本，无需拷贝脚本到远程服务器
```sh
ssh user@example.com bash < /path/to/local/script.sh
```

远程执行交互式命令
```sh
ssh user@example.com python
```
如果是执行python命令，这个命令不会立即返回，所以会直接阻塞，这个时候我们要加一个`-t`选项，让远程主机创建一个tty伪终端连接。
```sh
ssh user@example.com -t python
```

> 这个远程执行命令的功能，让ssh有了更多的想象空间

# 4. SSH Tunnel端口转发
有时候我们无法直接从host1连上host2，但是如果有一个host3在中间作为桥梁，host1能通过ssh连上host3，host3再将请求转发给host2，这时我们就在host1和host2建立了SSH Tunnel。

![SSH Tunnel](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAIKqhKIZ9LoZAJCyeKKZ9B4fDBidCp-FAoqz9LSZ8BounH32D44nZBh2SWgwk7HBV6CI7AkLoICrB0La10000)

**本地端口转发**

```sh
# 将host2的5000端口映射到本地的8080端口
ssh -L host1:8080:host2:5000 user@host3 -N 
```
其中host1就是本地主机网卡的ip地址，如果没有指定host1地址，默认会监听所有网卡。
```sh
ssh -L 8080:host2:5000 user@host3 -N
```

**远程端口转发**

上面的本地端口转发是针对host1的。相反，远程端口转换就是将host3的端口开放给host1这样的客户端，让host1的请求能转发到host2。

```sh
ssh -R 0.0.0.0:8080:host2:5000 user@host1 -N
```
这条命令执行在host3上，把host2的5000端口映射到host1的8080端口。

这个场景和前面本地端口转发的区别在于：前面的本地端口转发host1能连上host3，但如果host3是内网，host1无法访问，而host3能访问host1，这时就应该让host3来建立与host1的通道。

畅想一下这个功能的作用：

1. host3是部署在国外的http代理，host1因为被GFW墙了无法访问host2和host3，这时可以让国外的代理主动与host1建立通道，之后就可以顺畅的访问host2和host3了。
2. 你在本机开发了一个Web应用，给别人测试，但是你现在在内网，外网无法访问你的Web应用，一种方式是找台公网ip主机把Web应用部署上去。这样很麻烦，但是使用ssh远程转发，可以让在内网建立与外网的通道，外网就可以访问了。
...

> 注意：因为host3需要连接host1，所以host1应该要有sshd服务应用。然后host3的sshd要将`AllowTcpForwarding`选项打开。

**动态端口转发**

本地端口转发、远程端口转发都需要固定单一的端口。动态转发更牛逼的一点就是无需指定被访问目标主机的端口号。这个端口号需要在本地通过协议指定，该协议就是简单、安全、实用的 SOCKS 协议。

动态转发通过参数 -D 指定，格式：-D [本地主机:]本地主机端口。相对于前两个来说，动态转发无需再指定远程主机及其端口。它们由通过 SOCKS协议 连接到本地主机端口的那个主机。

举例：在host1上执行`ssh -D 50000 user@host3 -N`。这条命令创建了一个SOCKS代理，所以通过该SOCKS代理发出的数据包将经过host3转发出去。

怎么使用？

1. 用firefox浏览器，在浏览器里设置使用socks5代理127.0.0.1:50000，然后浏览器就可以访问host3所在网络内的任何IP了。

2. 如果是普通命令行应用，使用proxychains-ng，参考命令如下：
```sh
brew install proxychains-ng
vim /usr/local/etc/proxychains.conf   # 在ProxyList配置段下添加配置 "socks5   127.0.0.1 50000"
proxychains-ng wget http://host2      # 在其它命令行前添加proxychains-ng即可
```
3. 如果是ssh命令，则用以下命令使用socks5代理：
```sh
ssh -o ProxyCommand='/usr/bin/nc -X 5 -x 127.0.0.1:5000 %h %p' user@host2
```

> 这个功能想象空间更大:grin:

# 5. ssh客户端配置文件

前面有提到过[`sshd_config`](https://linux.die.net/man/5/sshd_config)服务端配置文件，同样地，还有一个[`ssh_config`](https://linux.die.net/man/5/ssh_config)客户端配置文件

用户的ssh客户端配置文件默认在`~/.ssh/config`路径下。

比如原来用`ssh -i /path/to/private_key -p 2222 user@example.com`远程登录的命令，我们只需要在`~/.ssh/config`文件中添加下面的配置就行
```
Host example.com
  User user
  Hostname example.com
  # Non standard port
  Port 2222
  IdentityFile /path/to/private_key
```
以后就可以直接用`ssh example.com`命令登录了，这样可以方便的管理多个密钥对，多个Host配置

而且Host可以设置通配符
```
Host *
  User user
  IdentityFile ~/.ssh/id_rsa
  ServerAliveInterval 15
  ServerAliveCountMax 3
  # 可以添加其他的配置
```
更多的配置选项可以通过`man ssh_config`命令查看手册

# 6. 穿越跳板机

![跳板机](https://p.pstatp.com/origin/pgc-image/8af6c15e283d43e0a4921d1f8033f750)

[跳板机](https://en.wikipedia.org/wiki/Bastion_host)在企业网络安全方面经常使用，为了避免多次`ssh`跳转多次，可以使用ssh的跳板机功能。

```sh
ssh -J <bastion-host> <remote-host>
或者
ssh -J user@<bastion:port> <user@remote:port>
```
如果跳板机有多层，可以直接用逗号分隔
```sh
ssh -J <bastion1>,<bastion2> <remote>
```

`-J`选项提供了灵活性，当然我们也可以直接在`.ssh/config`文件中进行配置
```
### The Bastion Host
Host bastion-host-nickname
  HostName bastion-hostname

### The Remote Host
Host remote-host-nickname
  HostName remote-hostname
  ProxyJump bastion-host-nickname
```
这样就可以直接使用`ssh remote-host-nickname`命令穿越跳板机登录远程主机了

# 6. 连接保活

TCP连接有[超时时间](https://tools.ietf.org/html/rfc5482)，防火墙也可以[配置空闲连接的超时关闭](https://www.google.com/search?q=firewall+timeout)。

所以经常看到我们ssh连上服务器，放着几分钟后，连接就断开了，这个时候就需要我们配置一下[连接保活](https://patrickmn.com/aside/how-to-keep-alive-ssh-sessions/)

```
Host *
  # 是否开启TCP保活，开启后会自动发送TCP保活消息
  # 默认yes
  TCPKeepAlive yes
  # 间隔多少秒发送一次保活消息
  # 默认为0，不发送
  ServerAliveInterval 60
  # 最多发送多少次保活消息
  # 默认为3，超过3次还是会断开连接
  ServerAliveCountMax 10
```

# 7. ssh多路复用

通常来说，[多路复用(Multiplexing)](https://en.wikipedia.org/wiki/Multiplexing)就是在单个连接上处理多个请求的功能。这和Java中的NIO解决的问题一样。

[SSH Multiplexing](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing)也是在单个TCP连接上处理多个SSH会话。

通过SSH远程执行命令，比如说下面的两个命令
```sh
ssh user@example.com "ps aux"
ssh user@example.com "ping baidu.com"
```
如果没有多路复用，每次执行ssh连接到远程主机都会重新创建一个tcp连接，很明显创建TCP连接是很耗时的。
> [Ansible](https://www.ansible.com/)的基础就是通过SSH远程执行命令，来实现自动化管理集群的

我们在`~/.ssh/config`文件中添加如下配置：

```
Host example.com
  User user
  Hostname example.com
  ControlMaster auto
  ControlPath ~/.ssh/socket/cm-%r@%h:%p
  ControlPersist 10m
```
上面就是一个标准的多路复用配置，里面有三个重要的配置参数：
* ControlMaster：用来打开多路复用，设置成`auto`，SSH客户端会尝试使用已存在的连接，如果不存在就创建一个。
* ControlPath：用来指定复用连接的套接字存储位置，`%r`表示登录的用户名，`%h`表示登录的主机名，`%p`表示远程端口号
* ControlPersist：表示这个socket连接保持多久

```sh
$ ssh example.com
Last login: Tue Feb 23 13:19:09 2021 from 122.224.245.226
user@example.com$ exit

Shared connection to example.com closed.
$ ll .ssh/socket
srw-------  1 holmofy  staff  0  2 23 13:22 -user@example.com:22
```
从登录以及推出的提示可以看到`Connection to example.com closed.`变成了`Shared connection to example.com closed.。`.ssh/socket`目录下也多了一个socket文件。

> socket文件就是unix哲学——一切皆文件。

另外还有命令可以检查和关闭socket连接
```sh
ssh -O check example.com # 检查socket有没有建立
ssh -O exit example.com  # 强制关闭socket连接
```

refs:

^ https://jeremyxu2010.github.io/2018/12/ssh的三种端口转发
^ http://daemon369.github.io/ssh/2015/03/21/using-ssh-config-file
^ https://blog.scottlowe.org/2015/12/11/using-ssh-multiplexing/
^ https://patrickmn.com/aside/how-to-keep-alive-ssh-sessions/
^ https://vqiu.cn/ssh-multiplexing/
^ https://www.redhat.com/sysadmin/ssh-proxy-bastion-proxyjump
^ https://zhuanlan.zhihu.com/p/74193910
