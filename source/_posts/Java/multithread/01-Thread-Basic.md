---
title: Java多线程复习与巩固(一)--线程基本使用
date: 2017-06-14
categories: JAVA
keywords:
- Java 多线程编程
---
# 1、进程与线程

在并发编程中，有两个基本的执行单元：进程和线程。在Java中，并发编程主要关心的是线程。当然，进程也很重要。


## 1.1、进程(Process)

进程有独立的执行环境，一个进程有一套私有的、完整的运行时资源，比如：每个进程都有自己的内存空间。

进程通常会被认为是一个应用程序的代名词。但实际上一个应用程序可能会包含多个协同工作的进程。比如你电脑里的360打开后肯定有两个或两个以上的进程：一个管理是后台服务模块，一个是主程序控制模块。而为了促进进程之间的通信，操作系统都会支持进程间通信(Inter-Process Communication IPC)的机制，例如：套接字、管道机制。IPC不仅可以用于同一系统上进程之间的通信，还可以用于不同系统上进程之间的通信，比如Java的RMI(Remote Method Invoke)就是不同系统间IPC的体现。

Java虚拟机的大多数实现都是作为一个进程运行的。Java应用程序可以使用`Runtime.exec()`执行指定命定来创建新进程，Java1.5之后还提供了更灵活的[ProcessBuilder](https://docs.oracle.com/javase/8/docs/api/java/lang/ProcessBuilder.htm)来创建进程，Java中以[Process](https://docs.oracle.com/javase/8/docs/api/java/lang/Process.html)对象表示创建的进程。

下面是用ProcessBuilder调用`unzip`命令执行解压的逻辑：

```java
@Slf4j
public class Demo {
    public void invokeUnzip(String password, Path targetDir, Path zipFile) {
        ProcessBuilder pb = new ProcessBuilder("unzip", 
                                               "-P", password, 
                                               "-d", targetDir.toAbsolutePath().toString(), 
                                               zipFile.toAbsolutePath().toString());
        try {
            FileUtils.deleteDirectory(targetDir.toFile());
            pb.redirectErrorStream(true);
            if (log.isDebugEnabled()) {
                log.debug("invoke unzip, command:" + String.join(" ", pb.command()));
            }

            Process unzipProcess = pb.start();
            try (BufferedReader stdOutReader = new BufferedReader(new InputStreamReader(unzipProcess.getInputStream()))) {
                stdOutReader.lines().forEach(log::info);
            }
            unzipProcess.waitFor();
        } catch (Exception e) {
            log.error("unzip file failed");
            ExceptionUtils.rethrow(e);
        }
    }
}
```

> 需要注意的是，由于创建的子进程没有终端控制台，所以它的标准IO流(stdin,stdout,stderr)都会被交由父进程处理(即我们的写的Java程序)。一方面这让我们能更灵活的控制子进程的IO，但如果我们没有对IO流进行处理将会导致子进程阻塞(子进程等待输入或输出导致进入sleep状态)。比如上面的例子中如果我们没有将子进程的输出消费掉(打印到日志中)，会导致子进程的阻塞；`FileUtils.deleteDirectory(targetDir.toFile())`这句也是为了防止程序运行时解压出来的文件已经存在在目录中，`unzip`命令会询问是否覆盖原文件而等待用户输入，这也会导致程序阻塞。

## 1.2、线程(Thread)

线程有个更形象的名字——“轻量级进程”。进程和线程都提供一个执行环境，但创建线程消耗的资源比创建进程少的多。

线程存在于一个进程中，每个进程至少包含一个线程(主线程)。同一进程中的线程共享该进程中的资源，比如内存资源或进程占用的文件资源等。这使得线程间通信比进程间通信更加简单，但也更容易出现问题。

> 正因为线程与进程有很多的相似之处，所以线程出现的问题以及解决方法和进程是类似的。在接下来的文章中我们会提到线程同步和线程死锁的问题，这和操作系统中的进程同步、进程死锁问题相似。进程同步已经由操作系统实现了，而线程同步需要我们程序员自己实现。

# 2、并发与并行

计算机系统通常有很多的活动进程和线程。但在单核处理器的系统中，任何**时刻**实际只有一个进程(或线程)执行。单核系统通过时间分片的方式处理进程(或线程)之间的共享。这就是早期的分时系统(Time Sharing System)。

后来多核处理器变得越来越普及，这大大增强了系统并行执行进程和线程的能力。因为多核处理器有多套处理设备(寄存器，ALU等)，所以同一时刻可以有多个进程(或线程)同时执行。

**并发(Concurrent)**是由于时间片非常短，CPU的流水线工作使得CPU看上去能够同时处理多个任务，但实际上只是同时处理不同任务的不同部分。

比如下面这个前趋图中：任务3的输入执行操作的同时($I_3$)，可以执行任务2的计算($C_2$)和任务1的打印操作($P_1$)。

![CPU并发工作的前趋图](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h85n3imgj20rm0d8423.jpg)

**并行(Parallel)**是由于有多核处理器，有多套处理设备，它可以**真正的同时**处理多个任务。

比如下面这个前趋图中，有4个处理机的CPU，在一号处理机处理一号任务输入的同时($I_1$)，二号处理机可以处理二号任务的输入($I_2$)...

![多核CPU并行的前趋图](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h862ck7vj20ic0cs3yh.jpg)

而且多核处理器的每个处理机又可以使用流水线并发处理多个任务，这就大大加强了计算机系统的处理能力。

# 3、Thread类

Java是纯面向对象的语言，所以线程也有与之对应的类——Thread。

## 3.1、Java中创建线程的两种方式

创建线程必须提供在该线程运行的代码，这在Java中有两种方式实现。

1. 实现Runnable接口。

   Runnable接口代表了线程中可执行的任务，在Runnable接口中只有一个方法：`run()`。实现这个接口的`run()`方法，并创建这个对象，作为参数交给Thread处理。

   ```java
   public class HelloRunnable implements Runnable{
       public void run(){
           System.out.println("Hello");
       }
       public static void main(String[] args){
           new Thread(new HelloRunnable()).start();
       }
   }
   ```

2. 继承Thread类，重写run方法。

   Thread类本身就实现了Runnable对象，但它的`run`方法只检查执行代理的runnable对象的`run`方法。

   ```java
       // Thread类默认的run方法
       public void run() {
           if (target != null) {
               // 代理了target的run方法
               target.run();
           }
       }
   ```

   我们可以继承Thread类，重写它的`run`方法，来提供执行任务：

   ```java
   public class HelloThread extends Thread{
       public void run(){
           System.out.println("Hello");
       }
       public static void main(String[] args){
           new HelloThread().start();
       }
   }
   ```

## 3.2、应该使用哪种方式创建线程呢？

   因为Java是单继承的，如果使用第二种方式重写run方法，那么意味着你就不能继承其他的类来扩展类的功能了。而如果使用第一种方式实现Runnable接口，对你的类没有什么影响，你想继承哪个类就继承哪个类，想继续实现哪个接口可以继续实现。

   所以一般使用实现Runnable接口的方式来创建线程。

   # 4、Thread类的相关方法与线程的状态

   先来看一张神图：

   ![java线程的生命周期](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h885pwmlj20qo0l140e.jpg)

上面这张图中涉及到了`Object.wait`和`Object.notify`这一对方法，这个放在[生产者与消费者](http://blog.csdn.net/holmofy/article/details/76553437)中讨论，我们先把Thread类中的常用方法解决掉！

Thread类和所有类一样有两种方法：类静态方法，实例成员方法(加了删除线的方法表示已经被弃用的方法)。

* 静态方法都是在本线程中执行，如：sleep()，yield()，interrupted()。
* 实例成员方法可以在其他线程执行，当然也可以在本线程中执行(但通常由其他线程调用)，如：interrupt()， join()， ~~destroy()~~， ~~resume()~~， ~~stop()~~， ~~suspend()~~ 。

## 4.1、暂停执行与sleep方法

`Thread.sleep`方法会导致当前线程在指定的时间内暂停执行，这使得处理器可以处理其他的线程任务。

`sleep`方法有两个重载版本：

```java
// 精确到毫秒
public static native void sleep(long millis)

// 精确到纳秒
public static void sleep(long millis, int nanos)
```

但实际上这个纳秒级的睡眠时间是无法精确的，因为它受到底层操作系统的限制(实际上还是调用C语言层面的sleep方法)。

```java
public static void sleep(long millis, int nanos)
throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    // 四舍五入操作
    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }

    sleep(millis);
}
```

另外调用`sleep`方法后的睡眠阶段可以调用`interrupt`方法来中断线程，这时`sleep`方法会抛出`InterruptedException`的异常(下面的线程中断机制会讲到)。所以我们调用`Thread.sleep`方法时经常要用`try catch`包裹。

```java
try{
	Thread.sleep(1000);
}catch(InterruptedException e){
	...
}
```

## 4.2、sleep与被弃用的suspend的区别

`Thread.suspend`方法也是暂停本线程的执行，会导致线程的挂起，`Thread.suspend`挂起必须要`Thread.resume`方法来唤醒。而`Thread.sleep`方法是定时挂起，它会在一段时间后自动还原成就绪态。而且使用`Thread.suspend`和`Thread.resume`方法非常容易造成死锁，因为`Thread.suspend`和`Thread.sleep`方法一样不会释放已经获取的锁。而后面要讲到的`Object.wait`方法在挂起后会释放锁。这也是`Thread.suspend`和`Thread.resume`方法被弃用的原因。

## 4.3、线程中断机制

通常我们会使用一个标志位来控制线程的终止：

```java
public class InterruptTest {
    static class Task implements Runnable {
        boolean stoped;

        public void run() {
            long start = System.currentTimeMillis();
            System.out.println("Thread start at " + start);
            try {
                while (!stoped) {
                    System.out.println("sleep...");
                    Thread.sleep(1600);
                }
            } catch (Exception e) { }
            long end = System.currentTimeMillis();
            System.out.println("Thread finish at " + end);
            System.out.println("Thread execute for " + (end - start));
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Task task = new Task();
        Thread child = new Thread(task);
        child.start();
        Thread.sleep(4000);
        task.stoped = true;
    }
}
```

运行结果：

```shell
Thread start at 1502876260431
sleep...
sleep...
sleep...
Thread finish at 1502876265234
Thread execute for 4803    # 执行时间超过4000毫秒
```

可以看出这种方式有个小问题，如果while循环中有`Thread.sleep`这样的阻塞方法，那么这个线程必须等到该方法返回后才能终止，所以这种自定义标志位的方法有时候并不能达到立马终止线程的目的。

Thread类在底层已经提供了一个类似的标志——中断标志，我们可以通过以下三个方法来对中断标志位进行操作。

|                 方法                  |                   方法描述                   |
| :---------------------------------: | :--------------------------------------: |
| public static boolean interrupted() | 测试当前线程是否已经中断，线程的中断状态由该方法清除。<br/>换句话说，如果连续两次调用该方法，则第二次调用将返回false。 |
|   public boolean isInterrupted()    |       测试线程是否已经中断。线程的中断状态不受该方法的影响。        |
|       public void interrupt()       |                  中断线程。                   |

前两个方法区别在于会不会清除中断状态：

```java
// 清除中断状态
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}
// 不清除中断状态
public boolean isInterrupted() {
    return isInterrupted(false);
}

private native boolean isInterrupted(boolean ClearInterrupted);
```

第二个方法一般由其他线程调用，该方法会将中断标志设为`true`。

如果调用`interrupt`方法时，线程正阻塞在某些阻塞方法时，这些阻塞方法将会立即抛出InterruptedException异常，并将中断状态清空(置为`false`)，这些方法包括前面提到的`Thread.sleep`还有后面要讲到的`Thread.join`和`Object.wait`等方法。

来个例子演示一下：

```java
package cn.hff.functor;

public class InterruptTest {
    static class Task implements Runnable {
        public void run() {
            long start = System.currentTimeMillis();
            System.out.println("Thread start at " + start);
            try {
                System.out.println("start doing something");
                while (!Thread.interrupted()) {
                    System.out.println("sleep...");
                    Thread.sleep(1600); // 中断后直接退出该方法
                }
                System.out.println("finish something");
            } catch (InterruptedException e) {
                System.out.println("Thread interrupt");
            }
            long end = System.currentTimeMillis();
            System.out.println("Thread finish at " + end);
            System.out.println("Thread execute for " + (end - start));
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Task task = new Task();
        Thread child = new Thread(task);
        child.start();
        Thread.sleep(4000);
        child.interrupt();
    }
}
```

执行结果：
```shell
Thread start at 1502876450763
start doing something
sleep...
sleep...
sleep...
Thread interrupt    # 中断后finish something不会打印
Thread finish at 1502876454764
Thread execute for 4001  # 基本能在4000毫秒后结束
```

如果Thread.sleep的异常在while循环内捕捉的话，还需要调用一次interrupt。因为`Thread.sleep`抛出异常时，中断标志为false。代码如下：

```java
public void run() {
	System.out.println("Thread start");
	while (!Thread.interrupted()) {
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {   // 当我们捕获异常时中断标志位已经为false
			System.out.println("Thread interrupt");
			Thread.currentThread().interrupt();// 还原中断标志位
		}
	}
	System.out.println("Thread finish");
}
```

> Java的中断机制让程序有更高的响应性，但需要我们正确理解的是，它并不会真正地中断一个正在运行的线程，只是发出中断请求，然后由线程自己在何时的时刻中断自己，这样的好处是线程能在结束任务前进行相应的收尾工作，比如关闭文件释放资源等。
>
> Java的中断机制[令初学者非常反感](https://www.yegor256.com/2015/10/20/interrupted-exception.html)，因为每次都try_catch会令代码不整洁，而且我们还不能简单的try_catch吞掉异常就不管了，还需要把异常标志位设回去，Guava提供了一个简单处理阻塞方法的工具类——[Uninterruptibles](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/Uninterruptibles.java)

## 4.4、join方法

thread.join把指定的线程加入到当前线程来执行。你可以用“并线(join)”这个词来进行理解：让调用该方法的线程等待thread线程执行完。

![Thread.join](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h88w76uoj20pq0gowfi.jpg)

和sleep方法一样，这个方法会也会抛出InterruptedException中断异常。

## 4.5、Java守护线程和setDaemon方法

看到“守护线程”这个概念，你可能会联想到Linux中的守护进程。Java程序由于运行在虚拟机上，虚拟机一般作为一个进程运行的，所以这里所说的守护线程和Linux中说的守护进程没什么关系，但在概念上也有很多与之类似的地方。

**如果一个程序主线程结束了，但还有非守护线程没有结束，那主线程会等待非守护线程的结束。**

**如果一个程序主线程结束了，非守护线程也结束了，但还有守护线程没有结束，主线程不会等待守护线程。**

Java里面最典型的守护线程就是垃圾回收线程(GC，garbage collector)。

我们先来写一个简单的例子：

```java
public class ThreadTest {
	static void printMessage(String msg) {
		String threadName = Thread.currentThread().getName();
		System.out.println(threadName + ": " + msg);
	}

	static class Task implements Runnable {
		public void run() {
			for (int i = 0; i < 10; i++) {
				try { Thread.sleep(500); } catch (InterruptedException e) {}
				printMessage("loop" + i);
			}
			printMessage("finish!");
		}
	}

	public static void main(String[] args)  throws InterruptedException {
		Thread thread = new Thread(new Task());
		thread.start();
		Thread.sleep(1000);
		printMessage("finish!");
	}
}
```

运行结果：

```shell
Thread-0: loop0
Thread-0: loop1
main: finish!
Thread-0: loop2
Thread-0: loop3
Thread-0: loop4
Thread-0: loop5
Thread-0: loop6
Thread-0: loop7
Thread-0: loop8
Thread-0: loop9
Thread-0: finish!
```

>  你会发现主线程已经结束了但它还在等待子线程的执行。

而如果我们添加`setDaemon(true)`后再执行一次：

```java
	public static void main(String[] args) throws InterruptedException {
		Thread thread = new Thread(new Task());
		thread.setDaemon(true);
		thread.start();
		Thread.sleep(1000);
		printMessage("finish!");
	}
```

运行结果：

```java
Thread-0: loop0
Thread-0: loop1
main: finish!
```

> 子线程后面的循环将不会执行。这就是守护线程与非守护线程的区别。

需要注意：`thread.setDaemon()`方法必须要在`thread.start()`之前调用，否则将会报IllegalThreadStateException异常。可以看一下setDaemon的源代码：

```java
    public final void setDaemon(boolean on) {
        checkAccess();
        if (isAlive()) {
            throw new IllegalThreadStateException();
        }
        daemon = on;
    }
```

另外我们可以手动调用`Thread.join`方法，让主线程等待守护子线程的执行

```java
	public static void main(String[] args) throws InterruptedException {
		Thread thread = new Thread(new Task());
		thread.setDaemon(true);
		thread.start();
		Thread.sleep(1000);
		printMessage("finish!");
		thread.join(); // 让主线程等待守护线程的执行
	}
```

# 5、总结性的小例子

```java
public class SimpleThreads {

	static void printMessage(String message) {
		String threadName = Thread.currentThread().getName();
		System.out.format("%s: %s%n", threadName, message);
	}

	private static class MessageLoop implements Runnable {
		String messages[] = {
				"message1", "message2", "message3", "message4"
		};

		public void run() {
			try {
				for (int i = 0; i < messages.length; i++) {
					Thread.sleep(4000);
					printMessage(messages[i]);
				}
			} catch (InterruptedException e) {
				printMessage("我还没干完呢!");
			}
		}
	}

	public static void main(String args[]) throws InterruptedException {
		long patience = 1000 * 15;

		long startTime = System.currentTimeMillis();
		printMessage("开始启动MessageLoop线程");
		Thread t = new Thread(new MessageLoop());
		t.start();

		printMessage("开始等待MessageLoop线程的完成");

		long interval = 0;
		while (t.isAlive()) {
			// 最多等MessageLoop线程1秒钟
			t.join(1000);
			interval = System.currentTimeMillis() - startTime;
			if ((interval > patience) && t.isAlive()) {
				printMessage("不想再等你了!");
				t.interrupt();
				// 不用等太久
				t.join();
			} else {
				printMessage("我已经等了" + interval + "毫秒了...");
			}
		}
		printMessage("已经结束了!");
	}
}
```



参考链接：

《[Java并发编程实战](https://book.douban.com/subject/10484692/)》

https://www.baeldung.com/java-process-api

https://www.yegor256.com/2015/10/20/interrupted-exception.html
