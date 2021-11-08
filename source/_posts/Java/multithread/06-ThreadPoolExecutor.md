---
title: Java多线程复习与巩固（六）--线程池ThreadPoolExecutor详解
date: 2017-06-19
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


##1. 为什么要使用线程池

线程创建与销毁都耗费时间，对于**大量的短暂任务**如果仍使用“创建->执行任务->销毁”的简单模式，将极大地降低线程的使用效率(一个线程仅仅处理一个短暂的任务就被销毁了)。在这种情况下，为了提高线程的使用效率，我们使用缓存池的策略让线程执行任务后不立即销毁而是等待着处理下一个任务。

##2. 使用Executors工具类创建线程池

Executors是线程池框架提供给我们的创建线程池的工具类，它里面提供了以下创建几类线程池的方法。

```java
// 创建固定线程数量的线程池
public static ExecutorService newFixedThreadPool();
// 创建单个线程的线程池(本质上就是容量为1的FixedThreadPool)
public static ExecutorService newSingleThreadExecutor();
// 创建无数量限制可自动增减线程的线程池
public static ExecutorService newCachedThreadPool();

// 创建(可计划的)任务延时执行线程池
public static ScheduledExecutorService newScheduledThreadPool();
// 单线程版的任务计划执行的线程池
public static ScheduledExecutorService newSingleThreadScheduledExecutor();
```

通过查看这几个方法的源码发现：前三个方法new了`ThreadPoolExecutor`对象，而后面两个方法new了`ScheduledThreadPoolExecutor`对象。

整个线程池框架的类继承图如下，其中ThreadPoolExecutor是本文的核心，`ScheduleThreadPoolExecutor`将放到后一篇文章中讲。

> [JDK文档](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)建议一般情况使用Executors去创建线程池

![线程池ThreadPoolExecutor相关类继承图](./ThreadPool.svg)

其中三个核心接口的方法如下：

