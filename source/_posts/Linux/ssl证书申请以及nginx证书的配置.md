---
layout: blog
title: ssl证书申请以及nginx证书的配置
date: 2018-02-05 14:49:23
tags:
categories: Linux运维
---

# 准备知识

* **SSL/TLS**：这两个分别是Secure Socket Layer(安全套接字层)，Transport Layer Security(传输层安全)的缩写。TLS是SSL的继承者，如果不是搞安全的专业人员，完全可以认为他们是一样的东西。

  > 关于这两者的差异可以参考https://kb.cnblogs.com/page/197396/


* key：因为SSL/TLS使用非对称加密算法进行认证，所以会有一对公钥、私钥。这里我们说的key通常指的是私钥。

  它长这个样子

  ![私钥](http://img.blog.csdn.net/20180205155616129?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* csr： Certificate Signing Request,即证书签名请求。这个文件是我们在申请数字证书的过程中使用[CSP(Cryptographic Service Provider,加密服务提供程序)](https://baike.baidu.com/item/CSP/10991199)生成私钥的时候一起生成的。

  它一般张这样：

  ![证书签名请求](http://img.blog.csdn.net/20180205155635667?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* crt：Certificate的缩写，这就是我们想要的证书。我们只要把CSR文件提交给[证书颁发机构(CA)](https://baike.baidu.com/item/CA/20721560)后，证书颁发机构使用其根证书私钥签名就生成了证书公钥文件，也就是颁发给用户的证书。

# 使用openssl自己颁发证书

> openssl的命令详细说明可以参考http://linux.51yip.com/search/openssl

**1.**生成私钥

```shell
openssl genrsa -out server.key -des3 -passout pass:123456 2048
```

> 生成2048位RSA私钥，并用DES3对称算法加密它，加密口令为123456，将生成的私钥输出到server.key文件中。

**2.**生成CSR请求

```shell
openssl req -new -key server.key -out server.csr
```

![生成CSR请求](http://img.blog.csdn.net/20180205155708122?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**3.**生成CA证书

```shell
openssl req -new -x509 -key server.key -out ca.crt -days 3650
```

> 证书的认证者总是CA或者是CA指定的第三方，这里我们只是生成一个CA证书自己玩玩。

**4.**用CA证书给自己颁发一个证书

```shell
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key -CAcreateserial -out server.crt
```

执行完成后server.crt就是我们要的证书了。

# 在Nginx中配置SSL证书

> [ngx_http_ssl_module](https://nginx.org/en/docs/http/ngx_http_ssl_module.html)模块的配置参考链接：https://nginx.org/en/docs/http/ngx_http_ssl_module.html

要在nginx中使用ssl需要在编译时安装ssl模块。可以用`nginx -V`命令检查当前nginx安装的模块

![查看Nginx安装的模块](http://img.blog.csdn.net/20180205155728971?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 如果没有安装过nginx，可以参考http://blog.csdn.net/holmofy/article/details/78639670

如果没有http_ssl_module，需要重新编译nginx源码。在执行`./configuration`的时候加上该模块

```shell
./configure --prefix=/usr/local/nginx --with-http_ssl_module
```

配置完成后用`make && make install`命令重新编译安装nginx。

安装完成后在`nginx.conf`文件中添加如下配置：

```nginx
server {
        listen 80;
        server_name  *.hufeifei.cn;
        # 告诉浏览器有效期内只准用 https 访问
        add_header Strict-Transport-Security max-age=15768000;
        # 永久重定向到 https 站点
        return 301 https://$server_name$request_uri;
    }

    # HTTPS server
    #
    server {
        listen       443 ssl;
        server_name  *.hufeifei.cn;

        # 证书文件路径
        ssl_certificate      /usr/local/nginx/ssl/server.crt;
        # 私钥文件路径
        ssl_certificate_key  /usr/local/nginx/ssl/server.key;
        ssl_password_file    /usr/local/nginx/ssl/passwd.pwd;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```

添加配置后，就可以启动了。

如果启动的时候有类似于下面的报错信息，请检查证书和私钥文件路径有没有写错。

![证书配置错误启动失败的情况](http://img.blog.csdn.net/20180205155759510?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

浏览器打开网站看看

![自己申请的证书，不受浏览器信任](http://img.blog.csdn.net/20180205155831257?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这是因为我们的证书是我们自己颁发的，浏览器不信任自己颁发的证书，如果要信任该证书需要在Internet选项中添加信任的证书。自己颁发的证书玩玩就行，实际上我们并不会用自己颁发的证书。

# 使用免费的SSL证书

我们要向获得受信任的证书，我们的csr必须得交给受信任的CA签名才行。要想获取这样的证书也很简单，国内有很多CA代理，不过一般需要请毛爷爷帮忙才行。

![CA代理商](http://img.blog.csdn.net/20180205155902945?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

用不起花钱的，也有免费的给你用：https://letsencrypt.org/

letsencrypt有很多第三方工具提供了向(sha)导(gua)式的证书生成工具，这里介绍一个[zerossl](https://zerossl.com/)

![ZeroSSL](http://img.blog.csdn.net/20180205155925766?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

开始制作免费证书：

![开始制作免费证书](http://img.blog.csdn.net/20180205155942312?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![生成证书签名请求](http://img.blog.csdn.net/20180205160000094?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在网站上添加相应的文件，验证网站是受你管理。

![验证网站的真实性](http://img.blog.csdn.net/20180205160024463?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![证书申请成功](http://img.blog.csdn.net/20180205160042124?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

证书申请成功，可以在nginx中配置证书了。

> 注意是配置最后生成的domain-crt.txt和domain-key.txt两个文件。而且这个证书的有效时间是90天，也就是说3个月后这个证书就过期了，到时候你可以用第一步生成的account-key.txt和domain-csr.txt再次申请。
>
> 除了这种方式手动申请，还有certbot提供的脚本自动申请：https://certbot.eff.org/，链接扔这了，你们自己玩吧。

