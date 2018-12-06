因为之前公司有人分享过G1回收器的内容，很多人听的云里雾里（包括我）。甚至有人问学GC有什么用，对写代码有帮助吗。我想这个问题不可置否。

《深入理解Java虚拟机》一书中有这么一句话：Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的“高墙”，墙外面的人想进去，墙里面的人却想出来。

这篇文章的目的就是为了突破这座藩篱，尽量让更多的人理解JVM的垃圾回收机制。

# 1、GC之前

早在1960年，[Lisp语言](https://en.wikipedia.org/wiki/Lisp_%28programming_language%29)中就有自动垃圾收集的算法，只是那时候还没有形成[*Garbage Collection*](https://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29)这个概念。直到1996年，GC因为[Java](https://en.wikipedia.org/wiki/Java_%28programming_language%29)的出现而一举成名。

在GC被广泛运用之前程序员们手动管理内存都会出现什么问题呢？

## 1.1、悬挂指针

[悬挂指针](https://en.wikipedia.org/wiki/Dangling_pointer)就是指针指向了一段已经被回收的内存。被回收的内存随时可能被再次分配出去，使用悬挂指针会出现一些不可预测的结果。

```c
void func(void)
{
   char *dp = NULL;
   ...
   {
       char c;
       dp = &c;
   }
    /* 变量c超出作用范围，栈内存已经被回收 */
    /* dp现在指向一段被回收的内存，是一个悬挂指针 */
   ...
}

int *func(void)
{
    int num = 1234;
    ...
    /* 返回函数执行栈上的一段内存 */
    /* 函数执行完后这段内存就被回收了 */
    return &num;
}

int *func(void)
{
    /* static变量不是分配在函数执行栈上的 */
    /* 函数返回不会回收这段内存 */
    /* 但多次调用这个函数返回的都是同一个地址 */
    /* 操作的内存是同一段内存，也可能会影响程序的执行 */
    static int num = 1234;
    ...
    return &num;
}

#include <stdlib.h>
void func()
{
    ...
    char *dp = malloc(size);
    ...
    free(dp);
    /* dp变成了一个悬挂指针 */
    /* 后续代码继续使用这个指针，将会出现不可预测的结果 */
    ...
    /* dp指向的内存可能已经分配给其他地方使用 */
    /* 二次释放，会导致正在使用的内存被释放(回收了不该回收的内存) */
    free(dp);
}

#include <stdlib.h>
void func()
{
    char *dp = malloc(size);
    ...
    free(dp);
    dp = NULL;
    /* 内存释放后立即把指针置为空,避免出现指针悬挂 */
    ...
}

/* 替换free函数，避免悬挂指针 */
void safefree(void **pp)
{
    assert(pp);                     /* debug模式下，pp为NULL直接abort */
    if (pp != NULL) {               /* 安全检查 */
        free(*pp);                  /* 释放内存块 */
        *pp = NULL;                 /* 将原指针置为NULL */
    }
}
```

> 悬挂指针(Dangling pointer)与野指针(Wild pointer)是两种不同的概念。野指针指的是：指针值没有初始化。没有初始化的指针指向的值是不确定的，直接使用野指针也会导致不可预测的结果，所以在声明一个值(不管是指针值还是普通值类型)的时候都尽量给一个初始值。

## 1.2、内存泄漏

[内存泄漏](https://en.wikipedia.org/wiki/Memory_leak)指的是不再需要的内存没有得到释放。

```c
void func(int n)
{
    int *p = malloc(sizeof(int) * n);
    ... // use pointer
    // 没有释放内存就返回了
}
int main(void)
{
    func(10);
    /* 函数中的指针p已经释放了，但p指向的内存没有释放 */
}
```

在C++中因为有析构函数，让我们可以在析构函数中释放内存

```cpp
#include<vector>
void func(int n)
{
    // std::vector和Java的ArrayList类似
    std::vector<int> arr(n);
    // use array
    // 函数返回时会自动调用栈对象的析构函数
    // 对象析构的时候可以释放内存。
}
```

但是一方面我们不能返回分配在栈上的对象(会导致悬挂指针)，另一方面栈的大小是有限的，不能在栈上分配过多的内存，否则极易StackOverflow。

```cpp
void func() {
    // 在栈上分配大数组肯定会栈溢出的
	int a[1024 * 1024]; //4M栈内存
	printf("%p", a);
}

int main() {
	func();
}
```

> `ulimit -s`可以查看类Unix系统下栈容量的最大值

所以一般在栈上分配基本类型、指针类型和一些函数作用域内的小结构体，需要作为返回值的对象以及大结构体对象都在堆上分配。

在堆上分配对象那就避免不了使用C++的`new`操作符。与`malloc`和`free`一样`new`也要和`delete`成对出现。但是在一个逻辑复杂的函数中难免会出现忘记释放内存就返回了：

```cpp
void func()
{
    big_object* p = new big_object();
    ...
    // 极有可能因为某些原因没有释放堆中的对象
    if(condition1) {
        throw std::runtime_error("error");
    }
    if(condition2){
        return;
    }
    ...
    // 最后的内存释放
    delete p;
}
```

C++有[智能指针](https://en.wikipedia.org/wiki/Smart_pointer)解决这些问题。

```cpp
void func() {
	big_object* p = new big_object(); // big_object在堆上分配内存
    // 智能指针在栈上,栈上的智能指针指向堆中的对象
	std::unique_ptr<big_object> smart_pointer = std::unique_ptr<big_object>(p);
	std::cout << smart_pointer->field << std::endl;
    // 函数执行完，会调用智能指针的析构函数
    // 析构函数会自动delete内部持有的指针对象
}

// 更方便的使用智能指针
void func() {
    // std::make_unique会自动构造big_object对象并以智能指针的形式返回
	auto smart_pointer = std::make_unique<big_object>();
	std::cout << smart_pointer->field << std::endl;
}
```

> C++11标准中除了引入了不共享内部持有对象的[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)，还提供了基于引用计数算法的[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)

看到这，你应该也有这样的体会：伟大的先辈们为内存管理做出了巨大的付出。

但是应用的开发速度随着互联网的迅猛发展逐步提高要求。不能在指针引用上花费太多精力，也不能因为内存泄漏导致应用跑了个把礼拜就宕机了。

> GC拯救了程序员，拯救了程序员的头发:smile:

# 2、GC算法

垃圾回收算法的设计主要考虑以下几个问题：

1、如何判断对象是垃圾对象

2、如何回收垃圾对象

3、何时回收垃圾对象

首先判断[垃圾对象](https://en.wikipedia.org/wiki/Garbage_%28computer_science%29)的方法有两种：[引用计数](https://en.wikipedia.org/wiki/Reference_counting)、[可达性分析](https://en.wikipedia.org/wiki/Unreachable_memory)

# 3、引用计数

引用计数就是为每个对象添加一个计数器，每当有一个地方引用到它，计数器值加1，当引用释放或失效时，计数器值减1，计数器值降为0对象就会被回收。

因为引用计数器降为0时就会被立即回收，不需要额外的操作，所以引用计数时间效率高。

微软COM技术中就定义了[IUnknown接口](https://docs.microsoft.com/en-us/windows/desktop/api/unknwn/nn-unknwn-iunknown)作为所有COM组件的基础接口，该接口就定义了引用的增加与释放的行为。前面提到的标准C++11中也提供了一个基于引用计数实现的[`std::shared_ptr`](https://en.cppreference.com/w/cpp/memory/shared_ptr)智能指针。[Python也使用引用计数](https://docs.python.org/3/c-api/refcounting.html)进行内存管理。

引用计数也有很多缺点，比如需要一个计数器字段，如果对象都很小只有一两个字段，引用计数方式的空间效率就不是很好了。另外有个致命的缺点就是引用计数无法检测到循环引用的问题，正因此主流的JVM都没有使用引用计数算法。

> C++11中提供了[`std::weak_ptr`](https://en.cppreference.com/w/cpp/memory/weak_ptr)可以用来解决循环引用的问题，[python有个分代垃圾收集器辅助引用计数回收垃圾](https://rushter.com/blog/python-garbage-collector/)。

![循环引用问题](http://ww1.sinaimg.cn/large/bda5cd74ly1fww39kcsqxj20p00av3yi.jpg)

[饱受诟病的IE6,IE7](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)中也使用引用计数进行垃圾回收，所以下面这样的代码就会导致内存泄漏，浏览器直接崩溃。

```javascript
var div;
window.onload = function() {
  div = document.getElementById('myDivElement');
  div.circularReference = div;
  div.lotsOfData = new Array(10000).join('*');
};
```

> 现在主流的JS引擎早已不再用引用计数算法了，[V8和Hotspot一样用分代收集算法](https://github.com/thlorenz/v8-perf/blob/master/gc.md)，值得一提的是V8的核心开发成员[Lars_Bak](https://en.wikipedia.org/wiki/Lars_Bak_%28computer_programmer%29)也是Hotspot团队的技术负责人。

# 4、可达性分析——引用树遍历

引用树遍历是可达性分析最主要的手段。

严格地说，引用树本质上是一个有根的Graph结构，而不是简单的树结构，因为对象引用极可能形成环路。

GC从根引用开始，顺着引用链遍历，找到所有的存活对象，同时把它们**标记(Mark)**下来，未标记的不可达对象就是垃圾对象。

![GC root](http://ww1.sinaimg.cn/large/bda5cd74ly1fww2bbjky3j20fc0bxgn1.jpg)

这些根引用被称为**GC Root**，在不同的编程语言和不同的场景下GC Root的定义也是不一样的。

> 1、引用树的遍历是BFS还是DFS？从内存的消耗角度考虑，应该选用DFS？
>
> 2、标记策略，每个对象头部加个标记位？Cache命中率低，Bitmap标记？

# 5、Mark-Sweep

标记清除算法是基于可达性分析最简单的一种回收策略。

![引用树遍历](http://ww1.sinaimg.cn/large/bda5cd74ly1fx8iwx1571g20bo08xtcf.gif)

Mark-Sweep算法的清除阶段很简单：遍历堆中的对象，把没标记的垃圾对象内存回收利用。这些回收的内存会被记录在一个[空闲列表(free list)](https://en.wikipedia.org/wiki/Free_list)中，下次创建对象申请内存的时候再从空闲列表中找到合适大小的内存块进行分配。

![标记清除](http://ww1.sinaimg.cn/large/bda5cd74ly1fww3xuvtzhj20k00ayjrm.jpg)

这种方式的缺点很明显：

1、 会造成大量的内存碎片。这些小块的内存会导致创建大对象时找不到连续的内存空间。

2、 创建对象时要从空闲列表中找到匹配的内存空间(First-Fit, Best-fit, Worst-fit)，影响对象的内存分配时间。

# 7、Copying

GC拷贝算法解决了Mark-Sweep算法的内存碎片的问题。

它将堆内存划分为From、To两个大小相等的区域，创建对象时只在其中一个区域内分配内存，等From区内存用完了，把标记存活的对象拷贝到To区，后续的对象内存就在To区分配(From，To转变身份)，下次内存用完再进行一次这样的过程。

![拷贝算法](http://ww1.sinaimg.cn/large/bda5cd74ly1fxewv4et6hj20ks0as0tt.jpg)

拷贝算法中内存块的状态是这样变化的：

![Copying](http://ww1.sinaimg.cn/large/bda5cd74ly1fww4eyhhd5j20k00b0wf1.jpg)

这样做的优点是不再有内存碎片，缺点也显而易见：对象移动后，需要更新对象的引用；内存使用率不高，一半的内存会空闲；对象存活率较高时，就会有大量的拷贝操作。

# 6、Mark-Compact

[标记整理算法](https://en.wikipedia.org/wiki/Mark-compact_algorithm)可以看作是标记清除和复制算法算法的组合。

> 也有人把compact翻译成压缩

标记整理与拷贝算法都解决了内存碎片的问题，区别在于Copying属于异地整理，Mark-Compact属于原地整理。

Mark-Compact把标记存活的对象往内存的一个方向靠拢，边界端后续的内存就全部记作空闲内存。

![Mark-Compact](http://ww1.sinaimg.cn/large/bda5cd74ly1fww488wqoyj20w00h8aaw.jpg)

这个算法缺点也很明显：前面有一块内存是垃圾对象，后续的对象都需要移动，存活对象较多时，移动耗时基本与内存大小成正比。

# 8、分代GC

> 事实上，目前为止都没有一个能“一统天下”的GC回收策略，每种回收策略都有各自的优缺点。

很多程序猿研究发现对象的生命周期和大多数生物有点类似：新生的对象越容易“夭折”，存活越久的对象越“顽强”。

基于这个认识，[分代GC(Generational GC)](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Generational_GC_%28ephemeral_GC%29)为对象引入“年龄”的概念，按照年龄段进行分代，不同分代的对象使用不同的GC回收策略：

* 对新生代(Young Generation)执行的GC称为新生代GC(Minor GC)；

* 对老年代(Old Generation)执行的GC称为老年代GC(Major GC)；

* 当新生代对象达到一定年龄要求后会迁移到老年代，这个“成年礼”称为晋升(Promotion)或老化(Tenuring)；

* 对整个堆内存(新生代和老年代)执行的GC称为Full GC。

# 9、Ungar分代

分代GC这个概念最早由[David Ungar](https://en.wikipedia.org/wiki/David_Ungar)于1984年在[论文](https://people.cs.umass.edu/~emery/classes/cmpsci691s-fall2004/papers/p157-ungar.pdf)中提出的，他将堆内存分为4个空间：Eden、两个Survivor、OldGen，其中Eden和两个Survivor合称为新生代空间。

> 注：论文中各空间名字并非如此，这样写只是为了贴合Hotspot VM

![David Ungar分代](http://ww1.sinaimg.cn/large/bda5cd74ly1fxb8p2gs6lj20ke06qaa4.jpg)

> Eden是伊甸园的意思，对象初生的地方——起这个名字的肯定是耶稣的虔诚信徒😂。

新创建的对象将在Eden上分配空间，Eden区满了，新生代GC将会被触发，存活的对象将被复制进Survivor-To区，上次GC存活对象存储在Survivor-From区，如果对象仍存活也会被复制进Survivor-To区。经历一次GC后，对象的年龄将会增大一岁。

![David Ungar](http://ww1.sinaimg.cn/large/bda5cd74ly1fxb7nmi8lwj20cl0ghwem.jpg)

当新生代对象年龄增长到一个指定的值后，对象将会晋升到老年代。当老年代满了就会触发老年代GC，Ungar在论文中使用标记清除算法回收老年代对象。在新生代晋升的对象把老年代填满之前，老年代GC都不会触发，所以老年代GC执行频率比新生代低。

**优点：**

由于新生代对象朝生夕死，存活的对象数不多，使用复制算法时复制操作不会特别损耗性能。同时Eden区空间通常大于Suvivor区，而且Minor GC后，大的Eden区仍作为对象分配的地方，所以不像传统复制算法那样浪费一半的空间。

**缺点：**

“很多对象年纪轻轻就会死”这个认知只适合于大多数情况，并不适用于所有程序。对于特殊的应用程序，会产生两个问题：新生代GC花费时间增多；老年代GC频繁运行。

> 新生代的对象被老年代引用，这个对象属于新生代还是老年代？往老年代复制，提前成年？
>
> 新生代GC Root是否需要包括老年代？
>
> 幸存者区满了怎么办？

# 10、JVM规范与内存结构

目前Oracle JDK和Open JDK使用的[JVM](https://en.wikipedia.org/wiki/Java_virtual_machine)都是Hotspot VM，市场上也有一些[其他虚拟机](https://en.wikipedia.org/wiki/Comparison_of_Java_virtual_machines)，它们大多遵循[JVM规范](https://docs.oracle.com/javase/specs/index.html)。

Java虚拟机规范中定义了程序执行期间使用的各种运行时数据区。其中一些数据区域是虚拟机启动时创建的，只在虚拟机退出时销毁。其他的数据区域归属于特定线程，线程数据区域是线程创建时创建退出时销毁。

![HotSpot JVM Architecture](http://ww1.sinaimg.cn/large/bda5cd74ly1fxpcdwwmmij20qo0k0wf3.jpg)

## 10.1、pc寄存器

JVM可以支持多线程，每个虚拟机线程都有自己的pc(program counter)寄存器。在任何时候，线程的当前方法如果不是native，则pc寄存器存储当前正在执行的JVM指令地址。

## 10.2、虚拟机堆栈

每个JVM线程创建时都会创建一个私有堆栈，JVM堆栈类似于传统C语言的堆栈：保存了局部变量和函数的入参，这些数据在函数返回时会随着栈帧弹出而回收。规范中允许JVM堆栈具有固定大小或动态伸缩。

- 如果线程运行时堆栈超出预定大小，JVM会抛出一个`StackOverflowError`。
- 如果可以动态扩展JVM堆栈，并且尝试进行扩展但内存不足，或者内存不足以为新线程创建初始JVM堆栈，JVM会抛出一个`OutOfMemoryError`。

> 在Hotspot中线程堆栈大小是固定的，不同的操作系统默认值不一样，也可以通过-Xss(-XX:ThreadStackSize)参数进行调节。

## 10.3、Java堆

Java堆是所有线程共享的内存，所有的类实例和数组都在此分配。Java堆在JVM启动时创建，并由垃圾回收器管理。

JVM规范里并没有规定Java堆的管理方式，所以不同的虚拟机实现对于Java堆的管理方式是不一样的。甚至于Hotspot中的不同垃圾回收器对Java堆的管理方式也是不一样的。

比如说在Oracle的JRockit虚拟机中，Java堆是这样划分的：

![JRockit](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo4pda6wqj20kt05c3yh.jpg)

IBM的J9虚拟机堆结构如下：

![IBM J9](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo4z0khvdj20jm056q2v.jpg)

Hotspot堆结构：

![HotSpot](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo51thjgxj20k80500sq.jpg)

因为CMS等垃圾回收器的GC时间与Java堆大小成正比，为了解决大内存的GC耗时问题，JDK7开始Hotspot引入了新垃圾回收器——Garbage First（G1）。G1的堆结构如下：

![Hotspot G1](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo5auekqmj20mj05dt8q.jpg)

> 收购Sun公司后，[Oracle致力于将JRockit的功能引入Hotspot](https://stackoverflow.com/questions/8068717/jrockit-jvm-versus-hotspot-jvm)。
>
> 下面会重点介绍Hotspot堆的细节

## 10.4、方法区

[方法区](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4)也是所有线程共享的内存。方法区中包括每个类的结构：运行时常量池，类的字段和方法等信息，方法、构造器和代码块的JVM指令。方法区在逻辑上是堆的一部分，但规范不要求对方法区进行垃圾回收（只是Hotspot等商用虚拟机都实现了该区域的自动内存管理），方法区的内存不要求连续，可以固定大小也可以动态伸缩。

## 10.5、运行时常量池

`.class`文件中有一个[`constant_pool`表](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4)，它包含了几种常量：编译时已知的字面量，运行时解析的方法引用和字段引用。

比如下面的这段Java代码：

```java
public class ConstantPoolTest {
    public void print() {
        System.out.println("Hello Java");
    }
}
```

`.class`文件的结构：

```java
public class ConstantPoolTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17   // java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19  // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20      // Hello Java
   #4 = Methodref          #21.#22  // java/io/PrintStream.println:(Ljava/lang/String;)V
   ...
   #18 = Class              #25     // java/lang/System
   #19 = NameAndType        #26:#27 // out:Ljava/io/PrintStream;
   #20 = Utf8               Hello Java
   #21 = Class              #28     // java/io/PrintStream
   #22 = NameAndType        #29:#30 // println:(Ljava/lang/String;)V
   #23 = Utf8               ConstantPoolTest
   #24 = Utf8               java/lang/Object
   #25 = Utf8               java/lang/System
   #26 = Utf8               out
   #27 = Utf8               Ljava/io/PrintStream;
   #28 = Utf8               java/io/PrintStream
   #29 = Utf8               println
   #30 = Utf8               (Ljava/lang/String;)V
{
   ...
  public void print();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2    // System类的静态字段引用
         3: ldc           #3    // 字符串常量
         5: invokevirtual #4    // 调用#4方法
         8: return
      LineNumberTable:
        line 7: 0
        line 8: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   LConstantPoolTest;
}
```

运行时常量池是方法区的一部分。[当类或接口被创建](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.3)时常量池会被一起创建。

## 10.6、本地方法栈

JVM可以使用传统的C堆栈以支持native方法的执行，另外本地方法栈也会被C语言实现的Java指令解释器(Hotspot中的JIT)使用。

# 11、Hotspot

> 声明：后续的Hotspot参数在不同的JDK版本中会有些许差异。

[Hotspot](https://en.wikipedia.org/wiki/HotSpot)虚拟机中最关键的三个组件是：Java堆、JIT即时编译器、垃圾回收器。

![Heapspot key Component](http://ww1.sinaimg.cn/large/bda5cd74ly1fxpcf1jowfj20qo0k074w.jpg)

## 11.1、JIT编译器

JIT及时编译器支持三种模式：`interpreted-only`、`compilation `、`mixed`，分别可以用`-Xint`、`-Xcomp`、`-Xmixed`选项开启。

在`interpreted-only`模式下，JVM不会编译字节码，所有的字节码由解释器临时解释执行。JIT的性能优势在这种模式下就无法得到体现。

在`compilation `模式下，在第一次调用方法时JVM会强制将方法编译成机器码，这些编译过的代码将会被缓存起来，下次调用将直接执行机器码。Hotspot提供了`-Xmaxjitcodesize`(`-XX:ReservedCodeCacheSize`)选项设置JIT编译代码的缓存大小（默认240M）。

在`mixed`模式下，只会把热点方法编译成机器码，除此之外所有字节码由解释器临时解释执行。

Hotspot默认使用混合模式：

![JVM](http://ww1.sinaimg.cn/large/bda5cd74ly1fxv3yl9hx3j20fi0290sm.jpg)

对比解释执行，编译的好处是会对方法中的代码进行优化：消除不必要的变量、循环外提、删除无用赋值等。在这个过程中会进行指令重排，在单线程环境下能保证原有语义，但是多线程环境下会影响程序的正常逻辑。

除了编译上的优化，JIT还有很多其他的优化策略。比如：

* 通过[逃逸分析](https://en.wikipedia.org/wiki/Escape_analysis)将原本需要在堆上分配的对象转换为栈分配，栈上分配对象的好处是随着函数返回，对象内存会自动回收，而不需要通过GC回收，间接地减少了GC的运行频率。

* 将调用频率高且体量小的方法进行内联，这和C++的内联函数作用一样。减少函数调用，可以减少因入栈消耗的时间。



JIT的内容不是本文的核心，有兴趣的可以自行谷歌或[参考官方文档](https://docs.oracle.com/javase/8/embedded/develop-apps-platforms/codecache.htm)。

下面主要讲得是Hotspot中的Java堆以及管理Java堆的垃圾回收器。

## 11.2、Hotspot分代

Hotspot使用Ungar分代策略管理Java堆，并提供了相应的参数设置各个分代的大小：

![JVM分代](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo5teibzaj20nb0c7wfs.jpg)

上图中`reserved`是操作系统保留的虚拟地址空间，在虚拟机刚运行时只会分配`-Xms`大小的物理内存，而且Java堆会通过以下策略尽可能的减少物理内存的消耗。

| 参数                         | 说明                                     | 默认值               |
| ---------------------------- | ---------------------------------------- | -------------------- |
| `-XX:MinHeapFreeRatio`       | 经过一次GC事件后允许的最小堆内存空闲比例 | 70%                  |
| `-XX:MaxHeapFreeRatio`       | 经过一次GC事件后允许的最大堆内存空闲比例 | 40%                  |
| `-Xms`/`-XX:InitialHeapSize` | Java堆的初始容量                         | 根据系统配置动态选择 |
| `-Xmx`/`-XX:MaxHeapSize`     | Java堆的最大容量                         | Linux 2000M          |

如果经过GC后空闲内存比例小于40%，分代将会扩容至40%的空闲内存占比，直到分代允许的最大内存容量。

如果经过GC后空闲内存比例大于70%，分代将会收缩至70%的空闲内存占比，直到分代允许的最小内存容量。

但是任何一个分代的Resize都会导致Full GC，设置`-Xms=-Xmx`可以防止由`-Xms`到`-Xmx`增长过程中Resize操作导致的Full GC。

除了以上几个重要的参数以外，Hotspot还提供了很多其他参数配置。

比如说：

`-XX:SurvivorRatio`可以设置新生代中Eden区域Survivor区的比例（默认8）。

![survivorRatio](http://ww1.sinaimg.cn/large/bda5cd74ly1fxpdrde1vvj20vz06v75f.jpg)

`-XX:MaxTenuringThreshold`可以设置新生代到老年代的老化年龄（最大值是15，并行收集器默认15，CMS默认6）。

![对象生命周期](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo88x0s41g20qa0d5wg3.gif)

> 更多的JVM参数以及参数的默认值可以参考[官方文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)。

## 11.3、PermGen与Metaspace

永久代（PermGen）中主要存储：类元数据等静态数据。也就是JVM规范中的方法区所存储的区域。

永久代在[32位JVM上默认最大内存为64M，在64位JVM上默认最大内存为82M](https://www.baeldung.com/java-permgen-metaspace)。

虽然永久代可以通过`-XX:PermSize`和`-XX:MaxPermSize`进行调节，但由于永久代没有很好的垃圾回收机制，对于大量使用反射、动态代理、字节码框架的应用经常会由于**PermGen**内存不足导致OutOfMemoryError。

所以Java8中用Metaspace取代了PermGen。

![Metaspace](http://ww1.sinaimg.cn/large/bda5cd74ly1fxo9oiczhbj20e908cwep.jpg)

## 11.4、可选的垃圾回收器

Hotspot VM包括三种不同类型的垃圾收集器，每种收集器具有不同的性能特征与不同的适用场景。

### 11.4.1、串行收集器

串行收集器使用单个线程来执行所有垃圾收集工作。因为线程之间没有通信开销，所以回收效率较高。它最适合单处理器机器，因为它无法利用多处理器硬件。它对于具有小数据集（最大约100 MB）的多处理器应用程序也非常有效。默认情况下，JVM会根据硬件、操作系统以及JVM配置(-client)选用串行收集器，或者可以使用`-XX:+UseSerialGC`选项显式启用串行收集器。

![串行GC](http://ww1.sinaimg.cn/large/bda5cd74ly1fxrar3t3u9j20ev04hglm.jpg)

> 串行收集器在新生代使用复制算法，老年代使用压缩整理算法。

### 11.4.2、并行收集器

并行收集器（也称为[吞吐量](https://translate.google.cn/#view=home&op=translate&sl=en&tl=zh-CN&text=throughput)收集器）并行执行垃圾回收，这可以显着减少垃圾收集开销。它适用于在多处理器硬件上运行的具有中型到大型数据集的应用程序。默认情况下，JVM会根据硬件、操作系统以及JVM配置(-server)选用串行收集器，或者可以使用`-XX:+UseParallelGC`选项显式启用并行收集器。

![ParallelGC](http://ww1.sinaimg.cn/large/bda5cd74ly1fxrbbp2cdxj20hj04kaa4.jpg)



[在Java8之前`-XX:+UseParallelGC`默认不会启用老年代的并行GC](https://blogs.oracle.com/jonthecollector/our-collectors)，老年代仍然使用串行GC，需要使用`-XX:+UseParallelOldGC`开启老年代并行GC。

在Java8中`-XX:+UseParallelGC`默认会启用老年代并行GC，可以使用`-XX:-UseParallelOldGC`选项关闭老年代并行GC。

新生代并行GC默认使用**Parallel Scavenge**收集器，另外还有一个**ParNewGC**回收器。

ParNewGC主要配合CMS收集器使用，因为ParNewGC有CMS并发阶段所需要的一些同步操作。`-XX:+UseParNewGC`选项可以开启新生代的ParNewGC，此时老年代使用Serial Old。ParNewGC不能和ParallelOldGC一起使用（原因[在这](https://blogs.oracle.com/jonthecollector/our-collectors)，我也没怎么看明白)。在Java8中UseParNewGC只能和CMS配合使用。

### 11.4.3、并发收集器

前面的收集器在收集过程中用户线程会完全暂停(也叫Stop The World)，收集完成后用户线程才会继续运行。这个暂停时间可能会持续一秒以上，对响应速度有要求的应用可能会有不好的体验，所以Hotspot还提供了并发收集器。

**并发收集器允许应用线程与GC线程并发执行**。这也意味着并发标记过程会存在GC线程和应用线程切换CPU的损耗。它适用于具有中型到大型数据集，并且响应时间比吞吐量更重要的应用程序。

![Concurrent GC](http://ww1.sinaimg.cn/large/bda5cd74ly1fxtz1cj37vj20m105cab2.jpg)

Java HotSpot VM提供两个并发垃圾回收器：CMS和G1。

使用`-XX:+UseConcMarkSweepGC`选项可以启用CMS收集器。

使用`-XX:+UseG1GC`选项可以启用G1收集器。

**CMS优缺点：**

CMS整个过程中只有初始标记和重新标记阶段需要StopTheWorld，相对ParallelOldGC停顿时间较短。

但由于CMS使用标记清除算法，所以会产生大量内存碎片，当无法找到连续的内存空间分配时，不得不提前触发一次FullGC。针对这点CMS提供了`-XX:+UseCMSCompactAtFullCollection`选项(默认开启，Java8中已弃用)，当内存分配失败时使用Serial Old对内存进行整理。这样解决了内存碎片的问题，但相应地STW时间变得更长。

另外CMS无法处理浮动垃圾(Floating Garbage，清除阶段新产生的垃圾)，可能出现浮动垃圾在完成清除之前又把老年代塞满了，导致“Concurrent Mode Failure”从而触发另一次Full GC。

很明显CMS需要在对象填满老年代之前就开始初始标记，CMS提供了`-XX:CMSInitiatingOccupancyFraction`和`-XX:CMSTriggerRatio`选项来指定这个阈值。

> 所以Oracle在文档上也明确指出：在并行GC无法满足应用延时要求时才使用CMS收集器。
>
> G1收集器最初的开发目的就是为了替代CMS。G1使用分区方式管理内存，所以G1会同时管理年轻代和老年代。
>
> Hotspot团队对[G1](https://www.youtube.com/watch?v=6JcV7T9Z8SY)进行了许多性能上的优化，[G1已经成为Java9默认的垃圾回收器](http://blog.mgm-tp.com/2018/01/g1-mature-in-java9/)。

下图是Hotspot可用收集器的组合，其中连线上的选项参数是针对Java7，Java8参数有部分改动，详请参考[官方文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)。

![Valid GC combinations](http://ww1.sinaimg.cn/large/bda5cd74ly1fxrakmsv9vj20ko0fnjtq.jpg)

## 11.5、选择垃圾回收器

除非应用程序具有相当严格的暂停时间要求，否则应该让JVM自行选择垃圾回收器。如有必要，可以通过调整堆大小以提高性能。如果性能仍不符合目标，请使用以下指南作为选择收集器的起点。

- 如果应用程序具有较小的数据集（最大约100 MB），那么选项选择串行收集器`-XX:+UseSerialGC`。这种应用程序一般是客户端程序，服务器应用肯定不适用。
- 如果应用程序性能是第一优先级并且没有暂停时间要求或一秒以上的暂停是可接受的，那应该选择并行收集器`-XX:+UseParallelGC`。
- 如果响应时间比总吞吐量更重要，并且垃圾收集暂停必须保持短于大约1秒，则使用`-XX:+UseConcMarkSweepGC`或`-XX:+UseG1GC`并发收集器。



参考：

[《垃圾回收算法手册：自动内存管理的艺术》Richard Jones / Eliot Moss / Antony Hosking著](https://book.douban.com/subject/26740958/)

[《深入理解Java虚拟机》周志明 著](https://book.douban.com/subject/24722612/)

[《垃圾回收的算法与实现》中村成洋 / 相川光 著](https://book.douban.com/subject/26821357/)

Java虚拟机规范：https://docs.oracle.com/javase/specs/jvms/se8/html/index.html

GC调优指南：https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html

Java8虚拟机参数：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html

Java7虚拟机参数：https://docs.oracle.com/javase/7/docs/technotes/tools/solaris/java.html

GC基础教程：https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html

G1收集器入门：https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

G1收集器调优：https://www.oracle.com/technetwork/articles/java/g1gc-1984535.html

Java6虚拟机GC调优：https://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html

https://www.oracle.com/technetwork/java/javase/tech/index-jsp-136373.html

https://docs.oracle.com/javase/8/embedded/develop-apps-platforms/codecache.htm

https://blogs.oracle.com/jonthecollector/our-collectors

https://www.oracle.com/technetwork/cn/community/developer-day/2-jvm-tuning-1866448-zhs.pdf

http://www.micheltriana.com/blog/2010/12/29/garbage-collection-pt-3-generations

https://www.stechies.com/difference-between-permgen-metaspace/

https://www.sczyh30.com/posts/Java/jvm-gc-hotspot-implements/

https://en.wikipedia.org/wiki/Garbage-first_collector

https://en.wikipedia.org/wiki/Mark-compact_algorithm

https://www.baeldung.com/java-permgen-metaspace

https://en.wikipedia.org/wiki/Concurrent_mark_sweep_collector