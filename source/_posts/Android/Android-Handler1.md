---
title: Android Handler消息机制源码分析——第一部分：Looper与MessageQueue
date: 2017-03-05
tags:
    - Android
    - Handler
categories: Android
description: Android Handler消息机制源码分析——第一部分:Looper与MessageQueue
---

由于在Android中，网络请求不能运行在主线程中，同时一些耗时的操作也不建议运行在主线程中。因此多线程以及线程间通信在Android中显得更为重要了，而安卓SDK中也提供给我们很多的多线程机制，譬如：Handler，AsyncTask，以及基于这些机制而来的IntentService，AsyncService，Loader等常用类，其中AsyncTask又是基于Java的Concurrent框架而来的，虽然有些人对AsyncTask抱有怨言，但后面的文章中我们也会以学习的态度对AsyncTask进行深入剖析。这里我们先来对常用的Handler做更深层次的了解。Handler机制可以说是Android开发过程中最常用到的线程通信机制，比如说Activity.runOnUiThread方法，View.post,View.postDelayed等方法底层使用的都是该机制。但是在分析源码之前我们先来对整个框架机制中的几个相关类简单的说明一下。

## Handler机制相关类的概括
### Handler
首先要提到的就是Handler整个框架的上层建筑，也就是我们常用到的Handler类。Handler类可以说是整个框架中最外层的类，从整个过程的开始（消息的发送）到整个过程的终结（消息的处理），两端的事件都在Handler类中进行。这里面有几个重要方法分别是
```java
void handleMessage(Message msg)
void dispatchMessage(Message msg)
boolean sendMessageAtTime(Message msg, long uptimeMillis)
```
### Message
前面说到消息，这里就不得不先介绍代表消息的Message类了，这个类中存储着消息携带的所有信息，以及消息需要交给谁处理，甚至消息怎么处理的都在这个对象中保存着。这个类重要的主要是一下它的成员变量。其中最重要的几个分别是：
```java
/*package*/ long when;		//代表消息何时处理
/*package*/ Handler target;	//消息给谁处理
/*package*/ Message next;	//指向消息队列(MessageQueue)的下一个消息
```
### MessageQueue
中文名翻译过来就是消息队列，看到这个名字让我兴奋的想起Windows程序设计中需要理解透彻的消息循环机制，苦于Windows闭源，然而今天终于可以通过Android的消息机制一窥其端倪了。所谓消息队列，其实Android使用的是Message单链表来实现消息队列。Message类中的next成员变量就是用来连接Message的链条的。
### Looper
Looper是整个框架的核心类，相当于汽车的发动机，Looper.loop方法不断的将当前线程MessageQueue中的Message分发出去(也就是调用msg.target.dispatchMessage方法交给Handler处理)。

## 源码摘要分析
下面我们对该机制部分重点核心源码进行分析，首先分析整个框架的核心类Looper

