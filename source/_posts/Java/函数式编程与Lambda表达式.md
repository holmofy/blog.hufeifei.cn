---
title: Java函数式编程与Lambda表达式
date: 2017-08-21 16:50
categories: JAVA
---

[C++](http://en.cppreference.com/w/cpp/language/lambda)，[Java](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)，[C#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions)，[Python](https://docs.python.org/3/howto/functional.html)等各个编程语言早已经支持lambda表达式了，作为即将从业的大学生，现在学习Java的函数式编程应该为时不晚。

<!-- toc -->

# FunctionalInterface—函数式接口

在`java.util.function` 包中提供给我们一些最常用的函数式接口：

## 四个最基本的函数式接口

![function](函数式编程与Lambda表达式\function.svg)

* `Function<T, R>`：`R apply(T t);`；输入类型`T`返回类型`R`。
* `Consumer<T>`：`void accept(T t);`；输入类型`T`，“消费”掉，无返回。
* `Predicate<T>`：`boolean test(T t);`；输入类型`T`，并进行条件“判断”，返回`true|false`。
* `Supplier<T>`：`T get();`；无输入，“生产”一个`T`类型的返回值。

## 基本数据类型的函数式接口

上面的四个接口因为使用泛型，Java泛型不支持基本数据类型，又因为基本数据类型与引用类型频繁的拆装箱将会严重影响效率，所以有Java还提供了几个基本数据类型的函数式接口：

### 1、`double`类型的函数式接口

- `DoubleFunction<R>`：`R apply(double value);`
- `DoubleConsumer`：`void accept(double value);`
- `DoublePredicate`：`boolean test(double value);`
- `DoubleSupplier`：`double getAsDouble();`

### 2、`int`类型的函数式接口

* `IntFunction<R>`：`R apply(int value);`
* `IntConsumer`：`void accept(int value);`
* `IntPredicate`：`boolean test(int value);`
* `IntSupplier`：`int getAsInt();`

### 3、`long`类型的函数式接口

* `LongFunction<R>`：`R apply(long value);`
* `LongConsumer`：`void accept(long value);`
* `LongPredicate`：`boolean test(long value);`
* `LongSupplier`：`long getAsLong();`

### 4、`boolean`类型的函数式接口

* `BooleanSupplier`：`boolean getAsBoolean();`

>为了防止API的爆炸式增长，JDK中只提供了一些必要的基本数据类型的函数式接口，如long，double。而int作为最突出的基本类型（如数组索引，整形字符等）。所有其他的类型都可以转成这三个类型。而这里的BooleanSupplier主要是为了能更方便地做布尔运算而扩展。

## 一元函数式接口

引用类型：

* `UnaryOperator<T> extends Function<T, T>`：`T apply(T t);`

基本数据类型：

* `IntUnaryOperator`：`int applyAsInt(int operand);`
* `LongUnaryOperator`：`long applyAsLong(long operand);`
* `DoubleUnaryOperator`：`double applyAsDouble(double operand);`

## 用于类型转换的一元函数式接口

#### ①、引用类型转基本数据类型

* `ToDoubleFunction<T>`：`double applyAsDouble(T value);`
* `ToIntFunction<T>`：`int applyAsInt(T value);`
* `ToLongFunction<T>`：`long applyAsLong(T value);`

#### ②、`double`类型转其他类型

* `DoubleToIntFunction`：`int applyAsInt(double value);`
* `DoubleToLongFunction`：`long applyAsLong(double value);`

> 由`double`得到引用类型就是前面的`DoubleFunction<R>`：`R apply(double value);`

#### ③、`int`类型转其他类型

* `IntToDoubleFunction`：`double applyAsDouble(int value);`
* `IntToLongFunction`：`long applyAsLong(int value);`

> 由`int`得到引用类型就是前面的`IntFunction<R>`：`R apply(int value);`

#### ④、`long`类型转其他类型

* `LongToDoubleFunction`：`double applyAsDouble(long value);`
* `LongToIntFunction`：`int applyAsInt(long value);`

> 由`long`得到引用类型就是前面的`LongFunction<R>`：`R apply(long value);`

## 二元函数式接口

* `BiFunction<T, U, R>`：`R apply(T t, U u);`
* `BiConsumer<T, U>`：`void accept(T t, U u);`
* `BiPredicate<T, U>`：`boolean test(T t, U u);`

> 因为Supplier是没有输入，只有返回值，返回值只有一个，所以没有该类型的二元函数式接口

### 同类型输入的二元函数式接口

引用类型：

* `BinaryOperator<T> extends BiFunction<T,T,T>`：`T apply(T left, T right)`

基本数据类型：

* `IntBinaryOperator`：`int applyAsInt(int left, int right);`
* `LongBinaryOperator`：`long applyAsLong(long left, long right);`
* `DoubleBinaryOperator`：`double applyAsDouble(double left, double right);`

### 混合类型输入的二元函数式接口

* `ObjDoubleConsumer<T>`：`void accept(T t, double value);`
* `ObjIntConsumer<T>`：`void accept(T t, int value);`
* `ObjLongConsumer<T>`：`void accept(T t, long value);`

### 引用类型到基本数据类型的二元函数式接口

* `ToDoubleBiFunction<T, U>`：`double applyAsDouble(T t, U u);`
* `ToIntBiFunction<T, U>`：`int applyAsInt(T t, U u);`
* `ToLongBiFunction<T, U>`：`long applyAsLong(T t, U u);`

## 其他函数式接口

上面列出了JDK1.8在`java.util.function`中提供给我们的常用的函数式接口，这些接口都只有**一个抽象方法**(Java8中还提供了默认方法的特性，所以只要有一个abstract方法即可)。JDK8中还提供一个运行时注解`FunctionalInterface`，所有函数式接口定义时都会加上这个注解用来标记，有了该标记那么就可以使用Lambda表达式来表示该接口抽象方法的实现了，而这个抽象方法的实现在[lambda](https://en.wikipedia.org/wiki/Lambda_calculus)中叫做**“匿名函数”**（注意不是匿名内部类，匿名内部类允许有成员变量可以保存对象的状态，但匿名函数不保存对象状态）。

除了上面列举出来的`java.util.function`包中的43个函数式接口，我们平常用到的很多接口在JDK1.8中都被标记为函数式接口，常用的如：

```java
java.lang.Runnable
java.util.concurrent.Callable
java.io.FileFilter
java.io.FilenameFilter
java.util.Comparator
java.nio.DirectoryStream.Filter
```

# Lambda表达式语法

前面说了那么多函数式接口，下面来看看Java中Lambda表达式的语法规则：

|       参数列表       | 箭头符  |         函数体         |
| :--------------: | :--: | :-----------------: |
| `(int x, int y)` | `->` | `{ return x + y; }` |

Lambda表达式是一个匿名函数，我们可以像定义一个函数一样定义它，只不过我们不需要给出它的函数名：

```java
// 多个参数
(type1 arg1, type2 arg2...) -> { body }

(int a, int b) -> {  return a + b; }

// 可以显式声明参数的类型，也可以从上下文推断参数的类型。
(arg1, arg2...) -> { body }

(a, b) -> { return a + b; }

// 一个参数
(a) -> { return a * a; }
// 参数列表只有一个参数可以省略：参数列表的小括号
a -> { return a * a; }
// 方法体只有一条语句可以省略：方法体大括号以及return关键字
a -> a * a;

// 没有参数，需要用一对空括号来表示
() -> return 1;
```

而且我们可以用前面提到的那些函数式接口的引用还指向这些匿名函数：

```java
public class StreamTest {
    public static void main(String[] args) {
        Function<String, String> upcase = s -> s.toUpperCase();
        Consumer<String> print = s -> System.out.println(s);
        Predicate<String> isEmpty = s -> s.isEmpty();
        Supplier<String> getStr = () -> String.valueOf(Math.random());

        // 虽然可以这么用，但事实上我们很少这么用
        print.accept(upcase.apply("Hello Lambda"));
        print.accept("Hello Lambda is empty: " + isEmpty.test("Hello Lambda"));
        print.accept(getStr.get());
    }
}
/////////////////////////////////////////////////////////////////////////
// 上面的几个赋值语句还可以这么写：
Function<String, String> upcase = String::toUpperCase;// 未知对象的成员方法

Consumer<String> print = System.out::println; // 已知对象的成员方法
    // 所以也允许这种语法
	StringBuilder builder = new StringBuilder();
	Consumer<String> append = builder::append;

Predicate<String> isEmpty = String::isEmpty; // 对象成员方法
DoubleUnaryOperator sin = Math::sin; // 静态成员方法(类成员方法)

// 甚至还可以引用构造方法作为匿名函数
Function<String, String> newStr = String::new;
Supplier<String> getStr = String::new;
```

> 语法很灵(qi)活(pa)，总之只要能满足指定的参数个数以及相应的参数类型就行

前面说到`Runnable`接口也是函数式接口，所以我们也可以使用Lambda表达式来表示Runnable接口的实现：

```java
    public static void main(String[] args) {
        // 常规写法
        new Thread(new Runnable() {
            public void run() {
                printMessage("this is child thread");
            }
        }).start();
        // Lambda表达式写法
        new Thread(() -> printMessage("this is child thread")).start();
    }

    public static void printMessage(String msg) {
        System.out.println(Thread.currentThread().getName() + ":" + msg);
    }
```

运行结果：

```shell
Thread-0:this is child thread
Thread-1:this is child thread
```

# Stream API

![Stream](函数式编程与Lambda表达式\Stream.svg)

与前面介绍的函数式接口对应的，有四类Stream接口，它们都继承自BaseStream：

* Stream：代表通用的引用类型的流式操作接口
* IntStream：代表`int`类型的流式操作接口，避免了`int`和`Integer`之间频繁的拆装箱操作
* LongStream：代表`long`类型的流式操作接口，避免了`long`和`Long`之间频繁的拆装箱操作
* DoubleStream：代表`double`类型的流式操作接口，避免了`double`和`Double`之间频繁的拆装箱操作

这几个接口分别有几个对应的实现类，但是这几个实现类的访问属性是包级别的，我们无法直接创建这些类的示例对象。Java提供了一个StreamSupport工厂类，让我们能够创建这四个接口的对象。

![StreamSupport的工厂方法](http://img-blog.csdn.net/20170822153745126?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

示例：

```java
package cn.hff.functor;

import java.util.Spliterator;
import java.util.Spliterator.OfInt;
import java.util.Spliterators;
import java.util.stream.IntStream;
import java.util.stream.StreamSupport;

public class StreamTest {
    public static void main(String[] args) {
        int[] arr = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
        // 首先要创建一个int类型的Spliterator
        // 可以使用Spliterators工厂类为我们创建
        OfInt spliterator = Spliterators.spliterator(arr, 0, arr.length,
                Spliterator.ORDERED | Spliterator.IMMUTABLE);
        // 使用StreamSupport工厂类创建IntStream的实例对象
        IntStream intStream = StreamSupport.intStream(spliterator, false);
        intStream.map(i -> i * i)
                .filter(i -> i % 2 == 0)
                .forEach(System.out::println);
    }
}
```

运行结果：

```shell
0
4
16
36
64
```

但是Stream主要用来操作一些常用的数据源：Collection集合，数组，IO管道。这些数据源都为我们提供了创建Stream对象的方法，所以通常我们不会直接去使用底层的StreamSupport类去创建Stream对象(不然怎么体现Stream API的简洁高效呢(～￣▽￣)～)。

## 可创建Stream的数据源

Java8中在很多类中都提供了创建Stream对象的方法：

- 集合：`Collection.stream()`或`Collection.parallelStream()`
- 数组：`Arrays.stream()`
- 对象(或基本数据)序列：`Stream.of()`,`IntStream.of()`,`LongStream.of()`,`DoubleStream.of()`
- 字符流：`BufferdReader.lines()`
- 文件路径：`Files.walk()`,`Files.lines()`,`Files.find()`
- 随机数流：`Random.ints()`
- 其他：`BitSet.stream()`,`Pattern.splitAsStream()`,`JarFile.stream()`等

所以上面的示例还可以这么写：

```java
// 数组创建Stream的方式
int[] arr = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
Arrays.stream(arr).map(i -> i * i)
        .filter(i -> i % 2 == 0)
        .forEach(System.out::println);

// 直接对int序列创建Stream的方式
// 实际上这个方法就是调用了 Arrays.stream(arr);
IntStream.of(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
        .map(i -> i * i)
        .filter(i -> i % 2 == 0)
        .forEach(System.out::println);
```

> 如果我们需要开发一个第三方库，并为调用者提供类似于上面几种创建Stream的API，这时我们可能就需要用到StreamSupport类，可以参考Java中提供的实现。

## Stream的特点

* 无存储性：

  流不是存储元素的数据结构；相反，它需要从数据结构，数组，生成器函数或IO管道中获取数据并通过流水线地(计算)操作对这些数据进行转换。

* [函数式编程](https://en.wikipedia.org/wiki/Functional_programming)：

  Stream上操作会产生一个新结果，而不会去修改原始数据。比如`filter`过滤操作它只会根据原始集合中将未被过滤掉的元素生成一个新的Stream，而不是真的去删除集合中的元素。

* [惰性求值](https://en.wikipedia.org/wiki/Lazy_evaluation)：

  很多Stream操作(如filter,map,distinct等)都是惰性实现，这样做为了优化程序的计算。比如说，要从一串数字中找到第一个能被10整除的数，程序并不需要对这一串数字中的每个数字进行测试。流操作分为两种：中间操作(返回值仍为Stream，仍可执行操作)，终断操作(结束Stream操作)。中间操作都是惰性操作。

* 无限数据处理：

  集合的大小是有限的，但是流可以对无限的数据执行操作。比如可以使用`limit`或`findFirst`这样的操作让Stream操作在有限的时间内结束。

* 一次性消费：

  流只能使用(“消费”)一次，一旦调用终断操作，流就不能再次使用，必须重新创建一个流。就像迭代器一样，遍历一遍后，想要再次遍历需要重新创建一个迭代器。

## Stream操作的分类

<table><tbody><tr><td colspan="3" align="center" border="0">Stream操作分类</td></tr><tr><td rowspan="2" border="1">中间操作(Intermediate operations)</td><td>无状态(Stateless)</td><td>StatelessOp: unordered(), filter(), map(), mapToInt(), mapToLong(), mapToDouble(), flatMap(), flatMapToInt(), flatMapToLong(), flatMapToDouble(), peek();</td></tr><tr><td>有状态(Stateful)</td><td>DistinctOps: distinct();<br/>SortedOps: sorted();<br/>SliceOps: limit(), skip() </td></tr><tr><td rowspan="2" border="1">终断操作(Terminal operations)</td><td>非短路操作</td><td>ForEachOps: forEach(), forEachOrdered();<br/>ReduceOps:reduce(), collect(), max(), min(), count();<br/>toArray()</td></tr><tr><td>短路操作(short-circuiting)</td><td>MatchOps: anyMatch(), allMatch(), noneMatch();<br/>FindOps: findFirst(), findAny()</td></tr></tbody></table>



* 中间操作：返回一个新的Stream。中间操作都是惰性的，它们不会对数据源执行任何操作，仅仅是创建一个新的Stream。在终断操作执行之前，数据源的遍历不会开始。

  * 无状态操作：该操作中的元素不会受前后元素的影响，可以独立的处理每个元素的操作。比如map，filter等操作只对一个元素进行操作，结果不会因为数据源中元素的顺序或个数等因素而受影响。

  * 有状态操作：该操作中的元素会受前后元素的影响，比如sorted，distinct等操作要求对两个或多个元素进行比较判断，元素的顺序或个数将会直接影响操作的结果。

* 终断操作：遍历流并生成结果或者[副作用](https://en.wikipedia.org/wiki/Side_effect_(computer_science))。执行完终断操作后，Stream就会被“消费”掉，如果想再次遍历数据源，则必须重新创建新的Stream。大多数情况下，终断操作的遍历都是即时的——在返回之前完成数据源的遍历和处理，只有`iterator()`和`spliterator()`不是，这两个方法用于提供额外的遍历功能——让开发者自己控制数据源的遍历以实现现有Stream操作中无法满足的操作(实际上现有的Stream操作基本能满足需求，所以这两个方法目前用的不多)。

  * 短路操作：从数据源匹配到具有所需特性的元素后就结束操作。当出现无限数据时，短路操作会在有限的时间内结束。

    事实上短路操作我们一直都在使用，比如：&&, ||两个运算符就是短路操作。

    ```c
    (condition1 && condition2) 如果condition1为false,就不会再去判断condition2了
    (condition2 || condition2) 如果condition1为true,就不会再去判断condition2了
    ```

    Stream中的短路操作和这个类似。

  * 非短路操作：对数据源中的所有数据都执行操作。

## Stream操作详解

前面对Stream操作进行了分类(其实都是JavaDoc里的内容，这里只是整理了一下(●ˇ∀ˇ●))，下面对这些操作进行详细的介绍：

### ** 中间操作 **

#### 一、无状态操作

##### 1. 一大利器——map

map、mapToInt、mapToLong、mapToDouble、flatMap、flatMapToInt、flatMapToLong、flatMapToDouble：

这些个操作可以统称为Map(Map&Reduce的核心之一)：就是把一个数据转换成另一个数据。这些方法区别在于转换前的数据类型与转换后的数据类型。

map就是转换前后数据类型相同；

mapToInt就是将原始数据类型转成`int`类型；

mapToLong就是将原始数据类型转成`long`类型；

mapToDouble就是将原始数据类型转成`double`类型；

![map](函数式编程与Lambda表达式\map.svg)

flatMap就有点特殊了，flat字面意思平铺，实际上就是让我们把原始数据转换成该原始数据类型对应地Stream对象

flatMapToInt就是将原始数据类型转成IntStream类型

flatMapToLong就是将原始数据类型转成LongStream类型

flatMapToDouble就是将原始数据类型转成DoubleSteam类型

![flatMap](函数式编程与Lambda表达式\flatMap.svg)

示例：

```java
Stream.of(Arrays.asList(1, 2, 3, 4), Arrays.asList(5, 6, 7, 8))
        .flatMap(List::stream)
        .forEach(System.out::println);
// 输出结果为：1 2 3 4 5 6 7 8

IntStream.of(1, 2, 3, 4)
        .flatMap(i -> IntStream.of(i, 2 * i, 3 * i))
        .forEach(System.out::println);
// 输出结果为：1 2 3 2 4 6 3 6 9 4 8 12
```

##### 2. filter，过滤

过滤：接受原始数据中满足测试要求的元素。

![filter](函数式编程与Lambda表达式\filter.svg)

##### 3. peek，非消费型遍历

peek，遍历Stream中的元素，和forEach类似，区别是peek不会“消费”掉Stream，而forEach会消费掉Stream；peek是中间操作所以也是惰性的，只有在Stream“消费”的时候生效。

示例：

```java
public class StreamTest {
    public static void main(String[] args) {
        IntStream.of(1, 2, 3, 4)
                .peek(i -> System.out.println("next item:" + i))
                .forEach(i -> System.out.println("get item:" + i));

        System.out.println("-----------------------");
        IntStream.of(1, 2, 3, 4)
                .peek(i -> System.out.println("next item:" + i))
                .filter(i -> i % 2 == 0)
                .forEach(i -> System.out.println("get item:" + i));

        System.out.println("-----------------------");
        IntStream.of(1, 2, 3, 4)
                .filter(i -> i % 2 == 0)
                .peek(i -> System.out.println("next item:" + i))
                .forEach(i -> System.out.println("get item:" + i));
    }
}
```

输出结果：

```shell
next item:1
get item:1
next item:2
get item:2
next item:3
get item:3
next item:4
get item:4
-----------------------
next item:1
next item:2
get item:2
next item:3
next item:4
get item:4
-----------------------
next item:2
get item:2
next item:4
get item:4
```

> 因为peek操作是惰性的，所以会和forEach一起生效

#### 二、有状态操作

##### 1. distinct，去重

去重操作和数据库中的类似：去除原始数据中重复的元素(只保留一个)。

![distinct](函数式编程与Lambda表达式\distinct.svg)

##### 2. sorted，排序

sorted：对Stream中的元素进行排序。

它有两个重载方法：

```java
// 按Comparable接口实现定义的规则排序，前提是类实现Comparable接口
Stream<T> sorted();
// 按Comparator实现指定的规则排序，可以不实现Comparable接口
Stream<T> sorted(Comparator<? super T> comparator);
```

示例：

```java
IntStream.of(4, 2, 1, 3).sorted().forEach(System.out::println);
// 输出顺序：1 2 3 4
```

##### 3. limit&skip，截取操作

这两个功能相近，区别在于limit取头部的数据(或者说截取前面的元素)，skip取尾部的数据(跳过前面的元素)：

![limit_skip](函数式编程与Lambda表达式\limit_skip.svg)

示例：

```java
public class StreamTest {
    public static void main(String[] args) {
        IntStream.of(1, 2, 3, 4, 5, 6, 7, 8)
                .limit(2)
                .forEach(i -> System.out.print(i + " "));

        System.out.println();

        IntStream.of(1, 2, 3, 4, 5, 6, 7, 8)
                .skip(6)
                .forEach(i -> System.out.print(i + " "));
        System.out.println();

        IntStream.of(1, 2, 3, 4, 5, 6, 7, 8)
                .skip(2)
                .limit(4)
                .forEach(i -> System.out.print(i + " "));

        System.out.println();

        IntStream.of(1, 2, 3, 4, 5, 6, 7, 8)
                .limit(6)
                .skip(2)
                .forEach(i -> System.out.print(i + " "));
    }
}
```

输出结果：

```shell
1 2
7 8
3 4 5 6
3 4 5 6
```
### ** 终断操作 **

#### 三、短路操作

##### 1. anyMatch&allMatch&noneMatch

前面说过短路操作其实就和我们日常编程用到的`&&`和`||`运算符处理过程类似，遇到一个满足条件的就立即停止判断。所以看下面几个例子来理解一下`anyMatch`，`allMatch`，`noneMatch`的区别。

**第一个anyMatch**：

```java
public class StreamTest {
    public static void main(String[] args) {
        boolean anyMatch = Stream.of("Alibaba", "Tecent", "Baidu", "Huawei", "Sohu", "HTC")
                .map(s -> {
                    System.out.println(s);   // 转小写前先把源字符串打印一下
                    return s.toLowerCase();
                }).anyMatch(s -> s.startsWith("h"));
        System.out.println("result: " + anyMatch);
    }
}
```

> 注意：之前说过map操作是中间操作，中间操作都是惰性的，所以这里的map操作会在anyMatch执行的时候一起执行，anyMatch终止了map也就跟着终止了。

输出结果：

```shell
Alibaba
Tecent
Baidu
Huawei        # 匹配到第一个以h开头的字符串，之后的字符串就不再进行比较了
result: true  # anyMatch就是任意一个元素满足条件就返回true，
              # 只有整个流中的元素匹配失败才返回false
```

**第二个noneMatch**：

```java
public class StreamTest {
    public static void main(String[] args) {
        boolean anyMatch = Stream.of("Alibaba", "Tecent", "Baidu", "Huawei", "Sohu", "HTC")
                .map(s -> {
                    System.out.println(s);
                    return s.toLowerCase();
                }).noneMatch(s -> s.startsWith("h")); // 只修改了这个调用的方法
        System.out.println(anyMatch);
    }
}
```

输出结果：

```shell
Alibaba
Tecent
Baidu
Huawei
false  # noneMatch和anyMatch相反：
       # 只要任意一个元素匹配成功就返回false，
       # 只有整个流中的元素都匹配失败才返回true
```

**第三个allMatch**

```java
public class StreamTest {
    public static void main(String[] args) {
        boolean anyMatch = Stream.of("Alibaba", "Tecent", "Baidu", "Huawei", "Sohu", "HTC")
                .map(s -> {
                    System.out.println(s);
                    return s.toLowerCase();
                }).allMatch(s -> s.startsWith("h"));  // 同样只修改了这一行代码
        System.out.println(anyMatch);
    }
}
```

输出结果：

```shell
Alibaba
false     # allMatch只要任意一个元字符串匹配失败就直接返回false
          # 只有所有的元素都匹配成功才返回true
```

对这三个操作概括的说就是：

* **anyMatch**: 任意一个匹配成功返回true，全部都匹配失败返回false
* **allMatch**: 全部都匹配成功返回true，任意一个匹配失败返回false
* **noneMatch**: 全部都匹配失败返回true，任意一个匹配成功返回false

> 看到这个，顿时想起高中的真命题，假命题，逆否命题的概念，(￣▽￣)"

#### 四、非短路操作

##### 1. 消费性遍历：forEach&forEachOrdered

这两个方法都是用于对流中的元素进行遍历，区别在于forEachOrdered能保证并行遍历的有序性，而forEach并不能，但是正因forEachOrdered保证了并行遍历的有序性，所以在并行执行的情况下效率不如forEach。

举个例子：

```java
public class StreamTest {
    public static void main(String[] args) {
        System.out.println("-------------------------------------------");
        // 顺序遍历
        System.out.print("forEach: ");
        Stream.of("A", "B", "C", "D").sequential()
                .forEach(s -> System.out.print(s + " "));
        System.out.print("\nforEachOrdered: ");
        Stream.of("A", "B", "C", "D").sequential()
                .forEachOrdered(s -> System.out.print(s + " "));

        System.out.println("\n-------------------------------------------");
        // 并行遍历
        System.out.print("forEach: ");
        Stream.of("A", "B", "C", "D").parallel()
               .forEach(s -> System.out.print(s + " "));
        System.out.print("\nforEachOrdered: ");
        Stream.of("A", "B", "C", "D").parallel()
                .forEachOrdered(s -> System.out.print(s + " "));
    }
}
```

运行结果：

```shell
-------------------------------------------
forEach: A B C D
forEachOrdered: A B C D
-------------------------------------------
forEach: B A C D         # 对于并发的forEach，每次执行结果都不一样
forEachOrdered: A B C D
```

##### 2. 利器之二——reduce

reduct，max，min，count这四个操作归根结底都属于Reduce操作(Map&Reduce核心之一)，所以重点说说reduce这个核心操作(reduce，归约，这名字让我莫名地想到编译原理)。

在Stream中有三个reduce重载方法：

```java
// reduce有三个方法
Optional<T> reduce(BinaryOperator<T> accumulator);

T reduce(T identity, BinaryOperator<T> accumulator);

<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
```

这里以三个小例子来分别解释这三个reduce方法的功能

> 下面伪代码来自与官方文档，需要注意的是下面所说的“功能类似”，实际reduce实现并没有这么简单，因为Stream不仅可以顺序执行，还可以使用并行执行，这也是Stream的强大所在。

方法一：

```java
// 功能： 取出一系列字符串中最长的一个字符串
Optional<String> longest = Stream.of("Hello", "Lambda", "Hello", "Java")
        .reduce((s1, s2) -> s1.length() > s2.length() ? s1 : s2);
String longestStr = longest.get();
System.out.println(longestStr);
// 输出结果： Lambda

// 上面的功能可以使用max方法执行
// Optional<String> max = Stream.of("Hello", "Lambda", "Hello", "Java")
//                .max((s1, s2) -> s1.length() - s2.length());
// System.out.println(max.get());
// (s1, s2) -> s1.length() - s2.length() 在这里是一个比较器
// 实际上max方法也是这么实现的

// Optional<T> reduce(BinaryOperator<T> accumulator);
// 在功能上这个reduce类似于下面这段伪代码：
Optional<T> simpleReduce(BinaryOperator<T> accumulator)
     boolean foundAny = false;
     T result = null;
     for (T element : <this stream>) {
         // 第一个元素，作为初始值
         if (!foundAny) { foundAny = true; result = element; }
         else result = accumulator.apply(result, element);
     }
     // Stream中没有元素，则使用空Optional作为返回值
     // 使用Optional的一个原因就是为了防止返回null
     return foundAny ? Optional.of(result) : Optional.empty();
}
```

示例二：

```java
// T reduce(T identity, BinaryOperator<T> accumulator);
// 第二个方法和第一个方法的唯一的区别就是有一个初始值identity
// 第二个方法功能上和下面的代码类似
T simpleReduce(T identity, BinaryOperator<T> accumulator){
     T result = identity;
     for (T element : this stream)
         result = accumulator.apply(result, element)
     return result;
}

// A作为初始值与Stream中的四个元素进行比较
String longestStr = Stream.of("Hello", "Lambda", "Hello", "Java")
        .reduce("A", (str1, str2) -> str1.length() > str2.length() ? str1 : str2);
System.out.println(longestStr);
// 输出结果：Lambda
```

示例三：

```java
// 功能：计算一系列字符串的字符总数
int sumLength = Stream.of("Hello", "Lambda", "Hello", "Java")
      .reduce(0, (acc, str2) -> acc + str2.length(), (len1, len2) -> len1 + len2);
System.out.println(sumLength);
// 输出结果：20
// 引用类型的Stream没有sum方法，只有数值类型的IntStream,LongStream,DoubleStream有sum方法。
// 但是正如上面的例子，我们可以使用reduce方法实现类似sum的功能
// 事实上三种基本数据类型的Stream的sum方法也是使用reduce实现的

<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
<U> U simpleReduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner){
     T result = identity;
     for (T element : this stream)
         result = accumulator.apply(result, element)
     return result;
}
```


> max,min,sum等方法的实现可以查看几种Stream对应的实现类Pipeline中的源码，也是使用reduce实现，这里就不做过多的说明

##### 3. 又一大利器——collect

![Reduce，Collect](http://img-blog.csdn.net/20170822155218684?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

假如有一个需求把一系列字符串整理成一个集合，不熟悉collet的时候，我会使用reduce实现这么写：

```java
public class StreamTest {
    public static void main(String[] args) {
        Collection<String> ll = new LinkedList<String>();
        // 使用reduce API实现
        Collection<String> collection = Stream.of("Hello", "Lambda", "Hello", "Java")
                .reduce(ll,                // 初始集合
                 (collect, item) -> {      // 将元素添加到集合中
                    collect.add(item);
                    return collect;
                }, (collect1, collect2) -> {       // 将集合合并起来
                    collect1.addAll(collect2);
                    return collect1;
                });
        System.out.println(collection);
    }
}
// 运行结果：[Hello, Lambda, Hello, Java]
```

虽然能够实现，但是代码很冗长，这可不是Stream的风格。所以有了collect这个API。

collect有两个重载方法，先来看这个和reduce有点像的：

```java
<R> R collect(Supplier<R> supplier,
              BiConsumer<R, ? super T> accumulator,
              BiConsumer<R, R> combiner);
```

我们使用这个API实现上面的需求：

```java
public class StreamTest {
    public static void main(String[] args) {
        Collection<String> collect = Stream.of("Hello", "Lambda", "Hello", "Java").collect(
                    LinkedList<String>::new, // 初始一个集合
                    LinkedList::add, // 将元素添加到集合中
                    LinkedList::addAll); // 将集合合并起来
        System.out.println(collect);
    }
}
// 运行结果：[Hello, Lambda, Hello, Java]
```

虽然代码少了几行，但还不是最简单的。Stream还提供了一个更简单的collect方法：

```java
<R, A> R collect(Collector<? super T, A, R> collector);
```

我们再把上面的代码修改一下：

```java
public class StreamTest {
    public static void main(String[] args) {
        Collection<String> collect = Stream.of("Hello", "Lambda", "Hello", "Java")
                .collect(Collectors.toCollection(LinkedList::new));
        System.out.println(collect);
    }
}
// 运行结果：[Hello, Lambda, Hello, Java]
```

看到这你也许会好奇Collector是什么，Collectors又是什么。下面就来解决这个问题。

## Collector收集器

![Collector](函数式编程与Lambda表达式\Collector.svg)

上面的例子中用到了Collectors这个工具类，它是专门用来生成Collector接口的实例对象。所以在这之前我们先来看看Collector接口是用来干嘛的：

```java
public interface Collector<T, A, R> {
  /** 下面五个方法是让我们提供五个reduce操作所需要的参数 **/

  // 这三个方法，和我们前面reduce方法中的参数一致，
  Supplier<A> supplier();
  BiConsumer<A, T> accumulator();
  BinaryOperator<A> combiner();

  // finisher用于做最后转换的处理
  Function<A, R> finisher();
  // characteristics指明Collector的属性特征
  Set<Characteristics> characteristics();

  // 用于生成Collector实例对象的两个静态方法
  public static<T, R> Collector<T, R, R> of(Supplier<R> supplier,
                                            BiConsumer<R, T> accumulator,
                                            BinaryOperator<R> combiner,
                                            Characteristics... characteristics) {
     ... // 省略若干语句
     return new Collectors.CollectorImpl<>(supplier, accumulator, combiner, cs);
  }
  public static<T, A, R> Collector<T, A, R> of(Supplier<A> supplier,
                                               BiConsumer<A, T> accumulator,
                                               BinaryOperator<A> combiner,
                                               Function<A, R> finisher,
                                               Characteristics... characteristics){

     ... // 省略若干语句
     return new Collectors.CollectorImpl<>(supplier, accumulator, combiner, finisher, cs);
  }
  // characteristics的取值
  enum Characteristics {
    CONCURRENT, UNORDERED, IDENTITY_FINISH
  }
}
```

来个例子说明一下怎么用Collector类：

```java
public class StreamTest {
    public static void main(String[] args) {
        Collector<String, List<String>, List<String>> toList = Collector.of(
                LinkedList::new,  // 提供一个LinkedList
                List::add,        // 将元素添加到LinkedList中
                (list1, list2) -> { list1.addAll(list2); return list1; },  // 合并两个LinkedList
                Collections::unmodifiableList,  // 将LinkedList包装成UnmodifiedList
                Characteristics.CONCURRENT,     // 是否并发进行collect操作
                Characteristics.IDENTITY_FINISH,// 指定最后的finisher需不需要进行强制转换
                Characteristics.UNORDERED);     // 是否保证并发时的有序性
        List<String> unmodList = Stream.of("Hello", "Lambda", "Hello", "Java")
                .collect(toList);
        System.out.println(unmodList);
    }
}
```

也许看到这你已经晕了：为什么这么复杂。嗯，放心，Stream不会让我们编程变得这么复杂，因为我们大部分常用的Collector已经在Collectors工具类中封装好了（果然是送佛送到西(→_→)）。

![Collectors工具类](http://img-blog.csdn.net/20170822155354590?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里挑几个常用的说明一下：

### 1. 集合收集器

```java
// collectionFactory参数可以自己指定集合类，
// 支持所有的Collection接口的实现类
public static <T, C extends Collection<T>>
    Collector<T, ?, C> toCollection(Supplier<C> collectionFactory);
// 转成List(实际上是ArrayList)
public static <T> Collector<T, ?, List<T>> toList();
// 转成Set(实际上是HashSet)
public static <T> Collector<T, ?, Set<T>> toSet();
```

> 这三个估计是最简单的

### 2. Map映射收集器

```java
/////////////////////////////////////////////////////
// 转成普通的Map(默认创建HashMap)
public static <T, K, U> Collector<T, ?, Map<K,U>>
       toMap(Function<? super T, ? extends K> keyMapper,
             Function<? super T, ? extends U> valueMapper);
// 例子：
Map<String, Student> map = students.parallelStream()
    .collect(Collectors.toMap(Student::getId, Function.identity()));
//这里的Function.identity返回一个"t->t"的匿名函数(恒等式)
// 这种静态方法用来表示该类函数式接口中最普遍的实现。
// 在其他函数接口中也有一些,如Predicate.isEqual(相等断言)

/////////////////////////////////////////////////////
// mergeFunction用于处理两个key相同的情况
public static <T, K, U> Collector<T, ?, Map<K,U>>
       toMap(Function<? super T, ? extends K> keyMapper,
             Function<? super T, ? extends U> valueMapper,
             BinaryOperator<U> mergeFunction);
// 例子：
Map<String, String> phoneBook = people.stream()
          .collect(toMap(Person::getName, // 有重名的人
                     Person::getAddress,
                     (s, a) -> s + ", " + a));//把两个名字相同的地址连成一个字符串

/////////////////////////////////////////////////////
// mapSupplier提供Map的实现类
public static <T, K, U, M extends Map<K, U>> Collector<T, ?, M>
       toMap(Function<? super T, ? extends K> keyMapper,
             Function<? super T, ? extends U> valueMapper,
             BinaryOperator<U> mergeFunction,
             Supplier<M> mapSupplier);

/////////////////////////////////////////////////
// 转成ConcurrentMap(默认创建ConcurrentHashMap)
public static <T, K, U> Collector<T, ?, ConcurrentMap<K,U>>
       toConcurrentMap(Function<? super T, ? extends K> keyMapper,
                       Function<? super T, ? extends U> valueMapper);
public static <T, K, U> Collector<T, ?, ConcurrentMap<K,U>>
       toConcurrentMap(Function<? super T, ? extends K> keyMapper,
                       Function<? super T, ? extends U> valueMapper,
                       BinaryOperator<U> mergeFunction);
public static <T, K, U, M extends ConcurrentMap<K, U>> Collector<T, ?, M>
       toConcurrentMap(Function<? super T, ? extends K> keyMapper,
                       Function<? super T, ? extends U> valueMapper,
                       BinaryOperator<U> mergeFunction,
                       Supplier<M> mapSupplier)
```

> 也许你和我一样看到这一大坨泛型都想吐了

### 3. 分组操作

分组操作有两种：

* 根据条件满足与否(true|false)分为两组：

```java
public static <T>
    Collector<T, ?, Map<Boolean, List<T>>> partitioningBy(Predicate<? super T> predicate);
public static <T, D, A>
    Collector<T, ?, Map<Boolean, D>> partitioningBy(Predicate<? super T> predicate,
                                                    Collector<? super T, A, D> downstream);
```

* 根据Function指定的分组规则分为多组：

```java
// Map
public static <T, K> Collector<T, ?, Map<K, List<T>>> groupingBy(
                         Function<? super T, ? extends K> classifier);
public static <T, K, A, D> Collector<T, ?, Map<K, D>> groupingBy(
                         Function<? super T, ? extends K> classifier,
                         Collector<? super T, A, D> downstream);
public static <T, K, D, A, M extends Map<K, D>> Collector<T, ?, M> groupingBy(
                         Function<? super T, ? extends K> classifier,
                         Supplier<M> mapFactory,
                         Collector<? super T, A, D> downstream);
// ConcurrentMap
public static <T, K> Collector<T, ?, ConcurrentMap<K, List<T>>> groupingByConcurrent(
                         Function<? super T, ? extends K> classifier);
public static <T, K, A, D> Collector<T, ?, ConcurrentMap<K, D>>  groupingByConcurrent(
                         Function<? super T, ? extends K> classifier,
                         Collector<? super T, A, D> downstream);
public static <T, K, A, D, M extends ConcurrentMap<K, D>> Collector<T, ?, M> groupingByConcurrent(
                         Function<? super T, ? extends K> classifier,
                         Supplier<M> mapFactory,
                         Collector<? super T, A, D> downstream)
```

> 又是一大坨的泛型

### 4. 字符串拼接

```java
public static Collector<CharSequence, ?, String> joining();
public static Collector<CharSequence, ?, String> joining(CharSequence delimiter);
public static Collector<CharSequence, ?, String> joining(CharSequence delimiter,
                                                         CharSequence prefix,
                                                         CharSequence suffix);
```

> 这个功能还是不错的，前缀后缀都帮你解决了。

例子：

```java
public class StreamTest {
    public static void main(String[] args) {
        String collect = Stream.of("Alibaba", "Tecent", "Baidu", "Huawei", "Sohu", "HTC")
                .collect(Collectors.joining(", ", "{ ", " }"));
        System.out.println(collect);
    }
}
// 运行结果：{ Alibaba, Tecent, Baidu, Huawei, Sohu, HTC }
```



> 参考链接：
>
> * https://java-latte.blogspot.jp/2014/03/stream-lambda-in-java-8.html
> * https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html