---
title: HTTP协议简史
date: 2021-06-20 18:50
categories: 网络
tags: 
- HTTP
- Network
---

我一直觉得，想深入学习某一项技术，必不可少的是了解这项技术的历史，因为知道历史，才能知道技术发展的原因。HTTP协议已经发展到2.0了，基于UDP的HTTP3.0也已经在议程中了。HTTP协议作为整个互联网的基础，已经经历了这么多版本的发展，所以这篇文章想聊一聊HTTP协议的历史。

# HTTP萌芽

1989年，在欧核研究组织工作的[伯纳斯·李博士](https://en.wikipedia.org/wiki/Tim_Berners-Lee)发明了[Mesh系统](https://www.w3.org/History/1989/proposal.html)。这个系统由四部分组成：
* 用来表示超文本文档的的格式：[HTML](https://developer.mozilla.org/en-US/docs/Web/HTML)，这个如今已由W3C组织发布到了HTML5标准了
* 用来交换超文本的简单协议：HTTP，这个是这篇文章的主角。
* 用来显示超文本文档的客户端：WorldWideWeb，这个慢慢演变成了我们电脑里的Chrome、FireFox、IE、Edge等浏览器。
* 一个提供可访问文档的服务器：httpd，这个就是大名鼎鼎的Apache服务器的前身。

这个时候的HTTP协议非常简单，后来被称为HTTP/0.9。因为请求只有一行，有时也被叫做单行协议。

# HTTP/0.9

最早的HTTP协议是没有版本号的，只要通过TCP能连接到服务器，协议、服务器、端口号都不需要。

```http
GET /path/to/page.html
```

响应体也极其简单，只包含响应文档。

```html
<HTML>
这是一个非常简单的HTML页面
</HTML>
```

HTTP0.9请求没有HTTP Header，也就意味着HTTP协议只能传输HTML文件，无法传输其他类型的文件。响应体也没有状态码或错误码，出了问题，也不知道具体是什么原因。

# HTTP/1.0

```http
GET /mypage.html HTTP/1.0
User-Agent: NCSA_Mosaic/2.0 (Windows 3.1)

200 OK
Date: Tue, 15 Nov 1994 08:12:31 GMT
Server: CERN/3.0 libwww/2.17
Content-Type: text/html
<HTML>
一个包含图片的页面
  <IMG SRC="/myimage.gif">
</HTML>
```



https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/Evolution_of_HTTP

https://hpbn.co/brief-history-of-http/

https://developers.google.com/web/fundamentals/performance/http2

https://hpbn.co/http2/
