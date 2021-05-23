---
title: TextureView、SurfaceTexture、Surface
date: 2017-03-26 23:34
tags:
categories: Android
description: TextureView、SurfaceTexture、Surface之间的关系
---
上篇文章我们说了SurfaceView，接下来我们对Texture进行一下分析。
SurfaceView由于使用的是独立的绘图层，并且使用独立的线程去进行绘制。前面的文章中也说到SurfaceView不能进行Transition，Rotation，Scale等变换，这就导致一个问题SurfaceView在滑动的时候，SurfaceView的刷新由于不受主线程控制导致SurfaceView在滑动的时候会出现黑边的情况，看下面的效果图。
![SurfaceView滑动出现黑边](http://img-blog.csdn.net/20170326232946909?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这个Demo里的SurfaceView是上篇文章中定义的DrawSurfaceView，不了解的同学可以看上一篇文章

如果这里的SurfaceView是VideoView，由于视频刷新的速度比这个DrawSurfaceView速度大好几倍，那么这个黑边的情况将更恶劣，同学可以将DrawSurfaceView的刷新延时DELAY设置的更小一点，看的效果更明显。这里要讲到的TextureView就是用来解决这种问题的。

## 先来简单介绍一下TextureView
TextureView是API 14添加进来的，也就是Android4.0之后才能使用这个类，而SurfaceView从API 1开始就已经有了。
从字面意思来看TextureView是用来绘制纹理的View，官方文档给出解释是说，TextureView专门用来渲染像视频或OpenGL场景之类的数据，而且TextureView只能用在具有硬件加速的Window中,如果使用的是软件渲染，TextureView什么也不显示。也就是说对于没有GPU的设备，TextureView完全不可用。好在现在的移动设备基本都有GPU进行硬件加速渲染（连我手里这款破旧的华为测试机都有(^o^)）。

### TextureView的相关类SurfaceTexture
TextureView在使用的时候涉及到这么几个类：SurfaceTexture，Surface。
Surface就是SurfaceView中使用的Surface，就是内存中的一段绘图缓冲区。
SurfaceTexture是什么呢，官方文档给出的解释是这样的：

SurfaceTexture用来捕获视频流中的图像帧的，视频流可以是相机预览或者视频解码数据。SurfaceTexture可以作为android.hardware.camera2, MediaCodec, MediaPlayer, 和 Allocation这些类的目标视频数据输出对象。可以调用``updateTexImage()``方法从视频流数据中更新当前帧，这就使得视频流中的某些帧可以跳过。

TextureView可以通过``getSurfaceTexture()``方法来获取TextureView相应的SurfaceTexture。但是最好的方式还是使用``TextureView.SurfaceTextureListener``监听器来对SurfaceTexture的创建销和毁进行监听，因为getSurfaceTexture可能获取的是空对象。

下面我们使用TextureView来实现类似于上篇文章中的1秒中更新一次界面的Demo
```java
package cn.hufeifei.drawview;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Paint;
import android.graphics.SurfaceTexture;
import android.os.Handler;
import android.os.Message;
import android.util.AttributeSet;
import android.view.Surface;
import android.view.TextureView;

/**
 * Created by Holmofy on 2017/3/26.
 */

public class DrawTextureView extends TextureView implements TextureView.SurfaceTextureListener {
    private static final long DELAY = 1000;
    private Surface mSurface;
    private Paint mPaint;
    private int mCount;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            mCount++;
            Canvas canvas = mSurface.lockCanvas(null);
            canvas.drawColor(0xFFFFFFFF);//擦除原来的内容
            canvas.drawText("Hello World" + mCount, getWidth() >> 1, getHeight() >> 1, mPaint);
            mSurface.unlockCanvasAndPost(canvas);
            sendEmptyMessageDelayed(0x0001, DELAY);
        }
    };

    public DrawTextureView(Context context) {
        this(context, null);
    }

    public DrawTextureView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DrawTextureView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mPaint = new Paint();
        mPaint.setTextAlign(Paint.Align.CENTER);
        mPaint.setColor(0xFFFF0000);
        mPaint.setTextSize(50);
        this.setSurfaceTextureListener(this);
    }

    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        mSurface = new Surface(surface);
        mHandler.sendEmptyMessageDelayed(0x0001, DELAY);
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height) {

    }

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
        mHandler.removeCallbacksAndMessages(null);
        mSurface = null;
        return true;
    }

    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {

    }
}
```
我们把它放到同样的View结构树中
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

    <cn.hufeifei.drawview.DrawTextureView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#ffffffff" />
</LinearLayout>
```
效果图如下
![TextureView刷新重绘](http://img-blog.csdn.net/20170326233106020?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
可以看到与Surface不同的是，Texture刷新的是它所在Window的Surface。

接下来我们使用DrawTextureView替换之前的SurfaceView来解决滑动黑边问题，下面是效果图
![TextureView解决SurfaceView滑动黑边问题](http://img-blog.csdn.net/20170326233147442?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 实现代码[点击这里](http://download.csdn.net/detail/holmofy/9794709)