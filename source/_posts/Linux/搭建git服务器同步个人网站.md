---
title: 搭建git服务器同步个人网站
date: 2017-06-15
tags:
categories: Linux运维
---


废话不多说，直接上正题。

## 装git

这里以我的CentOS 7为例，其他发行版可以查看git官网提供的命令https://git-scm.com/download/linux

```shell
yum install git
```

## 建git用户

创建git用户，是为了专门管理服务器上的git服务。在Linux最好每个用户管理某一个模块的功能，这样安全性更高。当然如果你不在意这些，你可以跳过这一步，在后面的步骤中你可以使用已存在的用户代替。

```shell
useradd git -s /usr/bin/git-shell -p 123Xyz
## p 指定密码，这个可以按你的想法设置
```

> -s 参数为了指定git用户使用的shell，这里使用git-shell。如果不指定的话默认是bash，别人可能会通过git用户来登录到你的服务器。
>
> -p 参数为了指定git用户的密码，不过待会儿我们会在服务器上为本地PC配置公钥，这样可以免得每次git同步都需要输密码。

## 服务器上创建裸仓库

你想在哪个目录下创建仓库，就在哪个目录下执行以下的命令。

在这里我现在刚创建的git用户的家目录下创建一个文件夹把它作为仓库的目录。

```shell
[root@VM_235_40_centos git]# pwd
/home/git
[root@VM_235_40_centos git]# mkdir www
[root@VM_235_40_centos git]# cd www
```

接下来我就在`/home/git/www/`目录下创建仓库

```shell
git init --bare html.git
## 里的html是我仓库的名字，这个可以任取
## -bare 是代表创建裸仓库，这个参数一定记得带上
```

## 服务器上为本地PC配置公钥

如果你的PC是Windows的，你可以使用git-bash执行下面的命令。

## 1. 为本地PC生成公钥密钥对

```shell
ssh-keygen
```

> ssh-keygen随 SSH 软件包提供，在Windows上git-bash会提供这个命令。

## 2. 查看生成的公钥和密钥

```shell
cd ~/.ssh
ls -al
```

```shell
## 果有以下两个文件，
## d_rsa代表使用rsa算法生成的密钥
## d_rsa.pub代表使用ras算法生成的公钥
id_rsa  id_rsa.pub
```

## 3.在服务器上配置公钥

在你要登录的用户的家目录下创建`.ssh/authorized_keys  `文件。

比如我要使用的是git用户，那我要创建一个`/home/git/.ssh/authorized_keys`文件

把上面公钥文件`id_rsa.pub`的内容拷贝到`authorized_keys`文件中。如果你要为多个PC配置公钥，那每个公钥分一行就行。

> 如果你和我一样服务器没有图形界面，可能不好拷贝，你可以使用下面这个简单的方法：
>
> ```shell
> echo 'ssh-rsa XXXXXXXXXXXXXXXXXX User@Host' > authorized_keys
> # 单引号中间是你id_rsa.pub文件的内容
> ```

## 4. 测试以上配置

在你的PC上使用git-bash输入以下命令，测试以上配置是否成功。

```shell
## ww.hufeifei.cn是我服务器的域名，你也可以使用ip地址
## home/git/www/html.git是前面创建的裸仓库的绝对路径
## est是你要从服务器clone到本地主机的目录
git clone git@www.hufeifei.cn:/home/git/www/html.git test
```

如果上面的命令执行后出现以下结果，说明前面的配置都OK，否则对照前面的步骤看看是否有遗漏或错误。

```shell
$ git clone git@www.hufeifei.cn:/home/git/www/html.git test
Cloning into 'test'...
warning: You appear to have cloned an empty repository.
```

> 在执行这个命令的目录下会出现test目录



## 动同步站点目录

现在你使用git推送任何网页文件，都没法儿看到效果，因为你只是把文件传到了git上，比如我的文件就会被传到`/home/git/www/`目录下，现在我们需要把这些文件同步到`www`服务器的目录上。

我的服务器使用的网站服务器软件是`apache httpd`，它的网页目录在`/var/www/html/`目录下(当然这个可以自己配置，我这里使用默认的)。

1. 进入裸仓库，我的仓库目录为`/home/git/www/html.git`

   ```shell
   [root@VM_235_40_centos www]# cd html.git
   [root@VM_235_40_centos html.git]# pwd
   /home/git/www/html.git
   [root@VM_235_40_centos html.git]# ls
   HEAD  branches  config  description  hooks  info  objects  refs
   ```

2. 配置git的钩子功能

   在`hooks/post-receive`文件中输入以下脚本（如果没有这个文件就创建一个）

   你可以使用vi编辑器输入以下内容：

   ```shell
   #!/bin/bash
   git --work-tree=/var/www/html checkout -f
   ```

   > `post-receive`就是git服务器在接收到客户端的push后要执行的脚本。
   >
   > `/var/www/html`就是我要同步的网站目录
   >
   > `git checkout -f `是强制检出git的意思

   既然是脚本，我们就比如让它可执行，赋予它权限：

   ```shell
   # 将post-receive文件赋给git用户，
   # 如果你使用的是其他用户，就赋给其他用户
   chown git:git post-receive
   # 将post-receive改为可执行文件
   chmod +x post-receive
   ```

3. 将网站目录赋给git用户

   现在你是无法直接将git同步到网站目录下的，因为你没有网站目录的权限，网站目录权限默认归`root`用户所有，我们现在要把权限赋给`git`用户(如果你前面使用的是其他用户，这里就用其他用户)

   ```shell
   chown -R git:git /var/www/html
   ```

   > -R 参数是递归的意思，就是讲目录下的所有目录全部赋给git用户。



## 下来你就可以往git服务器上同步网站了

```shell
## 本地PC网站目录下的所有文件添加到git中
git add .
## 文件提交到仓库，引号内是提交的相关信息
git commit -m 'message'
## 定远程git服务器地址，如果你使用刚刚clone的目录，你就可以跳过这一步
## it remote add origin git@www.hufeifei.cn:/home/git/www/html.git
## 本地的git推送到git服务器上，如果前面的公钥没配的话可能要输密码
git push -u origin master
## 看git当前状态
git status
```

## 限问题解决

如果期间出现`Permission denied`类似的错误，肯定是服务器目录的权限没有配置，毕竟Linux对权限限制的很死。

> 可以使用`chown`命令来修改目录的所有者，解决类似的问题
