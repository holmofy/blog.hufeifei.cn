---
title: Java多线程复习与巩固（七）--任务调度线程池ScheduledThreadPoolExecutor
date: 2017-06-20
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
* [ThreadPoolExecutor最佳实践--如何选择线程数](https://blog.hufeifei.cn/2018/07/Java/ThreadPoolExecutor-best-practice-thread-size/)
* [ThreadPoolExecutor最佳实践--如何选择队列](https://blog.hufeifei.cn/2018/08/Java/ThreadPoolExecutor-best-practice-queue/)

---

前篇：[《Java多线程复习与巩固(六)--线程池ThreadPoolExecutor详解》](http://blog.csdn.net/holmofy/article/details/77411854)

##  为什么要使用ScheduledThreadPoolExecutor

在[《Java多线程复习与巩固(二)--线程相关工具类Timer和ThreadLocal的使用》](https://blog.hufeifei.cn/2017/06/Java/multithread/02-Thread-Utility/)提到过，Timer可以实现**指定延时调度任务**，还可以实现**任务的周期性执行**。但是Timer中的所有任务都是由一个TimerThread执行，也就是说**Timer是单线程**执行任务。单线程执行任务有一个致命的缺点：**当某些任务的执行特别耗时，后续的任务无法在预定的时间内得到执行，前一个任务的延迟或异常将影响到后续的任务；另外TimerThread没有做异常处理，一个任务出现异常将会导致整个Timer线程结束**。

由于Timer单线程的种种缺点，这个时候我们就需要让线程池去执行这些任务。

##  使用Executors工具类

Executors是线程池框架提供给我们的创建线程池的工具类，FixedThreadPool，SingleThreadExecutor，CachedThreadPool都是[上一篇文章中的ThreadPoolExecutor对象](http://blog.csdn.net/holmofy/article/details/77411854)。

他还有另外两个方法：

```java
// 创建(可计划的)任务延时执行线程池
public static ScheduledExecutorService newScheduledThreadPool();
// 单线程版的任务计划执行的线程池
// 和Timer有点类似，但区别在于出现异常后SingleThreadScheduledExecutor会重新创建一个工作线程
public static ScheduledExecutorService newSingleThreadScheduledExecutor();
```

从下面的继承图我们知道ScheduledThreadPoolExecutor就是ScheduledExecutorService接口的实现类。

![线程池ThreadPoolExecutor相关类继承图](http://img-blog.csdn.net/20170819141036970?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##  构造ScheduledThreadPoolExecutor对象

先看一下ScheduledThreadPoolExecutor的几个构造函数

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
    ...
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }
    ...
}
```

从上面的代码可以看出ScheduledThreadPoolExecutor都是直接调用的父类ThreadPoolExecutor的构造函数。

我们结合[上一篇对ThreadPoolExecutor构造参数的解释](http://blog.csdn.net/holmofy/article/details/77411854#t2)对ScheduledThreadPoolExecutor的几个参数进行分析，主要有以下几个参数比较特殊：

* maximumPoolSize：线程池允许的最大线程数为`Integer.MAX_VALUE`，也就意味着ScheduledThreadPoolExecutor对线程数没有限制。这个是必须的，因为一旦对线程数有了限制，必定会存在任务等待调度的情况，有等待就可能会存在任务延时，所以最大线程数不能有限制。
* keepAliveTime和unit：0 NANOSECONDS，0纳秒，也就是说一旦有空闲线程会立即销毁该线程对象。
* workQueue：DelayedWorkQueue是ScheduledThreadPoolExecutor的内部类，它也是实现按时调度的核心。

##  二叉堆DelayedWorkQueue

DelayedWorkQueue和`java.util.concurrent.DelayQueue`有着惊人的相似度：

* DelayedWorkQueue实现了一个容量无限的二叉堆，DelayQueue底层使用PriorityQueue实现二叉堆各种操作。
* DelayedWorkQueue存储了`java.util.concurrent.RunnableScheduledFuture`接口的实现类，DelayQueue存储`java.util.concurrent.Delayed`接口的实现类，这两个接口有以下的继承关系（其中`ScheduledThreadPoolExecutor`内部类`ScheduledFutureTask`就实现了`RunnableScheduledFuture`接口）

![Future继承图](http://img-blog.csdn.net/20170819141457532?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##  为什么使用二叉堆

大学学过数据结构的应该学过堆排序吧：堆排序就是用小顶堆(或大顶堆)实现最小(或最大)的元素往堆顶移动。这里的**DelayedWorkQueue就是使用二叉堆获取堆中延时最短的任务**。具体的比较策略让我们看下面这个方法：

**ScheduledThreadPoolExecutor.ScheduledFutureTask.compareTo()**

```java
        public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                // 优先比较任务执行的时间
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                // 时间相同比较任务的先后顺序(FIFO)
                // 这个sequenceNumber在创建ScheduledFutureTask的时候
                // 由一个AtomicLong生成
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }
```

##  为什么不用DelayQueue的二叉堆实现

`java.util.concurrent.DelayQueue`就是根据延时获取元素的，那为什么不直接用`DalayQueue`而重新定义一个`DelayedWorkQueue`呢。这个问题本质上就是在问`DelayQueue`与`DelayedWorkQueue`的区别，我们看一下`DelayedWorkQueue`注释中的一段话：

```java
    static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {

        /*
         * A DelayedWorkQueue is based on a heap-based data structure
         * like those in DelayQueue and PriorityQueue, except that
         * every ScheduledFutureTask also records its index into the
         * heap array. This eliminates the need to find a task upon
         * cancellation, greatly speeding up removal (down from O(n)
         * to O(log n)), and reducing garbage retention that would
         * otherwise occur by waiting for the element to rise to top
         * before clearing. But because the queue may also hold
         * RunnableScheduledFutures that are not ScheduledFutureTasks,
         * we are not guaranteed to have such indices available, in
         * which case we fall back to linear search. (We expect that
         * most tasks will not be decorated, and that the faster cases
         * will be much more common.)
         *
         * All heap operations must record index changes -- mainly
         * within siftUp and siftDown. Upon removal, a task's
         * heapIndex is set to -1. Note that ScheduledFutureTasks can
         * appear at most once in the queue (this need not be true for
         * other kinds of tasks or work queues), so are uniquely
         * identified by heapIndex.
         */
        ...
    }
```

大致翻译过来：

```java
DelayedWorkQueue类似于DelayQueue和PriorityQueue，是基于“堆”的一种数据结构。
区别就在于ScheduledFutureTask记录了它在堆数组中的索引，这个索引的好处就在于：
取消任务时不再需要从数组中查找任务，极大的加速了remove操作，时间复杂度从O(n)降低到了O(log n)，
同时不用等到元素上升至堆顶再清除从而降低了垃圾残留时间。
但是由于DelayedWorkQueue持有的是RunnableScheduledFuture接口引用而不是ScheduledFutureTask的引用，
所以不能保证索引可用，不可用时将会降级到线性查找算法(我们预测大多数任务不会被包装修饰，因此速度更快的情况更为常见)。

所有的堆操作必须记录索引的变化 ————主要集中在siftUp和siftDown两个方法中。一个任务删除后他的headIndex会被置为-1。
注意每个ScheduledFutureTask在队列中最多出现一次(对于其他类型的任务或者队列不一定只出现一次)，
所以可以通过heapIndex进行唯一标识。
```

这里有几个地方可能有疑问：

**1. remove操作的时间复杂度从O(n)降低到了O(log n)**

```java
        public boolean remove(Object x) {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                // 因为在heapIndex中存储了索引
                // indexOf的时间复杂度从线性搜索的O(n)
                // 降低到了常量O(1)
                int i = indexOf(x);
                if (i < 0)
                    return false;

                // heapIndex标记为-1,表示已删除
                setIndex(queue[i], -1);
                int s = --size;
                RunnableScheduledFuture<?> replacement = queue[s];
                queue[s] = null;
                // siftUp和siftDown操作完全二叉树时间复杂度为O(log n)
                // 综合前面的O(1)+O(log n) ==> O(log n)
                if (s != i) {
                    siftDown(i, replacement);
                    if (queue[i] == replacement)
                        siftUp(i, replacement);
                }
                return true;
            } finally {
                lock.unlock();
            }
        }
        private int indexOf(Object x) {
            if (x != null) {
                if (x instanceof ScheduledFutureTask) {
                    // 如果是ScheduledFutureTask，可用heapIndex直接索引
                    int i = ((ScheduledFutureTask) x).heapIndex;
                    if (i >= 0 && i < size && queue[i] == x)
                        return i;
                } else {
                    // 否则使用线性查找
                    for (int i = 0; i < size; i++)
                        if (x.equals(queue[i]))
                            return i;
                }
            }
            return -1;
        }
```

**2. 任务的包装修饰**

包装修饰主要是指两个`ScheduledThreadPoolExecutor.decorateTask`方法。这部分内容放在文末“扩展ScheduledThreadPoolExecutor的功能”时讲。

##  任务的提交

```java
    public void execute(Runnable command) {
        schedule(command, 0, NANOSECONDS);
    }
    public Future<?> submit(Runnable task) {
        return schedule(task, 0, NANOSECONDS);
    }
    public <T> Future<T> submit(Runnable task, T result) {
        return schedule(Executors.callable(task, result), 0, NANOSECONDS);
    }
    public <T> Future<T> submit(Callable<T> task) {
        return schedule(task, 0, NANOSECONDS);
    }
```

我们看到原来ThreadPoolExecutor中的几个提交方法都被重写了，最终调用了个的都是`schedule`方法，并且这几个方法的延时都为0纳秒。

##  schedule

既然前面任务的提交全部都是交给schedule方法执行，那么让我们看一下schedule相关的几个方法

> 下面的几个方法也是`ScheduledExecutorService`接口扩展的几个方法

下面**需要注意的主要是`scheduleAtFixedRate`和`scheduleWithFixedDelay`两个方法的区别**：

```java
    // 触发时间
    private long triggerTime(long delay, TimeUnit unit) {
        // 时间统一使用纳秒单位
        return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
    }
    long triggerTime(long delay) {
        // 当前时间加上延迟时间
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    }
    // 在指定的时间执行一次，没有返回值
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<?> t = decorateTask(command,
            // 将Runnable接口对象封装成ScheduledFutureTask
            new ScheduledFutureTask<Void>(command, null, // Runnable给的返回值为null
                                          triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
    // 在指定的时间执行一次，有返回值
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            // 将Callable接口对象封装成ScheduledFutureTask
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }

    // 创建并执行一个周期性的任务，这个任务在initialDelay时间后生效
    // 第一次initialDelay,然后initialDelay+period,再然后initialDelay + 2 * period
    // 依此类推往下执行
    // 1. 如果执行过程中出现异常，后续的执行将会终止
    //    否则后续的任务会一直执行除非任务调用cancel方法取消或者线程池终止了
    // 2. 如果该任务任意一次执行超过了它的周期，那么后续的执行计划将会推迟
    //    绝对不会一个任务同时由两个线程执行
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        // 周期执行的任务
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          // 正数：固定周期执行
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

    // 创建并执行一个周期性的任务,任务在initialDelay时间后生效
    // 后续的执行时间在前一次任务执行完成后延时delay时间后执行
    // 第一次执行时间在initialDelay
    // 如果第一次执行耗时T1,那么第二次执行时间在initialDelay+T1+delay,
    // 如果第二次执行耗时T2,那么第三次执行时间在initialDelay+T1+T2+2*delay
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        // 延迟执行的任务
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          // 负数：固定延迟执行
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
    // 下面两个方法时留给子类实现的，默认直接返回task
    protected <V> RunnableScheduledFuture<V> decorateTask(
        Runnable runnable, RunnableScheduledFuture<V> task) {
        return task;
    }
    protected <V> RunnableScheduledFuture<V> decorateTask(
        Callable<V> callable, RunnableScheduledFuture<V> task) {
        return task;
    }
```

![fixRate与fixDelay的区别](http://tva1.sinaimg.cn/large/bda5cd74gy1ftalspuvhpj20t10h9q87.jpg)

总结来说就是**fixRate是以任务开始时间计算间隔，而fixDelay是以任务结束时间计算间隔**。

##  delayedExecute

上面的几个方法都是将`runnable`或`callable`包装成`ScheduledFutureTask`对象，最终都是丢给`delayedExecute`方法去执行：

```java
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        // 如果线程池已经SHUTDOWN，则拒绝任务
        if (isShutdown())
            reject(task);
        else {
            // 入队
            super.getQueue().add(task);
            // 再次检查
            if (isShutdown() &&
                // 检查线程池当前状态是否能继续执行任务
                // shutdown状态下是否把未完成的任务执行完
                !canRunInCurrentRunState(task.isPeriodic()) &&
                // 不能执行则移除任务
                remove(task))
                // 移除失败则取消任务
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
    // 这个方法和ThreadPoolExecutor.prestartCoreThread方法基本一致
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            // 添加核心线程
            addWorker(null, true);
        else if (wc == 0)
            // wc==0,说明corePoolSize==0,也就是所有的线程都是普通线程
            // 添加普通线程
            addWorker(null, false);
    }
```

## 0. ScheduledFutureTask.run

添加线程后，线程肯定会从阻塞队列中获取任务，并执行任务的run方法，也就是ScheduledFutureTask的run方法：

```java
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        ...
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
                // 不是周期性执行，则直接执行
                ScheduledFutureTask.super.run();

            // 否则就是周期性执行：执行完一个周期后，重置任务的状态
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime(); // 设置下一次运行的时间
                reExecutePeriodic(outerTask);
            }
        }
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                // 是调用scheduleAtFixedRate创建的任务，固定周期
                // 直接将上一次的时间加上周期
                time += p;
            else
                // 是调用scheduleWithFixedDelay创建的任务，固定延迟
                // 当前时间加上延迟
                time = triggerTime(-p);
        }
    }
