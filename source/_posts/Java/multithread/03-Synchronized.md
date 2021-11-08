---
title: Java多线程复习与巩固（三）--线程同步
date: 2017-06-16
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

## 、多线程容易出现的问题

因为同一个进程内的多个线程共享进程的资源，而进程之间，资源的获取是互斥的，所以线程间通信比进程间通信更简单。我们可以直接通过**共享资源的访问**来实现线程间通信，这种通信方式十分有效(速度快)，但也容易产生错误，如：**线程干扰**和**内存一致性错误**。

看一下下面这个例子：这个程序有两个线程，一个线程对计数器进行10000次加一操作，一个线程对计数器进行10000次减一操作，两个线程执行完后，计数器值原本应该等于0。但主线程在两个线程执行完后，打印计数器的值几乎很难得到0这个结果。

```java
public class ThreadCommunicate {
    static class Counter {
        private int c = 0;

        public void increment() {
            c++;
        }

        public void decrement() {
            c--;
        }

        public int value() {
            return c;
        }
    }

    static class IncrementTask implements Runnable {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        }
    }

    static class DecrementTask implements Runnable {
        public void run() {
            for (int i = 0; i < 10000; i++) {
                counter.decrement();
            }
        }
    }

    // Counter就是两个线程的共享资源
    private static Counter counter = new Counter();

    public static void main(String[] args) throws InterruptedException {
        // 创建两个线程同时对共享资源进行读写
        Thread i = new Thread(new IncrementTask());
        Thread d = new Thread(new DecrementTask());
        i.start();
        d.start();
        i.join();
        d.join();
        System.out.println(counter.value());
    }
}
```

## 、问题出现的原因

问题就出在`c++`和`c--`这两个操作上。

了解过汇编的都应该知道，一个`++`自增操作会分为以下几步：

```asm
  mov         eax,dword ptr [c]  ;根据c的地址从内存取出c的值放到寄存器中
  add         eax,1              ;执行加一操作
  mov         dword ptr [c],eax  ;把寄存器的值放回c地址所在的内存
```

另外从Java的反编译代码也可以看出来，`c++`，`c--`不止一步：

```asm
  public void increment();
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // 获取字段c的值
       5: iconst_1      #                   // 获取常量1的值
       6: iadd          #                   // 执行加操作
       7: putfield      #2                  // 把相加的结果放回字段c
      10: return

  public void decrement();
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // 获取字段c的值
       5: iconst_1      #                   // 获取常量1的值
       6: isub          #                   // 执行减操作
       7: putfield      #2                  // 把相加的结果放回字段c
      10: return
```

即使是非常简单的操作，在底层处理时也会分解成若干步。

不论是在单处理机CPU还是多处理机CPU中，两个线程执行指令的前后顺序是不确定的，如果出现下面的这种情况：

