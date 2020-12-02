# 1、单机部署

应用早期发展体量小，所有的应用都可以部署在一台机器上，由Nginx进行反向代理，将不同域名或者不同路径的请求路由到相应的应用。

![](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUAo2YrEBR9Iq2tAJCyeqRLJ24ujAaijukBAoqz9LL22y9GKghaK5ABzqZFpAc8HB0LTeFfenu82mrYhH22fbrgHc5kGawgNd9-BHPKHM9KH0LN6O5N6AAfgkRWSKlDIWEu50000)

```nginx
server {
    listen 80;
    server_name  app1.example.com;
    location / {
        proxy_pass http://localhost:8081/;
    }
}
server {
    listen 80;
    server_name  app2.example.com;
    location / {
        proxy_pass http://localhost:8082/;
    }
}
server {
    listen 80;
    server_name  app3.example.com;
    location / {
        proxy_pass http://localhost:8083/;
    }
    # 代理同一域名下不同路径的请求
    location /app4/ {  
        # 将app3.example.com/app4/下的请求代理给8084端口的应用
        proxy_pass http://localhost:8084/;
    }
}
```

# 2、独立部署

当业务发展更加壮大后，为避免各个应用间相互影响，需要对每个应用独立部署。

![](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuR8ABKujibBGBSfCpoZHjLC8ACglg0nEBIfBBUBYoijFILMeXb1AI39AG445XMY_zCoyYbYkMYwe2iU20eYy8Oe21LsWkT50ISDOgqGWgPTQaPXRa9EgbvoVYqMp4NRDHc3DHWGq6sFNR0pMR8og6GWTKlDIWA400000)

```nginx
server {
    listen 80;
    server_name  app1.example.com;
    location / {
        proxy_pass http://ip.app1:8080/;
    }
}
server {
    listen 80;
    server_name  app2.example.com;
    location / {
        proxy_pass http://ip.app2:8080/;
    }
}
server {
    listen 80;
    server_name  app3.example.com;
    location / {
        proxy_pass http://ip.app2:8083/;
    }
}
```

# 3、反向代理与正向代理

上面提到Nginx作为反向代理，那是不是也有正向代理呢？

