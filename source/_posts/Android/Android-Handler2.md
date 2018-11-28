---
title: Android Handler消息机制源码分析——第二部分：Message与Handler
date: 2017-03-07 09:28
tags:
    - Android
    - Hanlder
categories: Android
description: Android Handler消息机制源码分析——第二部分 Message与Handler
---
Android Handler消息机制源码分析——第二部分：Message与Handler

# Message
```java
/**
 * Message代表了一个消息，消息中包含了描述信息与若干数据，
 * Message主要通过Handler对象进行发送。
 * Message中有两个int字段arg1和arg2以及Object类型的obj字段供你使用
 * 你可以在这里设置一些额外的信息。
 * 如果要进行其他的数据传出可以使用setData()方法，
 * data是Bundle类型的字段，你可以在这个字段中设置你想要传输的数据
 *
 * 注意：虽然Message的构造方法是共有的，
 * 但是最好还是使用Message.obtain()方法来获取Message对象
 * 因为使用该方法可以复用Message回收池中的Message对象
 */
public final class Message implements Parcelable {
    /**
     * 用户自定义消息码，用来标识每个信息。
     * 一个消息队列可能处理多个Handler发出的消息，
     * 如果每个Handler对自己的消息都有自己的命名规则，
     * 就可以很好的避免与其他Handler发送的消息标识产生冲突
     */
    public int what;

    /**
     * arg1和arg2这两个字段能够用来保存一些简单的信息
     * 如果你想要传递更多的数据最好使用setData(Bundle)方法，
     * 如果是简单的数据使用arg1和arg2就足够了，可以避免创建对象(Bundle还是蛮耗资源的)
     */
    public int arg1;

    public int arg2;

    /**
     * 你想发送给接受者的任意对象，
     * 当使用Messenger发送跨进程的消息时，
     * 那这个对象还需要实现Parcelable接口了
     * 对于其他更复杂的数据最好还是使用setData(Bundle)进行传递
     *
     * <p>注意：在Android 2.2之前还不支持Parcelable接口
     */
    public Object obj;

    /**
     * Messenger涉及跨进程通信，这里不做深入研究
     * Optional Messenger where replies to this message can be sent.  The
     * semantics of exactly how this is used are up to the sender and
     * receiver.
     */
    public Messenger replyTo;

    /**
     * 这是一个可选字段，也是用来标识发送的信息。
     * 这个字段只在使用Messenger跨进程通信才会使用，其他情况下它都是-1
     */
    public int sendingUid = -1;

    /** If set message is in use.
     * This flag is set when the message is enqueued and remains set while it
     * is delivered and afterwards when it is recycled.  The flag is only cleared
     * when a new message is created or obtained since that is the only time that
     * applications are allowed to modify the contents of the message.
     *
     * It is an error to attempt to enqueue or recycle a message that is already in use.
     */
    /*package*/ static final int FLAG_IN_USE = 1 << 0;       //标志位：标识消息正在使用中

    /** If set message is asynchronous */
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1; //标志位：标识该消息是异步消息

    /** Flags to clear in the copyFrom method */
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

    /*package*/ int flags;	//存储上面的几个标志位

    /*package*/ long when;  //消息需要在何时处理

    /*package*/ Bundle data;  //消息所携带的数据，更直接的说这应该是个数据包(小数据没必要存在这里)

    /*package*/ Handler target; //消息的处理者

    /*package*/ Runnable callback; //消息的回调(有这个字段通常就不会去执行target的handleMessage方法)

    // 存储下一个消息(这就是MessageQueue的消息队列的存储机制，实际上就是个单链表)
    /*package*/ Message next;

    // 这几个字段是Message的回收池，
    // 或者叫做缓冲池(这就是建议使用Message.obtain方法获取Message的原因)
    private static final Object sPoolSync = new Object();
    private static Message sPool;          //缓冲池事实上也是使用链表实现的
    private static int sPoolSize = 0;

    private static final int MAX_POOL_SIZE = 50;

    private static boolean gCheckRecycle = true;

    //////////////////////////////////////////////////////////////////////////////////////////////
    /**
     * 从全局的缓冲池中创建一个Message对象，
     * 使用该方法避免了重复创建Message对象
     * 使用全局缓冲池达到对象复用的目的
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {    //取出缓冲池链表中的第一个Message对象
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    /**
     * 将传入的Message对象信息拷贝到缓冲池中的一个对象中
     */
    public static Message obtain(Message orig) {
        Message m = obtain();
        m.what = orig.what;
        m.arg1 = orig.arg1;
        m.arg2 = orig.arg2;
        m.obj = orig.obj;
        m.replyTo = orig.replyTo;
        m.sendingUid = orig.sendingUid;
        if (orig.data != null) {
            m.data = new Bundle(orig.data);
        }
        m.target = orig.target;
        m.callback = orig.callback;

        return m;
    }
    /**
     * 下面的obtain功能类似
     */
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;

        return m;
    }
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;

        return m;
    }
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }
    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;

        return m;
    }

    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;

        return m;
    }
    public static Message obtain(Handler h, int what,
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;

        return m;
    }
    //////////////////////////////////////////////////////////////////////////////////////////////

    /** @hide */
    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;
        }
    }

    /**
     * 将消息回收进缓冲池中
     * 正在使用的Message无法回收(也就是已经入队的消息或正在处理的消息)
     */
    public void recycle() {
        if (isInUse()) {   //正在使用无法回收
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    /**
     * 不检查是否正在使用，直接回收该消息
     * 这个是包内函数主要供MessageQueue和Looper在处理完消息后使用
     */
    void recycleUnchecked() {
        // 数据归零
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }

    /**
     * 从指定的Message中拷贝到该Message，对于Bundle类型的data字段执行的是浅拷贝
     * 没有拷贝o对象的when时间戳和target、callback字段
     */
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }

    /**
     * 返回该消息的处理时间，毫秒为单位的时间戳
     */
    public long getWhen() {
        return when;
    }

    public void setTarget(Handler target) {
        this.target = target;
    }

    /**
     * 获取这个消息的处理对象也就是Handler的实现类
     * 对象必须要实现handleMessage(Message)方法.
     */
    public Handler getTarget() {
        return target;
    }

    /**
     * 获取回调接口，当该消息被处理的时候该回调接口会被调用
     */
    public Runnable getCallback() {
        return callback;
    }

    /**
     * 获取存储数据的Bundle对象，这个对象是通过setData方法设置的。
     * 注意：当通过Messenger进行跨进程数据传输时，
     * 你还需要通过调用setClassLoader(ClassLoader)方法为你的Bundle设置类加载器
     * 类加载器主要是用来在其他进程来实例化Bundle中的对象
     * @see #peekData()
     * @see #setData(Bundle)
     */
    public Bundle getData() {
        if (data == null) {
            data = new Bundle();
        }

        return data;
    }

    /**
     * 与getData()不同，这个可能返回null
     * @see #getData()
     * @see #setData(Bundle)
     */
    public Bundle peekData() {
        return data;
    }

    /**
     * Sets a Bundle of arbitrary data values. Use arg1 and arg2 members
     * as a lower cost way to send a few simple integer values, if you can.
     * @see #getData()
     * @see #peekData()
     */
    public void setData(Bundle data) {
        this.data = data;
    }

    /**
     * 可以通过该方法来指定Message的处理者
     */
    public void sendToTarget() {
        target.sendMessage(this);
    }

    /**
     * 判断该消息是否为异步消息，这里涉及到上一篇MessageQueue中讲到的同步消息拦截器(SyncBarrier)。
     * SyncBarrier可以拦截同步消息，所以可以通过setAsynchronous方法将消息设置为异步的。
     *
     *这个方法是用来查看消息是否为异步的。
     * @hide
     */
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    /**
     * 是否将将消息设置为异步的
     * 对于为什么要异步消息，可以查看上篇MessageQueue.enqueueSyncBarrier方法与next方法
     * @hide
     */
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }

    //消息是否正在使用中(消息已在消息队列中或消息正在处理)
    /*package*/ boolean isInUse() {
        return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
    }

    /*package*/ void markInUse() {
        flags |= FLAG_IN_USE;
    }

    /** Message构造方法(但是还是建议使用Message.obtain()方法来获取Message对象)
    */
    public Message() {
    }

    @Override
    public String toString() {
        return toString(SystemClock.uptimeMillis());
    }

    String toString(long now) {
        StringBuilder b = new StringBuilder();
        b.append("{ when=");
        TimeUtils.formatDuration(when - now, b);

        if (target != null) {
            if (callback != null) {
                b.append(" callback=");
                b.append(callback.getClass().getName());
            } else {
                b.append(" what=");
                b.append(what);
            }

            if (arg1 != 0) {
                b.append(" arg1=");
                b.append(arg1);
            }

            if (arg2 != 0) {
                b.append(" arg2=");
                b.append(arg2);
            }

            if (obj != null) {
                b.append(" obj=");
                b.append(obj);
            }

            b.append(" target=");
            b.append(target.getClass().getName());
        } else {
            b.append(" barrier=");
            b.append(arg1);
        }

        b.append(" }");
        return b.toString();
    }

    // 下面都是Parcelable接口的实现，这里就不再往下解释了
    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };

    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest);
        dest.writeInt(sendingUid);
    }

    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source);
        sendingUid = source.readInt();
    }
}
```