![CPU执行顺序](http://tva1.sinaimg.cn/large/bda5cd74ly1g2hdq1y4pxj20qx0l4whn.jpg)

通过上图可以看出，两个线程同时执行一轮后，c的值结果等于-1。用一张动态图来形象描述一下这种情况：

![线程竞争共享资源的坏情况](http://tva1.sinaimg.cn/large/bda5cd74ly1g2hdqqf2pfg20hj09ngvd.gif)

很显然下面这张图才是我们想要的结果：

![线程竞争共享资源的好情况](http://tva1.sinaimg.cn/large/bda5cd74ly1g2hdr2zxkeg20hj09nqd9.gif)

所以真正能得到等于0的序列只有两种：

![可以得出0的两种序列](http://img-blog.csdn.net/20170614235441302?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这么多条指令的排列中只有2条能够得到0这个结果，难怪上面的程序几乎得不到0这个结果。

很明显两个线程需要**按顺序**来读写共享变量才不会出问题，一旦读写过程出现交错就可能出现问题，而这个按顺序来访问变量就是我们接下来提到的"同步"。

## 、同步与异步

![同步和异步](http://tva1.sinaimg.cn/large/bda5cd74ly1g2hdtzz8kzj20ma0gl74i.jpg)

“同步”和“异步”，在各个领域中都有这两个词的出现。通俗的讲：**同步就是在一条线上执行，异步就是分成多条线执行。**

很显然对于上面的问题，我们应该要将两个线程并成一条线，让它按次序执行，这就是我们要讲的“线程同步”。先来看一张动态图来初步了解线程同步的基本原理：

![线程同步的原理图](http://tva1.sinaimg.cn/large/bda5cd74ly1g2hdugsxjng20hj09nx6p.gif)

## 、使用线程同步解决问题

线程同步的方式有很多，下面我们介绍Java语言中最简单的线程同步的实现——使用`synchronized`关键字。`synchronized`关键字有两种使用方法：同步方法、同步代码块

## 4.1、同步方法

修改后的Counter类代码如下：

```java
    static class Counter {
        private int c = 0;

        public synchronized void increment() {
            c++;
        }

        public synchronized void decrement() {
            c--;
        }

        public int value() {
            return c;
        }
    }
```

> 因为`increment`方法和`decrement`方法对共享资源进行了修改属于**写操作**，而`value`方法没有进行修改仅仅只是读取属于**读操作**。对于读操作我们不需要加同步锁。

使用`synchronized`关键字修饰的方法有以下特点：

- 首先，同一对象上的两个`synchronized`方法的调用不可能交织。当一个线程正在执行一个对象的`synchronized`方法时，调用同一个对象的`synchronized`方法的所有其他线程都会阻塞(挂起)，直到第一个线程执行完同步方法。（解决了线程干扰问题）

- 第二，当一个`synchronized`方法退出时，会与后续的`synchronized`方法自动建立[happens-before](https://en.wikipedia.org/wiki/Happened-before)关联，在这一点上`synchronized`与`volatile`关键字功能有些类似：确保CPU寄存器或Cache高速缓存内的数据能够及时写回内存，从而保证了对象状态的改变对所有线程是可见的。（解决了内存一致性问题）

- 第三，`synchronized`对实例方法(非static方法)加同步锁，锁住的是实例对象(this)。`synchronized`对类静态方法(static方法)加同步锁，锁住的是Class实例(Xxx.class)。

  ```java
  // 锁住的是Xxx.class
  public static synchronized void function() {}

  // 锁住的是this实例对象
  public synchronized void function() {}
  ```


构造方法不能加同步锁，构造方法加`synchronized`关键字产生语法错误。因为对构造方法加同步锁没有任何意义，两个线程可以同时创建同一个类的两个实例对象，这没有任何影响。

> 多线程的时候不要在构造方法中将this引用共享出去，可能会出现异常。比如你在构造方法中将this引用添加到集合中：`List.add(this)`，其他的线程就可以从集合中获取这个对象的引用，但这个对象并没有完成初始化，有的字段可能为null，这时就可能会发生空指针异常(也可能会发生其他运行时异常)。

## 4.2、同步代码块

还有一种方式是使用同步代码块的方式，这种方法与`synchronized`方法在功能上基本一致。不同之处在于：同步代码块可以通过`synchronized (xxx)`锁住任意对象，另外这种方法能有效的控制同步锁的粒度，避免了对大范围的代码加锁。代码如下：

```java
    static class Counter {
        private int c = 0;

        public void increment() {
            synchronized (this) {
                c++;
            }
        }

        public void decrement() {
            synchronized (this) {
                c--;
            }
        }

        public int value() {
            return c;
        }
    }
```

> 注意：如果`increment`和`decrement`方法`synchronized`锁住的不是同一个对象，无法实现这两个方法的线程同步。

事实上只要锁住的是同一个对象就可以实现同步：

```java
    static class Counter {
        private Object lock = new Object();
        private int c = 0;

        public void increment() {
            synchronized (lock) {
                c++;
            }
        }

        public void decrement() {
            synchronized (lock) {
                c--;
            }
        }

        public int value() {
            return c;
        }
    }
```

## 3、synchronized使用注意事项

* 只能锁定对象，不能锁定基本数据类型(int,float等8种基本数据类型)；
* 被锁定的对象数组中的单个对象不会被锁定；
* 同步方法可以视作整个方法的synchronized(this){...}同步代码块 (但它们最终生成的二进制字节码是不一样的)
* 静态同步方法会锁定类的Class对象，因为静态方法没有实例对象可以锁定；
* 如果要锁定一个类对象，请谨慎考虑使用synchronized(Xxx.class)显式锁定，还是synchronized(obj.getClass())，两种方式对子类的影响不同；
* 内部类的同步是独立于外部类的，因为内部类本质上就是加了外部类的类名作为前缀OutClass$InnerClass，不论是成员内部类还是静态内部类，成员内部类之所以能访问外部类对象的属性和方法，主要是因为它持有外部类对象的引用；
* synchronized不是方法签名的组成部分，所以不能出现在接口的方法声明中；
* Java的synchronized线程锁是可重入的，也就是说持有锁的线程遇到同一个锁的同步点时是能继续的（比如一个同步方法调用同一个类中的另一个同步方法，或者递归调用同一个方法）。

> 关于synchronized的实现原理参考[下一篇文章](https://blog.csdn.net/Holmofy/article/details/73302423)