```

## 1. ScheduledThreadPoolExecutor的其他配置项

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
    /**
     * false：在线程池SHUTDOWN后取消已存在的周期任务
     * true: 线程池SHUTDOWN后，继续执行已存在的周期任务
     */
    private volatile boolean continueExistingPeriodicTasksAfterShutdown;

    /**
     * false: 在线程池SHUTDOWN后取消已存在的非周期性任务
     * true: 线程池SHUTDOWN后，继续执行已存在的非周期性任务
     */
    private volatile boolean executeExistingDelayedTasksAfterShutdown = true;

    /**
     * true: 调用ScheduledFutureTask.cancel方法后将任务从队列中remove
     */
    private volatile boolean removeOnCancel = false;

    // 省略这三个属性的getter/setter方法
}
```

## 2. 继承ScheduledThreadPoolExecutor对任务进行包装

ThreadPoolExecutor提供了beforeExecute,afterExecute,terminated三个钩子方法让我们重载以进行扩展。

ScheduledThreadPoolExecutor也提供了两个方法给我们扩展，下面是JDK文档提供的一个简单例子：

```java
public class CustomScheduledExecutor extends ScheduledThreadPoolExecutor {

  static class CustomTask<V> implements RunnableScheduledFuture<V> { ... }

  // 我们可以在这两个方法中对任务进行修改或包装
  protected <V> RunnableScheduledFuture<V> decorateTask(
               Runnable r, RunnableScheduledFuture<V> task) {
      return new CustomTask<V>(r, task);
  }
  protected <V> RunnableScheduledFuture<V> decorateTask(
               Callable<V> c, RunnableScheduledFuture<V> task) {
      return new CustomTask<V>(c, task);
  }
  // ... add constructors, etc.
}
```

