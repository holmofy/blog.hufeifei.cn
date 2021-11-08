---
title: ArrayList与Vector
date: 2017-04-23
categories: JAVA
keywords:
- ArrayList vs. Vector
---

Vector作为JDK1.0开始就已经存在的元老级数据结构，在JDK的版本升级过程中可谓是修修补补，与JAVA1.2中新增的ArrayList这个后起之秀相比，Vector就显得有点赘余了。但是对于新手来说就很有可能将这两个类混淆使用，这里对这两个类进行区别(主要体现在扩容策略和线程安全上)。

<!-- more -->

##相同点
1. 都使用数组实现，提供的操作也基本一致
  在ArrayList与Vector中都有一个`` Object[] elementData``用来保存数据
  看看它们的方法：
  ![ArrayList](https://s.pc.qq.com/tousu/img/20211020/8038031_1634737887.jpg)![Vector](https://s.pc.qq.com/tousu/img/20211020/5865753_1634737934.jpg)
2. 都实现了List接口
  对于ArrayList来说List接口仿佛就是为它而设计的，而Vector类完全是为了迎合JAVA1.2中出现的List接口而去实现的，这也导致了Vector中的很多方法功能上是重复的。
```java
boolean add(E e);   -->   void addElement(E obj);
boolean remove(Object o);   -->   boolean removeElement(Object obj);
 void clear();   -->   void removeAllElements();
E get(int index);   -->   E elementAt(int index);
E set(int index, E element);   -->   void setElementAt(E obj, int index);
void add(int index, E element);   -->   void insertElementAt(E obj, int index);
E remove(int index);   -->   void removeElementAt(int index);
ListIterator<E> listIterator();   -->   Enumeration<E> elements();
```

##不同点
## 默认构造对象的初始容量不同
两个类都可以通过构造函数指定初始容量在ArrayList中也是很有讲究的，这个在扩容策略中讲。这里要说的是是无参构造函数函数调用时，创建的数据对象数组不相同。
1. Vector默认初始容量为10
```java
    public Vector() {
        this(10);
    }
```
2. ArrayList默认初始是空数组
```java
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

## 扩容策略
扩容策略主要表现grow这个方法上
1. Vector的扩容策略
```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;

    // capacityIncrement这个变量在构造Vector对象的时候指定，默认为0
    // 如果指定了capacityIncrement变量，扩容方式就是：
    // 		newCapacity = oldCapacity + capacityIncrement；
    // 未指定capacityIncrement，扩容方式为：
    // 		newCapacity = oldCapacity * 2 ;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
Vector可以通过构造方法来指定扩容增量（也可以叫扩容因子）
```java
    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }
    // 默认扩容增量为0，也就是说上面的grow方法默认的扩容策略为：
    // 		newCapacity = oldCapacity * 2 ;
    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }
```
2. ArrayList的扩容策略
  在讲ArrayList的扩容策略之前，先说说两个常见的场景
* 场景一
```java
ArrayList<Integer> list1 = new ArrayList<>();
list1.add(1);
list1.add(2);
list1.add(3);
```
* 场景二
```java
ArrayList<Integer> list2 = new ArrayList<>(0);
list2.add(1);
list2.add(2);
list2.add(3);
```
对于上面两种场景，ArrayList数组容量的扩容过程，每一步数组的大小是多少呢。带着这个问题看下面的代码可能效果会更好。
首先来看看扩容的有效代码：
```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;

    // ArrayList的扩容策略：newCapacity = oldCapacity * 3/2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
}
```
但是对于初始容量为0的ArrayList对象来说，怎么办 ``0 * 3/2`` 永远都是0。 这就要说道``ensureCapacityInternal``方法了。
```java
    private void ensureCapacityInternal(int minCapacity) {
        // 当发现初始数组是空数组，会取DEFAULT_CAPACITY和minCapacity之中的最大值，而DEFAULT_CAPACITY为10。
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 容量不足，需要扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 每次都会调用ensureCapacityInternal确保容量足够
        elementData[size++] = e;
        return true;
    }
```
这段代码就解释了场景一的扩容过程。
```java
// 创建对象的时候，数组是DEFAULTCAPACITY_EMPTY_ELEMENTDATA
ArrayList<Integer> list = new ArrayLis<>();
list.add(1);   // 添加第一个元素的时候会取10作为最小容量，从而数组容量直接从0跳到10
list.add(2);   // 添加第二个元素，容量足够，无需扩容，数组容量仍是10
```
对于场景二的构造方法直接指定初始容量的构造方法
```java
 public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // 这里的EMPTY_ELEMENTDATA也是空数组，
        // 但是和DEFAULTCAPACITY_EMPTY_ELEMENTDATA不是一个对象
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```
**注意EMPTY_ELEMENTDATA和DEFAULTCAPACITY_EMPTY_ELEMENTDATA虽然都是空数组，但是并不是同一个对象**
现在场景二就好理解了
```java
// 这时创建对象，指定的数组是EMPTY_ELEMENTDATA这个空数组
ArrayList<Integer> list2 = new ArrayList<>(0);  // 容量为0
list2.add(1);  // 因为0*3/2 < 0+1 ，所以此时容量为1
list2.add(2);  // 1*3/2 < 1+1 ，所以此时容量为2
list2.add(3);  // 2*3/2 = 3 ，所以此时容量为3
list2.add(4);  // 3*3/2 = 4  ,此时容量为4
list2.add(5);  // 4*3/2 = 6 > 5 ，此时容量为6
```
## Vector线程安全，ArrayList线程不安全
Vector的大部分方法都是线程同步的，相应的Vector的效率也就不如ArrayList，因此Vector更适合多线程的应用场景；ArrayList线程不安全，多个线程同时操作可能会出现问题，在单线程的情况下效率会比Vector更高，在多线程情况下可以使用``Collections.synchronizedList``方法进行包装，从而达到线程安全的目的。
