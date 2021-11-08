---
title: Hessian初体验
date: 2017-12-24
categories: J2EE
tags: 
- Hessian
---

前面两篇文章[写了一个RMI的例子](http://blog.csdn.net/holmofy/article/details/78881331)并[简单分析了一下RMI的原理](http://blog.csdn.net/holmofy/article/details/78881885)。

这里简单概括一下RMI的优缺点；

优点：

* 面向对象的远程服务模型；客户端可以像调用本机对象一样去调用远程对象的方法。
* 基于TCP协议，执行速度非常快。

缺点：

* 使用Java特有的序列化协议JRMP，不能使用在多语言实现的异构系统上。

在J2EE中WebService使用SOAP协议解决了异构系统通信的问题。因为SOAP协议基于HTTP协议和XML实现的，并且在ASP.NET，PHP等其他语言中都有实现。但是由于XML冗余度较高，导致WebService执行效率偏低。

[caucho提供的Hessian](http://hessian.caucho.com/)就是基于这两者之间实现：**基于Http协议以二进制格式进行数据传输**。

> alibaba开源的dobbo中也提供了Hessian协议的支持

Hessian的官网中提供了Java，Flash，Python，C++，.NET，PHP，Ruby，OC等各种语言对Hessian的实现。这篇文章会用Java来实现一个简单的例子。

##服务端

**1.**导入Hessian以及相关的依赖

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>cn.hff</groupId>
	<artifactId>HessianDemo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>war</packaging>

	<dependencies>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>3.0.1</version>
			<scope>provided</scope>
		</dependency>
        <!-- 添加Hessian的依赖 -->
		<dependency>
			<groupId>com.caucho</groupId>
			<artifactId>hessian</artifactId>
			<version>4.0.38</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
            <!-- 配置编译级别 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.6.1</version>
				<configuration>
					<compilerVersion>1.8</compilerVersion>
				</configuration>
			</plugin>

			<!-- 配置Tomcat插件 -->
			<plugin>
				<groupId>org.apache.tomcat.maven</groupId>
				<artifactId>tomcat7-maven-plugin</artifactId>
				<configuration>
					<port>8080</port>
					<path>/</path>
					<uriEncoding>UTF-8</uriEncoding>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

**2.**定义服务接口

```java
package cn.hff.service;

public interface ICalcService {

	int add(int a, int b);

	int minus(int a, int b);

}
```

**3.**实现服务接口

```java
package cn.hff.service;

public class CalcServiceImpl implements ICalcService {

	public int add(int a, int b) {
		int result = a + b;
		System.out.printf("%d + %d = %d %n", a, b, result);
		return result;
	}

	public int minus(int a, int b) {
		int result = a - b;
		System.out.printf("%d - %d = %d %n", a, b, result);
		return result;
	}

}
```

**4.**在web.xml中配置服务

```xml
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">

	<display-name>Hessian-Test</display-name>

	<servlet>
		<servlet-name>Hessian</servlet-name>
		<servlet-class>com.caucho.hessian.server.HessianServlet</servlet-class>
		<init-param>
			<param-name>home-api</param-name>
			<param-value>cn.hff.service.ICalcService</param-value>
		</init-param>
		<init-param>
			<param-name>home-class</param-name>
			<param-value>cn.hff.service.CalcServiceImpl</param-value>
		</init-param>
	</servlet>

	<servlet-mapping>
		<servlet-name>Hessian</servlet-name>
		<url-pattern>/calc</url-pattern>
	</servlet-mapping>
</web-app>
```

**5.**运行服务端应用

```shell
mvn clean tomcat7:run
```



如果这个时候使用浏览器访问这个服务，会得到如下的响应：

![浏览器访问](http://img-blog.csdn.net/20171224151307535?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 因为调用服务传入的参数可能是对象类型，这时候传入的数据可能会非常大，GET请求很明显不符合要求，所以Hessian要求调用服务必须的使用POST请求

##客户端

```java
package cn.hff.client;

import java.net.MalformedURLException;

import com.caucho.hessian.client.HessianProxyFactory;

import cn.hff.service.ICalcService;

public class HessianClient {

	private static final String url = "http://localhost:8080/calc";

	public static void main(String[] args) throws MalformedURLException, ClassNotFoundException {

		HessianProxyFactory factory = new HessianProxyFactory();

		ICalcService service = (ICalcService) factory.create(url);

		System.out.println(service.add(3, 3));

		System.out.println(service.minus(3, 3));
	}

}
```

测试结果：

![调用结果](http://img-blog.csdn.net/20171224151317617?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
