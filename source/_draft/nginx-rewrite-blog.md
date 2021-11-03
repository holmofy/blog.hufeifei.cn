博客url迁移

最近把hexo博客的`permalink: :year/:month/:day/:title/`改成了`permalink: :year/:month/:title/`，但是seo收录的连接被下掉了。为了方便seo需要将原来的老链接重定向到新连接。

```nginx
server {
    # ...

    location ~ "^/\d{4}/\d{2}/\d{2}/.*" {
        rewrite "^/(\d{4})/(\d{2})/\d{2}/(.*)$" /$1/$2/$3 permanent;
    }
}
```



![](https://s.pc.qq.com/tousu/img/20211103/8316238_1635910328.jpg)