---
title: RMI简单示例
date: 2017-12-14
categories: JAVA
---

RMI是Java1.1中实现的一种RPC通信机制。使用RMI可以让一个Java虚拟机中的对象调用另外一个虚拟机中对象的方法，它大大增强了Java开发分布式应用的能力。

这篇文章使用RMI实现一个简单的例子。

# 服务端代码

**1.**声明一个服务接口，提供给远程调用，该接口必须继承自`java.rmi.Remote`接口。

```java
package cn.hff.service;

import java.rmi.Remote;
import java.rmi.RemoteException;

/**
 * 提供一个简单的远程调用接口，该接口必须继承Remote接口，用于标记该接口是远程服务提供。
 */
public interface ICalcService extends Remote {

	/**
	 * @throws RemoteException
	 *             远程方法必须声明该异常
	 */
	int add(int a, int b) throws RemoteException;

	/**
	 * @throws RemoteException
	 *             远程方法必须声明该异常
	 */
	int minus(int a, int b) throws RemoteException;

}
```

**2.**实现该服务接口

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

**3.**调用RMI的API将服务暴露给客户端调用

```java
package cn.hff.server;

import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;

import cn.hff.service.CalcServiceImpl;
import cn.hff.service.ICalcService;

public class Server {

	public static final String SERVICE_NAME = "calc";

	public static void main(String[] args) throws RemoteException, MalformedURLException, AlreadyBoundException {

		// 创建服务对象
		ICalcService service = new CalcServiceImpl();

		// 将服务暴露给远程调用
		Remote stub = UnicastRemoteObject.exportObject(service, 0);

		// 获取本地RMI注册表对象
		Registry registry = LocateRegistry.getRegistry();

		// 将存根以指定服务名绑定到注册表中
		registry.rebind(SERVICE_NAME, stub);

		System.out.println("ok. calc service binded");

	}
}
```

上面的绑定服务的方法可以简写成下面这种形式：

```java
	public static void main(String[] args) throws RemoteException, MalformedURLException, AlreadyBoundException {

		ICalcService service = new CalcServiceImpl();

        // Naming类已经帮我们把各种操作封装好了
        // 这里可以指定端口,rmiregistry默认端口为1099
		Naming.rebind("rmi://127.0.0.1:1099/calc", service);

		System.out.println("ok. calc service binded");
	}
```



# 客户端代码

```java
package cn.hff.client;

import java.rmi.NotBoundException;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

import cn.hff.service.ICalcService;

public class Client {

	public static final String SERVICE_NAME = "calc";

	public static void main(String[] args) throws RemoteException, NotBoundException {

		// 获取服务注册表
		Registry registry = LocateRegistry.getRegistry("127.0.0.1", 0);

		// 从注册表中查找指定服务
		Remote stub = registry.lookup(SERVICE_NAME);

		// 调用服务
		ICalcService service = (ICalcService) stub;

		int add = service.add(3, 3);

		System.out.println(add);

		int minus = service.minus(3, 3);

		System.out.println(minus);

	}
}
```

和前面一样，查找服务的步骤也已经封装在Naming中：

```java
Remote stub = Naming.lookup("rmi://127.0.0.1:1099/calc");
```

# 编译运行

**1.**编译服务端的三个类，并在类路径下执行`rmiregistry`命令运行本地注册表，然后再执行Server的main方法。

```shell
# 编译服务端代码
javac cn/hff/service/ICalcService.java cn/hff/service/CalcServiceImpl.java cn/hff/server/Server.java

# 运行本地注册表，注意开启后不要关闭
# 可以用"rmiregistry &"让它在后台运行
# rmiregistry还可以指定端口
# 这个端口就是服务端绑定服务、客户端查找服务的端口
rmiregistry
```

运行Server类的main方法：

```shell
java cn.hff.server.Server
```

![服务端开启](http://img-blog.csdn.net/20171223175946721?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**2.**编译并运行客户端程序

```shell
# 编译
javac cn/hff/client/Client.java
# 运行
java cn.hff.client.Client
```

![客户端调用](http://img-blog.csdn.net/20171223180005361?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