***下面源码中我会用区间注释来标注哪些是该机制的重点部分***
## Looper
```java
/**
 * Looper是个final类，所以你无法继承并重载它的方法
 */
public final class Looper {
    private static final String TAG = "Looper";

    //////////////////////////////////////////////////////////////////////////////////////////
    //该静态成员中保存了所有用到Handler消息机制的线程的Looper对象
    //ThreadLocal是java.lang包中的类，具体内容可以查看JDK源码
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    //另外单独搞一个成员变量来保存主线程的Looper
    private static Looper sMainLooper;

    /////////////////////////////////////////////////////////////////////////////////////////
    //消息队列
    final MessageQueue mQueue;
    final Thread mThread;

    private Printer mLogging;

      /***********************************************************************************
      * 初始化当前线程的Looper
      * 线程中要创建Handler必须先创建该线程的Looper对象，因为Handler需要当前线程Looper的引用
      * 正确的调用顺序为Looper.prepare->Looper.loop->Looper.quit
      ***********************************************************************************/
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        //如果当前线程已经创建了Looper，那么将抛出异常
        //也就是说一个线程只能有一个Looper对象，Looper.prepare()只能调用一次
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //如果当前线程没有Looper则，创建一个并保存在当前线程的局部变量中。
        sThreadLocal.set(new Looper(quitAllowed));
    }

    /***********************************************************************************
     * 初始化主线程的Looper，该方法在Android系统的入口类
     * ActivityThread类中已经被调用了，所以开发者不需要调用这个方法
     ***********************************************************************************/
    public static void prepareMainLooper() {
        //false代表主线程Looper的消息队列MessageQueue不允许退出
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            //将主线程Looper单独保存在sMainLooper静态变量中
            sMainLooper = myLooper();
        }
    }

    /**
     * 返回应用主线程的Looper
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    /***********************************************************************************
     * 运行当前线程的Looper(也就是消息队列开始遍历)，
     * 调用该方法后一定要调用quit结束消息循环
     ***********************************************************************************/
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            //调用looper前必须调用Looper.prepare()来创建当前线程的Looper对象
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        //Binder机制涉及native层(C/C++)的进程间通讯(IPC)，这里暂时不深入探讨
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

	//无限消息循环
        for (;;) {
            //消息出队//next方法可能会阻塞(类似Windows程序设计中GetMessage)
            Message msg = queue.next();
            if (msg == null) {
                //没有消息说明当前线程的消息队列MessageQueue正在退出
                return;
            }

            // 打日志
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            //将消息分发给msg.target(实际上就是Handler)
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // 确保分发过程中，线程的标识ID没有错
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            //回收消息(置0的置0，归null的归null)
            msg.recycleUnchecked();
        }
    }

    /**
     * 分会当前线程的Looper对象，如果当前线程没有相对应的Looper对象则返回null
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

    public void setMessageLogging(Printer printer) {
        mLogging = printer;
    }

    /**
     * 返回当前线程相对应的消息队列MessageQueue对象，
     * 调用该方法前如果没有初始化Looper，则发生空指针异常(不会返回null)
     * 因为这里返回的就是当前线程Looper.mQueue成员变量
     */
    public static MessageQueue myQueue() {
        return myLooper().mQueue;
    }

    /**
     * 注意这里的构造方法是私有的，
     * 也就是说开发者只能调用Looper.prepare方法创建Looper对象
     */
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    /**
     * 检查Looper对象是否为当前线程的Looper
     * @hide AndroidSDK 隐藏了该API，开发者一般无法调用(当然这里只是说一般情况)
     */
    public boolean isCurrentThread() {
        return Thread.currentThread() == mThread;
    }

    /**
     * 退出Looper消息循环
     * 调用该方法将终止loop方法，Looper不再处理消息队列中的任何消息
     *
     * 调用quit方法后发送的任何消息都将发送失败
     * 比如说Handler.sendMessage等方法将返回false
     *
     * 注意该方法是不安全，因为Looper终止后消息队列中还有消息没有分发出去，
     * 因此最好使用quitSafely方法来代替，以确保所有将发生的工作有序完成。
     *
     * @see #quitSafely
     */
    public void quit() {
        mQueue.quit(false);
    }

    /**
     * Looper安全退出循环
     * 调用该方法后，Looper会将消息队列中已经存在的剩余消息分发并处理掉。
     * 但是，Looper终止后，余下的延迟消息(DelaydMessage)将不被分发出去
     *
     * 调用quit方法后发送的任何消息都将发送失败
     * 比如说Handler.sendMessage等方法将返回false
     */
    public void quitSafely() {
        mQueue.quit(true);
    }

    /**
     * 发送一个同步Barrier拦截器到消息队列中
     *
     * 消息队列运行遇到同步拦截器时，同步消息将会暂停执行，
     * 直到拦截器被释放，也就是调用removeSyncBarrier方法(该方法中需要指定拦截器的标识ID)
     *
     * 该方法会立即拦截当前时间后的同步消息的执行，直到满足一定条件后释放该拦截器
     * 但是异步消息不受拦截器的阻碍继续执行
     *
     * 该方法必须与removeSysncBarrier配套调用，该方法返回的int作为Barrier拦截器的标识
     * 在removeSyncBarrier传入该int标识来释放拦截器让消息队列恢复正常运作。
     * 如果没有移除相应的Barrier拦截器，可能会导致应用崩掉
     *
     * @hide AndroidSDK 隐藏了该API，开发者一般无法调用
     */
    public int postSyncBarrier() {
        return mQueue.enqueueSyncBarrier(SystemClock.uptimeMillis());
    }


    /**
     * 移除消息队列中的同步Barrier拦截器
     *
     * @int 这个int值必须是postSyncBarrier返回的
     *
     * @throws IllegalStateException 没有找到int值对应的Barrier则抛出异常
     *
     * @hide 隐藏API
     */
    public void removeSyncBarrier(int token) {
        mQueue.removeSyncBarrier(token);
    }

    /**
     * 返回Looper对应的线程对象
     */
    public Thread getThread() {
        return mThread;
    }

    /**获取当前线程的消息队列**/
    /** @hide */
    public MessageQueue getQueue() {
        return mQueue;
    }

    /**
     * 该Looper的线程是否是空闲的(是否处在等待状态)
     * 该方法的返回值其实是不确定性的，可能得到返回值后，Looper立马就有了消息需要处理
     * @hide
     */
    public boolean isIdling() {
        return mQueue.isIdling();
    }

    public void dump(Printer pw, String prefix) {
        pw.println(prefix + toString());
        mQueue.dump(pw, prefix + "  ");
    }

    public String toString() {
        return "Looper (" + mThread.getName() + ", tid " + mThread.getId()
                + ") {" + Integer.toHexString(System.identityHashCode(this)) + "}";
    }
}
```


