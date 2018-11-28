---
title: Java多线程复习与巩固(四)--synchronized的实现
date: 2017-06-15 19:54
categories: JAVA
---

# 小小的回顾

在上一篇文章的例子中有一个Counter类：

```java
static class Counter {
    private int c = 0;
    public void increment() { c++; }
    public void decrement() { c--; }
    public int value() { return c; }
}
```

为了实现线程同步我们使用了`synchronized`关键字，而`synchronized`关键字有两种用法：

1. 同步方法：

   ```java
   static class Counter {
       private int c = 0;
       public synchronized void increment() { c++; }
       public synchronized void decrement() { c--; }
       public int value() { return c; }
   }
   ```

2. 同步代码块：

   ```java
   static class Counter {
       private int c = 0;
       public void increment() { synchronized(this){c++;} }
       public void decrement() { synchronized(this){c--;} }
       public int value() { return c; }
   }
   ```

# 反编译代码

我们把上面三段代码反编译一下，并取出`increment`和`decrement`两个方法的反编译代码：

1. 未加同步

   ```asm

     public void increment();
       descriptor: ()V
       flags: ACC_PUBLIC
       Code:
         stack=3, locals=1, args_size=1
            0: aload_0
            1: dup
            2: getfield      #2                  // Field c:I
            5: iconst_1
            6: iadd
            7: putfield      #2                  // Field c:I
           10: return
         LineNumberTable:
           line 3: 0

     public void decrement();
       descriptor: ()V
       flags: ACC_PUBLIC
       Code:
         stack=3, locals=1, args_size=1
            0: aload_0
            1: dup
            2: getfield      #2                  // Field c:I
            5: iconst_1
            6: isub
            7: putfield      #2                  // Field c:I
           10: return
         LineNumberTable:
           line 4: 0
   ```

2. 同步方法

   ```assembly
     public synchronized void increment();
       descriptor: ()V
       flags: ACC_PUBLIC, ACC_SYNCHRONIZED
       Code:
         stack=3, locals=1, args_size=1
            0: aload_0
            1: dup
            2: getfield      #2                  // Field c:I
            5: iconst_1
            6: iadd
            7: putfield      #2                  // Field c:I
           10: return
         LineNumberTable:
           line 3: 0

     public synchronized void decrement();
       descriptor: ()V
       flags: ACC_PUBLIC, ACC_SYNCHRONIZED
       Code:
         stack=3, locals=1, args_size=1
            0: aload_0
            1: dup
            2: getfield      #2                  // Field c:I
            5: iconst_1
            6: isub
            7: putfield      #2                  // Field c:I
           10: return
         LineNumberTable:
           line 4: 0
   ```

3. 同步代码块

   ```asm

     public void increment();
       descriptor: ()V
       flags: ACC_PUBLIC
       Code:
         stack=3, locals=3, args_size=1
            0: aload_0
            1: dup
            2: astore_1
            3: monitorenter
            4: aload_0
            5: dup
            6: getfield      #2                  // Field c:I
            9: iconst_1
           10: iadd
           11: putfield      #2                  // Field c:I
           14: aload_1
           15: monitorexit
           16: goto          24
           19: astore_2
           20: aload_1
           21: monitorexit
           22: aload_2
           23: athrow
           24: return
         Exception table:
            from    to  target type
                4    16    19   any
               19    22    19   any
         LineNumberTable:
           line 3: 0
         StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
             offset_delta = 19
             locals = [ class Counter, class java/lang/Object ]
             stack = [ class java/lang/Throwable ]
           frame_type = 250 /* chop */
             offset_delta = 4

     public void decrement();
       descriptor: ()V
       flags: ACC_PUBLIC
       Code:
         stack=3, locals=3, args_size=1
            0: aload_0
            1: dup
            2: astore_1
            3: monitorenter
            4: aload_0
            5: dup
            6: getfield      #2                  // Field c:I
            9: iconst_1
           10: isub
           11: putfield      #2                  // Field c:I
           14: aload_1
           15: monitorexit
           16: goto          24
           19: astore_2
           20: aload_1
           21: monitorexit
           22: aload_2
           23: athrow
           24: return
         Exception table:
            from    to  target type
                4    16    19   any
               19    22    19   any
         LineNumberTable:
           line 4: 0
         StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
             offset_delta = 19
             locals = [ class Counter, class java/lang/Object ]
             stack = [ class java/lang/Throwable ]
           frame_type = 250 /* chop */
             offset_delta = 4
   ```

