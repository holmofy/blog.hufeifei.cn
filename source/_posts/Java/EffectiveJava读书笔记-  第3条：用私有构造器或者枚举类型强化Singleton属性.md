---
title: EffectiveJava读书笔记-  第3条：用私有构造器或者枚举类型强化Singleton属性
date: 2018-02-14 17:10
categories: JAVA
---

# 用私有构造器或枚举类型强化Singleton属性

[单例模式(Singleton Pattern)](https://en.wikipedia.org/wiki/Singleton_pattern)无疑是笔试面试中被问得最多的问题之一。单例模式虽然看似简单，但是仍有很多东西值得思考。

GOF是这么定义单例模式的：

>  **确保一个类只有一个实例，并提供一个全局访问点。**

通常实现单例都需要我们私有化构造器，让对象无法在外部创建，同时提供一个外部访问的方法返回这个单例对象。

通常单例分为两大类实现：饿汉式和懒汉式。

## 饿汉式单例

> 所谓“饿汉式单例”就是**在类加载器加载这个类的时候就立马创建这个类的单例对象**。

### 1. 使用静态常量域提供外部访问

```java
public class Singleton {
    public static final Singleton INSTANCE = new Singleton();
    private Singleton() {/* 私有化构造器 */}
    public void doSomething() {
        ...
    }
}
```

### 2. 使用静态工厂方法提供外部访问

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {/* 私有化构造器 */}
    public static Singleton getInstance() {
        return INSTANCE;
    }
    public void doSomething() {
        ...
    }
}
```

静态工厂方法相对于静态常量域的好处是可以在不改变API的前提下，可以改变该类是否为单例的想法。

### 3. 防止反射调用私有构造器

上面的私有构造方法仍有缺少保护，外部的调用者仍可以使用反射机制`AccessibleObject.setAccessible()`方法来访问私有构造方法：

```java
public class SingletonTest {
    @Test
    public void testReflect()
            throws NoSuchMethodException, SecurityException, InstantiationException,
            IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton newInstance = constructor.newInstance();
        Assert.assertNotEquals(Singleton.getInstance(), newInstance);
    }
}
```

所有我们要对构造方法更狠一点：

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {
        if(INSTANCE != null)
          throw new IllegalStateException("The object can only be created once");
    }
    public static Singleton getInstance() {
        return INSTANCE;
    }
    public void doSomething() {
        ...
    }
}
```

### 4. 防止反序列化导致的多个实例

如果我们的Singleton类实现了Serializable接口，上面构造器检测抛异常的方式也无法阻止反序列化创建新实例。

```java
public class SingletonTest {

    @Test
    public void testSeriliable() {
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("D:/singleton.obj"))) {
            oos.writeObject(Singleton.getInstance());
        } catch (Exception ignore) {}

        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream("D:/singleton.obj"))) {
            Object newInstance = ois.readObject();
            Assert.assertNotEquals(Singleton.getInstance(), newInstance);
        } catch (Exception ignore) {}
    }
}
```

我们只需要定义一个`readResolve`即可：

```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
        if (INSTANCE != null)
            throw new RuntimeException("The object can only be created once");
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        System.out.println("do something");
    }

    // 访问修饰符可以任意
    private Object readResolve() {
        return INSTANCE;
    }
}
```

> 这个方式是《Effective Java》中推荐的做法。关于readResolve的原理，可以参考[Java对象序列化规范](https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serialTOC.html)或者[StackOverflow对这个问题的讨论：用Java如何高效的实现单例](https://stackoverflow.com/questions/70689/what-is-an-efficient-way-to-implement-a-singleton-pattern-in-java/71399#71399)

### 5. 使用单元素枚举类实现单例

使用枚举类实现单例是《Effective Java》中推荐的最佳方法：

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("do something");
    }
}
```

这种方式和最开始的使用静态常量域的方式差不多，但是它更简洁；由于枚举类的特性，它能绝对地防止多次实例化，并且无偿地提供了序列化的机制，这种方式完全避免了前面的反射和反序列化的问题。

## 懒汉式单例

> 所谓“懒汉式单例”就是**在加载这个类的时候不立即创建对象，而是等到第一次用到单例对象的时候临时创建单例对象**。对于一些大对象来说，**懒加载**还是很有必要的。

### 1. 使用静态工厂方法实现懒汉式单例

很显然为了能实现懒汉式单例，我们肯定不能**直接**使用静态常量了，所以只能用静态工厂方法实现懒汉式单例了。

```java
public class Singleton {
    private static Singleton INSTANCE;
    private Singleton() {/* 私有化构造器 */}
    public static Singleton getInstance() {
        if(INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
    public void doSomething() {
        ...
    }
}
```

### 2. 同步方法解决多线程问题

上面的单例在单线程环境下确实没啥毛病，但是在多线程环境下**可能**就会出现问题：可能会有多个进程同时通过 `(INSTANCE == null)`的条件检查，于是，多个实例就创建出来，如果在C++里面创建的对象没有销毁就会导致内存泄漏(多线程的世界真可怕(╯︵╰))，不过好在java天生支持多线程同步，我们可以在静态工厂方法上添加`synchronized`关键字实现线程同步访问：

```java
public class Singleton {
    private static Singleton INSTANCE;

    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }

    public void doSomething() {
        ...
    }
}
```

### 3. 使用同步代码块减小锁粒度

线程同步问题是解决了，但是**每次调用getInstance方法的时候都去检查同步锁**肯定会影响程序执行的效率，虽然现在JVM对`synchronized`的优化做的越来越好，但是调用次数多了整体效率肯定下降，所以我们有必要减小锁粒度。

第一步：

```java
    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                INSTANCE = new Singleton();
            }
        }
        return INSTANCE;
    }
