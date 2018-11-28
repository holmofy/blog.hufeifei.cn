# 避免创建不必要的对象

**不必要的对象指的就是可重用的对象**，我觉得可重用的对象在这里以下几个含义：

**重复使用的状态不变的对象，可以重用。**

String以及基本类型的包装类的对象都属于**不可变对象**，它们的状态本身就不可变，重复使用的次数比较多的地方，就有必要将这些对象缓存起来重复利用。

最具有说明性的就是JDK中对[包装类提供了缓冲池](https://blog.csdn.net/Holmofy/article/details/73612591#%E5%8C%85%E8%A3%85%E7%B1%BB%E7%9A%84%E7%BC%93%E5%AD%98%E6%B1%A0)：JDK对Byte、Short、Integer、Long这些常用整形均提供了`[-128, 127]`共256缓冲对象(int缓存池大小可修改)，对Character类型缓存了`[0, 127]`共128个ASCII码缓冲对象，Boolean也缓存了TRUE和FALSE两个对象。缓存的这些对象都是最常用的，可以有效的避免自动拆装箱过程中创建很多不必要的对象。

**重复使用的同一状态的对象，可以重用**

EffectiveJava书中给出了一个例子：

```java
public class Person {
    private final Date birthDate;
  
    // 判断某人是否在生育高峰期出生
    public boolean isBabyBoomer() {
        Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
        gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0, 0);
        Date boomStart = gmtCal.getTime();
        gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0, 0);
        Date boomEnd = gmtCal.getTime();
        return birthDate.compareTo(boomStart) >= 0 
          && birthDate.compareTo(boomEnd) < 0;
    }
}
```

`isBabyBoomer`方法中起始时间和结束时间都是固定的，但是每次调用这个方法都会创建和之前状态相同的两个对象。我们完全可以把这两个表示时间段的对象存起来，没必要每次都创建。要知道创建对象申请内存也需要花费一段时间的，如果能避免这些重复对象的创建，性能上也能得到显著的提升。

```java
public class Person {
    private final Date birthDate;
    
    // 生育高峰期的时间段
    private static final Date BOOM_START;
    private static final Date BOOM_END;
    static {
        Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
        gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0, 0);
        BOOM_START = gmtCal.getTime();
        gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0, 0);
        BOOM_END = gmtCal.getTime();
    }
  
    public boolean isBabyBoomer() {
        return birthDate.compareTo(BOOM_START) >= 0
          && birthDate.compareTo(BOOM_END) < 0;
    }
}
```

**频繁使用，重新创建对象耗费资源较大的对象，可以重用**

最好的例子就是线程对象和数据库连接对象。线程相对普通对象更消耗系统资源

# 消除过期的对象引用

