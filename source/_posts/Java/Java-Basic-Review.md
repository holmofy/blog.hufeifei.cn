---
title: Java语言基础复习与巩固
date: 2017-05-14
categories: JAVA
mathjax: true
---

[TOC]

## 八种基本数据类型

Java语言中有八种基本数据类型：

​	`byte`,`short`,`int`,`long`,`float`,`double`,`char`,`boolean`

C语言因为依赖与CPU平台，数据类型所占字节数可能会因硬件平台以及编译器的不同而有所改变。因为Java是不依赖于任何硬件平台的软件平台，所以它在设计的时候每种数据类型的所占的字节数是固定的(除boolean以外)。


|  数据类型   | 所占字节数 | 备注                              |
| :-----: | :---: | :------------------------------ |
|  byte   | 1Byte | 8位整数范围： $-2^{7}$ ~ $2^{7}-1$    |
|  short  | 2Byte | 16位整数范围： $-2^{15}$ ~ $2^{15}-1$ |
|   int   | 4Byte | 32位整数范围 $-2^{31}$ ~ $2^{31}-1$  |
|  long   | 8Byte | 64位整数范围 $-2^{63}$ ~ $2^{63}-1$  |
|  float  | 4Byte | 单精度的32位IEEE 754浮点数标准            |
| double  | 8Byte | 双精度的64位IEEE 754浮点数标准            |
|  char   | 2Byte | 16位的Unicode字符范围：\u0000 ~ \uffff |
| boolean |  不确定  |                                 |


> 注意：Java中boolean比较特殊：单个boolean在编译时映射成int；boolean数组则会被编译成byte数组。用1表示true，0表示false。参考[JVM规范](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3.4)

## 基本数据类型的包装类

Java中为上面的八种基本类型提供了包装类，从而让基本类型变成引用类型。有了这些包装类我们就可以在ArrayList，HashMap等Java集合类中存储基本的数据类型了(因为集合类型底层都是使用Object引用类型)。

