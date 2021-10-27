---
title: Java多线程复习与巩固（二）--线程相关工具类的使用
date: 2017-06-15
categories: JAVA
keywords:
- Java 多线程编程
---

**系列文章：**
* [Java多线程复习与巩固（一）--线程基本使用](https://blog.hufeifei.cn/2017/06/Java/multithread/01-Thread-Basic/)
* [Java多线程复习与巩固（二）--线程相关工具类的使用](https://blog.hufeifei.cn/2017/06/Java/multithread/02-Thread-Utility/)
* [Java多线程复习与巩固（三）--线程同步](https://blog.hufeifei.cn/2017/06/Java/multithread/03-Synchronized/)
* [Java多线程复习与巩固（四）--synchronized的实现](https://blog.hufeifei.cn/2017/06/Java/multithread/04-Synchronized-Implement/)
* [Java多线程复习与巩固（五）--生产者消费者问题（第一部分）](https://blog.hufeifei.cn/2017/06/Java/multithread/05-Provider-Consumer/)
* [Java多线程复习与巩固（六）--线程池ThreadPoolExecutor详解](https://blog.hufeifei.cn/2017/06/Java/multithread/06-ThreadPoolExecutor/)
* [Java多线程复习与巩固（七）--任务调度线程池ScheduledThreadPoolExecutor](https://blog.hufeifei.cn/2017/06/Java/multithread/07-ScheduledThreadPoolExecutor/)
* [Java多线程复习与巩固（八）--原子性操作与原子变量](https://blog.hufeifei.cn/2017/06/Java/multithread/08-Atomic/)
* [Java多线程复习与巩固（九）--volatile关键字与CAS操作](https://blog.hufeifei.cn/2017/06/Java/multithread/09-volatile-CAS/)

# 1、定时器(Timer类)

如果我们需要让某个任务在另一个线程中周期性的执行，或者让它在某个时刻执行一次。这时我们可能会写这样的代码：

周期任务：

```java
// 周期任务
public class PeriodTask implements Runnable{
    private long period = 1000;
    private boolean running = true;
    public void cancel(){
        running = false;
    }
    public void setPeriod(long period){
        this.period = period;
    }
    public long getPeriod(){
        return this.period;
    }
    public void run(){
        while(running){
            // 周期循环执行某个任务
            try{ Thread.sleep(period); }catch(InterruptException e){}
            doTask();
        }
    }
}
```

定时任务：

```java
// 定时任务
public class TimeTask implements Runnable{
    private long delay;
    public void setDelay(long delay){
        this.delay = delay;
    }
    public long getDelay(){
        return this.delay;
    }
    public void run(){
        // 延迟执行某个定时任务
        try{ Thread.sleep(delay); }catch(InterruptException e){}
        doTask();
    }
}
```

这两者的功能就像Linux中的`crontab`命令(指定的时间周期执行若干次)和`at`命令(定时执行一次)一样。

其实Java中已经将这些功能封装在一个Timer中。

![Timer相关类之间的关系](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h945u9dbj20j10et3z6.jpg)

但是`TaskQueue`和`TimerThread`是包内私有的，无法直接使用，我们能用的只有`Timer`和`TimerTask`这两个类。

`TimerTask`就表示我们要执行定时的任务，它有四种状态：

 * Virgin(处女状态)：代表TimerTask刚创建，没有被添加到定时器中。

 * Scheduled(计划调度状态)：代表该TimerTask已经添加到Timer的计划表中了(被添加到TimerQueue中)

 * Cancelled(已取消状态)：代表该TimerTask已经被取消，不能再被Timer定时器调用。

 * Executed(已执行状态)：代表该TimerTask已经被Timer执行完了。

状态之间的转换图如下：

![TimerTask的状态转换](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h95jwmmhj20g70ch0tc.jpg)

> 除了上图出现的三个可用的方法，Timer中还有一个Timer.purge方法，它负责将TimerQueue中已经取消的任务清除掉，也就是把Cancelled状态的任务清除。另外如果你没调用这个方法清除Cancelled状态的任务，TimerThread会自动帮你把Cancelled和Executed状态的TimerTask任务清除掉。

## 1.1、示例：

```java
import java.util.Timer;
import java.util.TimerTask;

public class TimerTest {
	static void printMessage(String msg) {
		long currTime = System.currentTimeMillis();
		System.out.println(currTime + ": " + msg);
	}

	static class Task1 extends TimerTask {
		public void run() {
			printMessage("task1");
		}
	}

	static class Task2 extends TimerTask {
		public void run() {
			printMessage("task2");
		}
	}

	static class Task3 extends TimerTask {
		public void run() {
			printMessage("task3");
		}
	}

	public static void main(String[] args) throws InterruptedException {
		Timer timer = new Timer();
		TimerTask task1 = new Task1();
		timer.schedule(task1, 3000);// 延迟三秒执行
		printMessage("task1 scheduled");
		TimerTask task2 = new Task2();
		timer.schedule(task2, 2000, 1000);// 2秒后以1秒周期执行
		printMessage("task2 scheduled");
		TimerTask task3 = new Task3();
		timer.schedule(task3, 1000, 2000);// 1秒后以2秒周期执行
		printMessage("task3 scheduled");
		Thread.sleep(6000);// 主线程睡眠十秒
		task2.cancel();// 取消定时任务二
		timer.purge();
		printMessage("task2 cancelled");
		Thread.sleep(6000);// 主线程睡眠6秒
		task1 = new Task1();
		timer.schedule(task1, 2000, 1000);// 2秒后以1秒周期执行
		printMessage("task1 scheduled again");
		Thread.sleep(6000);
		timer.cancel();
	}
}
```

> 虽然Java 5.0以后有了ScheduledThreadPoolExecutor也能进行定时任务的执行，但是它是用线程池实现的，而Timer是单线程的(就是上面类图中的TimerThread)，单线程的缺点是如果有一个定时任务特别耗时，将会导致后续的任务延迟，不能在预定的准确时间得到执行，所以在[稍后的文章中还会提到ScheduledThreadPoolExecutor](https://blog.csdn.net/holmofy/article/details/79344914)。当然对于一些小功能来说没必要使用线程池，Timer足以应付。

# 2、线程局部变量(ThreadLocal类)

**线程局部变量用于为每个线程维护一个变量的副本，使得每个线程都可以访问自己线程中的副本对象，而不会对其他线程中的副本造成影响，而且在访问这些变量时为我们提供了统一的访问方式**。

线程局部变量ThreadLocal有一个子类：InheritableThreadLocal。InheritableThreadLocal类会继承父线程的已经存储的副本，也就是说子线程会和父线程共享父线程中已有的副本，但这也使得子线程访问父线程要考虑同步问题，而且现在大多数系统会使用线程池技术，这已经不仅仅是InheritableThreadLocal能够解决的了，所以这个类实际上也不常用。

ThreadLocal可以简单的看作`Map<Thread,Value>`的结构，但实际上是Thread对象内部维护了一个`Map<ThreadLocal,Value>`的字段(`ThreadLocalMap`)来保存这个线程拥有的局部变量，这样做的原因是线程在销毁时，这个线程对象及其相应的局部变量都能被GC回收。这从Entry继承自WeakReference也能看得出来。

```java
public class Thread implements Runnable {
  //...
  ThreadLocal.ThreadLocalMap threadLocals = null;
  ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
  //...
}

public class ThreadLocal<T> {
  //...
  static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
      }
    }
    //...
  }
}
```

> 从这一点来看ThreadLocal更应该翻译成"线程上下文变量"。

下面具体看看ThreadLocal怎么实现的。先看一张Thread与ThreadLocal之间的关系图。

![ThreadLocal与Thread的关系](http://tva1.sinaimg.cn/large/bda5cd74ly1g2h97s5mioj20qk0is40j.jpg)

ThreadLocalMap以ThreadLocal对象作为Key，将线程要存的副本作为Value（ThreadLocalMap是本质上是一个哈希表，不过它比HashMap简单多了，它以“再哈希法”来解决哈希碰撞的问题，并以2倍增长的方式扩容）：

ThreadLocal类很简单只有四个可用的方法`initialValue`, `get`, `set`, `remove`：

```java
public class ThreadLocal<T> {
  // ...
  // 这个方法需要我们继承ThreadLocal然后进行重写。
  protected T initialValue() { return null; }
  public T get() {
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      // 以ThreadLocal为Key，取出Entry
      ThreadLocalMap.Entry e = map.getEntry(this);
      if (e != null) {
        @SuppressWarnings("unchecked")
        // 获取Entry中的Value值
        T result = (T)e.value;
        return result;
      }
    }
    // 为空时对变量进行初始化
    return setInitialValue();
  }
  private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else
      createMap(t, value);
    return value;
  }
  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      // 如果当前线程已经有ThreadLocalMap则直接将value存进入
      map.set(this, value);
    else
      // 当前线程还没有ThreadLocalMap对象，则创建一个。
      createMap(t, value);
  }
  void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
      // 删除当前线程中当前ThreadLoca对应的Entry。
      m.remove(this);
  }
  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }
}
```

## 2.1、示例：

示例一：

Spring中日期转换工具：因为[SimpleDateFormat不是线程安全的](https://www.javacodegeeks.com/2010/07/java-best-practices-dateformat-in.html)，多线程同时解析可能会出现问题，所以使用ThreadLocal为每个线程创建一个副本，让每个线程使用不同的SimpleDateFormat从而保证线程安全性。

```java
public class DateConverter implements Converter<String, Date> {

    private static final String PATTERN = "yyyy-MM-dd HH:mm:ss";

    private static final ThreadLocal<SimpleDateFormat> LOCAL = new ThreadLocal<>() {
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat(PATTERN);
        }
    };

    public Date convert(String source) {
        try {
            return LOCAL.get().parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }

}
```

示例二：

Spring提供的[动态数据源](https://spring.io/blog/2007/01/23/dynamic-datasource-routing/)能让我们方便的在多个数据库连接中随意切换，而不影响代码结构，这也需要我们使用ThreadLocal对象。

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

// AbstractRoutingDataSource有一个Map<Object, DataSource>字段保存了多个数据库连接
// 当调用getConnection()获取数据库连接时，会去调用determineCurrentLookupKey()方法拿到key
// 从而决定使用那个DataSource获取数据库连接。
public class DataSourceRouter extends AbstractRoutingDataSource {

   @Override
   protected Object determineCurrentLookupKey() {
      // 我们可以用任何对象作为Key
      return DataSourceSelector.dataSourceKey();
   }
}

// 这个类保存每个线程对应的数据库连接的Key
public class DataSourceSelector {
   
   private static final ThreadLocal<Integer> dataSourceId = 
            new ThreadLocal<Integer>();

   public static Integer dataSourceKey() {
      return dataSourceId.get();
   }
	
   public static void selectDataSource(int serverId) {
      dataSourceId.set(serverId);
   }
  
   public static void selectDataSourceByUser(User user){
      dataSourceId.set(user.getServerId());
   }

   public static void clear() {
      dataSourceId.remove();
   }
}
```

Spring的AbstractRoutingDataSource为我们提供了一个简便的分库策略。比如我们要对用户分库：

```java
@Data
public class User {
   private int id;
   private UserType type;
   // private ...
   public int getServerId() {
     // 按照用户的id分成8个数据库，但要求多个数据库的用户id全局唯一。
     return id % 8;
     // 或者根据用户的类型分库
     // 不同的UserType对应不同的serverId
     // return type.serverId();
   }
}
```



参考链接：

https://www.javacodegeeks.com/2010/07/java-best-practices-dateformat-in.html

https://spring.io/blog/2007/01/23/dynamic-datasource-routing/