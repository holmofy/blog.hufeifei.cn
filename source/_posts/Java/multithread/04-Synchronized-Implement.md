---
title: Java多线程复习与巩固（四）--synchronized的实现
date: 2017-06-17
categories: JAVA
keywords:
- Java多线程,synchronized实现,CAS锁,偏向锁,锁升级
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

# 1、温故知新

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

# 2、反编译代码

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

   ```asm
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

1. 普通方法和`synchronized`方法在方法内部没有任何区别，仅仅是`synchronized`方法比普通方法多了一个[`ACC_SYNCHRONIZED`标志位](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.6)，该标志位表示访问该方法需要同步访问(synchronized access)
2. `synchronized`同步代码块中由于要加载this引用，多了很多指令，而关键的两个指令是`monitorenter`，`monitorexit`。

# 3、JVM规范中的Monitor

Oracle官网提供的[JVM规范](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter)中对`monitorenter`和`monitorexit`两个指令有以下介绍：

**monitorenter**指令用于进入对象的`monitor`监视器，该指令的操作数是栈中的对象引用(*objectref*)，这个对象引用必须是引用类型，不能是基本类型。每个对象都关联着一个`monitor`监视器，当且仅当`monitor`监视器有线程所有者(owner)才会被锁住，线程通过执行`monitorenter`指令尝试获取对象关联的`monitor`监视器的所有权：

* 如果一个对象(*objectref*)关联的`monitor`监视器的`entry`为0，该线程进入`monitor`监视器并将它的`entry`设置为1。该线程则是`monitor`的所有者。
* 如果一个线程已经拥有对象(*objectref*)关联的`monitor`监视器，就让它再次进入该`monitor`监视器，并将`entry`加1(可重入锁)。
* 如果另一个线程已经拥有了对象(*objectref*)关联的`monitor`监视器，那么该线程将会阻塞，直到`monitor`监视器的`entry`为0时再次尝试获取所有权。

注意：

* 如果引用对象为null，`monitorenter`指令将会抛出空指针异常。
* `monitorenter`指令可以配合一个或多个`monitorexit`指令来实现`synchronized`代码块。虽然它们提供了锁定语义，但在`synchronized`**方法中**并不会执行这两个指令，而是在调用`synchronized`方法时`monitor`进入，在方法返回时`monitor`退出，这个将由JVM方法调用和返回指令隐式地处理。
* Java的同步机制除了要实现`monitorenter`和`monitorexit`这样的操作，还应该包括等待`monitor`监视器(Object.wait)，通知等待`monitor`监视器的其他线程(Object.notify和Object.notifyAll)。JVM指令中不会提供这些操作的支持。
* JVM规范并没有规定如何实现`monitor`与对象的关联。

**monitorexit**用于退出对象的`monitor`监视器：

* 执行`monitorexit`指令的线程必须是该`monitor`监视器的所有者。执行该指令时，该线程会将`monitor`的`entry`减一。如果`entry`进入次数的结果为0，该线程将退出`monitor`不再是它的所有者。其他进入`monitor`的阻塞线程可以尝试获取该`monitor`监视器。

JVM规范文档说的非常清楚明白，`synchronized`关键字是由`monitor`监视器实现的。

# 4、Hotspot中的锁及其优化

> Hotspot介绍可以参考[Wiki](https://en.wikipedia.org/wiki/HotSpot)
>
> Hotspot源码可以从[官网](http://download.java.net/openjdk/jdk8/)或[这里](http://download.csdn.net/detail/holmofy/9873419)下载
>
> Hotspot偏向锁的介绍可参考[这里](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)

### 4.1、轻量级锁、重量级锁、偏向锁

在Java Hotspot中，每一个对象前面都有一个类指针和一个头字段。头字段中存储了一个哈希码(HashCode)以及一个标志位，该标志位用于标识对象的年龄(新生代，老年代等)，同时它也被用来实现轻量锁。下面这张图展示了头字段的位置以及不同对象状态下的字段值。

![object header](http://tva1.sinaimg.cn/large/bda5cd74ly1g2jjj6iyfqg20h707xwey.gif)

图的右侧描述了标准的对象锁定过程。只要对象没被锁住，头字段的后两位将会置为01。当一个方法在对象上同步时，头字段和对象指针会被存储到当前线程栈帧的`lock record`上。然后VM通过`compare-and-swap`操作尝试**将`lock record`的指针存入对象头字段**中。如果成功（对象头字段与预期的`hash～age～01`相等），那么当前线程就获取了锁。由于`lcok record`总是按字边界对齐，头字段最后两位为00并以此标记对象被锁定。

![](http://tva1.sinaimg.cn/large/bda5cd74ly1g2jm0udbqvj20bx0gdt9m.jpg)

如果由于对象之前已经锁住导致`compare-and-swap`操作失败，虚拟机首先会测试对象头字段是否指向当前线程的方法堆栈。这种情况下，说明当前线程已经拥有对象锁，然后能安全地继续接下来的执行逻辑。对于这种递归锁住一个对象的情况，`lock record`会被初始化为0而非对象的头字段。除非两个不同的线程并发竞争同一个对象的锁，`thin lock`才必须升级膨胀成重量级的`monitor`去管理等待的线程。

`thin lock`比升级的锁消耗更小，但它的性能也会受影响，因为每个`compare-and-swap`操作在多处理器上必须原子性地执行，然而大多数对象只会被一个特定线程加解锁。

在Java6中，这个问题被所谓的免存储偏向锁技术解决了。由于大部分对象大多数情况最多由一个线程锁定，**为了避免过多无谓的CAS操作**，我们允许线程偏向某个线程，这也是"偏向锁"(`Biased Lock`)这个名字的来由。

只有第一次获取锁时会执行一次`compare-and-swap`将上锁线程的ID存入对象头字段中，这时我们称**这个对象偏向这个线程**。将来同一线程的加解锁操作都不再需要任何原子操作或头字段修改操作，执行栈中的`lock record`也不会被初始化为0了，因为线程不会再去检查偏向锁的对象。 

当线程在偏向另一个线程的对象上同步时(产生多线程竞争同一个锁的情况)，偏向锁将会被撤销，使对象看上去是以常规方式锁定的一样。遍历拥有偏向锁线程的堆栈，关联的`lock record`将会按照`thin lock`的策略进行调整，并将`lock record`的指针置入对象头字段。当访问对象的哈希码时，偏向锁也被撤销，因为哈希码位与线程ID共享。

明确设计为多个线程之间共享的对象不适用于偏向锁定，比如生产者/消费者共同操作的队列。因此，如果某个类的实例在过去经常发生偏向锁撤销，则会禁用偏向锁，这叫做**批量撤销(bulk revocation)**。如果在禁用了偏向锁的类实例上调用锁定代码，则它将执行标准的轻量级锁。新分配的类实例会被标记为不可偏置。

类似的机制称为**批量偏置(bulk rebiasing)**，它优化了类的对象被不同的线程加解锁但从不并发的情况。它会使类的所有实例的偏向锁暂时无效，而不是禁用偏置锁。类中的`epoch`值用于指示偏向锁有效性的时间戳。在对象分配时将该值复制到对象头中。然后，批量偏置可以有效地实现为适当类中的`epoch`的增加。下一次要锁定此类的实例时，代码会在对戏那个头字段中检测到不同的值，并将对象重新映射到当前线程。

### 4.2、Hotspot源码分析

[oopDesc](https://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/oop.hpp)类是对象类的顶级基类：

![oopDesc](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuU9ApaaiBbR8pozmIIrEvkA2CXHiT7KL4ekA4Ylz8G8e4fbvGFrvoQamEIweIeIA_AGiHoGnJ0KbbGMfA2gu75BpKe0k0W00)

> Java8之后Method和Class等数据不再继承自oopDesc体系，而是被作为MetaspaceObj移入元数据区管理。

这里的oop不是面向对象编程，而是[`Ordinary Object Pointers`](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)的缩写。

意思就是对象的托管指针。正常情况下它与机器原生指针大小相同。Java应用程序和GC子系统会仔细跟踪托管指针，以便可以回收垃圾对象。此过程还可能涉及垃圾回收过程中存活对象的重新定位（复制算法与整理算法）。

> 关于垃圾回收的内容可以参考[这篇文章](https://blog.csdn.net/Holmofy/article/details/84862210)

**oopDesc的定义**如下：

```cpp
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
  // ...
}
```

每个对象头部都有一个markOop类型的4字节头字段。

![Mark Word](http://tva1.sinaimg.cn/large/bda5cd74ly1g3djka7cboj20f509p3z2.jpg)

这个头字段里就记录着前面说到的轻量级锁、重量级锁、偏向锁等信息。

接下来我们就看`synchronized`整个同步过程是如何操作这个头字段的。

在Hotspot中`synchonized`同步由[`ObjectSynchronizer`](https://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/runtime/synchronizer.hpp)实现。

```cpp
class ObjectSynchronizer : AllStatic {
  // 有偏向锁优化的同步
  static void fast_enter  (Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS);
  static void fast_exit   (oop obj,    BasicLock* lock, Thread* THREAD);

