---
title: 使用Certbot申请Let'sEncrypt免费证书
date: 2018-08-12
tags:
categories: Linux运维
---

半年前[在自己的网站上配了个SSL证书](https://blog.csdn.net/holmofy/article/details/79261123)，当时是用[ZeroSSL](https://zerossl.com/)进行证书申请的。但是证书三个月就会过期，每次都去手动申请，着实让人头痛。后来到[Let's Encrypt官网](https://letsencrypt.org)看了下，在它提供的[ACME协议](https://letsencrypt.org/docs/acme-protocol-updates/)客户端列表中，最推荐使用[Certbot](https://certbot.eff.org/)。

![官方给出的ACME客户端列表](http://tva1.sinaimg.cn/large/bda5cd74gy1fr7vee3qb5j211y0k70us.jpg)

Certbot是有个最大的好处是，能自动化部署[Let's Encrypt](https://letsencrypt.org/)证书。

到[Certbot官网](https://certbot.eff.org/)，你可以根据自己的服务器操作系统以及使用的WebServer进行选择。

![](http://tva1.sinaimg.cn/large/bda5cd74gy1fr7vjvmtnjj211y0kgtab.jpg)

它会给出相应的certbot的安装和配置命令。连nginx都帮你配好！

![CentOS7下Nginx配置ssl证书](http://tva1.sinaimg.cn/large/bda5cd74gy1fr7vmwiw7ej211y0kgdhu.jpg)

##独立申请证书

certbot提供的全自动化的配置是挺不错的，但是个人总觉得全自动的隐藏过多细节，心里有点不踏实。

所以还是用独立申请(与使用哪个WebServer无关)。

```shell
certbot certonly --standalone --email 1938304905@qq.com -d www.hufeifei.cn -d blog.hufeifei.cn
```

![命令执行详情](http://tva1.sinaimg.cn/large/bda5cd74gy1fu7b1noepuj20oc0k4wg3.jpg)

生成证书和私钥后在nginx的配置文件中手动配置上面的两个文件，具体可以参考[nginx文档](https://nginx.org/en/docs/http/configuring_https_servers.html)

##过期重申

`certbot renew `命令能帮我们检查证书还有多少天过期，如果接近到期时间，会帮我们更新证书。

同时因为更新证书要用到80端口，所以在更新之前要将nginx停掉，certbot提供了两个钩子参数`--pre-hook`和`--post-hook`让我们执行命令前后关闭或开启nginx。

```shell
certbot renew --pre-hook "/usr/local/nginx/sbin/nginx -s stop" --post-hook "/usr/local/nginx/sbin/nginx"
```

正常没过期的时候执行这个命令：

![没过期的时候执行该命令](http://tva1.sinaimg.cn/large/bda5cd74gy1fu7bi6exc7j20qm06cwei.jpg)

##定期重申

为了省去每三个月都上服务器执行一次命令的繁琐，我们可以在corntab中加一个定时任务定期执行上面的`renew`命令：

每个月的1号执行一次重申命令。

![crontab](http://tva1.sinaimg.cn/large/bda5cd74gy1fu7bvwiz4jj20o906ggln.jpg)



参考：https://certbot.eff.org/docs/using.html#certbot-commands
