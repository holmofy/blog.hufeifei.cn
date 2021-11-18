---
title: 由配置文件到Diamond配置中心引发的考究
date: 2020-04-15
categories: JAVA
tags: 
- Diamond
- Config
- JAVA
mathjax: true
keywords:
- Diamond
- 阿里配置中心
---

对于配置中心，我也不算是第一次接触。在我从上一家公司离职之际正好赶上项目进行容器化——将原来直接部署在ECS的应用迁移到K8S。而应用的配置都被迁移到了[Nacos](https://nacos.io/zh-cn/)上，所以我也算是经历过传统到前沿的过度阶段吧。入职阿里后，趁着了解Diamond来说说我对传统配置文件的形式和配置中心的形式的一些感受吧。

# 1、软件的老基友——配置文件

在软件开发过程中，经常会碰到各种配置文件。还记得大一用Win32写的魔塔游戏，用ASCII码自己手动编辑游戏的地图文件，到最后大三大四玩Java Web时碰到SSH、SSM等框架的各种配置文件，再到如今经常会碰到的日志配置和SpringBoot的`properties`配置文件等。

所以我对配置文件的作用大致总结是：

**对系统的功能预留出外部的控制线头，可以通过修改配置文件控制应用的部分行为**。

与配置文件相反对应的就是常被拿来吐槽的“硬编码”。配置文件解决的就是**修改应用的行为而不需要重新编译程序**。

但是每次改动配置后，需要重启应用，仍是一个特别烦心的操作，特别是服务集群机器比较多，手工一台台重启，真的会令人崩溃。好在上一家公司中，我所负责的产品只部署在两台Web上，一些配置需要调整还是能让人接受的。

# 2、将配置存在数据库中

在讨论注册中心之前，我觉的有必要提一下我之前负责的产品中提供的一种配置方式——数据库配置。在我上一家公司负责的系统中有一张`system_config`的表，存储了功能开关、功能阈值的等配置。这种配置方式的好处是应用不需要重启，所有需要用到配置的地方，临时访问数据库去读取配置。而对于轮询调度的后台worker为了降低频繁的读取配置导致的I/O消耗，我们提供了一个Scheduler低频率地从数据库更新配置到内存中，其他的调度器则使用这个内存中的配置。

基于这个数据库配置这个思想，当时我还实现了一个通用的服务调用开关。这也缘于去年双十一(2019年)，淘宝官方对isv严格的封网策略，为了能根据需要在双十一当晚临时对调度任务进行控制，同时又不能重启服务，所以在调度入口处做了个切面。

之所以提到数据库中存储配置，是因为看了[Diamond文档](http://mw.alibaba-inc.com/products/diamondserver/_book/programer-magazine-configserver.html)的一些介绍，我觉得Diamond和这个有点类似。只是集团把这个功能抽离成了一个中间件。我猜测[diamond-server](http://gitlab.alibaba-inc.com/middleware-diamond/diamond-server)就是个提供配置表增删改查的Http服务。

> 看过里面ConfigServlet正好与client-sdk中的ClientWorker.getServerConfig相对应，验证了我的猜想，而配置表就是domain中的ConfigInfo。

@DiamondListener所谓的推送式配置也只是客户端定时进行http长轮询实现的。

> 可以参看client-sdk中的ClientWorker#LongPullingRunnable长轮训任务怎么实现的。

# 3、配置中心解决了什么问题

关于这个问题我摘录了[坤宇](https://www.atatech.org/users/150359)前辈在[他的文章](https://www.atatech.org/articles/58676)中提到的几段话。

> 在集中式开发时代，配置文件基本足够用了，因为那时配置的管理通常不会成为一个很大的问题，简单一点来说，系统上了生产之后，如果需要修改一个配置，登录到这台生产机器上，vi修改这个配置文件，然后reload一下并不是什么很大的负担。
>
> ...
>
> 在分布式系统中，一次构建、发布、上线是非常非常重的一个过程，它不像单机时代那样重启一台机器、一个进程就可以了，在分布式系统中，它涉及到将软件包(例如war)分发到可能超过几千台机器，然后将几千台机器上的应用进程一一重启这么一个过程，超过2000台机器的一个应用一次完整的发布过程需要多长时间，相信很多核心系统的小二都深有体会。

总结来说：在分布式系统中，为了避免修改配置后需要手动重启集群中的所有机器，需要将应用集群中的配置集中到一个中间服务中，这样的一个中间件就是“配置中心”。另外一个更优秀的配置中心还应该有一个齐全的权限控制功能。

> 关于配置中心的权限控制这个问题，我还是深有体会。因为上一家公司使用是配置文件，配置文件就和线上应用放在一起，而MySQL、Redis等数据库的连接配置包括密码都明文存在配置文件中，同时线上服务是有数据库的写权限的。这也就以为着有服务器权限的开发人员都可以操作线上数据库的内容，这个问题细思极恐。而如果配置中心能够做到对配置文件的访问进行控制，这个问题就迎刃而解了。

# 4、阿里的配置中心Diamond

作为一个比较晚入职的新员工，在我入职时，市面上已有的配置中心其实已经很多了。比如阿里开源的[Nacos](https://github.com/alibaba/nacos)，携程的[Apollo](https://github.com/ctripcorp/apollo)，Spring的[Spring Cloud Config](https://github.com/spring-cloud/spring-cloud-config)。而Diamond是阿里内部使用的配置中心中间件，经历了[很多年双十一的考验](https://www.atatech.org/search?q=Diamond%E5%8F%8C%E5%8D%81%E4%B8%80%E6%80%BB%E7%BB%93)。下面会简单记录一下我这两天接触Diamond的源码以及相应的一些理解。

### 4.1、对Environment的扩展——EnvironmentPostProcessor

和Spring Framework中的`BeanPostProcessor`一样，SpringBoot提供了一个`EnvironmentPostProcessor`，可以`Environment`进行扩展。具体可以参考`ConfigFileApplicationListener`这块代码，里面会回调所有的`EnvironmentPostProcessor`。

### 4.2、将Diamond配置加载到`Environment`中

Diamond所说的配置持久化，就是把配置存在数据库的表中，Diamond-Server实际上是一个对配置进行增删改查的一个Http服务。我们业务部门使用diamond-client-sdk调用http服务读取Diamond-Server中的配置，然后注册到应用的Environment中。为了高可用Diamond-Server通常是集群部署，`ServerListManager`负责获取Diamond-Server的地址，默认从`http://jmenv.tbsite.net:8080/diamond-server/diamond`获取。

Diamond有个`DiamondEnvironmentPostProcessor`就是实现了`EnvironmentPostProcessor`，从Diamond-server中读取配置后更新到应用的`Environment`对象中。这样我们就可以像平常一样用Spring的`@Value`注解或SpringBoot的`@ConfigurationProperties`注解了。

> 关于@Value和@ConfigurationProperties注解的内容可以参看[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config)

```java
@Component
public class DiamondDemo {

    /**
     * 这里和平常一样可以用@Value注解获取.properties文件中的配置
     * 而Diamond会从Diamond-Server配置中心获取配置并合并到环境的配置中。
     */
    @Value("${string}")
    private String stringValue;

    @Value("${number}")
    private int intValue;

    @Value("${boolean}")
    private boolean booleanValue;

    public String getStringValue() {
        return stringValue;
    }

    public int getIntValue() {
        return intValue;
    }

    public boolean getBooleanValue() {
        return booleanValue;
    }
}
```

下面是Diamond获取配置的时序图(可以对着源码看一下)：

![](http://www.plantuml.com/plantuml/svg/fPBDIaCn48NtVOeiAzo-W0kf-0C55uNY0uIRyJAGp8J9YUZR6strHg-8ucv2JhvlE6HRu0qrnTMLoWDFjnpfCkV8emUht7412PdRR2xSDVka4cxaaKqbaM2l1NlJaKfHS-Skp-SkjQPv7fpF-MprC-hNkkDy2hQRJ0QcyZ_XFJYMHlVX7Vbyq6eZhOE7tuN1JQOrlmwZfgo5GOE3LMgQ7j4n6suY72jUOi29jEBZ-NVR760iKpCAIF15j8n_tjn5zU5a_tiNRrKiIju9jFifZGvwGiDlIK9DyGK0)

### 4.3、推送式配置——监听Diamond中配置的修改

可以看到Diamond中有个DiamondAutoConfiguration类实现了`BeanPostProcessor`接口，里面会将所有带`@DiamondListener`注解且实现了`DiamondDataCallback`接口的Spring Bean注册到Diamond中。而`ClientWorker`会创建后台任务定时去检查Diamond-Server的配置是否有更新，如有更新会回调之前注册的DiamondListener。

![Diamond推送式配置](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuNB9JCpDpqjnB2t9TyxFIyjCBorABCdCprFGrRLJm2bffL2KcfvPN99Q15NY0-BafHOLQoIb9kPf4cKiq9J45BXEZPJ4aaJF51s5zAByqW8G8mSgeydba9gN0dGi0000)

推送形式的配置示例：

```java
package com.alibaba.aliexpress.luofei.diamond;

import java.io.ByteArrayInputStream;
import java.util.Properties;

import com.alibaba.boot.diamond.annotation.DiamondListener;
import com.alibaba.boot.diamond.listener.DiamondDataCallback;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.bind.BindResult;
import org.springframework.boot.context.properties.bind.Bindable;
import org.springframework.boot.context.properties.bind.Binder;
import org.springframework.boot.context.properties.bind.PropertySourcesPlaceholdersResolver;
import org.springframework.boot.context.properties.source.ConfigurationPropertySources;
import org.springframework.core.env.Environment;
import org.springframework.core.env.PropertiesPropertySource;

/**
 * 通过 @DiamondListener注解，监听相关的配置项
 */
@DiamondListener(dataId = "com.taobao.middleware:configFromListener.properties")
public class DiamondDataCallbackDemo implements DiamondDataCallback {

    @Autowired
    private ConfigBean configBean;

    @Autowired
    private Environment environment;

    private String dataCahe;

    public String getReceivedData() {
        return dataCahe;
    }

    @Override
    public void received(String data) {
        try {
            dataCahe = data;
            Properties properties = new Properties();
            properties.load(new ByteArrayInputStream(data.getBytes()));
            System.out.println("received from diamond listener: " + properties);

            // 把properties的值注入到ConfigBean里

            Bindable<ConfigBean> bindable = Bindable.ofInstance(configBean);
            Binder binder = new Binder(
                ConfigurationPropertySources.from(new PropertiesPropertySource("diamond-demo", properties)),
                new PropertySourcesPlaceholdersResolver(this.environment));
            BindResult<ConfigBean> result = binder.bind("", bindable);
            if (!result.isBound()) {
                System.out.println("Bind the data to configBean fail. Properties:" + properties);
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



参考链接：

一篇好TM长的关于配置中心的文章：https://www.atatech.org/articles/58676

软负载&配置中心-Diamond 2017双十一总结：https://www.atatech.org/articles/93883

软负载领域和实践谈：https://www.atatech.org/articles/79325

Diamond文档：http://mw.alibaba-inc.com/products/diamondserver/_book/
