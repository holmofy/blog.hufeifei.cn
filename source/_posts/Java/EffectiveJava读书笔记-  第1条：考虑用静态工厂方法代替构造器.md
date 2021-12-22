---
title: EffectiveJava读书笔记-  第1条：考虑用静态工厂方法代替构造器
date: 2018-02-12
categories: JAVA
---

## 考虑静态工厂方法代替构造器

静态工厂方法相对于构造器的好处：



**1. 静态工厂方法有名字，相比构造器创建的对象更语义化**

最好的例子就是并发库中的`Executors`工具类了。

Executors中有多个创建线程池的方法：

```java
public static ExecutorService newFixedThreadPool(int nThreads);
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory);
public static ExecutorService newSingleThreadExecutor();
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory);
public static ExecutorService newCachedThreadPool();
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory);
```

我们都知道这几个方法最终创建的对象都是ThreadPoolExecutor对象。

如果要你自己通过ThreadPoolExecutor的构造函数创建线程池，而你又不[了解ThreadPoolExecutor构造函数中几个参数的意义以及ThreadPoolExecutor的一些内部实现原理](http://blog.csdn.net/holmofy/article/details/77411854)的话，你很难知道创建的线程池在将来的使用中会有什么样的行为。

而通过这几个静态工厂方法的方法名我们就已经大概知道创建了一个什么样的线程池。所以将来开发的过程中多考虑使用这种语义明确的静态工厂方法吧。

**2. 不必在每次调用的时候都创建一个新对象**

书上举了`Boolean.valueOf(boolean)`这个例子，还说这种方式类似于享元模式。其实个人比较喜欢把享元模式叫做**对象池**模式，这种设计模式通常用来**存储“创建起来比较耗时或耗资源”的大对象**，比如数据库连接池以及上面刚说到的线程池。Boolean中缓存的TRUE，FALSE并不是什么大对象，但却是我们**经常使用到的小对象**，为了避免频繁创建和GC回收对象，Boolean类就保存了这两个对象的引用。除了Boolean这个包装类，Byte、Short、Integer、Long中都对`[-128,127]`区间内的对象进行了缓存。

但是我觉得下面这个例子更能贴合我们的实际开发：

`Charset.forName(String charsetName)`

Charset是`java.nio.charset`包下面的类，这个方法是用于根据字符集的名字创建Charset对象的。

这个静态工厂方法做了一个很简单但又很复杂的LRU缓存：这个缓存使用两个数组存储两组键值对，key为charsetName字符串，value为Charset对象。

当然我在这里举这个例子并不是想说这个缓存算法有多么的高明，只是觉得我们可以用合适的缓存算法来**避免重复的创建对象，同时避免对象一直占据内存不得回收**。要想简单的实现一个对象缓存，可以使用JDK自带的LinkedHashMap或WeakHashMap两个类，前者基于LRU算法，后者基于GC回收算法。

**3. 可以返回返回类型的任何子类的对象**

这个就不用多说了，前面的提到的Executors类中还有两个java8新增的几个静态工厂方法：

```java
public static ExecutorService newWorkStealingPool();
public static ExecutorService newWorkStealingPool(int parallelism);
```

这两个方法返回的是`ForkJoinPool`类型的线程池。

书中举的例子是`java.util.EnumSet`，EnumSet顾名思义：集合内的元素是枚举类型。

根据具体的枚举类型，可以得到枚举类中的所有枚举值，进一步就确定这个集合最大的容量了。EnumSet就直接把所有枚举值放到一个数组，然后通过类似于BitSet的位图算法并借助枚举类值的ordinal作为索引来标记集合中是否有对应的枚举值。按照枚举类的大小它分成了两种实现，枚举值个数小于64的直接用一个`long`进行标记，这就是`RegularEnumSet`的实现；枚举值个数大于64的，则用`long[]`进行标记，这就是`JumboEnumSet`的实现。

书中还提到了Java中经常见到的SPI机制，为此我还专门写了[一个示例来使用`java.util.ServiceLoader`类来加载SPI服务实现](http://blog.csdn.net/holmofy/article/details/79318219)，书中说的SPI就是基于工厂方法实现的，而ServiceLoader则是使用反射以及`META-INF/services/`目录下的文件约定实现的。

**4. 在创建参数化实例时，使代码变得更加简洁**

其实这个优点算不上是优点，所以书上说的为HashMap提供静态工厂也迟迟在JDK中没有落实。

不过在Java9中已经有了这些静态工厂方法。

关于这些方法的具体介绍可以参考这篇文章：http://blog.csdn.net/rickiyeat/article/details/78169656



静态工厂方法的缺点：

**1. 如果将类的构造器私有化，那么这个类就不能子类化**

正如书中所说，不能子类化本质上是好处：**鼓励使用复合，而不是继承**。

**2. 与其他静态方法没有实质上的区别**

静态工厂方法本质上就是静态方法，如果不在文档中声明你可能并不知道这是一个静态工厂方法。所以在命名静态工厂方法的时候需要遵守一定的命名习惯，这些“习惯”其实就是JDK中大多数静态工厂方法命名的方式：

* valueOf：严格的说，这个方法应该是类型转换；可以参考JDK中基本类型的包装类型。
* of：valueOf的简洁版，可以参考EnumSet的实现。
* getInstance：返回的示例对象是根据参数创建的，这个静态方法也常被被用于单例模式。
* newInstance：和getInstance一样，但是newInstance返回的每个实例对象都是新创建。
* getXxx：Xxx是类名。和getInstance一样。
* newXxx：Xxx是类名。和newInstance一样。