```

上面的做法可行吗，很显然是不可以滴，多个线程仍然会进入 `(INSTANCE == null)`条件，这里的同步只是让多个线程排队去创建对象而已。

第二步：

```java
    public static Singleton getInstance() {
        synchronized (Singleton.class) {
            if (INSTANCE == null) {
                INSTANCE = new Singleton();
            }
        }
        return INSTANCE;
    }
```

这种做法，和使用静态代码块差不多，每次调用getInstance方法的时候仍然会去检查同步锁。

第三步：

```java
    public static Singleton getInstance() {
        // DCL
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
```

这个[**Double Checked Locking**](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)总该差不多了吧，不好意思，还不够。

在**多处理器共享内存(shared memory multiprocessors)**或者**编译器优化(optimizing compilers)进行指令重排**的情况下仍有可能会导致创建多个对象。

对于这个问题我这有两种解释：

**1.** 多处理器共享内存：

处理器p1创建完Singleton对象，并把它赋值给INSTANCE变量走出了同步代码块，但是INSTANCE变量并没有立即反映到内存上(处理器直接操作cache高速缓存，并不直接操作内存)，这时处理器p2进入同步代码块后发现INSTANCE仍为null，就会创建另一个Singleton对象。

**2.** 编译器优化时进行指令重排：

`INSTANCE = new Singleton();`这句话大概会分三步走：

1. `new`：要求操作系统进行内存分配
2. `Singleton()`：调用类的构造函数对分配的内存进行初始化
3. `=`：将新创建的对象的地址赋值给INSTANCE变量

但是JVM在将字节码翻译成机器码的过程中可能会对指令进行重新排列(学过编译原理的应该都知道编译器会对指令进行优化重排)，这个时候第2步和第3步的先后顺序就不确定了，如果线程1按照`1->3->2`的顺序执行，线程2得到的就是一个还未初始化的实例对象，然后就报错了。

这个时候我们用上[`volatile`关键字](https://en.wikipedia.org/wiki/Volatile_(computer_programming))就能解决了。

> 关于volatile关键字的一些用法我之前也写过一篇文章：http://blog.csdn.net/Holmofy/article/details/73824757

第四步：

```java
public class Singleton {
    // 使用volatile关键字
    private static volatile Singleton INSTANCE;

    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }

    public void doSomething() {
        ...
    }
}
```

### 4. 使用私有静态内部类保存单例

上面的方法也太麻烦了吧，一个单例都要搞老半天，有没有更简单的方法。

《Effective Java》第一版推荐的方法：

```java
public class Singleton {

    // 静态内部类包装实例
    private static class InstanceHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton() {}

    public static Singleton getInstance() {
        return InstanceHolder.INSTANCE;
    }

    public void doSomething() {
        System.out.println("do something");
    }

}
```

因为使用了私有静态内部类，所以只有当第一次调用getInstance方法时才会加载这个静态内部类，然后才会去创建对象；读取的时候又没有进行线程同步不影响性能(简直完美了)。



## 什么时候单例不是单例

之前在StackOverflow中看到有讨论不同类加载器下单例模式会出现问题，然后在Oracle官网找到了[这篇文章](http://www.oracle.com/technetwork/articles/java/singleton-1577166.html)

**1. 两个或多个JVM中有多个单例对象**

由于程序在不同的JVM上运行，很明显每个JVM都会有自己的Singleton实例。但是在基于分布式技术的系统（如EJB，RMI和Jini）可以让不同的JVM中的两个对象保持相同的状态。

**2. 不同的类加载器会创建多个单例对象**

一个JVM可以有多个ClassLoader，当两个ClassLoader加载一个类时，实际上有两个class副本，然后每个class都有它自己的Singleton实例。有一些Servlet容器(比如[iPlanet](http://www.mathewsystems.com/))每个Servlet都有自己的类加载器，那么两个不同的Servlet将访问不同的Singleton对象。如果你的程序中也有自定义ClassLoader，那么务必注意这个问题。



参考链接：

StackOverflow: https://stackoverflow.com/questions/70689/what-is-an-efficient-way-to-implement-a-singleton-pattern-in-java/71399#71399

The "Double-Checked Locking is Broken" Declaration：http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html

How to Simply Singleton：https://www.javaworld.com/article/2073352/core-java/simply-singleton.html

When is a Singleton not a Singleton? http://www.oracle.com/technetwork/articles/java/singleton-1577166.html