![三个核心接口的方法](http://img-blog.csdn.net/20170819141127562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

##3. 构造ThreadPoolExecutor对象

`java.util.concurrent.ThreadPoolExecutor` 类是线程池中最核心的类之一，因此如果要透彻地了解Java中的线程池，必须先了解这个类。下面分析一下ThreadPoolExecutor类的核心源码的具体实现。

在ThreadPoolExecutor类中提供了四个构造方法：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        //代码省略
    }
    ...
}
```

 　　从上面的代码可以得知，ThreadPoolExecutor继承了AbstractExecutorService(实现ExecutorService接口)类，并提供了四个构造器，前三个构造器最终都会辗转调用第四个构造器。

下面解释一下第四个构造器中各个参数的含义：

- corePoolSize：核心池的大小。

  * 核心池中的线程会一致保存在线程池中(即使线程空闲)，除非调用`allowCoreThreadTimeOut`方法允许核心线程在空闲后一定时间内销毁，该时间由构造方法中的`keepAliveTime`和`unit`参数指定；
  * 在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了`prestartAllCoreThreads()`或者`prestartCoreThread()`方法，从这两个方法的名字就可以看出，是**“预创建线程”**的意思，即在没有任务到来之前就创建corePoolSize个线程(prestartAllCoreThreads)或者一个线程(prestartCoreThread)；

- maximumPoolSize：线程池允许的最大线程数。这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程。

  * 默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，**当线程池中的线程数目达到corePoolSize后，就会把新加入的任务放到缓存队列当中**，缓存队列由构造方法中的`workQueue`参数指定，**如果入队失败(队列已满)则尝试创建临时线程**，但临时线程和核心线程的总数不能超过maximumPoolSize，**当线程总数达到maximumPoolSize后会拒绝新任务**；所以有两种方式可以让任务绝不被拒绝：

    ① 将maximumPoolSize设置为`Integer.MAX_VALUE`(线程数不可能达到这个值)，`CachedThreadPool`就是这么做的；

    ② 使用无限容量的阻塞队列(比如LinkedBlockingQueue)，所有处理不过来的任务全部排队去，`FixedThreadPool`就是这么做的。

- keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。

  * 默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用——当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会被销毁，直到线程池中的线程数不超过corePoolSize。但是如果调用了`allowCoreThreadTimeOut(true)`方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；

- unit：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：

  ```java
  TimeUnit.DAYS;              //天
  TimeUnit.HOURS;             //小时
  TimeUnit.MINUTES;           //分钟
  TimeUnit.SECONDS;           //秒
  TimeUnit.MILLISECONDS;      //毫秒
  TimeUnit.MICROSECONDS;      //微妙
  TimeUnit.NANOSECONDS;       //纳秒
  ```
  > 并发库中所有时间表示方法都是以`TimeUnit`枚举类作为单位

- workQueue：一个阻塞队列(BlockingQueue接口的实现类)，用来存储等待执行的任务，一般来说，这里的阻塞队列有以下几种选择：

  ```java
  ArrayBlockingQueue    // 数组实现的阻塞队列，数组不支持自动扩容。所以当阻塞队列已满
                        // 线程池会根据handler参数中指定的拒绝任务的策略决定如何处理后面加入的任务

  LinkedBlockingQueue   // 链表实现的阻塞队列，默认容量Integer.MAX_VALUE(不限容)，
                        // 当然也可以通过构造方法限制容量

  SynchronousQueue      // 零容量的同步阻塞队列，添加任务直到有线程接受该任务才返回
                        // 用于实现生产者与消费者的同步，所以被叫做同步队列

  PriorityBlockingQueue // 二叉堆实现的优先级阻塞队列

  DelayQueue          // 延时阻塞队列，该队列中的元素需要实现Delayed接口
                      // 底层使用PriorityQueue的二叉堆对Delayed元素排序
                      // ScheduledThreadPoolExecutor底层就用了DelayQueue的变体"DelayWorkQueue"
                      // 队列中所有的任务都会封装成ScheduledFutureTask对象(该类已实现Delayed接口)
  ```

  > 有关BlockingQueue的内容可以参考[《Java集合框架总结与巩固》](http://blog.csdn.net/holmofy/article/details/71215548)

- threadFactory：线程工厂，主要用来创建线程；默认情况都会使用Executors工具类中定义的默认工厂类`DefaultThreadFactory`。可以实现ThreadFactory接口来自己控制创建线程池的过程(比如设置创建线程的名字、优先级或者是否为Deamon守护线程)

- handler：表示当拒绝处理任务时的策略，有以下四种取值(默认为AbortPolicy)：

  ```java
  ThreadPoolExecutor.AbortPolicy: 丢弃任务并抛出RejectedExecutionException异常。
  ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
  ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
  ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
  ```


  > 可通过实现RejectedExecutionHandler接口来自定义任务拒绝后的处理策略

##4. 线程池的状态转换

ThreadPoolExecutor类中有一个`ctl`属性，该属性是AtomicInteger类型，本质上就是32bit的int类型。这个32bit字段中存储了两个数据：

![ThreadPoolExecutor.ctl](./ThreadPoolExecutor_ctl.svg)

其中三个高字节位存储了线程池当前的运行状态，线程池状态有以下几个：

```java
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

* RUNNING：接受新任务并处理队列中的任务
* SHUTDOWN：不接受新任务但处理队列中的任务
* STOP：不接受新任务也不处理队列中的任务并终断正在处理中的任务
* TIDYING：所有任务已经终止，workerCount等于0，线程池切换到TIDYING后将会执行`terminated()`钩子方法
* TERMINATED：`terminated()`方法已执行完成

整个过程的状态转换图如下：

![线程池状态转换图](./RunState.svg)

我们可以调用的两个改变线程池状态的方法分别是：

```java
// 进入SHUTDOWN状态
public void shutdown();
// 进入STOP状态
public List<Runnable> shutdownNow();
```

另外ThreadPoolExecutor提供了一些方法来查询这些状态：