## 3. ScheduledThreadPoolExecutor尚有的缺点

ScheduledThreadPoolExecutor是使用纳秒为单位进行任务调度，它底层使用的是`System.nanoTime()`来获取时间：

```java
    final long now() {
        return System.nanoTime();
    }
```

这个时间是相对于JVM虚拟机启动的时间，这个纳秒值在$2^{63}纳秒 \approx 292年$后会溢出(几乎可以忽略溢出问题)，ScheduledThreadPoolExecutor也对溢出进行了处理：

```java
    long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    }
    private long overflowFree(long delay) {
        Delayed head = (Delayed) super.getQueue().peek();
        if (head != null) {
            // 溢出会影响compareTo方法的比较
            long headDelay = head.getDelay(NANOSECONDS);
            if (headDelay < 0 && (delay - headDelay < 0))
                delay = Long.MAX_VALUE + headDelay;
        }
        return delay;
    }
```

既然ScheduledThreadPoolExecutor已经处理了，那还有什么问题吗。问题就在于我们无法使用`yyyy-MM-dd HH-mm-ss`这种**精确时间点**的方式进行任务的调度。

不过在[**SpringTask**](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/integration.html#scheduling) 以及 [**Quartz**](http://quartz-scheduler.org/)等框架中已经解决了这个问题，并提供了**cron表达式**来精确任务的调度时间。后续如果有机会对这些框架的原理进行分析。

> SpringTask既可以单独使用也可以整合Quartz使用，除了Quartz还有一个轻量级的[Cron4j](http://www.sauronsoftware.it/projects/cron4j/manual.php)可以实现任务调度，不过Cron4j并没有用线程池(估计那时候java5还没出来)，每个任务都会去创建一个新线程。