# Handler
```java
/**
 * Handler是发送和处理消息的地方，也就是整个Handler框架的头跟尾。
 * 在一个线程中你可以创建多个Handler对象，
 * 每个Handler示例对象都关联了该线程的Looper和MessageQueue。
 * Handler在创建的时候就绑定了它所在的线程的MessageQueue(不需要开发者绑定)
 *
 * <p>发送消息主要一下两类：
 * (1) 发送一个Runnable在指定的时间执行
 * (该方法事实上是通过调研那个getPostMessage方法将Runnable封装到Message对象中)
 * boolean post(Runnable r)
 * boolean postAtTime(Runnable r, long uptimeMillis)
 * boolean postDelayed(Runnable r, long delayMillis)
 * boolean postAtFrontOfQueue(Runnable r)
 * (2) 发送一个消息在指定的时候分发到Handler.handleMessage(Message)方法中
 * boolean sendMessage(Message msg)
 * boolean sendEmptyMessage(int what)
 * boolean sendEmptyMessageDelayed(int what, long delayMillis)
 * boolean sendEmptyMessageAtTime(int what, long uptimeMillis)
 * boolean sendMessageDelayed(Message msg, long delayMillis)
 * boolean sendMessageAtTime(Message msg, long uptimeMillis)
 *
 * 以上两中类型的方法最终都会辗转调用最后那个方法sendMessageAtTime
 * 不同的是前者回调的是Message.callable.run方法(也就是指定的Runnable对象的run方法)
 * 后者回调的是Message.target.handleMessage方法(也就是开发者继承Handler后重载的handleMessage)
 */
public class Handler {
    /*
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
    private static final String TAG = "Handler";

    /**
     * 类似于代理模式
     * Handler与Callback的关系类似于Thread与Runnable的关系
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }

    /**
     * 处理消息的方法
     * 该方法就是子类需要实现的方法
     */
    public void handleMessage(Message msg) {
    }

    /**
     * 消息分发
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            //调用Message.callback.run方法(这就是post类型的方法处理入口)
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    // 这里mCallback就是前面的Callback接口
                    return;
                }
            }
            handleMessage(msg); //最后调用Handler.handleMessage方法
        }
    }

    /**
     * 默认构造函数
     */
    public Handler() {
        this(null, false);
    }

    /**
     * 看了dispatchMessage方法，这个构造就很容易理解
     */
    public Handler(Callback callback) {
        this(callback, false);
    }

    /**
     * 开发者提供一个Looper从而替换默认的Looper
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }

    /**
     * Handler默认是同步的，可以通过该构造修改默认是否默认异步。
     * 异步的Handler发送每个消息时会调用Message.setAsynchronous(boolean)设置消息为异步
     * 异步消息的作用主要是不受SyncBarrier同步拦截器的影响
     *
     * @hide
     */
    public Handler(boolean async) {
        this(null, async);
    }

    /**
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();  //获取当前线程的Looper(默认Looper)
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    /**
     * @hide
     */
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    public String getMessageName(Message message) {
        if (message.callback != null) {
            return message.callback.getClass().getName();
        }
        return "0x" + Integer.toHexString(message.what);
    }
    //////////////////////////////////////////////////////////////////////////////////////////////
    //obtainMessage都是间接调用Message.obtain(对象复用的功能前面也说过了)
    public final Message obtainMessage()
    {
        return Message.obtain(this);
    }

    public final Message obtainMessage(int what)
    {
        return Message.obtain(this, what);
    }

    public final Message obtainMessage(int what, Object obj)
    {
        return Message.obtain(this, what, obj);
    }

    public final Message obtainMessage(int what, int arg1, int arg2)
    {
        return Message.obtain(this, what, arg1, arg2);
    }

    public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
    {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
    //////////////////////////////////////////////////////////////////////////////////////////////

    //////////////////////////////////////////////////////////////////////////////////////////////
    // post类型-->Runnable填充Message.callback字段
    // post方法最终会辗转调用 sendMessageAtTime(Message msg, long uptimeMillis) 方法
    // post类型与send类型的区别可以参考dispatchMessage方法
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    public final boolean postAtTime(Runnable r, long uptimeMillis)
    {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    public final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
    {
        // getPostMessage来填充Message.callback字段
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }

    public final boolean postDelayed(Runnable r, long delayMillis)
    {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }

    public final boolean postAtFrontOfQueue(Runnable r)
    {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
    //////////////////////////////////////////////////////////////////////////////////////////////
    public final boolean runWithScissors(final Runnable r, long timeout) {
        if (r == null) {
            throw new IllegalArgumentException("runnable must not be null");
        }
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout must be non-negative");
        }

        if (Looper.myLooper() == mLooper) {
            r.run();
            return true;
        }

        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }

    public final void removeCallbacks(Runnable r)
    {
        mQueue.removeMessages(this, r, null);
    }

    public final void removeCallbacks(Runnable r, Object token)
    {
        mQueue.removeMessages(this, r, token);
    }
    //////////////////////////////////////////////////////////////////////////////////////////////
    // send方式的方法最终都辗转调用sendMessageAtTime(Message msg, long uptimeMillis)方法
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis); //消息入队
    }

    //将消息做为消息队列的头结点
    public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

    ////////////////////////////////////////////////////////////////////////////////////////////////////////
    // 消息入队
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis); //调用MessageQueue.enqueueMessage
    }
    //////////////////////////////////////////////////////////////////////////////////////////////
    public final void removeMessages(int what) {
        mQueue.removeMessages(this, what, null);
    }

    public final void removeMessages(int what, Object object) {
        mQueue.removeMessages(this, what, object);
    }

    public final void removeCallbacksAndMessages(Object token) {
        mQueue.removeCallbacksAndMessages(this, token);
    }

    public final boolean hasMessages(int what) {
        return mQueue.hasMessages(this, what, null);
    }

    public final boolean hasMessages(int what, Object object) {
        return mQueue.hasMessages(this, what, object);
    }

    public final boolean hasCallbacks(Runnable r) {
        return mQueue.hasMessages(this, r, null);
    }

    public final Looper getLooper() {
        return mLooper;
    }

    public final void dump(Printer pw, String prefix) {
        pw.println(prefix + this + " @ " + SystemClock.uptimeMillis());
        if (mLooper == null) {
            pw.println(prefix + "looper uninitialized");
        } else {
            mLooper.dump(pw, prefix + "  ");
        }
    }

    @Override
    public String toString() {
        return "Handler (" + getClass().getName() + ") {"
        + Integer.toHexString(System.identityHashCode(this))
        + "}";
    }

    // 下面与Messnger相关的都属于跨进程通信
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }

    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }

    //处理msg.callback情况
    private static void handleCallback(Message message) {
        message.callback.run();
    }

    final MessageQueue mQueue;
    final Looper mLooper;
    final Callback mCallback;
    final boolean mAsynchronous;
    IMessenger mMessenger;

    private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }

        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {
                return false;
            }

            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
}
```