```java
// 非运行状态：SHUTDOWN,STOP,TIDYING,TERMINATED
public boolean isShutdown() {
    return ! isRunning(ctl.get());
}
// 正在终止状态：SHUTDOWN,STOP,TIDYING
public boolean isTerminating() {
    int c = ctl.get();
    return ! isRunning(c) && runStateLessThan(c, TERMINATED);
}
// 终止状态：TERMINATED
public boolean isTerminated() {
    return runStateAtLeast(ctl.get(), TERMINATED);
}
```

##5. 任务的提交

任务提交主要有三种方式：

* execute(Runnable command)：定义在Executor接口中
* submit的三个重载方法：定义在ExecutorService接口中
* invoke(invokeAll,invokeAny)提交方式：定义在ExecutorService接口中

## 5.1 submit提交方式

submit方法的实现源码在ThreadPoolExecutor的基类AbstractExecutorService中：

```java
    // 将Runnable和Callable包装成FutureTask对象
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```

submit最终都会调用execute方法去执行任务，区别在于submit方法返回一个FutureTask对象(顾名思义FutrueTask就是未来将执行的任务，可以通过FutureTask对象获取任务执行的结果)。

### 5.1.1 FutrueTask

FutureTask实现Future接口，Future接口及其相关类继承图如下：

![Future继承图](./Future.svg)

FutureTask类中定义了如下的状态：

```java
    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;// 任务已被创建(new的时候默认状态为0)
    private static final int COMPLETING   = 1;// 任务即将完成(已获取返回值或已捕获异常)
    private static final int NORMAL       = 2;// 任务正常完成(以返回值的形式完成任务)
    private static final int EXCEPTIONAL  = 3;// 任务异常完成(任务执行过程发生异常并被捕获)
    private static final int CANCELLED    = 4;// 任务已被取消(任务还没被执行就被取消了,可能在排队)
    private static final int INTERRUPTING = 5;// 任务正在中断(任务执行时被取消)
    private static final int INTERRUPTED  = 6;// 任务已经中断(INTERRUPTING的下一个状态)
```

FutureTask的状态转换图如下(其中绿色标注的是外部可调用的方法，其他方法均有内部调用，RUNNING状态是我附加的状态，表示该任务已经被运行)：

![FutureTask状态转换图](./FutrueTaskState.svg)

Future接口定义了以下几个方法：

```java
// 尝试取消任务，根据任务的执行状态分一下几种情况
// 任务完成或任务已取消：该方法调用失败返回false
// 运行状态：任务正在执行，根据传入mayInterruptIfRunning参数判断是否应该终断任务
// 排队状态：任务正在排队，则将任务状态设置为CANCELLED,线程池不会执行已经被取消的任务
boolean cancel(boolean mayInterruptIfRunning);
// 任务是否被取消,CANCELLED,INTERRUPTING,INTERRUPTED状态下返回true
boolean isCancelled() { return state >= CANCELLED; }
// 任务是否被执行,只有NEW状态返回false
boolean isDone(){ return state != NEW; }
// 获取任务执行结果,该方法会一直阻塞等待任务的执行结果
V get() throws InterruptedException, ExecutionException;
// 获取任务执行结果,该方法会等待timeout时间,
// 如果没有结果就抛出超时异常(TimeoutException)
V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```

示例：

