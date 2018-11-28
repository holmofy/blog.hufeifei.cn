---
title: Java多线程复习与巩固(五)--生产者消费者问题（第一部分）
date: 2017-06-26 14:54
categories: JAVA
---

# 生产者消费者问题（第一部分）

**生产者消费者问题**也称为**有限缓冲问题**，是线程同步的一个经典问题：生产者线程和消费者线程共享一块固定大小的缓存，生产者负责生成产品然后存入共享缓冲区中，消费者负责从共享缓冲区中取出产品进行消费。该问题的关键在于生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区空时消耗数据。

要解决这个问题就必须：**让生产者在缓冲区满时休眠，等到下次消费者消耗缓冲区中的数据的时候，生产者才能被唤醒，开始往缓冲区添加数据。同样地，让消费者在缓冲区空时进入休眠，等到生产者往缓冲区添加数据之后，再唤醒消费者。**

> 解决生产者消费者问题的方法有很多，这里先介绍最简单的一种，后续的文章中会陆续给出其他的解决方案

## Object的`wait`和`notify`方法

`Object.wait()`和`Thread.sleep()`方法在功能上很相似，它们都会导致线程挂起，但是`Thread.sleep()`可以指定线程被挂起的时间，当然`Object.wait()`也有一个重载的方法也可以指定被挂起的时间，可`Thread.sleep()`挂起时不会释放线程占有的资源(不会释放锁)，而`Object.wait()`会暂时释放线程所占有的资源(会释放锁)。因此`Object.wait()`调用后其他线程就可以进入`synchronized`同步代码块执行了。

而`Object.notify()`就是用来唤醒因调用`Object.wait()`而挂起的一个线程，另外还有一个`Object.notifyAll()`方法用来唤醒所有因调用`Object.wait()`而挂起的线程。

使用`Object.wait()`方法和`notifyAll()`方法来实现线程的休眠和唤醒。

```java
import java.util.Random;

public class ProducerConsumer {
    private static final int BUFFER_SIZE = 100;
    static int[] buffer = new int[BUFFER_SIZE];
    static int head, tail = 0;
    static int count = 0;

    static class Producer implements Runnable {
        Random random = new Random();

        public void run() {
            while (true) {
                synchronized (buffer) {
                    while (count >= buffer.length) {
                        // 外层需要套一个while循环，
                        // 以为buffer.wait()可能会被错误的唤醒
                        // 如果缓冲区已满则等待
                        try {
                            buffer.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 生成一个随机数作为生产的产品
                    int product = random.nextInt(10);
                    System.out.println("Producer: 我生产了一个随机数" + product);
                    // 将产品放入共享缓冲区中
                    buffer[tail] = product;
                    // 尾部指针加一
                    tail = (tail + 1) % buffer.length;
                    count++;
                    // 提醒消费者消费
                    buffer.notifyAll();
                }
            }
        }
    }

    static class Consumer implements Runnable {
        public void run() {
            while (true) {
                synchronized (buffer) {
                    while (count <= 0) {
                        try {
                            buffer.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    // 取出共享缓冲区中的产品
                    int product = buffer[head];
                    head = (head + 1) % buffer.length;
                    count--;
                    System.out.println("Consumer: 我消费了一个随机数" + product);
                    buffer.notifyAll();
                }
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
    }
}
```

## 对共享缓冲区进行封装

> 操作系统中有个概念叫管程(Monitors)，它是这么定义的：一个管程定义了一个数据结构和能为并发进程所执行（在该数据结构上）的一组操作，这组操作能同步进程和改变管程中的数据。

管程这个概念仿佛与我们下面的封装不谋而合。

```java
import java.util.Random;

public class ProducerConsumer {
    // 你也可以将这个类设计成泛型，让它有通用性
    static class Buffer {
        private int[] buffer;
        private int head = 0, tail = 0;
        private int count = 0;

        public Buffer(int size) {
            buffer = new int[size];
        }

        public synchronized void put(int data) {
            while (count >= buffer.length) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            buffer[tail] = data;
            tail = (tail + 1) % buffer.length;
            count++;
            notifyAll();
        }

        public synchronized int take() {
            while (count <= 0) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            int data = buffer[head];
            head = (head + 1) % buffer.length;
            count--;
            notifyAll();
            return data;
        }
    }

    static Buffer buffer = new Buffer(10);

    static class Producer implements Runnable {
        Random random = new Random();

        public void run() {
            while (true) {
                // 生成一个随机数作为生产的产品
                int product = random.nextInt(100);
                buffer.put(product);
                System.out.println("Producer: 我生产了一个随机数" + product);
            }
        }
    }

    static class Consumer implements Runnable {
        public void run() {
            while (true) {
                int product = buffer.take();
                System.out.println("Consumer: 我消费了一个随机数" + product);
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new Producer()).start();
        new Thread(new Consumer()).start();
    }
}
```

在`java.util.concurrent`包中有很多类似于上面的Buffer的数据结构，不同的是它们大都使用并发库中的`ReentrantLock`实现线程的互斥访问。通常它们都是`BlockingQueue`接口的实现类，`BlockingQueue.put()`和`BlockingQueue.take()`方法和上面的我写的例子中的`Buffer.put()`和`Buffer.take()`方法基本类似，不同之处是`BlockingQueue.put()`和`BlockingQueue.take()`把`InterruptedException`抛出来交给外部处理。

在后续的文章中我们会对ReentrantLock和这一系列BlockingQueue进行简单的使用和原理分析。

> 关于java.util.concurrent包中的集合类可以参考我的这篇文章[Java 集合框架总结与巩固](http://blog.csdn.net/holmofy/article/details/71215548)

