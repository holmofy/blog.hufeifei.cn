---
title: Java多线程复习与巩固（八）--原子性操作与原子变量
date: 2017-06-26 14:54
categories: JAVA
---

前面讲[线程同步](http://blog.csdn.net/holmofy/article/details/73264273)时，我们对多线程容易出现的问题进行了分析，在那个例子中，问题的根源在于`c++`和`c--`这两个操作在底层处理的时候被分成了若干步执行。当时我们用的是`synchronized`关键字来解决这个问题，而从[synchronize的实现原理](http://blog.csdn.net/holmofy/article/details/73302423)中我们知道`synchronized`通过`monitor`监视器来实现线程同步，这种同步方式要求线程等待`monitor`的拥有者线程释放后，才可能进一步执行，而线程等待可能会导致**线程上下文的切换(Context Switch)**，线程上下文的切换会带来极大的开销：保存和恢复线程当前的执行状态(如程序计数器，线程执行栈等)。这片文章中我们使用另一种方式来解决前面提出的多线程问题。


# 使用原子操作来解决多线程的问题

先贴出代码：

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadCommunicate {
    static class Counter {
        private AtomicInteger c = new AtomicInteger(0);

        public void increment() {
            c.getAndIncrement();
        }

        public void decrement() {
            c.getAndDecrement();
        }

        public int value() {
            return c.get();
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

    private static Counter counter = new Counter();

    public static void main(String[] args) throws InterruptedException {
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

程序运行结果：

```shell
0
```

上面代码中使用了`java.util.concurrent.atomic`包中的一个类**`AtomicInteger`**，使用的是类中的`getAndIncrement`和`getAndDecrement`方法，这两个方法类似于之前例子中的`c++`，`c--`操作。

`AtomicInteger`是对`int`类型的封装，**`AtomicInteger`类中的方法能保证对内存中的`int`值的操作都是原子性的**，换句话说就能保证一个线程在对`int`操作的过程中不会被另一个线程打断，从而使得两个线程不会发生[前面文章中](http://blog.csdn.net/holmofy/article/details/73264273)出现的指令交叉执行的现象。

> 对于单处理机CPU来说，[原子操作](http://baike.baidu.com/item/%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C)指的是一个不会被“线程调度机制”打断的操作，这种操作一旦开始，就一直占用CPU直到操作结束，中间不会有任何上下文切换(context switch，切换到另外的进程或线程)。
>
> 对于多处理机CPU来说，[原子操作](https://en.wikipedia.org/wiki/Linearizability)不仅仅具有前面的那些性质，还应包括“在一个处理机上的操作不会受其他处理机的影响”这一特性，比如说一个处理机修改内存的时候另一个处理机不能修改内存。

# `java.util.concurrent.atomic`包

像`AtomicInteger`这样的类还有很多，它们都在`java.util.concurrent.atomic`包中，这些类都是无锁的、线程安全的。

![原子操作类](http://img.blog.csdn.net/20170626181153640?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从功能上来说，上面这些类主要分为以下几种：

## 1. 单一值原子性封装

`AtomicBoolean`、`AtomicInteger`、`AtomicLong`、`AtomicReference`是对`volatile`修饰的单一值进行封装。

由于**`volatile`关键字只能保证多线程读取(get)、写入(set)操作的一致性，但不能保证多线程修改操作(++,--等操作)的原子性**。但`AtomicInteger`和`AtomicLong`类内部使用[CAS操作](http://blog.csdn.net/holmofy/article/details/73824757)保证了`getAndIncrement`(i++)，`getAndDecrement`(i--),`incrementAndGet`(++i),`decrementAndGet`(--i)等这类操作的原子性。

特别地，`AtomicBoolean`底层使用`int`存储，用`1`表示`true`，用`0`表示`false`，因为在Java中[`boolean`类型的字节长度是不确定的](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3.4)，单个的`boolean`编译时会被映射为`int`类型，`boolean`数组编译时才会被映射为`byte`类型的数组。用`1`表示`true`，用`0`表示`false`。

JDK没有提供`byte`、`short`、`float`、`double`、`char`的包装类，[Java官方文档](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html#package.description)给出的建议是使用已有的`AtomicInteger`和`AtomicLong`来自己实现相应的包装类。比如：

* 用`AtomicInteger`来存储`byte`数据，进行相应的强制转换即可；
* 用`AtomicInteger`来存储`float`数据，并使用`Float.floatToRawIntBits(float)`和`Float.intBitsToFloat(int)`方法进行转换；
* 用`AtomicLong`来存储`double`数据，并使用`Double.doubleToRawLongBits(double)`和`Double.longBitsToDouble(long)`进行转换。

Demo：

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicFloat extends Number {
    private int float2Int(float value) {
        return Float.floatToRawIntBits(value);
    }

    private float int2Float(int value) {
        return Float.intBitsToFloat(value);
    }

    private AtomicInteger bits;

    public AtomicFloat() {
        this(0f);
    }

    public AtomicFloat(float initialValue) {
        bits = new AtomicInteger(float2Int(initialValue));
    }

    public final boolean compareAndSet(float expect, float update) {
        return bits.compareAndSet(float2Int(expect), float2Int(update));
    }

    public final void set(float newValue) {
        bits.set(float2Int(newValue));
    }

    public final float get() {
        return int2Float(bits.get());
    }

    public float floatValue() {
        return get();
    }

    public final float getAndSet(float newValue) {
        return int2Float(bits.getAndSet(float2Int(newValue)));
    }

    public final boolean weakCompareAndSet(float expect, float update) {
        return bits.weakCompareAndSet(float2Int(expect), float2Int(update));
    }

    public double doubleValue() {
        return (double) floatValue();
    }

    public int intValue() {
        return (int) get();
    }

    public long longValue() {
        return (long) get();
    }
}
```
>  Guava的`com.google.common.util.concurrent.AtomicDouble`包有这些扩展类的实现，
>  如[AtomicDouble](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/AtomicDouble.java)、[AtomicDoubleArray](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/AtomicDoubleArray.java)
## 2. 数组值原子性封装

 `AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`是对数组类型的值进行原子性操作的封装。

> 这三个类在方法中多传入一个**索引作为参数**来访问数组中的元素，`AtomicIntegerArray`使用`int[]`存储，`AtomicLongArray`使用`long[]`存储，`AtomicReferenceArray`使用`T[]`泛型数组存储。

## 3. 对象字段原子性封装

`AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater`是对类对象的某个字段进行原子操作。

这三个类都是抽象类，但是它们都提供了一个工厂方法`newUpdater(Class<U> tclass, String fieldName)`来创建内部实现类的实例。

这三个类主要用在**已经封装好的类**，我们**无法对这个类的代码进行修改**，但是**却要保证里面某些字段的操作是原子性的**。

## 4. 对象标志原子性封装

`AtomicMarkableReference`、`AtomicStampedReference`是对`AtomicReference`类的扩展。

这两个类的区别在于`AtomicMarkableReference`使用`boolean`与引用类型的值进行关联，这个布尔值用来标识这个引用对象是否被；而`AtomicStampedReference`使用`integer`与引用类型的值进行关联，你可以使用这个`integer`代表引用数据更新的**版本**数值。

> [下一篇文章](http://blog.csdn.net/Holmofy/article/details/73824757)讲CAS操作的ABA问题时会提到这两个类的用处

下面的代码是JDK1.8中`AtomicMarkableReference`、`AtomicStampedReference`的部分代码(1.8之前实现有所不同)：

```java
public class AtomicMarkableReference<V> {
    private static class Pair<T> {
        final T reference;
        final boolean mark;
        private Pair(T reference, boolean mark) {
            this.reference = reference;
            this.mark = mark;
        }
        static <T> Pair<T> of(T reference, boolean mark) {
            return new Pair<T>(reference, mark);
        }
    }
    private volatile Pair<V> pair;
    public AtomicMarkableReference(V initialRef, boolean initialMark) {
        pair = Pair.of(initialRef, initialMark);
    }
    ...
}

public class AtomicStampedReference<V> {
    private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
    private volatile Pair<V> pair;
    public AtomicStampedReference(V initialRef, int initialStamp) {
        pair = Pair.of(initialRef, initialStamp);
    }
    ...
}
```


> 在Java1.8中还增加了`DoubleAccumulator`、`DoubleAdder`、`LongAccumulator`、`LongAdder`这四个类用于并发累积计数。