```java
public class ThreadPoolExecutorTest {
    public static void main(String[] args) {
        // 线程池大小固定为4，处理不完的任务全部入队
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(4, 4, 0, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>());
        Scanner sc = new Scanner(System.in);
        System.out.println("----------------------------------------");
        System.out.println("输入0使用submit提交任务到线程池");
        System.out.println("输入正整数使用取消上次提交的任务并允许中断");
        System.out.println("输入负整数使用取消上次提交的任务不允许中断");
        System.out.println("输入其他字符关闭线程池并退出程序");
        System.out.println("----------------------------------------");
        Future<String> task = null;
        while (true) {
            if (sc.hasNextInt()) {
                int command = sc.nextInt();
                if (command == 0) {
                    // 提交任务
                    task = threadPool.submit(new Callable<String>() {
                        @Override
                        public String call() throws Exception {
                            printMessage("Task start");
                            Thread.sleep(1800);
                            printMessage("Task finish");
                            return "result";
                        }
                    });
                } else if (command > 0 && task != null) {
                    // 强制取消任务
                    boolean success = task.cancel(true);
                    System.out.println(task + ":task.cancel(true)->"
                            + (success ? "success" : "failed"));
                } else if (command < 0 && task != null) {
                    // 取消任务
                    boolean success = task.cancel(false);
                    System.out.println(task + ":task.cancel(false)->"
                            + (success ? "success" : "failed"));
                }
            } else {
                threadPool.shutdownNow();
                break;
            }
        }
    }
    private static void printMessage(String msg) {
        String name = Thread.currentThread().getName();
        System.out.println(name + ":" + msg);
    }
}
```

## 5.2 execute提交方式

submit最终都会调用execute方法去执行任务，所以我们重点需要分析execute方法：

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            // 如果核心线程未满，则创建核心线程并将任务交由新线程处理(即使有核心线程空闲)
            // 所以前corePoolSize个任务都会一次交给新创建的核心线程执行
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 往下走说明核心线程已全部创建完毕

        // 如果线程池仍是RUNNING状态，将任务加入工作队列(也就是构造时提供的阻塞队列)
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();

            if (! isRunning(recheck) && remove(command))
            // 如果线程池已经SHUTDOWN，将任务回滚移除工作队列并拒绝该任务
                reject(command);

            else if (workerCountOf(recheck) == 0)
            // 该情况用于核心线程数(corePoolSize)为0
            // 或者允许核心线程超时(allowCoreThreadTimeOut)的时候
                addWorker(null, false);
        }
        // 入队失败则尝试创建普通临时线程(非核心线程)
        else if (!addWorker(command, false))
            // 如果仍无法创建线程
            // 可以断定线程池已经SHUTDOWN(关闭)或者SATURATED(饱和)
            // 所以拒绝该任务
            reject(command);
    }
```

根据源码可以得出以下执行流程：

1. 如果正在运行的线程数小于corePoolSize，Executor总是添加一个新线程，而不是任务排队。
2. 如果已有corePoolSize或更多线程在运行，则Executor总是优先选择任务排队而不会添加新线程。
3. 如果任务请求不能排队，则创建一个新线程，但是线程总数量不超过maximumPoolSize，如果不能创建新线程任务将会被拒绝。

这里有几点需要重点注意：

- 第五行`workerCountOf(c) < corePoolSize`：如果核心线程没有创建完则新任务交由新创建的核心线程处理，但是我们如果调用`prestartAllCoreThreads`方法预先把所有核心线程创建完成，则这一分支不会执行。
- 第十三行`workQueue.offer(command)`：offer方法不会阻塞(名不符其实，阻塞队列不阻塞呵呵)，如果入队成功会立即返回true否则返回false。入队成功与否取决于`workQueue`的性质。比如：①单链表实现的**LinkedBlockingQueue**默认容量为`Integer.MAX_VALUE`(等价于无限容量)，所以此时该方法不会返回false也不会创建临时线程(都去排队去了)，当然如果创建LinkedBlockingQueue时指定了capacity，offer方法就可能返回false，但我们一般不会这么干；②而数组实现的**ArrayBlockingQueue**不允许扩容，所以队列已满则会返回false进入二十三行尝试创建临时线程，如果总线程数不超过maximumPoolSize则能创建临时线程，但会导致后来的任务没排队反而能得到执行(不公平)，如果超出maximumPoolSize创建临时线程失败则会拒绝任务，两种情况都不好，所以ArrayBlockingQueue用的不是很多；③使用小顶堆实现的`PriorityBlockingQueue`会根据任务的优先级来选择执行顺序；④使用没有容量的同步队列`SynchronousQueue`，如果没有空闲线程接收任务会立即返回false，所以大部分情况会创建新的临时线程。
- 第十七行`workerCountOf(recheck) == 0`：这句判断间接表示核心线程数为0的情况，核心线程数为0只会发生在两种条件下：①线程池本身已经指定核心数为0(构造方法指定或`setCorePoolSize`方法指定)，②调用`allowCoreThreadTimeOut`方法允许核心线程超时导致核心线程数位0。

## 5.3 invoke提交方式

ExecutorService中定义了两组invoke方法：

```java
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) 
                              throws InterruptedException;