![基本数据类型的包装类](http://img-blog.csdn.net/20170623011407927?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

原本JDK1.5之前这些基本数据类型要变成包装类，必须得通过相应的构造函数来对基本数据类型进行包装。这就导致一个问题：如果我往ArrayList中存入一万个整型的1，则需要new出一万个Integer对象。这无疑会消耗很大的内存（Java对象不像C语言结构体那么干净，Java对象头部有很多字段，诸如monitor等信息）。

于是JDK5.0后提供了自动拆装箱机制。自动拆装箱机制分为两个操作：自动拆箱(auto boxing)和自动装箱(auto unboxing)

* 自动拆箱：就是程序编译时，遇到数值类型复制给相应的包装类引用，会自动调用相应类型包装类的`valueOf`方法来获取包装类对象。

* 自动拆箱：就是程序编译时，遇到包装类对象复制给相应的数值类型，会自动调用包装类对象的xxxValue方法来获取相应的数值类型。

比如下面这段简短的代码：

```java
public class NumberTest {
	public static void main(String[] args){
		Integer i1 = 10;
		int i2 = i1;

		Boolean b1 = true;
		boolean b2 = b1;

		Character c1 = 'c';
		char c2 = c1;
	}
}
```

使用以下Windows命令对其进行编译后并进行反编译：

```powershell
C:\Users\Holmofy\Desktop>javac NumberTest.java

C:\Users\Holmofy\Desktop>dir|find "NumberTest"
2017/05/23  22:50               599 NumberTest.class
2017/05/23  22:48               198 NumberTest.java

C:\Users\Holmofy\Desktop>javap -c NumberTest > NumberTest.javap

C:\Users\Holmofy\Desktop>dir|find "NumberTest"
2017/05/23  22:50               599 NumberTest.class
2017/05/23  22:48               198 NumberTest.java
2017/05/23  22:51             1,162 NumberTest.javap

C:\Users\Holmofy\Desktop>
```

打开得到的反编译文件NumberTest.javap：

```java
Compiled from "NumberTest.java"
public class NumberTest {
  public NumberTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       5: astore_1
       6: aload_1
       7: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
      10: istore_2
      11: iconst_1
      12: invokestatic  #4                  // Method java/lang/Boolean.valueOf:(Z)Ljava/lang/Boolean;
      15: astore_3
      16: aload_3
      17: invokevirtual #5                  // Method java/lang/Boolean.booleanValue:()Z
      20: istore        4
      22: bipush        99
      24: invokestatic  #6                  // Method java/lang/Character.valueOf:(C)Ljava/lang/Character;
      27: astore        5
      29: aload         5
      31: invokevirtual #7                  // Method java/lang/Character.charValue:()C
      34: istore        6
      36: return
}
```

确实如此。

## 包装类的缓存池

前面说自动装箱调用的相应包装类的valueOf静态方法。那看看这些静态方法的实现：

Byte

```java
    public static Byte valueOf(byte b) {
        final int offset = 128;
        return ByteCache.cache[(int)b + offset];
    }
```

Short

```java
    public static Short valueOf(short s) {
        final int offset = 128;
        int sAsInt = s;
        if (sAsInt >= -128 && sAsInt <= 127) { // must cache
            return ShortCache.cache[sAsInt + offset];
        }
        return new Short(s);
    }
```

Integer

```java
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

Long

```java
    public static Long valueOf(long l) {
        final int offset = 128;
        if (l >= -128 && l <= 127) { // will cache
            return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
    }
```

Float

```java
    public static Float valueOf(float f) {
        return new Float(f);
    }
```

Double

```java
    public static Double valueOf(double d) {
        return new Double(d);
    }
```

Character

```java
    public static Character valueOf(char c) {
        if (c <= 127) { // must cache
            return CharacterCache.cache[(int)c];
        }
        return new Character(c);
    }
```

Boolean

```java
    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }
```

通过查看源码发现，总结出下面几个结论：

1. Byte，Short，Integer，Long这四种整数包装类型都缓存了`-128 ~ 127`(即256个数据)。
2. Float，Double这两个浮点包装类型没有缓存，valueOf直接调用构造方法进行包装。
3. Character包装类型缓存了`0 ~ 127`总共128个ASCII码值。
4. Boolean包装类型缓存了true(TRUE)和false(FALSE)。

有了这些缓存数据，从而避免了前面所说的大量new对象的问题。但也正因为这些缓存数据，也导致了以下的现象：

```java
public class NumberTest{
	public static void main(String[] args){
		Integer i1 = new Integer(100);
		Integer i2 = new Integer(100);
		System.out.println(i1 == i2);

		Integer i3 = 100;
		Integer i4 = 100;
		System.out.println(i3 == i4);

		Integer i5 = 200;
		Integer i6 = 200;
		System.out.println(i5 == i6);
	}
}
```

运行结果：

```shell
false
true
false
```

很明显这是Integer缓存搞的鬼：100是缓存对象，而200不是缓存对象。

也许你不会觉得这对程序有什么影响，但是如果你了解过IdentityHashMap这个集合类，你就不会这么想了。

> 关于IdentityHashMap的介绍可以查看[Java集合框架总结和巩固](http://blog.csdn.net/holmofy/article/details/71215548)

## Integer缓存池的自定义配置

通过查看Byte，Short，Integer，Long源码你会发现，它们缓存都包装在一个相应的类中ByteCache、ShortCache、IntegerCache、LongCache。其中ByteCache，ShortCache，LongCache这三个内部类代码基本一致：都是缓存`-127~128`的数据。

> 以下代码均来自jdk1.8.0_131，其他版本不能保证完全相同

```java
	// java.lang.Byte.ByteCache
	private static class ByteCache {
        private ByteCache(){}

        static final Byte cache[] = new Byte[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Byte((byte)(i - 128));
        }
    }
	// java.lang.Short.ShortCache
    private static class ShortCache {
        private ShortCache(){}

        static final Short cache[] = new Short[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Short((short)(i - 128));
        }
    }
	// java.lang.Long.LongCache
    private static class LongCache {
        private LongCache(){}

        static final Long cache[] = new Long[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Long(i - 128);
        }
    }
```

但IntegerCache类的内容就丰富了。

```java
	// java.lang.Integer.IntegerCache
	private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

可以看出Java会去查找虚拟机的`java.lang.Integer.IntegerCache.high`属性来决定创建多大的缓存池，如果该属性小于127则使用默认的127作为缓存池的最大界限，否则就使用`java.lang.Integer.IntegerCache.high`属性中的值。我们可以通过java命令的`-XX:AutoBoxCacheMax=<size>`来设置JVM的`java.lang.Integer.IntegerCache.high`属性。

比如下面这个例子：

```java
public class Test {
	public static void main(String[] args) {
		Integer a = 256;
		Integer b = 256;
		System.out.println(a == b);
	}
}
```

按照前面的解释，正常情况下运行会输出`false`。在运行时进行如下配置就会输出`true`。

![Integer缓存池的配置](http://img-blog.csdn.net/20170802125934475?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 包装类中的其他工具方法

在这些基本数据类型的包装类中还提供给我们许多的工具方法。

1. 解析字符串

   在这些包装类中都有`parseXxxx`这样的静态方法，使用这些静态方法能够将字符串类型的值解析成相应的数据类型。并且对于Byte，Short，Integer，Long这四种整数包装类型还可以在解析时指定基数(也就是使用8进制，使用10进制还是16进制)。

2. 转换成字符串

   包装类中还有toString静态方法，能将基本数据类型转换成相应的字符串值。

3. 其他：Boolean中的逻辑运算方法、Character中的静态工具方法最多(如判断数字字符、空白字符、西欧字符，转大写、小写)，这些工具方法正等待着各位自己去发掘(避免重复造轮子哦)。


![基本数据类型的工具方法](http://img-blog.csdn.net/20170623011508265?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 基本数据类型的格式化输出

在Java中提供了两个类PrintStream和PrintWriter支持常见数据的格式化输出。

> 关于PrintStream和PrintWriter的更多详细内容可以参考[JavaIO总结与巩固](http://blog.csdn.net/holmofy/article/details/72626387)

比如我们常用的System.out.print等方法也是该类中的方法。

除了这些方法，还提供了format，printf等方法。

```java
PrintStream format(String format, Object ... args);
PrintStream format(Locale l, String format, Object ... args)
PrintStream printf(String format, Object ... args)
PrintStream printf(Locale l, String format, Object ... args)
PrintWriter format(String format, Object ... args);
PrintWriter format(Locale l, String format, Object ... args)
PrintWriter printf(String format, Object ... args)
PrintWriter printf(Locale l, String format, Object ... args)
```

这些方法都是直接或间接地调用了`java.util.Formatter`中的`format`方法。在使用上这些方法和C语言中的`printf`方法用法上还是有很多的差别。

|  转换符  |               备注                |
| :---: | :-----------------------------: |
|   d   |   decimal。以十进制输出的整数，java不支持i    |
|   f   |   float。以十进制输出的浮点数，java中没有lf    |
|   s   | string。输出字符串，java有字符串连接，这个用的比较少 |
|   n   |    newline。换行，java推荐使用%n代替\n    |
| ty,tY |   time year。ty—两位数年份，tY—四位数年份   |
|  tB   |       以地区的方式显示月份，如：May，五月       |
|  tm   |      time month。两位数月份不够补零       |
| td,te |  time day。td—两位数日期不够补零，te—不会补零  |
|  tl   |            十二进制的小时数             |
|  tM   |     time minutes。两位数分钟数不够补零     |
|  tp   |   以地区的方式显示上下午，如：AM\|PM，上午\|下午   |
|  tD   |     time Date。等价于：%tm%td%ty     |
|  08   |       以8位字符输出，不够前面补零，并右对齐       |
|   +   |           输出正负号(包括符号)           |
|   -   |               左对齐               |
|   ,   |      以地区的方式输出分隔符，如：122,345      |
|  .3   |            输出时保留三位小数            |
| 10.3  |      以10位字符输出，右对齐，并保留三位小数       |

示例：

```java
import java.util.Calendar;
import java.util.Locale;

public class TestFormat {

    public static void main(String[] args) {
      long n = 461012;
      System.out.format("%d%n", n);                  //  -->  "461012"
      System.out.format("%08d%n", n);                //  -->  "00461012"
      System.out.format("%+8d%n", n);                //  -->  " +461012"
      System.out.format("%,8d%n", n);                //  -->  " 461,012"
      System.out.format("%+,8d%n%n", n);             //  -->  "+461,012"

      double pi = Math.PI;
      System.out.format("%f%n", pi);                 //  -->  "3.141593"
      System.out.format("%.3f%n", pi);               //  -->  "3.142"
      System.out.format("%10.3f%n", pi);             //  -->  "     3.142"
      System.out.format("%-10.3f%n", pi);            //  -->  "3.142"
      System.out.format(Locale.FRANCE,
                        "%-10.4f%n%n", pi);          //  -->  "3,1416"

      Calendar c = Calendar.getInstance();
      System.out.format("%tB %te, %tY%n", c, c, c);  //  -->  "May 29, 2016"
      System.out.format("%tl:%tM %tp%n", c, c, c);   //  -->  "2:34 am"
      System.out.format("%tD%n", c);                 //  -->  "05/29/16"
    }
}
```
> 关于字符格式化的更多内容可以查看JDK中的java.text.*`包中的相关描述。

## Java中的数学运算

java像大多数语言一样不仅提供了`+`,`-`,`*`,`/`,`%`这样的基本数学运算，还提供了一个Math类来进行更复杂的数学运算。

调用Math类中的数学方法我们可以使用两种方式：

1. 因为Math类属于`java.lang`包下的，所以我们无需另外导包，可以直接Math.xxx()进行调用。
2. 因为Math类的方法都是`static`静态的，所以我们可以使用`import static java.lang.Math.*`方式导入Math类的静态方法。这样我们调用的时候可以不用加类名，直接调用xxx()方法。

### Math类中的常量和最基本方法

- `Math.E`, 自然对数的底数
- `Math.PI`, 圆周率

<table width="100%" border="1" cellpadding="4" cellspacing="3" summary="基础方法摘要"><caption><b>最基础的方法</b></caption><thead><tr><th id="method">
                方法
            </th><th width="50%" id="desc">
                方法描述
            </th></tr></thead><tbody><tr><td headers="method"><code>
double abs(double d)
<br>
float abs(float f)
<br>
int abs(int i)
<br>
long abs(long lng)
</code></td><td headers="desc">
                取绝对值
            </td></tr><tr><td headers="method"><code>
double ceil(double d)
</code></td><td headers="desc">
                向上取整
            </td></tr><tr><td headers="method"><code>
double floor(double d)
</code></td><td headers="desc">
                向下取整
            </td></tr><tr><td headers="method"><code>
double rint(double d)
</code></td><td headers="desc">
                四舍五入取整
            </td></tr><tr><td headers="method"><code>
long round(double d)
<br>
int round(float f)
</code></td><td headers="desc">
                四舍五入取整
            </td></tr><tr><td headers="method"><code>
double min(double arg1, double arg2)
<br>
float min(float arg1, float arg2)
<br>
int min(int arg1, int arg2)
<br>
long min(long arg1, long arg2)
</code></td><td headers="desc">
                返回两个参数的较小值
            </td></tr><tr><td headers="method"><code>
double max(double arg1, double arg2)
<br>
float max(float arg1, float arg2)
<br>
int max(int arg1, int arg2)
<br>
long max(long arg1, long arg2)
</code></td><td headers="desc">
                返回两个参数的较大值
            </td></tr></tbody></table>

### 指数运算与幂运算

<table width="100%" border="1" cellpadding="4" cellspacing="3" summary="幂运算与指数运算"><caption><b>幂运算与指数运算</b></caption><tbody><tr><th id="method">
方法
</th><th width="50%" id="desc">
方法描述
</th></tr><tr><td headers="method"><code>
double exp(double d)
</code></td><td headers="method">
自然底数e的d次幂
</td></tr><tr><td headers="method"><code>
double log(double d)
</code></td><td headers="desc">
d的自然对数。ln(d)
</td></tr><tr><td headers="method"><code>
double pow(double base, double exponent)
</code></td><td headers="desc">
b的e次幂
</td></tr><tr><td headers="method"><code>
double sqrt(double d)
</code></td><td headers="desc">
开平方根。根号d
</td></tr></tbody></table>

### 三角函数运算

<table width="100%" border="1" cellpadding="4" cellspacing="3" summary="三角函数摘要"><caption><b>三角函数摘要</b></caption><tbody><tr><th id="method">
方法
</th><th width="50%" id="desc">
方法描述
</th></tr><tr><td headers="method"><code>
double sin(double d)
</code></td><td headers="desc">
求正弦值
</td></tr><tr><td headers="method"><code>
double cos(double d)
</code></td><td headers="desc">
求余弦值
</td></tr><tr><td headers="method"><code>
double tan(double d)
</code></td><td headers="desc">
求正切值
</td></tr><tr><td headers="method"><code>
double asin(double d)
</code></td><td headers="desc">
反正弦值。arcsine(sin(x))==x
</td></tr><tr><td headers="method"><code>
double acos(double d)
</code></td><td headers="desc">
反余弦值。arccos(cos(x))==x
</td></tr><tr><td headers="method"><code>
double atan(double d)
</code></td><td headers="desc">
反正切值。arctan(tan(x))==x
</td></tr><tr><td headers="method"><code>
double atan2(double y, double x)
</code></td><td headers="desc">
将直角坐标系<code>(x, y)</code> 转成极坐标系
<code>(r, theta)</code> 并返回 <code>theta</code>的值。
</td></tr><tr><td headers="method"><code>
double toDegrees(double d)
<br>
double toRadians(double d)
</code></td><td headers="desc">
将弧度制转换成角度制
</td></tr></tbody></table>

### 产生伪随机数

Math.random()函数底层使用`java.util.Random`类随机生成一个大于等于零并且小于一的双精度浮点数：`0.0 <= Math.random() < 1.0`。这和C语言的rand()函数还是有区别，rand函数生成的是0到32767(RAND_MAX)之间的数。这个方法比较鸡肋，我们完全可以使用`java.util.Random`或`java.security.SecureRandom`这两个类来生成随机数。



## Java中的字符串

### Java字符串与C/C++字符串对比

字符串是程序设计中最常用到的数据类型。

Java中有一个专门的类String用来代表字符串，而不像C/C++中字符串仅仅是一个以`\0`结尾的字符数组（C语言的字符串常量与字符数组还是有很大的区别的）。

> C/C++：
>
> ```C
> char str[] = "Hello World";// 字符串长度等于实际字符长度加上一个\0结尾符
> // sizeof(str) == 12 == 11+1
> ```

> Java：
>
> ```java
> //Java数组封装了数组长度，cArr.length = 11;
> char[] cArr = {'H','e','l','l','o',' ',' W','o','r','l','d','!'};
> String str1 = new String(cArr);// String类封装了char[]
> String str2 = "Hello World!";
> // Java中不支持char[] str = "Hello World";因为字符串常量也是String类对象。
> ```

正因C/C++语言字符串是基本数据类型，导致了C++中各种对字符串类的封装：如STL中的`std::string`，MFC中的`CString`，Qt中的`QString`...各种库中都有自己实现的封装。

另外Java中char类型占两个字节，使用Unicode字符集UTF-16编码；而C/C++的char类型占一个字节，使用ASCII编码。

> 关于编码的问题可以查看我的这篇文章：[从ASCII、ISO-8859、GB2312、GBK到Unicode的UCS-2、UCS-4、UTF-8、UTF-16、UTF-32](http://blog.csdn.net/holmofy/article/details/72846118)

### Java字符串的不可变性

尽管Java的String类中提供了像`replace()`这样的修改字符串的方法，但实际上Java的String类是一个不可变类型：一旦创建，对象的字符内容不可修改。那些`replace()`之类的方法都是重新创建一个String类对象作为返回值。

那我们平时经常使用“`+`”操作符作为来进行字符串连接，是怎样实现的呢？

### Java的字符串连接底层实现

下面我们写一个简单的小Demo来看一下String的字符串连接：

```java
public class StringTest{
	public static void main(String[] args){
		String a = "Hello";
		String b = new String("Hello");
		System.out.println(a == b);
		String c = "Hello";
		System.out.println(a == c);
		String d = c + "World!";
		String e = "Hello" + "World!";
		System.out.println(d == e);
		String f = "HelloWorld!";
		System.out.println(e == f);
	}
}
```

我们编译运行，然后反编译一下看看底层实现

```powershell
C:\Users\Holmofy\Desktop>javac StringTest.java

C:\Users\Holmofy\Desktop>dir|find "StringTest"
2017/06/02  20:01             1,233 StringTest.class
2017/05/26  16:04               441 StringTest.java

C:\Users\Holmofy\Desktop>javap -c StringTest > StringTest.javap

C:\Users\Holmofy\Desktop>dir|find "StringTest"
2017/06/02  20:01             1,233 StringTest.class
2017/05/26  16:04               441 StringTest.java
2017/06/02  20:02             5,397 StringTest.javap
```

我们先不讨论运行结果，先来看一下StringTest.javap这个反编译出来的文件：

```java
Compiled from "StringTest.java"
public class StringTest {
  public StringTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String Hello
       2: astore_1
       3: new           #3                  // class java/lang/String
       6: dup
       7: ldc           #2                  // String Hello
       9: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
      12: astore_2
      13: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      16: aload_1
      17: aload_2
      18: if_acmpne     25
      21: iconst_1
      22: goto          26
      25: iconst_0
      26: invokevirtual #6                  // Method java/io/PrintStream.println:(Z)V
      29: ldc           #2                  // String Hello
      31: astore_3
      32: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      35: aload_1
      36: aload_3
      37: if_acmpne     44
      40: iconst_1
      41: goto          45
      44: iconst_0
      45: invokevirtual #6                  // Method java/io/PrintStream.println:(Z)V
      48: new           #7                  // class java/lang/StringBuilder
      51: dup
      52: invokespecial #8                  // Method java/lang/StringBuilder."<init>":()V
      55: aload_3
      56: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      59: ldc           #10                 // String World!
      61: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      64: invokevirtual #11                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      67: astore        4
      69: ldc           #12                 // String HelloWorld!
      71: astore        5
      73: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      76: aload         4
      78: aload         5
      80: if_acmpne     87
      83: iconst_1
      84: goto          88
      87: iconst_0
      88: invokevirtual #6                  // Method java/io/PrintStream.println:(Z)V
      91: ldc           #12                 // String HelloWorld!
      93: astore        6
      95: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      98: aload         5
     100: aload         6
     102: if_acmpne     109
     105: iconst_1
     106: goto          110
     109: iconst_0
     110: invokevirtual #6                  // Method java/io/PrintStream.println:(Z)V
     113: return
}
```

可以看出字符串连接实际上是new一个StringBuilder对象，然后调用它的append方法，最后调用toString方法转换成String类型的不可变对象。

我们将结果运行出来：

```powershell
C:\Users\Holmofy\Desktop>java StringTest
false
true
false
true
```

从运行结果可以得出以下结论：

```java
"Hello" != new String("Hello");    // 字符串常量与new出来的String对象不是同一个对象
"Hello" == "Hello";    // 两个字符串常量是同一个String对象

String c = "Hello";
c + "World!" != "Hello" + "World!";
"Hello" + "World!" == "HelloWorld!";
// c + "World!"运行时会调用StringBuilder.append进行字符串连接
// "Hello" + "World!"不会在运行时进行字符串连接，而是在编译时转成"HelloWorld!"常量
```

### String字符串常量池的底层实现

Java中所有的字符串常量都存储在字符串常量池中。比如上面Demo中的"Hello"，"HelloWorld!"这种字面量。

对于我们通过构造方法new出来的String对象都是创建在Java的堆内存中，这样的对象一旦失去引用后，我们就无法再次使用了，只能等着GC回收这个对象。

其实String中有个`intern()`方法，可以将我们new出来的字符串对象，加入到字符串常量池中，这样我们下次再使用这样的字面量时，Java会从池中返回这个对象的引用。

举个例子：

```java
StringBuilder builder = new StringBuilder();
// 随机生成一个100以内的加法运算
builder.append((int)(Math.random()*100));
builder.append('+');
builder.append((int)(Math.random()*100));
builder.append('=');
String question = builder.toString(); // 此时的question对象并不在常量池中
question.intern();// 这句话能把生成的加法算式加入到字符串常量池中
```

我们先看一下Java文档中是怎么描述这个方法的：

```java
/**
 * Returns a canonical representation for the string object.
 * <p>
 * A pool of strings, initially empty, is maintained privately by the
 * class {@code String}.
 * <p>
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
 * <p>
 * It follows that for any two strings {@code s} and {@code t},
 * {@code s.intern() == t.intern()} is {@code true}
 * if and only if {@code s.equals(t)} is {@code true}.
 * <p>
 * All literal strings and string-valued constant expressions are
 * interned. String literals are defined in section 3.10.5 of the
 * <cite>The Java&trade; Language Specification</cite>.
 *
 * @return  a string that has the same contents as this string, but is
 *          guaranteed to be from a pool of unique strings.
 */
public native String intern();
```

如果该在字符串常量池中已经有一个和该字符串值相等(equal)的字符串对象，则intern方法会返回池中的这个对象；如果没有，则将该字符串加入到字符串常量池中。

所以两个字符串只有当`s.equal(t)`成立才会有`s.intern() == t.intern()`。

intern方法是native方法，接下来我们看一下它的C/C++实现。

注意我的JDK和JVM源码是`openjdk-8-src-b132-03_mar_2014`这个版本的，不同版本源码可能会有所差别。

首先找到java/lang/String.c文件：

```c++
#include "jvm.h"
#include "java_lang_String.h"

JNIEXPORT jobject JNICALL
Java_java_lang_String_intern(JNIEnv *env, jobject this)
{
    return JVM_InternString(env, this);
}
```

这个文件中就一个intern方法，底层调用的JVM的InternString方法。

找到Jvm.cpp：

```c++
JVM_ENTRY(jstring, JVM_InternString(JNIEnv *env, jstring str))
  JVMWrapper("JVM_InternString");
  JvmtiVMObjectAllocEventCollector oam;
  if (str == NULL) return NULL;
  oop string = JNIHandles::resolve_non_null(str);
  oop result = StringTable::intern(string, CHECK_NULL);
  return (jstring) JNIHandles::make_local(env, result);
JVM_END
```

找到JVM_ENTRY和JVM_END这两个宏定义(interfaceSupport.hpp文件中)

```c++
#define JVM_ENTRY(result_type, header)                               \
extern "C" {                                                         \
  result_type JNICALL header {                                       \
    JavaThread* thread=JavaThread::thread_from_jni_environment(env); \
    ThreadInVMfromNative __tiv(thread);                              \
    debug_only(VMNativeEntryWrapper __vew;)                          \
    VM_ENTRY_BASE(result_type, header, thread)

#define JVM_END } }
```

展开宏定义，就可以看到JVM_InternString的定义了：

```c++
extern "C" {
  jstring JNICALL JVM_InternString(JNIEnv *env, jstring str) {
    JavaThread* thread=JavaThread::thread_from_jni_environment(env);
    ThreadInVMfromNative __tiv(thread);
    debug_only(VMNativeEntryWrapper __vew;)
    VM_ENTRY_BASE(result_type, header, thread)
    // 前面几句不用管
    JVMWrapper("JVM_InternString");
    JvmtiVMObjectAllocEventCollector oam;

    // 下面两句是用来判空处理的
    if (str == NULL) return NULL;
    oop string = JNIHandles::resolve_non_null(str);// 将jstring对象转成oop对象

    oop result = StringTable::intern(string, CHECK_NULL);
    return (jstring) JNIHandles::make_local(env, result);// 将oop对象还原成jstring对象
  }
}
```

最关键的部分就是`StringTable::intern`方法，先让我们看一下StringTable类的定义：

```c++
class StringTable : public Hashtable<oop, mtSymbol> {
    friend class VMStructs;

private:
  // 单例模式
  static StringTable* _the_table;

  static bool _needs_rehashing;

  static volatile int _parallel_claimed_idx;

  static oop intern(Handle string_or_null, jchar* chars, int length, TRAPS);
  oop basic_add(int index, Handle string_or_null, jchar* name, int len,
                unsigned int hashValue, TRAPS);

  oop lookup(int index, jchar* chars, int length, unsigned int hashValue);

  // 将构造方法私有
  StringTable() : Hashtable<oop, mtSymbol>((int)StringTableSize,
                              sizeof (HashtableEntry<oop, mtSymbol>)) {}

  StringTable(HashtableBucket<mtSymbol>* t, int number_of_entries)
    : Hashtable<oop, mtSymbol>((int)StringTableSize, sizeof (HashtableEntry<oop, mtSymbol>), t,
                     number_of_entries) {}

public:
  // 获取StringTable单例
  static StringTable* the_table() { return _the_table; }

  static uint bucket_size() { return sizeof(HashtableBucket<mtSymbol>); }

  // 由JVM调用该方法来创建字符串缓冲池单例
  static void create_table() {
    assert(_the_table == NULL, "One string table allowed.");
    _the_table = new StringTable();
  }
  ...  // 省略若干方法

  // 查找(这个方法与Java中的Hashtable的get方法类似)
  static oop lookup(Symbol* symbol);
  static oop lookup(jchar* chars, int length);

  // 存入(这个方法与Java中的Hashtable的put方法类似)
  static oop intern(Symbol* symbol, TRAPS);
  static oop intern(oop string, TRAPS);
  static oop intern(const char *utf8_string, TRAPS);
  ...  // 省略若干方法
}
```

从源码可以看出：**StringTable也就是所谓的字符串常量池实际上就是一个Hashtable，而且被设计成单例，通过`lookup()`来查找元素，`intern()`方法来添加元素。**

### String类中的几个工具方法

对于String类中的许多成员方法，我们可能都非常熟悉。

`substring`获取子串（截取字符串中的一部分），包含start，不包含end。

`startsWith`和`endsWidth`来判断字符串头部和尾部的字符串

`indexOf`从前往后进行字符串匹配，`lastIndexOf`从后往前进行字符串匹配

`trim`去除字符串两端的空白字符(空格、换行等空白字符，即ASCII码0~32[空格])

`split`字符串切分

`concat`字符串连接，并返回一个新字符串，与C语言中的`strcat`函数类似

`compareTo`用于字符串之间的比较(该方法实现自`Comparable`接口)，与C语言中的`strcmp`函数类似，按字母的编码顺序进行比较(字典序)，如果大于传入的字符串则返回值大于0，如果相等则返回值等于0，如果小于传入的字符串则返回值小于0

`equals`用于比较字符串内容是否相同(该方法重写自`Object`类)，相同返回true，不相同返回false

`compareToIgnoreCase`忽略大小写比较两个字符串

`equalsIgnoreCase`忽略大小写比较两字符串内容是否相同

`contentEquals`比较两个字符序列内容是否相同，传入参数只要是`CharSequence`接口的实现类即可

`getBytes`获取字符串编码的字节数组，可以指定字符集编码

`toLowerCase`转小写，`toUpperCase`转大写

`replace`和`replaceAll`用于替换子串(可以使用正则)

`replaceFirst`替换第一次出现(从前到后)的子串

`matches`字符串是否能匹配正则

...

上面的方法我们都经常使用，但是`String`类的一些静态方法可能会被我们忽略掉。

```java
// 这和C语言中的sprintf函数类似，用于将若干对象格式化成一个字符串
String format(String format, Object... args)
```

Java8中对`StringBuilder`类进行了一层封装——`StringJoiner`类，于是String类就多了个静态方法

> StringJoiner类真的很“鸡肋”，主要是为了1.8提供的函数式编程而设计的。

```java
String join(CharSequence delimiter, CharSequence... elements)
String join(CharSequence delimiter, Iterable<? extends CharSequence> elements)
```

## StringBuffer与StringBuilder

前面频繁提到了`StringBuffer`和`StringBuilder`这两个类，但很多初学者对这两个类区分不清：**StringBuffer是线程安全的(方法基本上都是`synchronized`同步方法)，StringBuilder是线程不安全的(没有使用`synchronized`进行同步)；正因如此StringBuilder在效率上比StringBuffer要高**。这一点上和`Vector`与`ArrayList`的关系是一样的。

![CharSequence](http://img-blog.csdn.net/20170623011629190?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 前面也提到Java中String的字符串连接就是使用StringBuilder实现的，但StringBuilder是Java5之后才添加的，在Java5之前都是使用StringBuffer类来实现的。Java中每次字符串连接都会重新创建一个StringBuilder类，所以不会出现多线程访问共享资源时发生的问题。

除了前面所说的StringBuffer与StringBuilder之间的区别比较重要外，它们的扩容策略也值得我们探究，这也可能会成为影响我们程序内存效率的因素。

> 它们的扩容策略是相同的

首先我们先看一下这两个类的构造方法：

```java
// StringBuilder
public StringBuilder() {
    super(16);
}
public StringBuilder(int capacity) {
    super(capacity);
}
public StringBuilder(String str) {
    super(str.length() + 16);
    append(str);
}
public StringBuilder(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```

```java
// StringBuffer
public StringBuffer() {
    super(16);
}
public StringBuffer(int capacity) {
    super(capacity);
}
public StringBuffer(String str) {
    super(str.length() + 16);
    append(str);
}
public StringBuffer(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```

> 它们的构造方法完全一致。
>
> 我们能通过传入`capacity`来设置它的初始容量，如果不指定则默认为16(一个char占2字节，就相当于32个字节)。如果我们能大致确定要连接的字符数量，最好还是指定一下它的初始容量，这样可以避免频繁的创建新数组和数组拷贝。

这两个类的扩容策略是相同的，扩容策略的实现都被封装在它们的基类`AbstractStringBuilder`中：

```java
    void expandCapacity(int minimumCapacity) {
        // 扩容策略为：newCapacity = value.length * 2 + 2
        int newCapacity = value.length * 2 + 2;
        if (newCapacity - minimumCapacity < 0)
            newCapacity = minimumCapacity;
        if (newCapacity < 0) {
            if (minimumCapacity < 0) // overflow
                throw new OutOfMemoryError();
            newCapacity = Integer.MAX_VALUE;
        }
        value = Arrays.copyOf(value, newCapacity);
    }
```
