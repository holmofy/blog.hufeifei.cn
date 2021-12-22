---
title: Spring高级篇-Profile
date: 2018-2-4
categories: J2EE
---

通常开发测试与上线生产使用不同的环境配置，我们可以使用@Profile注解实现。

## 在类上使用@Profile注解

**开发环境配置**

```java
package cn.hff;

import javax.sql.DataSource;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseBuilder;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType;

/**
 * 开发环境下的配置
 *
 * @author Holmofy
 *
 */
@Configuration
@Profile("dev")
public class DevProfileConfig {

	@Bean(destroyMethod = "shutdown")
	public DataSource dataSource() {
		// 开发环境下使用内嵌数据库
		return new EmbeddedDatabaseBuilder()
				// 使用纯Java开发的DERBY数据库作为内嵌数据库
				.setType(EmbeddedDatabaseType.DERBY)
				// 创建表结构的sql文件
				.addScript("classpath:schema.sql")
				// 插入测试数据的sql文件
				.addScript("classpath:test-data.sql")
				.build();
	}
}
```

因为要使用Derby作为内嵌数据库，所以要导入Derby的依赖：

```xml
<dependency>
  <groupId>org.apache.derby</groupId>
  <artifactId>derby</artifactId>
  <version>10.14.1.0</version>
  <scope>test</scope>
</dependency>
```

>   Spring支持HSQL,H2,Derby三种内嵌数据库，我们只需把数据库的依赖jar包放在类路径下，Spring会自动为我们创建数据库以及数据库连接，在连接关闭的时候数据库就会被删除，这个很适合在开发环境下使用。

**生产环境配置**

```java
package cn.hff;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;

import com.alibaba.druid.pool.DruidDataSource;

/**
 * 生产环境下的配置
 *
 * @author Holmofy
 *
 */
@Configuration
@Profile("prod")
@PropertySource("classpath:db.properties")
public class ProdProfileConfig {

	@Autowired
	private Environment env;

	@Bean
	public DataSource dataSource() {
		DruidDataSource ds = new DruidDataSource();
		ds.setUrl(env.getProperty("jdbc.url"));
		ds.setDriverClassName(env.getProperty("jdbc.driverClass"));
		ds.setUsername(env.getProperty("jdbc.username"));
		ds.setPassword(env.getProperty("jdbc.password"));
		// 其他配置
		return ds;
	}
}
```

这里使用阿里的Druid连接池：

```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.1.6</version>
</dependency>
```

另外在类路径下的`db.properties`文件中配置数据源的详细配置。

## 在方法上使用@Profile注解

Spring3.2开始，@Profile注解就可以在方法级别上使用了：

```java
package cn.hff;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseBuilder;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType;

import com.alibaba.druid.pool.DruidDataSource;

@Configuration
@PropertySource("classpath:db.properties")
public class DataSourceConfig {

	@Autowired
	private Environment env;

	/**
	 * 开发环境才激活
	 */
	@Bean("dataSource")
	@Profile("dev")
	public DataSource embeddedDerbyDataSource(){
		return new EmbeddedDatabaseBuilder()
				.setType(EmbeddedDatabaseType.DERBY)
				.addScript("classpath:schema.sql")
				.addScript("classpath:test-data.sql")
				.build();
	}

	/**
	 * 生产环境才激活
	 */
	@Bean("dataSource")
	@Profile("prod")
	public DataSource mysqlDataSource() {
		DruidDataSource ds = new DruidDataSource();
		ds.setUrl(env.getProperty("jdbc.url"));
		ds.setDriverClassName(env.getProperty("jdbc.driverClass"));
		ds.setUsername(env.getProperty("jdbc.username"));
		ds.setPassword(env.getProperty("jdbc.password"));
		return ds;
	}

}
```

## XML配置方式

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/jdbc
		http://www.springframework.org/schema/jdbc/spring-jdbc-4.3.xsd
		http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		http://www.springframework.org/schema/context/spring-context-4.3.xsd">

	<context:property-placeholder location="classpath:db.properties"/>

	<beans profile="dev">
		<jdbc:embedded-database id="dataSource">
			<jdbc:script location="classpath:schema.xml"/>
			<jdbc:script location="classpath:test-data.xml"/>
		</jdbc:embedded-database>
	</beans>

	<beans profile="prod">
		<bean id="dataSource"
			class="com.alibaba.druid.pool.DruidDataSource"
			destroy-method="close"
			p:url="${jdbc.url}"
			p:driverClassName="${jdbc.driverClass}"
			p:username="${jdbc.username}"
			p:password="${jdbc.password}">
		</bean>
	</beans>

</beans>
```

## 激活Profile

激活Profile需要两个独立的properties属性：`spring.profiles.active`和`spring.profiles.default`。如果设置了`spring.profiles.active`的值，就用它的值来确定哪个profile是激活的，如果没有设置，则使用`spring.profiles.default`设置的值。如果两个值都没有设置，则所有的profile都不会被激活，也就是至创建哪些没有定义在profile中的bean。

我们有以下几种方式设置这两个值：

* 作为DispatcherServlet的初始化参数

  ```xml
  <servlet>
  	<servlet-name>CoreServlet</servlet-name>
  	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  	<init-param>
  		<param-name>spring.profiles.default</param-name>
  		<param-value>dev</param-value>
  	</init-param>
  	<load-on-startup>1</load-on-startup>
  </servlet>
  ```

  > web.xml中配置DispatcherServlet初始化参数

* 作为Web应用的上下文参数

  ```xml
  <web-app>
  	...
  	<context-param>
  		<param-name>spring.profiles.default</param-name>
  		<param-value>dev</param-value>
  	</context-param>
  	...
  </web-app>
  ```

  > web.xml中配置context-param

* 作为JNDI的条目

* 作为环境变量

* 作为JVM的系统属性

* 在测试类上使用@ActiveProfiles注解激活指定的profile

  ```java
  package cn.hff;

  import java.sql.Connection;
  import java.sql.SQLException;

  import javax.sql.DataSource;

  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.test.context.ActiveProfiles;
  import org.springframework.test.context.ContextConfiguration;
  import org.springframework.test.context.junit4.SpringRunner;

  @RunWith(SpringRunner.class)
  @ContextConfiguration(classes = DataSourceConfig.class)
  @ActiveProfiles("dev")
  public class ProfilesTest {

  	@Autowired
  	DataSource ds;

  	@Test
  	public void test() throws SQLException {
  		Connection conn = ds.getConnection();
  		System.out.println(conn);
  		conn.close();
  	}

  }
  ```



> 在SpringBoot中通常会把服务打成jar包部署，可以在命令后面指定激活哪个profile
>
> ```shell
> java -jar XXX.jar --spring.profiles.active=prod
> ```
>
>



> 参考：
>
> Spring In Action
>
> Spring Framework Documentation