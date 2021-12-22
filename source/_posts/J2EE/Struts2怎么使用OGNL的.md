---
title: Struts2怎么使用OGNL的
date: 2017-08-7
categories: J2EE
---

> 这几天整理笔记，发现了以前的几篇文章没发出来

在之前的一篇文章中我们看到了OGNL的强大功能。

> [OGNL](http://commons.apache.org/proper/commons-ognl/index.html)并不是专门为Struts2框架而设计的，它是用于获取和设置Java对象属性的一种独立的表达式语言。
>
> 所以在看这篇文章之前建议先把[之前的一篇文章](http://blog.csdn.net/holmofy/article/details/78385677)看完。

Struts2框架就是使用OGNL完成数据的设置与访问的：

数据访问主要体现在**JSP页面中我们可以使用OGNL表达式访问Action中的数据**；

数据的设置主要体现在**Struts2框架自动帮你把请求参数注入到Action中**。



下面这篇文章将会根据源码详细分析Struts2是如何运用OGNL表达式的。

> 先来看一张图，本文会逐步分析源码来解释这张图包含的意义。

![ValueStack原理图](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp39gtbwj20mb0ch0tg.jpg)

## 一、官方文档导致的错误理解

[官方文档](http://struts.apache.org/core-developers/)中对OGNL有一段描述：Struts2框架将OGNL的上下文context设置为ActionContext，并将ValueStack设置为OGNL的root对象，而且还给出了一个树状图。

![contextMap](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp493ypuj20t50dqgmf.jpg)

看完这文档，按照[前一篇文章](http://blog.csdn.net/holmofy/article/details/78385677)的介绍的OGNL取值方式去思考，会发现ActionContext和ValueStack(唯一实现类OgnlValueStack)完全不符合OGNL对context和root的要求：context必须是Map类型的映射，root可以是任意Object类型(但是根据调用`actionContext.get("root")`并不能获取到ValueStack对象)。

经过查看源码发现：ActionContext虽然没有实现Map，但是内部有一个名为context的Map对象；OgnlValueStack也并没有实现所谓的值栈功能，而是由内部的CompoundRoot实现，而且OgnlValueStack也有一个名为context的Map对象。

![ActionContext](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp5b1jwkj20hy088gma.jpg)

![ValueStack](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp5qp61zj20q8090t9w.jpg)

## 二、ActionContext和ValueStack中真正的context

看到上面的代码，我首先想到的是，这两个context不会就是同一个吧。那就写个Action类测试一下试试看吧：

```java
public class OgnlTestAction extends ActionSupport {

    @Override
    public String execute() throws Exception {
        ActionContext ctx = ActionContext.getContext();

        ValueStack stack = ctx.getValueStack();

        Map<String, Object> map1 = ctx.getContextMap();

        Map<String, Object> map2 = stack.getContext();

        System.out.println(map1 == map2); // true

        System.out.println(map1.getClass()); // class ognl.OgnlContext

        return SUCCESS;
    }
}
```

哟呵，真是同一个对象，而且根据`map1.getClass()`得到类型也的确实是实现了Map接口的OgnlContext，这就比较符合Ognl的规则了。



再仔细看看下面OgnlValueStack中的setRoot方法，发现里面调用了`Ognl.createDefaultContext`创建了一个OgnlContext对象，这才是真正的context。

![OgnlValueStack](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp68p6nsj20vz0f7mzl.jpg)

## 三、CompoundRoot才是真正的值栈实现

CompoundRoot源码很简单，就是使用List模拟了一个栈的数据结构。

> CompoundRoot：2.3版本中继承自ArrayList，在2.5之后换成了CopyOnWriteArrayList。

![ComponentRoot](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp74fwxqj20hm0gq3zm.jpg)

>从上面的代码可以看出CompoundRoot效率很低下，push和pop操作分别使用`add(0,o)`和`remove(0)`实现，对于使用数组实现的List而言，这两个操作都伴随着大数据量的拷贝操作。
>
>好在CompoundRoot中并不会存放很多数据，一次请求过程中最多都不会存储超过10个元素，所以性能影响并不是明显。但是开发时应该注意不应该往CompoundRoot中push过多的对象。

## 四、为什么OGNL表达式能直接访问栈顶的数据呢

看过前一篇文章的应该知道，在OGNL表达式中访问List对象需要使用`[n]`这样的索引语法。但是平常使用在页面中使用OGNL表达式都是可以直接访问Action中的属性。这个到底是怎么实现的呢？

首先需要知道的是：客户端请求一个Action后，该Action会被放在值栈的栈顶(也就是CompoundRoot的栈顶)，如果配置了Action chain，后面的Action会推入栈顶。

然后再看一下刚刚的OgnlValueStack中创建OgnlContext的方法，发现该方法调用时传入了一个CompoundRootAccessor对象，这个**CompoundRootAccessor**是解密的关键。

![root](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp7rjk50j20uf05adgn.jpg)

## 五、CompoundRootAccessor让OGNL总是能直接访问栈顶元素

CompoundRootAccessor实现了PropertyAccessor接口，OGNL中可以通过实现PropertyAccessor接口来自定义对象的访问规则

```java
public class CompoundRootAccessor implements PropertyAccessor, MethodAccessor, ClassResolver {
	...// 省略若干代码

    public Object getProperty(Map context, Object target, Object name) throws OgnlException {
        CompoundRoot root = (CompoundRoot) target;
        OgnlContext ognlContext = (OgnlContext) context;

        // 这个地方
        if (name instanceof Integer) {
            Integer index = (Integer) name;
            return root.cutStack(index);
        } else if (name instanceof String) {
            // name为top时，取栈顶元素
            if ("top".equals(name)) {
                if (root.size() > 0) {
                    return root.get(0);
                } else {
                    return null;
                }
            }
			// 遍历栈中的所有元素
            for (Object o : root) {
                if (o == null) {
                    continue;
                }

                try {
                    // 找到栈中第一个不为空的元素进行属性匹配
                    if ((OgnlRuntime.hasGetProperty(ognlContext, o, name)) || ((o instanceof Map) && ((Map) o).containsKey(name))) {
                        return OgnlRuntime.getProperty(ognlContext, o, name);
                    }
                } catch (OgnlException e) {
                    if (e.getReason() != null) {
                        final String msg = "Caught an Ognl exception while getting property " + name;
                        // 如果查找属性时出现异常，则抛出异常停止匹配
                        throw new XWorkException(msg, e);
                    }
                } catch (IntrospectionException e) {
                    // 如果是Ognl内部调用反省机制API出现异常，则继续匹配栈中下一个元素
                }
            }

            // 没找到抛异常
            if (context.containsKey(OgnlValueStack.THROW_EXCEPTION_ON_FAILURE))
                throw new NoSuchPropertyException(target, name);
            else
                return null;
        } else {
            return null;
        }
    }
    // 还有一个setProperty方法，由于篇幅原因这里就不列出来了。
}
```

通过查看源码发现，前面所谓的“访问栈顶元素”其实是**遍历List直到找到相应的属性**。

真正的访问栈顶可以通过“top”的ognl表达式来获取。

这也导致了一个严重的问题：**如果你的Action中有名字为“top”的属性，这个属性将访问不了，这是因为通过“top”会直接返回栈顶元素**。

下面写个例子可以验证这个问题：

```java
public class TopTestAction extends ActionSupport{
    private String top = "Action中的top属性";
    .. // 省略getter/setter方法

    public String execute() throws Exception {
        return SUCCESS;
    }
    // 重写toString方法
    public String toString(){
        return "栈中的TopTestAction对象";
    }
}
```

在jsp页面中尝试访问该属性：

```xml
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="s" uri="/struts-tags"%>
<html>
<body>
	<h4>struts2 OGNL</h4>
	<s:property value="top" />
</body>
</html>
```

struts.xml文件中的action配置这里就不贴了。

测试结果：

![top](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp8faz5jj207103qaa0.jpg)

## 六、为什么通过EL表达式能访问值栈中的数据

除了能在Struts2的标签中能使用OGNL表达式访问值栈，EL表达式中也能访问值栈中的数据。

比如：

```java
public class LoginAction extends ActionSupport {

    private String username;
    private String password;
 	..// 省略getter/setter代码
	public String execute() throws Exception {
        return SUCCESS;
    }
}
```

在JSP页面中使用EL表达式访问：

```xml
<%@ page language="java" contentType="text/html; charset=UTF-8"
	pageEncoding="UTF-8" isELIgnored="false"%>
<!-- 这个地方需要将isELIgnored="false"设置上，Struts2貌似把EL表达式功能给默认关闭了 -->
<html>
<body>
	<table border="1">
		<tr><td>username:</td><td>${username}</td></tr>
		<tr><td>password:</td><td>${password}</td></tr>
	</table>
</body>
</html>
```

测试结果：

![actionTest](http://tva1.sinaimg.cn/large/bda5cd74gy1fqdp90if5uj207d03eq2y.jpg)

为什么能在EL表达式中访问Action中的属性呢，Struts2是如何实现的。

先自己思考一下：

如果要我实现，第一种方式是将值栈设置为Request的一个Attribute，但是实现起来相当复杂，而且代码可维护性不高；第二种能想到的就是用代理类包装Request，重写默认的getAttribute方法，这样既不破坏原有代码，而且修改起来也比较方便。

Servlet规范中定义了包装类应该继承自`HttpServletRequestWrapper`，查看一下它有多少子类，我擦，还真发现Struts2定义了一个包装类：

```java
public class StrutsRequestWrapper extends HttpServletRequestWrapper {
	// 省略构造函数
    /**
     * Gets the object, looking in the value stack if not found
     * 如果原始的request.getAttribute没有获取到对象，则尝试在值栈中查找对象
     * @param key The attribute key
     */
    public Object getAttribute(String key) {
        if (key == null) {
            throw new NullPointerException("You must specify a key value");
        }

        if (disableRequestAttributeValueStackLookup || key.startsWith("javax.servlet")) {
            // 防止与Servlet标准中的一些属性冲突，而影响Servlet容器的正常工作，
            // 所有遇到以java.servlet开头的key直接返回
            return super.getAttribute(key);
        }

        ActionContext ctx = ActionContext.getContext();
        // 根据key获取原始的getAttribute中的对象
        Object attribute = super.getAttribute(key);

        // 如果getAttribute没有找到，则尝试从值栈中查找
        if (ctx != null && attribute == null) {
            boolean alreadyIn = isTrue((Boolean) ctx.get(REQUEST_WRAPPER_GET_ATTRIBUTE));

            // 注意: 不能让以#开始的key进行查找，否则当
            // key为#attr.foo或者#request.foo时将会发生死循环
            if (!alreadyIn && !key.contains("#")) {
                try {
                    // 尝试在值栈中查找
                    ctx.put(REQUEST_WRAPPER_GET_ATTRIBUTE, Boolean.TRUE);
                    ValueStack stack = ctx.getValueStack();
                    if (stack != null) {
                        // 根据key在值栈中查找对象
                        attribute = stack.findValue(key);
                    }
                } finally {
                    ctx.put(REQUEST_WRAPPER_GET_ATTRIBUTE, Boolean.FALSE);
                }
            }
        }
        return attribute;
    }
}
```

> 通过查看上面这段代码发现：在EL表达式中并不能使用`#xxx`的方式访问context下的内容(主要是几个内置对象的访问)。这个功能也没必要，EL自带的四个范围对象已经具有这个功能了，如果能使用`#xxx`就导致功能重复。而且使用`#att.foo`或`#request.foo`访问request中的数据，反而会导致死循环。