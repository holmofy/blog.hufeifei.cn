---
title: Struts2通配符和使用上的坑
date: 2017-08-5
categories: J2EE
---

Struts2和Servlet相比有几个牛逼的地方。对OGNL表达式的整合以及通配符的运用就是其中两个。

而J2EE标准中，与这两个对应的分别是JSP中EL表达式的运用，以及urlPattern中的通配符。

[前面的一篇文章](http://blog.csdn.net/holmofy/article/details/78385677)中，讲述了OGNL的使用(OGNL在功能上把EL秒成渣(～￣▽￣)～ )。

这篇文章就来说说Struts2中的通配符以及它的各种坑。

##最基本的使用

## 1. `*`通配符

和Servlet标准中urlPattern通配符功能一样，就是一个控制器类处理多个url的请求：通过合并一些相似的url映射，减少action映射的数量。

```xml
<action name="/edit*" class="struts.webapp.example.Edit{1}Action">
    <result name="failure">/mainMenu.jsp</result>
    <result>{1}.jsp</result>
</action>
```

上面name属性中的`*`通配符就是允许匹配任意以`/edit`开头的url，比如`/editSubscription`, `/editRegistration`，**但是需要注意的是`/editSubscription/add`这种url是无法匹配的**。

在&lt;action&gt;标签的其他属性中，甚至子标签&lt;result&gt;中以及result的子标签&lt;param&gt;都可以使用`{n}`这种方式去替换url映射中由通配符表示的部分，其中n的范围是0到9，特殊地，当n=0时，表示完整的请求路径。

比如`/edit*`匹配`/editRegistration`请求，那么`{1}`就是`Registration`，`{0}`就是`/editRegistration`(这方面和正则表达式类似)。

但是**注意下面这个坑**：

```xml
<action name="List*s" class="actions.List{1}s">
  <result>list{1}s.jsp</result>
</action>
```

当url为`ListAccounts`，上面的配置会正常运行。当url为`ListSponsors`时，由于`ListSponsors`中间也出现了s，所以最终会得到下面的匹配结果：

```xml
<action name="ListSpons" class="actions.ListSpons">
  <result>listSpons.jsp</result>
</action>
```

所以**`*`号最好不要出现在中间**(估计也没人会这么搞)。

## 2. `**` 通配符

前面说到`/edit*`通配符无法匹配`/editSubscription/add`这样的url。如果确实需要匹配这种url可以使用两个`*`号， 也就是`/edit**`。

官方文档中给出了一个通配符中特殊符号的列表：

| 特殊符号  | 意义                                       |
| :---: | ---------------------------------------- |
|  `*`  | 匹配除了斜杠（'/'）字符之外的零个或多个字符。                 |
| `**`  | 匹配零个或多个字符，包括斜杠（'/'）字符。                   |
| `\`字符 | 反斜线字符用作转义序列。<br>所以`\*`匹配字符星号（`*`），`\\`匹配字符反斜杠（`\`）。 |

> **注意：**如果需要让Action的name属性包含`/`，需要配置一个常量：
>
> `<constant name="struts.enable.SlashesInActionNames" value="true"/>`
>
> 虽然能够让Action的name包含`/`，但Struts2官方并不推荐使用这种配置(因为有副作用)，可以参看Struts2官网对Action name包含`/`的讨论：https://issues.apache.org/jira/browse/WW-1383
>
> 另外如果需要对Action name中能出现的字符进行限制，可以配置如下变量：
>
> ``<constant name = "struts.allowed.action.names" value = "[a-z{}]" */>`



> Struts2的通配符与[Java7中新增的FileSystem.getPathMatcher](https://docs.oracle.com/javase/8/docs/api/java/nio/file/FileSystem.html#getPathMatcher-java.lang.String-)中的通配符在功能上有点相似。
>
> 上面这中使用`*`或`?`的通配符模式有很多名字，有人把它叫做[`glob`](https://en.wikipedia.org/wiki/Glob_(programming))匹配模式，也有人把它叫做[Ant-style](https://docs.spring.io/spring/docs/5.0.2.BUILD-SNAPSHOT/javadoc-api/org/springframework/web/bind/annotation/RequestMapping.html#path--)...



##命名空间的配置

虽然不推荐Action的name属性包含`/`，但是对于使用`/`进行模块划分有更好的解决方案——package的namespace属性。

Struts2中，package标签的namespace属性是用来细分项目模块的，比如：

```xml
<!-- 没有配置namespace，则为默认命名空间 -->
<package name="default">
    <action name="foo" class="mypackage.simpleAction">
        <result name="success" type="dispatcher">greeting.jsp</result>
    </action>

    <action name="bar" class="mypackage.simpleAction">
        <result name="success" type="dispatcher">bar1.jsp</result>
    </action>
</package>
<!-- 配置namespace为"/"，则为根命名空间 -->
<package name="mypackage1" namespace="/">
    <action name="moo" class="mypackage.simpleAction">
        <result name="success" type="dispatcher">moo.jsp</result>
    </action>
</package>

<!-- 配置namespace为"/barspace"，是精确命名空间 -->
<package name="mypackage2" namespace="/barspace">
    <action name="bar" class="mypackage.simpleAction">
        <result name="success" type="dispatcher">bar2.jsp</result>
    </action>
</package>
```

> 命名空间的匹配优先级为：精确命名空间 > 根命名空间 > 默认命名空间



##命名空间中包含请求参数

从Struts2.1开始，框架可以从命名空间中提取请求参数，要使用该功能需要先配置一个常量：

```xml
<constant name="struts.patternMatcher" value="namedVariable"/>
```

定义命名空间时可以包含`{PARAM_NAME}`这样的模式串，从url中提取一些请求参数，比如：

```java
@Namespace{"/users/{userID}");
public class DetailsAction exends ActionSupport {
  private Long userID;
  public void setUserID(Long userID) {...}
}
```

如果请求url为`/users/10/detail`，Struts2框架会自动从url中提取出`10`作为`DetailsAction.userID`字段的值。



##Action Name中包含请求参数

除了上面的命名空间可以从url提取请求参数，Action也可以从url中提取请求参数，使用该功能需要配置两个常量：

```xml
<constant name="struts.enable.SlashesInActionNames" value="true"/>
<constant name="struts.mapper.alwaysSelectFullNamespace" value="false"/>
```

然后配置Action映射：

```xml
<package name="edit" extends="struts-default" namespace="/edit">
    <action name="/person/*" class="org.apache.struts.webapp.example.EditAction">
        <param name="id">{1}</param>
        <result>/mainMenu.jsp</result>
    </action>
</package>
```

当请求url为`/edit/person/123`时，`EditAction`的`id`字段将会被设置为`123`

> 命名空间和Action Name中携带参数，这两个运用的场合还是比较多，比如各大博客平台的url就是这么设计的，CSDN的博客url：blog.csdn.net/{USER_NAME}/article/details/{ARTICLE_ID}。在Struts2中使用这种携带参数的url可能还会蹑手蹑脚(确实不怎么好用)，但是在另一个更牛逼的MVC框架——SpringMVC，它把这种url中携带参数的方式发扬光大，并且还衍生出[RESTful](https://baike.baidu.com/item/RESTful)架构模式(听起来很牛逼的样子，其实本质上就是在url中携带参数，同时与Http协议的POST、GET、PUT 和 DELETE请求方式进行结合)。

##更牛逼的通配符----正则表达式

从Struts2.1.9开始可以在Action的name属性中定义正则表达式，这就大大的加强了Struts2的匹配能力，因为正则独立于框架独立于编程语言的，如果以前了解过正则，在Struts2的通配符映射的配置上基本不需要花什么学习成本。

## 使用前的配置

要使用正则作为通配符，需要配置三个常量：

```xml
<constant name="struts.enable.SlashesInActionNames" value="true"/>
<constant name="struts.mapper.alwaysSelectFullNamespace" value="false"/>
<constant name="struts.patternMatcher" value="regex" />
```

## 第一种形式：{FIELD_NAME}

这是最简单的一种匹配形式：url中的`{FIELD_NAME}`部分将会作为Action的字段。比如：

```xml
<package name="books" extends="struts-default" namespace="/">
    <action name="/{type}/content/{title}" class="example.BookAction">
    <result>/books/content.jsp</result>
    </action>
</package>
```

当请求的url为`/fiction/content/Frankenstein`时，`BookAction`的`type`字段会被设置为`fiction`，`title`字段会被设置为`Frankenstein`。

## 第二种形式：{FIELD_NAME: REGEX}

看到上面的例子，也许你会说这和正则有个毛关系。

别急，这第二种形式才开始和正则有关。

这种形式语法为：`{FIELD_NAME:REGEX}`，FIELD_NAME和上面第一种形式一样，作为Action的字段名；后面的REGEX，用来对url进行限制，只有满足该正则的url才能匹配该Action。比如：

```xml
<package name="books" extends="struts-default" namespace="/">
    <action name="/{type}/{author:.+}/list" class="example.ListBooksAction">
    <result>/books/list.jsp</result>
    </action>
</package>
```

上面的`.+`就是用来对url进行限制的正则。

> 有关正则表达式的使用，可以参考[这里](http://blog.csdn.net/Holmofy/article/category/7015920)

比如对于`/philosophy/AynRand/list`请求路径，ListBooksAction的`type`字段会被设置为`philosophy`，`author`字段会被设置为`AynRand`

在&lt;action&gt;标签的其他属性中，以及它的子标签中仍然可以使用`{n}`的符号获取匹配组，比如：

```xml
<package name="books" extends="struts-default" namespace="/">
    <action name="/books/{ISBN}/content" class="example.BookAction">
    <result>/books/{1}.jsp</result>
    </action>
</package>
```



##在action的method属性中使用通配符

前面说到使用`{n}`可以获取通配符的匹配组，这个匹配组可以用在&lt;action&gt;标签属性或子标签下等多个地方，而放在&lt;action&gt;的method属性下最能体现Struts2的灵活性。因为method属性是根据请求url变化而动态调用Action类下的方法的，官方称之为通配符方法(Wildcard Method)

## 1. 通配符方法

```xml
<action name="user_*" class="cn.hff.struts.UserAction" method="{1}">
    <result name="success">/user/{1}_success.jsp</result>
    <allowed-methods>add,delete,update,get</allowed-methods>
</action>
```

当请求的url为`/user_add`时，会调用UserAction的add方法。

> **注意：通配符方法中的一些坑**
>
> **1.**使用通配符方法进行映射，和使用`!`符号的“动态方法调用”可能会重叠，需要设置一个常量来禁用动态方法调用(Struts2.5默认是禁用状态)：
>
> ```xml
> <constant name="struts.enable.DynamicMethodInvocation" value="false" />
> ```
>
> **2.**对于Struts2.3之前的版本可以不配置&lt;allowed-methods&gt;，但是对于之后的版本，如果不配置&lt;allowed-methods&gt;将会出现如下异常：
>
> ```java
> Struts has detected an unhandled exception:
> Message:There is no Action mapped for namespace [/] and action name [user_login] associated with context path [/shop].
> ```
>
> 具体原因可以继续往下看

## 2. 动态方法调用

除了通配符方法，Struts2还提供了一种更便捷的方式进行Action的方法映射，而且官方还为他取了个牛逼哄哄的名字——动态方法调用(Dynamic Method Invocation)。不过官方文档中说，DMI方式存在安全性问题，所以Struts2中默认把这个功能关闭了(default.property文件中`struts.enable.DynamicMethodInvocation=false`)，如果需要使用需要设置常量将该功能打开。

```xml
<constant name="struts.enable.DynamicMethodInvocation" value="true" />
```

动态方法调用和下面这种通配符方法在功能上类似（和上面通配符方法的配置相似，只是把下划线`_`改成了`!`号）：

```xml
<action name="use!*" class="cn.hff.struts.UserAction" method="{1}">
    ...
</action>
```

> 当请求url为`use!register`时，会调用UserAction的register方法。

不过官方文档中说动态方法调用和通配符方法的实现并不相同，并且推荐首选通配符方法而不是动态方法调用。

```
The Wildcard Method feature is implemented differently. When a Wildcard Method action is invoked, the framework acts as if the matching action had been hardcoded in the configuration. The framework "believes" it's executing the action Category!create and "knows" it is executing the create method of the corresponding Action class. Accordingly, we can add for a Wildcard Method action mapping its own validations, message resources, and type converters, just like a conventional action mapping. For this reason, the Wildcard Method is preferred.
```

## 3. strict-method-invocation

在Struts2.3中在package标签中添加了一个属性来限制DMI，该选项告诉Struts2框架拒绝所有未通过`method`属性配置或者&lt;allowed-methods&gt;标签标明的方法。

> &lt;allowed-methods&gt;配置Action可访问的方法，中间用逗号隔开。

在Struts2.5中不仅仅限制了DMI的调用，还对Action的可使用的方法进行了限制，也就是默认开启了`strict-method-invocation`选项(在`struts-default.xml`文件中可以看到)

总结起来就是：

* Struts 2添加了一个`struts.enable.DynamicMethodInvocation`常量来开启DMI调用方式
* Struts 2.3在package标签中添加了一个`strict-method-invocation`属性以及一个`<allowed-methods>`标签来限制DMI可调用的方法。
* Struts 2.5将`strict-method-invocation`属性默认值设置成了true



>  **通配符的详细实现细节可以查看ActionMapper接口的几个实现类**



> 参考链接：
>
> Struts2官方文档：http://struts.apache.org/core-developers/wildcard-mappings.html