>  [Oracle官网](http://docs.oracle.com/javase/specs/jvms/se8/html/index.html)和[维基百科](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings)有相关指令的介绍

通过看底层字节码，可以看出以下几点：

1. 普通方法和`synchronized`方法在方法内部没有任何区别，仅仅是`synchronized`方法比普通方法多了一个`ACC_SYNCHRONIZED`标志位，该标志位表示访问该方法需要同步访问(synchronized access)
2. `synchronized`同步代码块中由于要加载this引用，多了很多指令，而关键的两个指令是`monitorenter`，`monitorexit`。

# JVM规范中的Monitor

Oracle官网提供的[JVM规范](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter)中对`monitorenter`和`monitorexit`有以下介绍：

**monitorenter**指令用于进入对象的`monitor`监视器，该指令的操作数是栈中的对象引用(*objectref*)，这个对象引用必须是引用类型，不能使基本类型。每个对象都关联着一个`monitor`监视器，当且仅当`monitor`监视器有线程所有者(owner)才会被锁住，线程通过执行`monitorenter`指令尝试获取对象关联的`monitor`监视器的拥有权：

* 如果一个对象(*objectref*)关联的`monitor`监视器的`entry`进入次数为0，该线程进入`monitor`监视器并将它的`entry`进入次数设置为1。该线程则是`monitor`的所有者。
* 如果一个线程已经拥有对象(*objectref*)关联的`monitor`监视器，就让它再次进入该`monitor`监视器，并将`entry`进入次数加一。
* 如果另一个线程已经拥有了对象(*objectref*)关联的`monitor`监视器，那么该线程将会阻塞，直到`monitor`监视器的`entry`进入次数为0时再次尝试获取所有权。

注意：

* 如果引用对象为null，`monitorenter`指令将会抛出空指针异常。
* `monitorenter`指令可以配合`monitorexit`指令来实现`synchronized`代码块。虽然它们提供了锁定语义，但在`synchronized`方法中并不会执行这两个指令，而是在调用`synchronized`方法时`monitor`进入，在方法返回时`monitor`退出。
* Java的同步机制除了要实现`monitorenter`和`monitorexit`这样的操作，还应该包括等待`monitor`监视器(Object.wait)，通知等待`monitor`监视器的其他线程(Object.notify和Object.notifyAll)。JVM指令中不会提供这些操作的支持。

**monitorexit**用于退出对象的`monitor`监视器：

* 执行`monitorexit`指令的线程必须是该`monitor`监视器的所有者。执行该指令时，该线程会将`monitor`的`entry`进入次数减一。如果`entry`进入次数的结果为0，该线程将退出`monitor`不再是它的所有者。其他进入`monitor`的阻塞线程可以尝试获取该`monitor`监视器。

JVM规范文档说的非常清楚明白，`synchronized`关键字是由`monitor`监视器实现的。

`monitor`底层又是如何实现JVM规范中提到的要求呢，接下来我们从JVM实现源码来了解一下Monitor的具体实现。

# Hotspot中Monitor的实现

> Hotspot介绍可以参考[Wiki](https://en.wikipedia.org/wiki/HotSpot)
>
> Hotspot源码可以从[官网](http://download.java.net/openjdk/jdk8/)或[这里](http://download.csdn.net/detail/holmofy/9873419)下载

在Java Hotspot中，每一个对象前面都有一个类指针和一个头字段。头字段中存储了一个哈希码(HashCode)标识值以及一个标志位，该标志位用于标识对象的年龄(新生代，老年代等)，同时它也被用来实现轻量锁。下面这张图展示了头字段的位置以及不同对象状态下的字段值。