  // 线程竞争激烈偏向锁获取失败，就会使用标准加锁过程
  static void slow_enter  (Handle obj, BasicLock* lock, TRAPS);
  static void slow_exit   (oop obj,    BasicLock* lock, Thread* THREAD);
  //...

  // 锁升级，由轻量级锁膨胀到Monitor重量级锁
  static ObjectMonitor* inflate(Thread * Self, oop obj);
```

fast_enter的实现如下：

```cpp
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
    // 启用了偏向锁优化
    if (!SafepointSynchronize::is_at_safepoint()) {
      // 当前线程尝试获取偏向锁，如果没获取成功，则撤销偏向
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        // 成功获取偏向锁
        return;
      }
      // 没有获取到偏向锁，那必定已经撤销了偏向
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    // 断言偏向锁已经撤销
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }

 // 关闭了偏向锁或偏向锁获取失败，则会使用标准上锁过程
 slow_enter (obj, lock, THREAD) ;
}
```

slow_enter的实现如下：

```cpp
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  // 保证偏向锁已经撤销
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  if (mark->is_neutral()) {
    // 如果是无锁状态，尝试获取轻量级锁
    lock->set_displaced_header(mark);
    // 用CAS操作将LockRecord的指针存入头字段
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      // 轻量级锁获取成功
      return ;
    }
    // 轻量级锁获取失败则调用下面的inflate膨胀升级成重量级锁 ...
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    // 轻量级锁重入
    // ...
    lock->set_displaced_header(NULL);
    return;
  }