## MessageQueue
```java
package android.os;

import android.util.Log;
import android.util.Printer;

import java.util.ArrayList;

/**
 * 这是一个底层类，用来保存Message消息列表，并通过Looper来分发出去
 * 消息是不能被直接添加到MessageQueue消息队列中
 * 需要通过与当前线程Looper相关联的Handler对象来添加消息
 *
 * 注意：MessageQueue是按照消息的时间进行排序的
 * <p>可以通过Looper.myQueue方法来获取当前线程的消息队列(不过该API已经被标记为hide了)
 */
public final class MessageQueue {
    // 标识该消息队列能否被quit退出
    private final boolean mQuitAllowed;

    @SuppressWarnings("unused")
    private long mPtr; // 该属性主要用在Native层(C/C++)作为指针

    ////////////////////////////////////////////////////////////////////////////////////////////////////
    //该属性是MessageQueue这个链表的头结点，Message中next来连接下一个结点，
    //注意：消息队列按时间排序的
    Message mMessages;

    //为了避免线程处理完所有的消息后等待新消息，
    //或者剩下的消息还没到分发处理的时候，
    //线程一直处于等待状态而浪费线程资源，
    //所以可以添加线程空闲时的处理方法，当线程空闲后会回调IdleHandler的接口方法
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    private IdleHandler[] mPendingIdleHandlers;
    private boolean mQuitting;//标识是否正在退出

    // 标识next方法是否阻塞等待nativePollOnce方法返回重新激活
    private boolean mBlocked;

    // Barriers are indicated by messages with a null target whose arg1 field carries the token.
    private int mNextBarrierToken;

    //C/C++的native层方法
    private native static long nativeInit();					// 消息队列的初始化
    private native static void nativeDestroy(long ptr);		// 消息队列资源释放
    private native static void nativePollOnce(long ptr, int timeoutMillis); //消息队列等待（类似于Java的Object.wait）
    private native static void nativeWake(long ptr);               // 消息队列的唤醒 (类似于Java的Object.notify )
    private native static boolean nativeIsIdling(long ptr);     // 消息队列是否闲置

    /**
     * 前面提到的回调接口，用于碰到当线程将要阻塞等待新消息的时候回调
     */
    public static interface IdleHandler {
        /**
         * 当消息队列处理完了消息，将要等待其他更多消息入队。
         * 这个时候线程将进入空闲状态，如果MessageQueue添加了空闲处理，则该方法会回调。
         *
         * 如果返回true表示继续让该IdleHandler保持活跃(再遇到空闲仍会回调)，false表示将其移除。
         * 如果消息队列中仍有消息，但这些消息的处理时间还没到，该方法也可能被回调
         */
        boolean queueIdle();
    }

    /**
     * 添加新的IdleHandler到消息队列中，如果IdleHandler.queueIdle方法返回false
     * 则这个IdleHandler将会自动从消息队列中移除，当然你也可以调用removeIdleHandler
     * 手动移除。
     * 注意：该方法在任何线程中调用都是线程安全的，
     * 所以在其他线程中也可以添加该线程的空闲处理对象
     */
    public void addIdleHandler(IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    /**
     * 移除IdleHandler
     */
    public void removeIdleHandler(IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }

    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }

    @Override
    protected void finalize() throws Throwable {
        try {
            dispose();
        } finally {
            super.finalize();
        }
    }

    // 通过调用nativeDestroy来销毁native层的MessgaeQueue
    private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }

    /***********************************************************************************
     * 消息出队
     ***********************************************************************************/
    Message next() {
        // 已经退出消息循环并销毁队列的资源后，这里直接返回。
        // 消息循环退出后native层的资源已经销毁了，所以Looper是不允许重新启动的
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);  //消息等待，类似于Java的Object.wait()方法

            synchronized (this) {
                // 获取当前时间，与Message中的时间进行比较，从而得出该消息是否能够被处理
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // msg.target为null说明该msg是个SyncBarrier同步消息拦截器
                    // (详细内容可以看enqueueSyncBarrier方法)
                    // 同步消息被拦截了，那就查找异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 如果下一个消息还没准备好，则计算延时(准备好后再唤醒它)
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 说明当前消息需要在此时处理
                        mBlocked = false;

                        //需要处理，则进行出队操作
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // 说明消息队列没有消息
                    nextPollTimeoutMillis = -1;
                }

                // 所有的消息处理完了，查看是否有退出消息(也就是是否调用过quit方法)
                // 如果有退出消息，则销毁资源返回null后开始退出
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 只有当队列为空或者第一个消息还没到处理的时候，IdleHanlder才会被调用
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // 如果没有设置IdleHandler空闲处理，则继续循环等待新消息
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();   //进行空闲处理
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                //如果queueIdle返回false，则将其移除
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
    /***********************************************************************************
     *消息队列的退出
     ***********************************************************************************/
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {//已经调用过quit了
                return;
            }
            mQuitting = true;

            if (safe) {
                //如果是安全退出，则删除当前时间后的所有消息
                removeAllFutureMessagesLocked();
            } else {
                //如果不是安全退出，则直接删除所有消息(这可能会导致有些必须处理的消息没有被处理)
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }

    //同步消息拦截(就是放一个target为null的空处理消息进去)
    int enqueueSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }

    //移除同步消息拦截器
    void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }

    /***********************************************************************************
     * 消息入队
     ***********************************************************************************/
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            //所有的消息必须要有处理对象，当然除了调用enqueueSyncBarrier入队的同步消息拦截器
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                // 调用quit方法后，消息无法入队，返回false入队失败
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                //原来的消息队列为空||新消息的时间为0(也就是立即执行)||新消息的执行时间比原来的消息都早
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                //否则的话需要把新消息插到队列中间

                // 通常新消息插到中间不需要唤醒队列，但如果该消息是拦截器等待的异步消息就不一样啦
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    //将消息插入到相应时间的合适位置
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr); // 唤醒MessageQueue
            }
        }
        return true;
    }

    // 是否符合条件的消息
    boolean hasMessages(Handler h, int what, Object object) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.what == what && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
    // 是否符合条件的消息
    boolean hasMessages(Handler h, Runnable r, Object object) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.callback == r && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
    // 消息队列是否闲置
    boolean isIdling() {
        synchronized (this) {
            return isIdlingLocked();
        }
    }

    private boolean isIdlingLocked() {
        // If the loop is quitting then it must not be idling.
        // We can assume mPtr != 0 when mQuitting is false.
        return !mQuitting && nativeIsIdling(mPtr);
     }

    //移除指定条件的消息(下面的我就不逐行解释了，无非是循环遍历，然后删除结点)
    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
    //移除指定条件的消息
    void removeMessages(Handler h, Runnable r, Object object) {
        if (h == null || r == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.callback == r
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    //移除指定条件的消息
    void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
    //删除所有的消息
    private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }

    //移除当前时间之后的所有消息(移除未来消息)
    private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {
                removeAllMessagesLocked();
            } else {
                Message n;
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }

    void dump(Printer pw, String prefix) {
        synchronized (this) {
            long now = SystemClock.uptimeMillis();
            int n = 0;
            for (Message msg = mMessages; msg != null; msg = msg.next) {
                pw.println(prefix + "Message " + n + ": " + msg.toString(now));
                n++;
            }
            pw.println(prefix + "(Total messages: " + n + ", idling=" + isIdlingLocked()
                    + ", quitting=" + mQuitting + ")");
        }
    }
}
```