// 等待timeout时间，即使一个任务都没完成也返回
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                              long timeout, TimeUnit unit) throws InterruptedException;

<T> T invokeAny(Collection<? extends Callable<T>> tasks)
                              throws InterruptedException, ExecutionException;
// 等待timeout时间，即使一个任务都没完成也返回
<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                              long timeout, TimeUnit unit)
                              throws InterruptedException, ExecutionException, TimeoutException;
```

- invokeAny取得率先完成的任务的返回值，当第一个任务结束后，会调用cancel方法取消其它任务。
- invokeAll等所有任务执行完毕后，取得全部任务的结果值。

**invokeAll存在的问题**

invokeAll有个严重的问题是，任务执行后不会抛出任务执行的异常。调用方需要手动调用`Future.get()`方法触发异常，而且FutureTask的异常是被ExecutionException包裹过的，所以调用`get`方法的时候是无法捕获到内部抛出的异常类型，要么通过捕获ExecutionException异常拿到它内部包装的异常，要么直接捕获所有的Exception。

```java
// AbstractExecutorService#invokeAll
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks) {
            RunnableFuture<T> f = newTaskFor(t);
            futures.add(f);
            execute(f);
        }
        for (int i = 0, size = futures.size(); i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                try {
                    f.get();
                // invokeAll把这个异常给忽略了。
                // 之所以要忽略，是要让所有的任务都执行完
                // 所以外部调用者仍需要调用get()方法触发这个异常。
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                }
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done) // 这部分只可能在execute抛异常的时候执行
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```

### 5.3.1 等待任务完成

invokeAll方式会导致调用线程阻塞直到所有任务完成，由于不知道哪个任务先执行完毕，使用这种方式效率不是很高。所以Java5还提供了一个`CompletionService `接口给我们用。`CompletionService`目前只有一个实现类——`ExecutorCompletionService`。

![ExecutorCompletionService架构图](http://tva1.sinaimg.cn/large/bda5cd74gy1ftaj3nepevj219a0fyn7o.jpg)

ExecutorCompletionService实际只是维护了一个队列，然后将完成的任务往队列里放，这个实现主要是依赖FutureTask的一个钩子方法done：

```java

public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor;
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue;

    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        // 任务执行完后会执行该钩子方法
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }
  ...
    // 提交任务时再用QueueingFuture包装
    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task);
        executor.execute(new QueueingFuture(f));
        return f;
    }

    public Future<V> submit(Runnable task, V result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task, result);
        executor.execute(new QueueingFuture(f));
        return f;
    }
