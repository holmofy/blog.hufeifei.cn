---
title: Java SPI机制
date: 2018-02-12 16:50
categories: JAVA
keywords:
- Java SPI
---

[SPI全称Service Provider Interface](https://en.wikipedia.org/wiki/Service_provider_interface)，是Java提供的一种让第三方实现或扩展的API。

java平台中很多功能都是以这种方式提供接口给开发者调用的，最典型的如：[JDBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity)，[JDNI](https://en.wikipedia.org/wiki/Java_Naming_and_Directory_Interface)，[JCE(Java加密扩展)](https://en.wikipedia.org/wiki/Java_Cryptography_Extension)，[JAXP](https://en.wikipedia.org/wiki/Java_API_for_XML_Processing)等，看JDK源码或者第三方源码的时候会经常碰到SPI，所以我觉得很有必要写个笔记把SPI记录下来。

# 定义SPI接口

这里我就拿JDBC的数据库驱动作为例子，这里Driver类进行了简化。

> 早期的JDBC需要使用DriverManager手动注册数据库驱动类，JDBC4.0之后要求数据库驱动必须在  `META-INF/services/java.sql.Driver`文件中包含驱动类的全类名，以实现驱动类的自动注册(自动注册实际上就是用SPI实现)。

```java
// 这个地方不要用java作为包名
package javaxx;

public interface Driver {

	void connect();

}
```

> 这里不要使用`java`作为包名，因为这个包名已经被JDK自己使用了，为了防止冲突Java平台不允许第三方使用这个作为包名，否则在后面的代码中会抛出SecurityException。
>
> ![包名不允许为java](http://img-blog.csdn.net/20180212181440818?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

用以下命令编译这个接口，并打成jar包

```powershell
:: 编译SPI接口
javac javaxx/Driver.java
:: 把SPI打成jar包，方便其他SPI的实现者添加依赖
jar cvf ../jar/driver-spi.jar javaxx/Driver.class
```

# 实现SPI接口

**MySQL驱动简单实现**

```java
package mysql;

import javaxx.Driver;

public class MysqlDriver implements Driver{

	public void connect(){
		System.out.println("MySQL Driver Connected");
	}

}
```

为了能让Java平台发现这个SPI实现我们需要在`META-INF/services`目录下以SPI接口全类名作为文件名，并在该文件中写下实现类全类名。

在这个例子中就是创建`META-INF/services/javaxx.Driver`文件，并在文件中写下以下内容

```bash
# 文件中“#”号开头的内容会被解析成注释
# MySQL Provider Implement
mysql.MysqlDriver
```

我们把这个SPI实现打成jar包

```powershell
:: 编译Mysql SPI实现
javac -classpath ../jar/driver-spi.jar mysql/MysqlDriver.java
:: 将编译的class文件打成jar包
jar -cvf ../jar/mysql-driver.jar mysql/ META-INF/
```

**Oracle驱动简单实现**

```java
package oracle;

import javaxx.Driver;

public class OracleDriver implements Driver {

	public void connect(){
		System.out.println("Oracle Driver Connected");
	}

}
```

在`META-INF/services/javaxx.Driver`文件中写下Oracle SPI的实现类：

```bash
# java.Driver Oracle Implement
oracle.OracleDriver
```

编译打包：

```powershell
:: 编译Oracle实现
javac -classpath ../jar/driver-spi.jar oracle/OracleDriver.java
:: 打成jar包
jar -cvf ../jar/oracle-driver.jar oracle/ META-INF/
```

# 使用ServiceLoader加载SPI实现

在`java.util`包下有个ServiceLoader工具类可以帮助我们加载类路径下所有的SPI接口的实现类

```java
package cn.hff;

import javaxx.Driver;
import java.util.ServiceLoader;

public class Client {

    public static void main(String[] args) {
    	ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
        // 遍历所有的Loader实现类
    	for(Driver d : loader) {
    		d.connect();
    	}
    }

}
```

我们编译一下上面的Client类：

```powershell
:: 编译
javac -classpath ../jar/driver-spi.jar cn/hff/Client.java
```

为了能让ServiceLoader找到MySQL和Oracle的SPI，运行时需要在将前面编译的jar包添加到类路径子下：

```powershell
:: 运行
java -classpath .;../jar/driver-spi.jar;../jar/mysql-driver.jar;../jar/oracle-driver.jar; cn.hff.Client
```

> 示例代码地址：https://gitee.com/holmofy/Java-SPI-Demo