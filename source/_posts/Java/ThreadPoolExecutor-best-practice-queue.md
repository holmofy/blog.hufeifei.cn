---
title: ThreadPoolExecutor最佳实践--如何选择队列
date: 2018-08-12
categories: JAVA
keywords:
- ThreadPoolExecutor最佳实践
- 线程池
- 队列
---

前一篇文章《[如何选择线程数](https://blog.csdn.net/Holmofy/article/details/81271839)》讲了如何决定线程池中线程个数，这篇文章讨论“如何选择工作队列”。

再次强调一下，ThreadPoolExecutor最核心的四点：

1、当有任务提交的时候，会创建核心线程去执行任务（即使有核心线程空闲）；

2、当核心线程数达到**corePoolSize**时，后续提交的都会进BlockingQueue中排队；

3、当BlockingQueue满了(offer失败)，就会创建临时线程(临时线程空闲超过一定时间后，会被销毁)；

4、当线程总数达到**maximumPoolSize**时，后续提交的任务都会被RejectedExecutionHandler拒绝。

# 1、BlockingQueue

线程池中工作队列由BlockingQueue实现类提供功能，BlockingQueue定义了这么几组方法：

<table BORDER CELLPADDING=3 CELLSPACING=1>
  <caption>Summary of BlockingQueue methods</caption>
   <tr>
     <td></td>
     <td ALIGN=CENTER><em>Throws exception</em></td>
     <td ALIGN=CENTER><em>Special value</em></td>
     <td ALIGN=CENTER><em>Blocks</em></td>
     <td ALIGN=CENTER><em>Times out</em></td>
   </tr>
   <tr>
     <td><b>Insert</b></td>
     <td>add(e)</td>
     <td>offer(e)</td>
     <td>put(e)</td>
     <td>offer(e, time, unit)</td>
   </tr>
   <tr>
     <td><b>Remove</b></td>
     <td>remove()</td>
     <td>poll()</td>
     <td>take()</td>
     <td>poll(time, unit)</td>
   </tr>
   <tr>
     <td><b>Examine</b></td>
     <td>element()</td>
     <td>peek()</td>
     <td><em>not applicable</em></td>
     <td><em>not applicable</em></td>
   </tr>
</table>

阻塞队列是最典型的[“生产者消费者”](https://blog.csdn.net/holmofy/article/details/76553437)模型：

* 生产者调用put()方法将生产的元素入队，消费者调用take()方法；
* 当队列满了，生产者调用的put()方法会阻塞，直到队列有空间可入队；
* 当队列为空，消费者调用的get()方法会阻塞，直到队列有元素可消费；

![生产者消费者模型](http://tva1.sinaimg.cn/large/bda5cd74gy1fu62gsnpioj20jw06zaa5.jpg)

但是需要十分注意的是：**ThreadPoolExecutor提交任务时使用offer方法(不阻塞)，工作线程从队列取任务使用take方法(阻塞)**。正是因为ThreadPoolExecutor使用了不阻塞的offer方法，所以当队列容量已满，线程池会去创建新的临时线程；同样因为工作线程使用take()方法取任务，所以当没有任务可取的时候线程池的线程将会空闲阻塞。

> 事实上，工作线程的超时销毁是调用`offer(e, time, unit)`实现的。

# 2、JDK提供的阻塞队列实现

JDK中提供了以下几个BlockingQueue实现类：

![BlockingQueue类图](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLSCh9JyxEp4iFB4qjJUNYGk4gsEZgAZWM5ILMeWXZKHHScPUSKPIVbrzQZ4k9JsPUTceA8ODSKdCIAt591XHbvXTbbbIYkTaXDIy5w2i0)

## 2.1、[ArrayBlockingQueue](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/ArrayBlockingQueue.html)

这是一个由**数组实现**的**容量固定**的有界阻塞队列。这个队列的实现非常简单：

```java
private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x; // 入队
    if (++putIndex == items.length) // 如果指针到了末尾
        putIndex = 0; // 下一个入队的位置变为0
    count++;
    notEmpty.signal(); // 提醒消费者线程消费
}
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null; // 出队置空
    if (++takeIndex == items.length) // 如果指针到了末尾
        takeIndex = 0; // 下一个出队的位置变为0
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal(); // 提醒生产者线程生产
    return x;
}
```

通过简单的指针循环实现了一个环形队列：

![环形队列](http://tva1.sinaimg.cn/large/bda5cd74gy1fu66bwa1vqj20f307i0u3.jpg)

下面有一张维基百科关于环形缓冲区的的动画，虽然动画描述内容与ArrayBlockingQueue实现有所差异，但贵在生动形象(着实找不到更好的动画了)。

![环形缓冲区](http://tva1.sinaimg.cn/large/bda5cd74gy1fu6688jbd2g20m80goe7i.gif)

> ArrayBlockingQueue主要复杂在迭代，允许迭代中修改队列(删除元素时会更新迭代器)，并不会抛出ConcurrentModificationException；好在大多数场景中我们不会迭代阻塞队列。

## 2.2、[SynchronousQueue](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/SynchronousQueue.html)

这是一个非常有意思的集合，更准确的说它并不是一个集合容器，因为**它没有容量**。你可以“偷偷地”把它看作`new ArrayBlockingQueue(0)`，之所以用"偷偷地"这么龌龊的词，首先是因为`ArrayBlockingQueue`在`capacity<1`时会抛异常，其次`ArrayBlockingQueue(0)`并不能实现`SynchronousQueue`这么强大的功能。

**正如SynchronousQueue的名字所描述一样——“同步队列”，它专门用于生产者线程与消费者线程之间的同步**：

- 因为它任何时候都是空的，所以消费者线程调用take()方法的时候就会发生阻塞，直到有一个生产者线程生产了一个元素，消费者线程就可以拿到这个元素并返回。
- 同样的，你也可以认为任何时候都是满的，所以生产者线程调用put()方法的时候就会发生阻塞，直到有一个消费者线程消费了一个元素，生产者才会返回。

另外还有几点需要注意：

- SynchronousQueue不能遍历，因为它没有元素可以遍历；
- 所有的阻塞队列都不允许插入null元素，因为当生产者生产了一个null的时候，消费者调用poll()返回null，无法判断是生产者生产了一个null元素，还是队列本身就是空。

**CachedThreadPool使用的就是同步队列**：

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

因为SynchronousQueue无容量的特性，所以CachedThreadPool不会对任务进行排队，如果线程池中没有空闲线程，CachedThreadPool会立即创建一个新线程来接收这个任务。

所以使用CachedThreadPool要注意避免提交长时间阻塞的任务，可能会由于线程数过多而导致内存溢出(OutOfOutOfMemoryError)。

## 2.3、[LinkedBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingQueue.html)

这是一个由**单链表实现**的**默认无界**的阻塞队列。LinkedBlockingQueue提供了一个可选有界的构造函数，而在未指明容量时，容量默认为Integer.MAX_VALUE。

> 按照官方文档的说法LinkedBlockingQueue是一种**可选有界(optionally-bounded)阻塞队列**。

**SingleThreadPool和FixedThreadPool使用的就是LinkedBlockingQueue**

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

因为FixedThreadPool使用无界的LinkedBlockingQueue，所以当没有线程空闲时，新提交的任务都会提交到阻塞队列中，由于队列永远也不会满，FixedThreadPool永远也不会创建新的临时线程。

但是需要注意的是，不要往FixedThreadPool提交过多的任务，因为所有未处理的任务都会到LinkedBlockingQueue中排队，队列中任务过多也可能会导致内存溢出。虽然这个过程会比较缓慢，因为队列中的请求所占用的资源比线程占用的资源要少得多。

## 2.4、其他队列

DelayQueue和PriorityBlockingQueue底层都是使用**二叉堆实现**的**优先级阻塞队列**。

区别在于：

* 前者要求队列中的元素实现Delayed接口，通过执行时延从队列中提取任务，时间没到任务取不出来；
* 后者对元素没有要求，可以实现Comparable接口也可以提供Comparator来对队列中的元素进行比较，跟时间没有任何关系，仅仅是按照优先级取任务。

> 当我们提交的任务有优先顺序时可以考虑选用这两种队列
>
> 事实上[ScheduledThreadPoolExecutor内部实现了一个类似于DelayQueue的队列](https://blog.csdn.net/Holmofy/article/details/79344914)。

除了这两个，BlockingQueue还有两个子接口BlockingDeque(双端阻塞队列)，TransferQueue(传输队列)

并且两个接口都有自己唯一的实现类：

![其他类型的队列](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuShCAqajIajCJbLmoibFpixCImyiJIrDnIBkabg88XvIb9XNd9PQ156Febl1HbSNLH-cFBf-X6gGV0rGWKzcNdPg2genI9fGbQ6Qvf2QbmBq7G00)

* LinkedBlockingDeque：使用双向队列实现的双端阻塞队列，双端意味着可以像普通队列一样FIFO(先进先出)，可以以像栈一样FILO(先进后出)
* LinkedTransferQueue：[它是ConcurrentLinkedQueue、LinkedBlockingQueue和SynchronousQueue的结合体](http://cs.oswego.edu/pipermail/concurrency-interest/2009-February/005888.html)，但是把它用在ThreadPoolExecutor中，和无限制的LinkedBlockingQueue行为一致。

![LinkedTransferQueue](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vL22nDBKr5uZlbv2TdP-Qbeuk752Nc5QUb5a74kS2KWeskBfe660ykZwOHAbu3b73EpqikBIfApIlnoSpBJar1Cm2X42ADQW-AgKYgq9pfa9gN0lGm0000)

# 3、让生产者阻塞的线程池

前面说到CachedThreadPool和FixedThreadPool都有可能导致内存溢出，前者是由于线程数过多，后者是由于队列任务过多。而究其根本就是因为任务生产速度远大于线程池处理任务的速度。

所以有一个想法就是让生产任务的线程在任务处理不过来的时候休息一会儿——也就是阻塞住任务生产者。

但是前面提到过ThreadPoolExecutor内部将任务提交到队列时，使用的是不阻塞的offer方法。

我提供的第一种方式是：重写offer方法把它变成阻塞式。

## 3.1、重写BlockingQueue的offer

这种处理方式是将原来非阻塞的offer覆盖，使用阻塞的put方法实现。

```java
public class ThreadPoolTest {

    private static class Task implements Runnable {
        private int taskId;

        Task(int taskId) {
            this.taskId = taskId;
        }

        @Override public void run() {
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException ignore) {
            }
            System.out.println("task " + taskId + " end");
        }
    }

    public static void main(String[] args) {
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(2, 2, 0, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(2) {
                    @Override public boolean offer(Runnable runnable) {
                        try {
                            super.put(runnable); // 使用阻塞的put重写offer方法
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        return true;
                    }
                });
        threadPool.submit(new Task(1));
        System.out.println("task 1 submitted");
        threadPool.submit(new Task(2));
        System.out.println("task 2 submitted");
        threadPool.submit(new Task(3));
        System.out.println("task 3 submitted");
        threadPool.submit(new Task(4));
        System.out.println("task 4 submitted");
        threadPool.submit(new Task(5));
        System.out.println("task 5 submitted");
        threadPool.submit(new Task(6));
        System.out.println("task 6 submitted");
        threadPool.shutdown();
    }

}
```

执行的过程中会发现Task5要等到线程池中的一个任务执行完成后，才能提交成功。

这种方式把BlockingQueue的行为修改了，这时线程池的maximumPoolSize形同虚设，因为ThreadPoolExecutor调用offer入队失败返回false后才会创建临时线程。现在offer改成了阻塞式的，实际上永远是返回true，所以永远都不会创建临时线程，maximumPoolSize的限制也就没有什么意义了。

## 3.2、重写拒绝策略

在介绍第二种方式之前，先简单介绍JDK中提供了四种拒绝策略：

![JDK提供的拒绝策略](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vL24hDIaqkIKrnhKXDBYt9pC_pICnBoKajui8beM2ZgwlWabcSd5YKuf-JabfS4f2VavbSYL-3Or9-4L4AdHDpWCoWqhnYk6h2xe0gvN98pKi1-Wy0)

- AbortPolicy——抛出RejectedExecutionException异常的方式拒绝任务。
- DiscardPolicy——什么都不干，静默地丢弃任务
- DiscardOldestPolicy——把队列中最老的任务拿出来扔掉
- CallerRunsPolicy——在任务提交的线程把任务给执行了

**ThreadPoolExecutor默认使用AbortPolicy**

> DiscardPolicy和DiscardOldestPolicy两种策略看上去都不怎么靠谱，除非真有这种特别的需求，比如客户端应用中网络请求拥堵(服务端宕机或网络不通畅)的话可以选择抛弃最老的请求，大多数情况还是使用默认的拒绝策略。

我们的第二种做法就是写一个自己的RejectedExecutionHandler。这种方式相对“温柔”一些，在线程池提交任务的最后一步——被线程池拒绝的任务，可以在拒绝后调用队列的`put()`方法，让任务的提交者阻塞，直到队列中任务被被线程池执行后，队列有了多余空间，调用方才返回。

```java
public class ThreadPoolTest {

    private static class Task implements Runnable {
        private int taskId;

        Task(int taskId) {
            this.taskId = taskId;
        }

        @Override
        public void run() {
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException ignore) {
            }
            System.out.println("task " + taskId + " end");
        }
    }

    private static class BlockCallerPolicy implements RejectedExecutionHandler {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            try {
                executor.getQueue().put(r);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(2, 2, 0, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2), new BlockCallerPolicy());
        threadPool.submit(new Task(1));
        System.out.println("task 1 submitted");
        threadPool.submit(new Task(2));
        System.out.println("task 2 submitted");
        threadPool.submit(new Task(3));
        System.out.println("task 3 submitted");
        threadPool.submit(new Task(4));
        System.out.println("task 4 submitted");
        threadPool.submit(new Task(5));
        System.out.println("task 5 submitted");
        threadPool.submit(new Task(6));
        System.out.println("task 6 submitted");
        threadPool.shutdown();
    }

}
```

使用这种方式的好处是线程池仍可以设置maximumPoolSize，当任务入队失败仍可以创建临时线程执行任务，只有当线程总数大于maximumPoolSize时，任务才会被拒绝。

# 4、Tomcat中的线程池

作为一个最常用的Java应用服务器之一，Tomcat中线程池还是值得我们借鉴学习的。

> 注意下面代码来自Tomcat8.5.27，版本不同实现可能略有差异

[*org.apache.catelina.core.StandardThreadExecutor*](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/catalina/core/StandardThreadExecutor.html)

```java
public class StandardThreadExecutor extends LifecycleMBeanBase
        implements Executor, ResizableExecutor {
    // Tomcat线程池默认的配置
    protected int threadPriority = Thread.NORM_PRIORITY;
    protected boolean daemon = true;
    protected String namePrefix = "tomcat-exec-";
    protected int maxThreads = 200;
    protected int minSpareThreads = 25;
    protected int maxIdleTime = 60000;
    ...
    protected boolean prestartminSpareThreads = false;
    protected int maxQueueSize = Integer.MAX_VALUE;

    protected void startInternal() throws LifecycleException {
        // 任务队列:这里你看到的是一个无界队列，但是队列里面进行了特殊处理
        taskqueue = new TaskQueue(maxQueueSize);
        TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());
        // 创建线程池，这里的ThreadPoolExecutor是Tomcat继承自JDK的ThreadPoolExecutor
        executor = new ThreadPoolExecutor(
            getMinSpareThreads(), getMaxThreads(), // 核心线程数与最大线程数
            maxIdleTime, TimeUnit.MILLISECONDS, // 默认6万毫秒的超时时间，也就是一分钟
            taskqueue, tf); // 玄机在任务队列的设置
        executor.setThreadRenewalDelay(threadRenewalDelay);
        if (prestartminSpareThreads) {
            executor.prestartAllCoreThreads(); // 预热所有的核心线程
        }
        taskqueue.setParent(executor);

        setState(LifecycleState.STARTING);
    }
    ...
}
```

[*org.apache.tomcat.util.threads.ThreadPoolExecutor*](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/tomcat/util/threads/ThreadPoolExecutor.html)

```java
public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {

    private final AtomicInteger submittedCount = new AtomicInteger(0);
    private final AtomicLong lastContextStoppedTime = new AtomicLong(0L);
    private final AtomicLong lastTimeThreadKilledItself = new AtomicLong(0L);

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        submittedCount.decrementAndGet(); // 执行完成后提交数量减一

        if (t == null) {
            // 如果有必要抛个异常让线程终止
            stopCurrentThreadIfNeeded();
        }
    }
    @Override
    public void execute(Runnable command) {
        execute(command,0,TimeUnit.MILLISECONDS);
    }
    public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet(); // 提交时数量加一
        try {
            super.execute(command);
        } catch (RejectedExecutionException rx) {
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    // 如果任务被拒绝，则强制入队
                    if (!queue.force(command, timeout, unit)) {
                        // 由于TaskQueue默认无界，所以默认强制入队会成功
                        submittedCount.decrementAndGet();
                        throw new RejectedExecutionException("Queue capacity is full.");
                    }
                } catch (InterruptedException x) {
                    submittedCount.decrementAndGet(); // 任务被拒绝，任务数减一
                    throw new RejectedExecutionException(x);
                }
            } else {
                submittedCount.decrementAndGet(); // 任务被拒绝，任务数减一
                throw rx;
            }
        }
    }
}
```

[*org.apache.tomcat.util.threads.TaskQueue*](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/tomcat/util/threads/TaskQueue.html)

```java
public class TaskQueue extends LinkedBlockingQueue<Runnable> {
    private volatile ThreadPoolExecutor parent = null;
    public boolean force(Runnable o) {
        if ( parent==null || parent.isShutdown() )
            throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
        // 因为LinkedBlockingQueue无界，所以调用offer强制入队
        return super.offer(o);
    }

    public boolean force(Runnable o, long timeout, TimeUnit unit) throws InterruptedException {
        if ( parent==null || parent.isShutdown() )
            throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
        return super.offer(o,timeout,unit);
    }
    @Override
    public boolean offer(Runnable o) {
        // 不是上面Tomcat中定义地ThreadPoolExecutor，不做任何检查
        if (parent==null) return super.offer(o);
        // 线程数达到最大线程数，尝试入队
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
        // 提交的任务数小于线程数，也就是有空余线程，入队让空闲线程取任务
        if (parent.getSubmittedCount() < parent.getPoolSize()) return super.offer(o);
        // 走到这说明线程池没有空闲线程
        // 这里返回false，改变了LinkedBlockingQueue默认的行为
        // 使得Tomcat可以创建临时线程
        if (parent.getPoolSize() < parent.getMaximumPoolSize()) return false;
        // 到这里说明临时线程也没有空闲，只能排队了
        return super.offer(o);
    }
}
```

Tomcat的线程池扩展了JDK线程池的功能，主要体现在两点：

* Tomcat的ThreadPoolExecutor使用的TaskQueue，是无界的LinkedBlockingQueue，但是通过taskQueue的offer方法覆盖了LinkedBlockingQueue的offer方法，改写了规则，使得线程池能在任务较多的情况下增长线程池数量——JDK是先排队再涨线程池，Tomcat则是先涨线程池再排队。
* Tomcat的ThreadPoolExecutor改写了execute方法，当任务被reject时，捕获异常，并强制入队。

<!--

限流：

[Tomcat](https://tomcat.apache.org/)，[Jetty](https://eclipse.org/jetty/)等应用服务器会为每个请求分配一个线程，为了避免线程资源的浪费，肯定会使用线程池进行管理。可以预想当服务器负载过重的时候，没有空余线程，新来的请求肯定得排队。但是前面说过排队任务过多可能导致内存溢出，Tomcat作为一个性能稳定的服务器肯定不会让这种事儿发生，而且Http服务器不同于普通的应用软件，**请求排队时间过长久久得不到处理，用户可等不了这么久**。所以Tomcat会拒绝在一定时间内处理不了的请求，这样服务器可以容易地向客户端响应一个错误，比如[HTTP的503错误“Service unavailable”](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.5.4) 。

-->



> 参考链接：
>
> 支持生产阻塞的线程池 ：http://ifeve.com/blocking-threadpool-executor/
>
> Disruptor框架：http://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf
>
> 线程池调整的重要性：https://blog.bramp.net/post/2015/12/17/the-importance-of-tuning-your-thread-pools/
>
> 线程池调整的重要性(译)：http://www.importnew.com/17633.html
>
> SynchronousQueue与TransferQueue的区别：https://stackoverflow.com/questions/7317579/difference-between-blockingqueue-and-transferqueue/7317650
>
> Tomcat配置线程池：https://tomcat.apache.org/tomcat-8.5-doc/config/executor.html