```

示例：

```java
void solve(Executor e, Collection<Callable<Result>> solvers) 
                       throws InterruptedException, ExecutionException {
    CompletionService<Result> ecs = new ExecutorCompletionService<Result>(e);
    for (Callable<Result> s : solvers)
        ecs.submit(s);
    int n = solvers.size();
    for (int i = 0; i < n; ++i) {
        Result r = ecs.take().get();
        if (r != null)
            doSomething(r);
    }
}
```

> invokeAny就是通过`CompletionService`实现，从而拿到第一个执行完成的任务的结果。

##6. 线程如何创建

刚刚的execute提交任务中调用到了addWorker方法来创建线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            // 获取线程池当前状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // 线程池已经SHUTDOWN不接受任务
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                // 获取线程池当前的工作线程数
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||// 超过了最大线程容量
                    // 创建核心线程，超过了核心线程数
                    // 创建普通线程，超过了最大可允许的线程数
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    // 拒绝
                    return false;
                // 尝试增加线程数WorkerCount++
                if (compareAndIncrementWorkerCount(c))
                    // 增加线程数成功，跳出循环
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    // 前后状态不一致，循环重试
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建Worker内部类对象，Worker内部会创建线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                // 线程创建成功
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 将工作线程加入集合
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start(); // 启动线程
                    workerStarted = true;
                }
            }
        } finally {
            // 如果线程创建或执行失败
            if (! workerStarted)
                // 执行失败的处理
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                // 线程创建失败了，必须从集合中删除该Worker对象
                workers.remove(w);
            // 并WorkCount--
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```

##7. 线程如何执行

线程执行我们肯定得找到run方法，我们看一下Worker类是怎么定义的：

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /** 实际的工作线程 */
        final Thread thread;
        /** 线程初始的任务，可能为null. */
        Runnable firstTask;
        /** 该线程执行的任务数量 */
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // AQS中的内容，这个暂时不讨论
            this.firstTask = firstTask;
            // 使用ThreadFactory创建线程，并指定runnable参数为this
            this.thread = getThreadFactory().newThread(this);
        }

        /** 因为创建线程时指定了runnable为this，所以会执行这个方法  */
        public void run() {
            runWorker(this);
        }

        ...
    }
```

ThreadPoolExecutor.runWorker方法

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 判断firstTask是否为空,否则从任务队列中取出任务
            while (task != null || (task = getTask()) != null) {
                w.lock(); // AQS中的内容暂时不做讨论
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 回调beforeExecute方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run(); //执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 回调afterExecute方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++; // 执行完成的任务数加一
                    w.unlock();
                }
                // 循环执行下一个任务
            }
            completedAbruptly = false;
            // 没有任务执行，终结线程
        } finally {
            //
            processWorkerExit(w, completedAbruptly);
        }
    }

    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            ...

            try {
                Runnable r = timed ?
                    // 超时功能使用阻塞队列实现
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

##8. 线程池的其他变量和方法

其他成员变量：

```java
 // 工作队列：用于任务排队等待
private final BlockingQueue<Runnable> workQueue;
 // 操作线程池数据时，用于保持同步的锁
private final ReentrantLock mainLock = new ReentrantLock();
 // 保存所有工作线程的集合
private final HashSet<Worker> workers = new HashSet<Worker>();
 // 调用awaitTermination方法时等待终止的条件
private final Condition termination = mainLock.newCondition();
 // 线程池达到的最大线程数，记录曾经出现过的最大线程数
private int largestPoolSize;
 // 已经完成任务的数量
private long completedTaskCount;
 // 用于创建线程的线程工厂
private volatile ThreadFactory threadFactory;
 // 任务的拒绝策略
private volatile RejectedExecutionHandler handler;
 // 线程存活时间
private volatile long keepAliveTime;
 // 是否允许核心线程超时
private volatile boolean allowCoreThreadTimeOut;
 // 核心线程数
private volatile int corePoolSize;
 // 线程池允许的最大线程数
private volatile int maximumPoolSize;
```

其他成员方法：

```java
//------------------------------------------------
// 与线程大小有关的方法：
// 这几个数量之间满足：CorePoolSize <= PoolSize <= LargestPoolSize <= MaximumPoolSize
// 核心线程数
public void setCorePoolSize(int corePoolSize);
public int getCorePoolSize();
// 最大线程数
public void setMaximumPoolSize(int maximumPoolSize);
public int getMaximumPoolSize();
// 当前线程池大小
public int getPoolSize();
// 线程池达到的最大线程数
public int getLargestPoolSize();

