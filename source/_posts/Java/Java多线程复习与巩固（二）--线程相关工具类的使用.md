---
title: Java多线程复习与巩固(二)--线程相关工具类的使用
date: 2017-06-14 20:30
categories: JAVA
---

# 定时器(Timer类)

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
            try{ Thread.sleep(period); }catch(InterruptException e){}
            task...
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
        try{ Thread.sleep(delay); }catch(InterruptException e){}
        task...
    }
}
```

这就像Linux中的`crontab`命令(指定的时间周期执行若干次)和`at`命令(定时执行一次)一样。

其实Java中已经将这些功能封装在一个Timer中。

![Timer相关类之间的关系](http://img.blog.csdn.net/20170614212002312?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

但是`TaskQueue`和`TimerThread`我们是看不到的，我们能用的只有`Timer`和`TimerTask`这两个类。

`TimerTask`就表示我们要执行定时的任务，它有四种状态：

 * Virgin(处女状态)：代表TimerTask刚创建，没有被添加到定时器中。

 * Scheduled(计划调度状态)：代表该TimerTask已经添加到Timer的计划表中了(被添加到TimerQueue中)

 * Cancelled(已取消状态)：代表该TimerTask已经被取消，不能再被Timer定时器调用。

 * Executed(已执行状态)：代表该TimerTask已经被Timer执行完了。

状态之间的转换图如下：

![TimerTask的状态转换](http://img.blog.csdn.net/20170614212034938?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 除了上图出现的三个可用的方法，Timer中还有一个Timer.purge方法，它负责将TimerQueue中已经取消的任务清除掉，也就是把Cancelled状态的任务清除。另外如果你没调用这个方法清除Cancelled状态的任务，TimerThread会自动帮你把Cancelled和Executed状态的TimerTask任务清除掉。

## 示例：

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

> 虽然Java 5.0以后有了ScheduledThreadPoolExecutor也能进行定时任务的执行，但是它是用线程池实现的，而Timer是单线程的(就是上面类图中的TimerThread)，单线程的缺点是如果有一个定时任务特别耗时，将会导致后续的任务不能在准确的时间得到执行，所以在[稍后的文章中还会提到ScheduledThreadPoolExecutor](https://blog.csdn.net/holmofy/article/details/79344914)。当然对于一些小功能来说没必要使用线程池，Timer足以应付。

# 线程局部变量(ThreadLocal类)

线程局部变量用于为每个线程维护一个变量的副本，使得每个线程都可以访问自己线程中的副本对象，而不会对其他线程中的副本造成影响，而且在访问这些变量时为我们提供了统一的访问方式。线程局部变量ThreadLocal有一个子类：InheritableThreadLocal。InheritableThreadLocal类会继承父线程的已经存储的副本，也就是说子线程会和父线程共享父线程中已有的副本，但这也使得子线程访问父线程要考虑同步问题，所以这个类实际上也不常用。

下面先看一张Thread与ThreadLocal之间的关系图。

![ThreadLocal与Thread的关系](http://img.blog.csdn.net/20170614212105209?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ThreadLocalMap以ThreadLocal对象作为Key，将线程要存的副本作为Value（ThreadLocalMap是本质上是一个哈希表，不过它比HashMap简单多了，它以“再哈希法”来解决哈希碰撞的问题）：

ThreadLocal类很简单只有四个可用的方法`initialValue`, `get`, `set`, `remove`：

```java
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
```

## 示例：

示例一：

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadLocalTest {
	static class ClientIdGenerator extends ThreadLocal<Integer> {
		AtomicInteger clientCount = new AtomicInteger();

		@Override
		protected Integer initialValue() {
			return clientCount.incrementAndGet();
		}
	}

	public static void main(String[] args) {
		ClientIdGenerator clientId = new ClientIdGenerator();
		for (int i = 0; i < 10; i++) {
			new Thread(new Runnable() {
				public void run() {
					try {
						Thread.sleep(10);
					} catch (InterruptedException e) {}
					System.out.println(Thread.currentThread().getName() + ": I am client" + clientId.get());
				}
			}).start();
		}
	}
}
```

实例二：

Spring中日期转换工具：因为SimpleDateFormat不是线程安全的，多线程同时解析可能会出现问题，所以需要使用ThreadLocal为每个线程创建一个副本

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



