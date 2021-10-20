---
title: 从ReentrantLock分析AQS的原理
date: 2017-04-23
categories: JAVA
---

毫不夸张的说[AQS(AbstractQueuedSynchronizer)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)可以说是JUC包的核心，Semaphore、ReentrantLock、ThreadPoolExecutor等各种工具都是基于AQS开发的，所以这篇文章从最简单的ReentrantLock着手来了解一下AQS的实现原理。

# ReentrantLock与synchronized关键字的对比

|          | ReentrantLock                        | synchronized                                                 |
| -------- | ------------------------------------ | ------------------------------------------------------------ |
| 实现原理 | 基于AQS实现                          | 基于JVM提供的monitorenter和monitorexit两个指令实现<br />具体指令实现由不同JVM提供，<br />比如OpenJDK中有重量级锁、轻量级锁、偏向锁的优化 |
| 释放方式 | 显示调用unlock()方法                 | 代码块结束后自动释放(monitorexit)                            |
| 锁类型   | 公平锁&非公平锁                      | 非公平锁                                                     |
| 灵活性   | 支持响应中断、超时、尝试获取锁       | 不灵活                                                       |
| 条件队列 | 支持多个条件队列<br />newCondition() | 只能关联一个条件队列                                         |
| 可重入性 | 可重入                               | 可重入                                                       |

示例代码：

```java
// **************************Synchronized的使用方式**************************
// 1.用于代码块
synchronized (this) {}
// 2.用于对象
synchronized (object) {}
// 3.用于方法
public synchronized void test () {}
// 4.可重入
for (int i = 0; i < 100; i++) {
	synchronized (this) {}
}

// **************************ReentrantLock的使用方式**************************
public void testLock () throw Exception {
	// 1.初始化选择公平锁、非公平锁
	ReentrantLock lock = new ReentrantLock(true);
	// 2.可用于代码块
	lock.lock();
	try {
		try {
			// 3.支持多种加锁方式，比较灵活; 具有可重入特性
			if(lock.tryLock(100, TimeUnit.MILLISECONDS)){ }
		} finally {
			// 4.手动释放锁
			lock.unlock()
		}
	} finally {
		lock.unlock();
	}
}
```

# ReentrantLock中的AQS

可以看早期的jdk中ReentrantLock的lock实现非常简单

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
}
```





```java
public class ReentrantLock implements Lock, java.io.Serializable {
  //...
    abstract static class Sync extends AbstractQueuedSynchronizer {
      //...
        abstract boolean initialTryLock();
      
        final void lock() {
            if (!initialTryLock())
                acquire(1);
        }
      //...
    }
	
		static final class NonfairSync extends Sync {

        final boolean initialTryLock() {
            Thread current = Thread.currentThread();
            if (compareAndSetState(0, 1)) { // first attempt is unguarded
                setExclusiveOwnerThread(current);
                return true;
            } else if (getExclusiveOwnerThread() == current) {
                int c = getState() + 1;
                if (c < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(c);
                return true;
            } else
                return false;
        }

        /**
         * Acquire for non-reentrant cases after initialTryLock prescreen
         */
        protected final boolean tryAcquire(int acquires) {
            if (getState() == 0 && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
    }
}

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
  //...
    public final void acquire(int arg) {
        if (!tryAcquire(arg))
            acquire(null, arg, false, false, false, 0L);
    }
  //...
}
```



https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