//------------------------------------------------
// 和任务数量有关的方法：
// 当前活跃的任务数量：这个数量等于当前工作线程个数(不包括空闲线程)
public int getActiveCount();
// 线程池已接受的任务总量
public long getTaskCount();
// 线程池已经完成的任务数量
public long getCompletedTaskCount();

//------------------------------------------------
// 其他getter/setter方法：
// threadFactory的getter/setter方法
public void setThreadFactory(ThreadFactory threadFactory);
public ThreadFactory getThreadFactory();
// handler的getter/setter方法
public void setRejectedExecutionHandler(RejectedExecutionHandler handler);
public RejectedExecutionHandler getRejectedExecutionHandler();
// allowCoreThreadTimeOut的getter/setter方法
public void allowCoreThreadTimeOut(boolean value);
public boolean allowsCoreThreadTimeOut();
// keepAliveTime的getter/setter方法
public void setKeepAliveTime(long time, TimeUnit unit);
public long getKeepAliveTime(TimeUnit unit);

//------------------------------------------------
// 其他方法
// 预启动一个核心线程
public boolean prestartCoreThread();
// 预启动所有核心线程
public int prestartAllCoreThreads();
// 获取等待队列
public BlockingQueue<Runnable> getQueue();
// 将task任务从等待队列中移除
public boolean remove(Runnable task);
// 将等待队列中的所有已经被取消的Future任务清除
public void purge();
// 在shutdown请求之后，调用awaitTermination方法
// 会阻塞等待任务完成进入TERMINATED状态
// timeout设置等待时间，unit设置时间单位
public boolean awaitTermination(long timeout, TimeUnit unit);
```

##9. 扩展ThreadPoolExecutor的功能

在ThreadPoolExecutor类中还有三个protected属性的空方法：

```java
// 任务执行之前：
// Thread t:执行任务的线程
// Runnable r:被执行的任务
protected void beforeExecute(Thread t, Runnable r) { }
// 任务执行之后：
// Runnable r:被执行的任务
// Throwable t:任务执行过程中抛出的异常
protected void afterExecute(Runnable r, Throwable t) { }
// 线程池进入TERMINATED状态后
protected void terminated() { }
```

根据JDK文档中的说明，我们可以重载这三个钩子方法来对ThreadPoolExecutor进行扩展。下面是官方文档提供的扩展示例：

```java
// 可以暂停执行的线程池
class PausableThreadPoolExecutor extends ThreadPoolExecutor {
    public PausableThreadPoolExecutor(...) { super(...); }

    private boolean isPaused;
    private ReentrantLock pauseLock = new ReentrantLock();
    private Condition unpaused = pauseLock.newCondition();

    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        pauseLock.lock();
        try {
            // 如果线程池被暂停，则任务等待不暂停的条件(等待resume方法的调用)
            while (isPaused)
                unpaused.await();
        } catch (InterruptedException ie) {
            t.interrupt();
        } finally {
            pauseLock.unlock();
        }
    }

    public void pause() {
        pauseLock.lock();
        try {
            isPaused = true;// 暂停
        } finally {
            pauseLock.unlock();
        }
    }

    public void resume() {
        pauseLock.lock();
        try {
            isPaused = false;// 恢复
            unpaused.signalAll(); // 激活所有等待的线程
        } finally {
            pauseLock.unlock();
        }
    }
}
```

##10. 更多种类的线程池MoreExecutors

[Guava](https://github.com/google/guava)是Google提供的一个最受欢迎的通用工具包。它提供了很多并发工具，其中包括几个便捷的`ExecutorService`实现类，这些实现类我们无法直接访问，Guava提供了一个唯一的访问入口——[MoreExecutors](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/MoreExecutors.java)工具类。

1、 应用执行完成(主线程以及其他非守护线程执行完)后自动关闭的线程池

```java
/* 执行完成后，等待terminationTimeout后关闭线程池 */
public static ExecutorService getExitingExecutorService(
    ThreadPoolExecutor executor, long terminationTimeout, TimeUnit timeUnit);
