---
title: Fess作为企业内网搜索引擎
date: 2025-10-13
tags:
categories: 网络
---

之前一直都是用[hexo-theme-webstack](https://github.com/HCLonely/hexo-theme-webstack/)做内网导航，这就像公司的门户网站一样，对内网所有的系统手动做索引。

<img width="1899" height="987" alt="image" src="https://github.com/user-attachments/assets/46258a8e-15d4-47f3-8b9a-353e5c931e96" />

但是手动做索引太慢了，而且维护成本高，自然而然地就有了从门户升级到搜索引擎的需求。目前看到最流行的就是[Fess](https://fess.codelibs.org/)这个企业级的搜索引擎。

[官方文档1](https://fess.codelibs.org/articles/1/document.html)中如何安装和配置搜索引擎爬虫的功能。

[官方文档7](https://fess.codelibs.org/articles/7/document.html)还提到了内网中需要带认证的系统如何爬取。

[具体可以看一下官方的教程](https://fess.codelibs.org/articles.html#list-of-published-articles)，在公司里部署一个这个搜索引擎，就可以对企业内部所有的文档做索引，方便员工进行信息检索。这就相当于公司内网的百度，可以指定爬取的网站必须是内网的域名或ip。
