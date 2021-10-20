---
title: EffectiveJava读书笔记-  第2条：遇到构造器有多个参数时要考虑用构建者模式
date: 2018-02-13
categories: JAVA
---

# 遇到构造器有多个参数时要考虑用建造者模式

静态工厂方法和构造器都有一个局限性：

**当构造的对象有大量的可选参数时，你可能需要定义很多个静态工厂方法或者构造器**。



**用setter方法替代多参数构造器的几个缺点**

书中提到多参数的构造器的一个替代方法，就是用JavaBean模式：使用无参构造器创建对象，然后调用setter方法设置每个必要参数以及一些可选参数。

**1. 构造的过程中，对象可能处于不一致的状态**

**2. 因为有setter方法，所以阻止了这个对象成为不可变(unmodifiable)对象**



为了避免以上的这些问题，就需要我们**把构造对象的过程抽离出来**，这也是建造者模式(Builder Pattern)的设计初衷。

对于建造者模式，JDK中的例子都不是很典型，所以我就举OkHttp中的一两个例子。

> 事实上，涉及到Http协议的库都大量的使用到建造者模式，因为Http中的可配置的参数实在太多了。

从创建OkHttpClient对象开始：

```java
OkHttpClient httpClient = new OkHttpClient.Builder()
  .cache(cache) // 缓存响应数据
  .connectionPool(connectionPool) // 连接池
  .cookieJar(cookieJar) // Cookie存储策略
  .addInterceptor(interceptor) // 拦截器
  ... // 其他可选的配置项
  .build(); // 最后创建OkHttpClient对象
```

创建OkHttpClient时，可以指定很多可选配置，上面只是举了几个最常用的配置选项，其他你没有进行配置的选项都会使用默认的配置。创建OkHttpClient对象后你就不能再修改这个OkHttpClient对象的属性了：OkHttpClient中不提供这些属性的setter方法；对于一些集合属性(比如拦截器)，它都是用Collections.unmodifiableXxx进行了包装，防止调用者进行修改。

创建请求也是一个典型的构建者模式：

```java
Request request = new Request.Builder()
  .get()
  .url("https://www.baidu.com")
  .addHeader("Accept-Language", "zh-CN")
  .addHeader("User-Agent", "Chrome/63.0.3239.84")
  .build();
```

> 这里设置请求头，可以设置若干个，很明显比用构造器灵活的多。