#if 0
  // The following optimization isn't particularly useful.
  if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
    // 已经升级成重量级锁，检查是否是当前线程获取了重量级锁
    lock->set_displaced_header (NULL) ;
    return ;
  }
#endif

  // 锁升级
  lock->set_displaced_header(markOopDesc::unused_mark());
  // 升级成ObjectMonitor后，点用其enter方法
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```

inflate的实现如下：

```cpp
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  // ...
  // for循环，处理多个线程同时调用inflate的情况。
  for (;;) {
      const markOop mark = object->mark() ;
      assert (!mark->has_bias_pattern(), "invariant") ;

      // mark是以下状态中的一种：
      // *  Inflated（已膨胀的重量级锁状态） - 直接返回
      // *  Stack-locked（轻量级锁状态）    - 膨胀成重量级锁
      // *  INFLATING（膨胀中）            - 忙等待直到膨胀完成
      // *  Neutral（无锁状态）            - 膨胀
      // *  BIASED（偏向锁）               - 非法状态，在这里不会出现

      // CASE: inflated
      if (mark->has_monitor()) {
          // 直接返回重量级锁ObjectMonitor
          ObjectMonitor * inf = mark->monitor() ;
          // ...
          return inf ;
      }

      // CASE: inflation in progress
      if (mark == markOopDesc::INFLATING()) {
         // 正在膨胀中，说明另一个线程正在进行锁膨胀，continue重试
         TEVENT (Inflate: spin while INFLATING) ;
         // 在该方法中会进行spin/yield/park等操作完成自旋动作 
         ReadStableMark(object) ;
         continue ;
      }

      // CASE: stack-locked
      if (mark->has_locker()) {
          // 当前轻量级锁状态，先分配一个ObjectMonitor对象，并初始化值
          ObjectMonitor * m = omAlloc (Self) ;
        
          m->Recycle();
          m->_Responsible  = NULL ;
          m->OwnerIsThread = 0 ;
          m->_recursions   = 0 ;
          m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;
          // 将锁对象的mark word设置为INFLATING (0)状态 
          markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
          if (cmp != mark) {
             omRelease (Self, m, true) ;
             continue ;       // Interference -- just retry
          }

          // 已经成功将mark-word设置成INFLATING (0)
          markOop dmw = mark->displaced_mark_helper() ;
          assert (dmw->is_neutral(), "invariant") ;

          // 将ObjectMonitor设置到mark-word中
          m->set_header(dmw) ;

          // Monitor的Owner为Lock Record
          m->set_owner(mark->locker());
          // ObjectMonitor关联对象
          m->set_object(object);

          // 将锁对象头设置为重量级锁状态
          guarantee (object->mark() == markOopDesc::INFLATING(), "invariant") ;
          object->release_set_mark(markOopDesc::encode(m));
          
          // ...
          return m ;
      }

      // CASE: neutral

      // 分配以及初始化ObjectMonitor对象
      ObjectMonitor * m = omAlloc (Self) ;
      // 初始化ObjectMonitor
      m->Recycle();
      m->set_header(mark);
      m->set_owner(NULL);
      m->set_object(object);
      m->OwnerIsThread = 1 ;
      m->_recursions   = 0 ;
      m->_Responsible  = NULL ;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;
      // 用CAS替换对象头的mark word为重量级锁状态
      if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr(), mark) != mark) {
          // 不成功说明有另外一个线程在执行inflate，释放monitor对象
          m->set_object (NULL) ;
          m->set_owner  (NULL) ;
          m->OwnerIsThread = 0 ;
          m->Recycle() ;
          omRelease (Self, m, true) ;
          m = NULL ;
          continue ;
      }
      // ...
      return m ;
  }
}
```

升级成ObjectMonitor锁之后，会调用它的enter方法。

[`ObjectMonitor`](https://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/runtime/objectMonitor.hpp)就是JVM规范中定义的与对象关联的监视器对象。

在ObjectMonitor中有以下几个重要字段：

```cpp
class ObjectMonitor {
  // ...
  volatile markOop _header;           // displaced object header word - mark
  void*   volatile _object;           // backward object pointer - strong root
  void*   volatile _owner;            // pointer to owning thread OR BasicLock
  volatile jlong _previous_owner_tid; // thread id of the previous owner of the monitor
  volatile intptr_t  _recursions;     // recursion count, 0 for first entry
  int OwnerIsThread ;                 // _owner is (Thread *) vs SP/BasicLock
  