/* 默认延迟120秒 */
public static ExecutorService getExitingExecutorService(ThreadPoolExecutor executor);
/* 对ScheduledExecutorService的包装 */
public static ScheduledExecutorService getExitingScheduledExecutorService(
    ScheduledThreadPoolExecutor executor, long terminationTimeout, TimeUnit timeUnit);
public static ScheduledExecutorService getExitingScheduledExecutorService(
    ScheduledThreadPoolExecutor executor)
```

这个工具方法修改传入的线程池的ThreadFactory使其生成守护线程，但是线程池使用守护进程会导致有些任务只执行了一半，为了让线程池更加可控，所以Guava使用`Runtime.getRuntime().addShutdownHook(hook)`注册了一个等待线程：

```java
addShutdownHook(
    MoreExecutors.newThread(
    "DelayedShutdownHook-for-" + service,
    new Runnable() {
        @Override
        public void run() {
            try {
                service.shutdown();
                service.awaitTermination(terminationTimeout, timeUnit);
            } catch (InterruptedException ignored) { }
        }
    }));
```

示例：

```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(5);
ExecutorService executorService = MoreExecutors.getExitingExecutorService(
    executor, 100, TimeUnit.MILLISECONDS); 
executorService.submit(() -> { while (true) {} }); // 执行一个无限期的任务
```

> 上面的代码中，如果我们直接在线程池中执行无限期的任务，会导致JVM无限期挂起。
>
> 当我们使用`getExitingExecutorService`进行包装后，程序执行完后，如果线程池100毫秒内没有执行完任务，将会直接关闭线程池。

2、当前线程执行的线程池

有时可能需要在当前线程或线程池中执行任务，具体取决于某些条件，为了使用统一的接口可能需要在当前线程执行的Executor。虽然这个实现起来并不难，但还是需要编写样板代码。所幸Guava已经提供了这样的方法。

```java
// 枚举单例
Executor executor = MoreExecutors.directExecutor();
// 每次都会创建一个新的对象
ExecutorService executor = MoreExecutors.newDirectExecutorService();
```

3、使用ListenableFuture异步增强的线程池

使用submit提交方式将会返回一个实现了Future接口的FutureTask对象，我们通过调用`Future.get`方法获取任务的执行结果，如果任务没有执行完，get方法将会导致调用线程的阻塞。为此Guava为我们提供了一个增强型的Future接口——`ListenableFuture`，能以异步回调的方式获取执行结果。

为了能让线程池返回`ListenableFuture`，MoreExecutors中提供了两个包装方法：

```java
public static ListeningExecutorService listeningDecorator(ExecutorService delegate);
public static ListeningScheduledExecutorService listeningDecorator(ScheduledExecutorService delegate)
```

> 实际上`ListenableFutureTask`和上面的ExecutorComletionService一样也是通过实现FutureTask的done方法实现。

使用示例：

```java
ExecutorService delegate = Executors.newCachedThreadPool();
// 包装成Guava的ListeningExecutorService
ListeningExecutorService executor = MoreExecutors.listeningDecorator(delegate);
// 提交有返回结果的任务
final ListenableFuture future = executor.submit(new Callable<Integer>() {
    public Integer call() throws Exception {
        int result = 0;
        Thread.sleep(1000);
      return result;
    }
});
future.addListener(new Runable() {
  public void run() {
      System.out.println("result:" + future.get());
  }
}, MoreExecutors.directExecutor());
// Futures工具类提供了工具方法用于任务正常或异常情况的处理。
Futures.addCallback(future, new FutureCallback<Integer>() {
    public void onSuccess(Integer result) {
        // 任务正常返回结果
        System.out.println("result:" + result);
    }
    public void onFailure(Throwable t) {
        // 任务抛异常了
        t.printStackTrace();
    }
}, MoreExecutors.directExecutor());
```

> 关于ListenableFuture的更多细节用法，参考[Guava的wiki](https://github.com/google/guava/wiki/ListenableFutureExplained)