![反向代理与正向代理](https://img.alicdn.com/tfs/TB1zpwopmR26e4jSZFEXXbwuXXa-640-231.png)

我们平时用的翻墙就属于正向代理：正常情况我们从国内网络(被笑称为世界上最大的局域网)无法访问谷歌，我们只需要在国外架设一台代理服务器（前提是这台代理服务器的ip没有被墙），让代理服务器代替我们访问谷歌。[SOCKS协议](https://www.ietf.org/rfc/rfc1928.txt)制定了客户端-代理服务器-目标服务器之间的工作原理。[Shadowsocks](https://shadowsocks.org/en/index.html)是最常用的代理服务器软件之一。

![SOCKS](https://img.alicdn.com/tfs/TB1kJpf17Y2gK0jSZFgXXc5OFXa-1452-520.png)

反向代理通常是面向内网的代理，用于保护内网的服务器。一个反向代理服务器通常还包含负载均衡、认证加密和缓存等功能。Nginx、Apache的Httpd以及HAProxy是目前最流行的反向代理服务器。

> 关于代理服务器器的认识可以参考：https://github.com/dariubs/awesome-proxy

# 4、集群部署与负载均衡

从摩尔定律到[阿达姆定律](https://en.wikipedia.org/wiki/Amdahl%27s_law)，单机的计算性能几乎已经被我们榨取干净了。当业务发展到更大的规模，需要突破单机限制，对应用采取集群式部署。而[负载均衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))就是指在这一组机器上分配任务的过程。

![](http://www.plantuml.com/plantuml/svg/bT2z2i8m4C3nFKyHkgTWktKeLLSHGPpKuKZFMiX7kQHIANrt8tR8K0Mp9F3ZduEhd0VolLQiE3caWWjgcxiN9J-G7Pv7f0SIOyRMoCVFbKIIZ8ppyxvfpN1b4xiHQEJhhjkgtdcaeE58gnHAtrhZthQQSSco7vR7Di_aGfJndmM4Tue3w4vXAgs-c9s5UWCAZCIbCelAxAyoZyhyUpcn2aqTDlm2)

可以根据[nginx文档](http://nginx.org/en/docs/http/load_balancing.html)配置相应的负载均衡算法和各个机器的负载权重可以配置

```nginx
upstream app1 {
  # app1应用下的ip或域名
  server srv1.app1.example.com;
  server srv2.app1.example.com;
}
upstream app2 {
  # app2应用下的ip或域名
  server srv1.app2.example.com;
  server srv2.app2.example.com;
}
upstream app3 {
  # app3应用下的ip或域名
  server srv1.app3.example.com;
  server srv2.app3.example.com;
}
server {
    listen 80;
    server_name  app1.example.com;
    location / {
        proxy_pass http://app1/;
    }
}
server {
    listen 80;
    server_name  app2.example.com;
    location / {
        proxy_pass http://app2/;
    }
}
server {
    listen 80;
    server_name  app3.example.com;
    location / {
        proxy_pass http://app3/;
    }
}
```

为了保证用户的数据安全，网站还需要配置HTTPS协议：

```nginx
upstream app1 {
  server srv1.app1.example.com;
  server srv2.app1.example.com;
}
upstream app2 {
  server srv1.app2.example.com;
  server srv2.app2.example.com;
}
upstream app3 {
  server srv1.app3.example.com;
  server srv2.app3.example.com;
}
server {
    listen 443;
    server_name  app1.example.com;
    location / {
        proxy_pass http://app1/;
    }
    ssl_certificate app1.example.com.crt;
    ssl_certificate_key app1.example.com.key;
}
server {
    listen 443;
    server_name  app2.example.com;
    location / {
        proxy_pass http://app2/;
    }
    ssl_certificate app2.example.com.crt;
    ssl_certificate_key app2.example.com.key;
}
server {
    listen 443;
    server_name  app3.example.com;
    location / {
        proxy_pass http://app3/;
    }
    ssl_certificate app3.example.com.crt;
    ssl_certificate_key app3.example.com.key;
}
```

# 5、阿里的统一接入

上面的网络架构还存在许多问题：

1. 代理服务器存在单点故障
2. 域名申请：每新建一个应用，注册一个三级域名，就需要在域名供应商后台添加一条解析记录。在配置文件中都要新添加一条新的`server`配置记录。
3. 证书申请：每添加一个域名，证书需要重新申请，而且证书若干年后会过期，那么多域名的证书不方便管理。
4. 负载均衡集群ip变更：集群中任意一台机器的ip变更后，需要修改upstream中的记录。

为了解决这么些问题，阿里开发了几个中间件：

* [AServer](http://gitlab.alibaba-inc.com/ali-devops/devops/wikis/aserver)：AServer是基于Nginx开发的 统一网关接入，为集团内部应用提供稳定、高效的网络接入，同时提供验签、限流、防刷、反外挂等辅助功能。目前有3700+AServer服务实例，双11大促使用机器一两千台，是基础网络服务。
	
	> 以前电商的两大主要流量入口分别是PC的Wagbridge和无线的Aserver，[2016年两个应用合并了，统称Aserver](https://app.aone.alibaba-inc.com/appcenter/app/detail?appId=39771)
* iDNS：用于DNS域名增减的工作流程管理平台
	
	> 
* KeyCenter：keyCenter是集团内部唯一的密钥管理系统,主要负责密钥的存储、使用、分发、更新等，并提供数据加密解密、签名验签API。为了避免密钥明文写在应用方程序或文件中，集团统一要求相关密钥均要保存在KeyCenter系统中，应用系统通过KeyCenter SDK产生相应支付宝签名。

* VIPServer：通过集中式的配置向客户提供路由信息，以非网关的形式实现负载均衡功能；支持多种映射策略（轮询、轮询+同机房、轮询+同网段）；通过健康探测机制，自动剔除不健康的机器，实现集群之间调用的透明化；对调用量、调用方等数据也有一定程度的反馈

![](https://img.alicdn.com/tfs/TB19a2V1QY2gK0jSZFgXXc5OFXa-1454-698.png)



Refs:
https://github.com/dariubs/awesome-proxy
https://www.atatech.org/articles/11296
https://www.atatech.org/articles/54341
https://www.atatech.org/articles/69725
https://www.atatech.org/articles/70365
https://www.atatech.org/articles/75956
https://www.atatech.org/articles/77671
https://www.atatech.org/articles/91046
https://www.atatech.org/articles/111481
https://www.atatech.org/articles/116078
https://www.atatech.org/articles/121230
https://www.atatech.org/articles/135529
https://www.atatech.org/articles/139064
https://www.atatech.org/articles/148882
https://www.atatech.org/articles/171468

https://www.atatech.org/articles/185810