  volatile int _Spinner ;             // for exit->spinner handoff optimization
  volatile int _SpinFreq ;            // Spin 1-out-of-N attempts: success rate
  volatile int _SpinClock ;
  volatile int _SpinDuration ;
  volatile intptr_t _SpinState ;      // MCS/CLH list of spinners
  
  volatile intptr_t  _count;
  volatile intptr_t  _waiters;        // number of waiting threads
  ObjectWaiter* volatile _WaitSet;    // LL of threads wait()ing on the monitor
  volatile int _WaitSetLock;          // protects Wait Queue - simple spinlock
  // ...
}
```

重点就是它的enter方法：

```cpp
void ATTR ObjectMonitor::enter(TRAPS) {
  Thread * const Self = THREAD ;
  void * cur ;

  // CAS操作尝试将获取锁
  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  if (cur == NULL) {
     // monitor之前没有被其他线程锁住，则当前线程成为monitor的所有者
     // ...
     return ;
  }

  // monitor已被当前线程获取，线程重入monitor，则entry数自增
  if (cur == Self) {
     // TODO-FIXME: check for integer overflow!  BUGID 6557169.
     _recursions ++ ;
     return ;
  }

  // 当前线程是之前持有轻量级锁的线程。
  // 由轻量级锁膨胀且第一次调用enter方法，那cur是指向Lock Record的指针
  if (Self->is_lock_owned ((address)cur)) {
    assert (_recursions == 0, "internal state error");
    _recursions = 1 ;
    // 设置owner字段为当前线程（之前owner是指向Lock Record的指针）
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }
 
  // ...

  // 线程没有获取到monitor，在线程挂起之前，先尝试自旋。
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
     // 自旋过程中获取到了锁，直接返回
     Self->_Stalled = 0 ;
     return ;
  }

  // ...

  {
    // ...
    // 无限for循环，处理挂起后被唤醒，但没有竞争到锁
    for (;;) {
      jt->set_suspend_equivalent();

       // 自旋尝试获取锁，没有获取到则入ObjectWaiter队列，最后调用park方法挂起自己
      EnterI (THREAD) ;

      if (!ExitSuspendEquivalent(jt)) break ;

      _recursions = 0 ;
      _succ = NULL ;
      exit (false, Self) ;

      jt->java_suspend_self();
    }
    Self->set_current_pending_monitor(NULL);
  }
  //...
}
```

JVM代码比较复杂，分析必有不详尽之处。可以参考[这篇文章](
https://www.javazhiyin.com/24370.html)深入考究。



参考链接：

https://wiki.openjdk.java.net/display/HotSpot/Synchronization

http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter

https://blogs.oracle.com/dave/biased-locking-in-hotspot

http://www.spring4all.com/article/16805

https://www.oracle.com/technetwork/java/6-performance-137236.html#2.1.1

https://www.cs.princeton.edu/picasso/mats/HotspotOverview.pdf

https://wiki.openjdk.java.net/display/HotSpot/CompressedOops

https://www.baeldung.com/jvm-compressed-oops

https://docs.oracle.com/javase/7/docs/technotes/guides/vm/performance-enhancements-7.html

https://www.javazhiyin.com/24370.html

https://www.hollischuang.com/archives/1910

https://my.oschina.net/apdplat/blog/208456