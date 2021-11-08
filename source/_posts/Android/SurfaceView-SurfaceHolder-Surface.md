---
title: SurfaceView、SurfaceHolder、Surface
date: 2017-03-26
tags:
- Android
- SurfaceView
categories: Android
keywords:
- SurfaceView
- SurfaceHolder
- Surface
description: SurfaceView、SurfaceHolder、Surface之间的关系
---

按照官方文档的说法，SurfaceView继承自View，并提供了一个独立的绘图层，你可以完全控制这个绘图层，比如说设定它的大小，所以SurfaceView可以嵌入到View结构树中，但是需要注意的是，由于SurfaceView直接将绘图表层绘制到屏幕上，所以和普通的View不同的地方就在与它不能执行Transition，Rotation，Scale等转换，也不能进行Alpha透明度运算。SurfaceView的Surface排在Window的Surface(也就是View树所在的绘图层)的下面，SurfaceView嵌入到Window的View结构树中就好像在Window的Surface上强行打了个洞让自己显示到屏幕上，而且SurfaceView另起一个线程对自己的Surface进行刷新。特别需要注意的是SurfaceHolder.Callback的所有回调方法都是在主线程中回调的。

SurfaceView、SurfaceHolder、Surface的关系可以概括为以下几点：
* SurfaceView是拥有独立绘图层的特殊View
* Surface就是指SurfaceView所拥有的那个绘图层，其实它就是内存中的一段绘图缓冲区。
* SurfaceView中具有两个Surface，也就是我们所说的双缓冲机制
* SurfaceHolder顾名思义就是Surface的持有者，SurfaceView就是通过过SurfaceHolder来对Surface进行管理控制的。并且SurfaceView.getHolder方法可以获取SurfaceView相应的SurfaceHolder。
* Surface是在SurfaceView所在的Window可见的时候创建的。我们可以使用SurfaceHolder.addCallback方法来监听Surface的创建与销毁的事件。


下面我们写一个Demo来对普通的View与SurfaceView进行区别。

## 普通的View
我们先写一个继承自View的DrawView，我们每过1秒进行一次重绘
```java
public class DrawView extends View {
    private static final int DELAY = 1000;

    private Paint mPaint;

    private int mCount;

    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            mCount++;
            invalidate();
            handler.sendEmptyMessageDelayed(0x0001, DELAY);
        }
    };

    public DrawView(Context context) {
        this(context, null);
    }

    public DrawView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DrawView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        mPaint = new Paint();
        mPaint.setTextAlign(Paint.Align.CENTER);
        mPaint.setColor(0xFFFF0000);
        mPaint.setTextSize(50);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        handler.sendEmptyMessageDelayed(0x0001, DELAY); //关联上Window后开始每1000ms重绘一次
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        handler.removeCallbacksAndMessages(null); //撤销关联时停止重绘，防止内存泄漏
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // 在View的中间绘制一个HelloWorld 并写上重绘的次数
        canvas.drawText("Hello World" + mCount, getWidth() >> 1, getHeight() >> 1, mPaint);
    }
}
```
然后我们把DrawView放到View结构树中
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:gravity="center"
        android:text="普通的View"
        android:textSize="20sp" />

    <cn.hufeifei.drawview.DrawView
        android:background="#ffffffff"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
</LinearLayout>
```
为了能够看到DrawView刷新的情况，我们在开发者模式下打开显示surface更新的选项
![打开显示surface更新的选项](http://img-blog.csdn.net/20170326232425934?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
执行结果
![普通View重绘](http://img-blog.csdn.net/20170326232535259?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由于这里是使用gif录制的结果图，我特意将gif帧率调节到33FPS，才有这样的效果图。
我们可以看到每隔1秒界面就刷新了一次，显然这里刷新的是Window的surface。
**（使用模拟器的同学需要注意了，由于模拟器在显示到PC屏幕的过程中有延迟，模拟器在绘制时可能使用了双缓冲，所以模拟器显示出来的可能界面刷新的频率不一样）
**
## SurfaceView的重绘
这次我们的DrawSurfaceView继承自SurfaceView
```java
public class DrawSurfaceView extends SurfaceView implements SurfaceHolder.Callback {
    private static final long DELAY = 1000;
    private final SurfaceHolder mHolder;
    private Paint mPaint;
    private int mCount;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            mCount++;
            Canvas canvas = mHolder.lockCanvas();
            canvas.drawColor(0xFFFFFFFF);//擦除原来的内容
            canvas.drawText("Hello World" + mCount, getWidth() >> 1, getHeight() >> 1, mPaint);
            mHolder.unlockCanvasAndPost(canvas);
            sendEmptyMessageDelayed(0x0001, DELAY);
        }
    };

    public DrawSurfaceView(Context context) {
        this(context, null);
    }

    public DrawSurfaceView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DrawSurfaceView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        mPaint = new Paint();
        mPaint.setTextAlign(Paint.Align.CENTER);
        mPaint.setColor(0xFFFF0000);
        mPaint.setTextSize(50);

        mHolder = getHolder();          //获取SurfaceView的ViewHolder对象
        mHolder.addCallback(this);    //为ViewHolder添加事件监听器
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        mHandler.sendEmptyMessageDelayed(0x0001, DELAY);
        Log.i("surfaceCreated", Thread.currentThread().getName());//打印当前线程的名字
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        Log.i("surfaceChanged", Thread.currentThread().getName());//打印当前线程的名字
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        mHandler.removeCallbacksAndMessages(null);
        Log.i("surfaceDestroyed", Thread.currentThread().getName());//打印当前线程的名字
    }
}
```
同样的我们把DrawSurfaceView放到View结构树中
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:gravity="center"
        android:text="SurfaceView"
        android:textSize="20sp" />

    <cn.hufeifei.drawview.DrawSurfaceView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
</LinearLayout>
```
![SurfaceView结果图](http://img-blog.csdn.net/20170326232725650?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到很明显的区别，SurfaceView并没有刷新Window，而只是刷新了SurfaceView所在的界面区域，可以看出SurfaceView使用的是自己的surface。
![SurfaceHolder.Callback回调方法](http://img-blog.csdn.net/20170326232803713?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
从打印的log来看SurfaceHolder.Callback方法也确实是在主线程回调的。


## 实现代码[点击这里](http://download.csdn.net/detail/holmofy/9794709)
