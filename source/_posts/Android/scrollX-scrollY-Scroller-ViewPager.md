---
title: View滑动效果常用属性详解2-使用scrollX|scrollY和Scroller实现自定义ViewPager
date: 2017-01-02
tags:
categories: Android
description: View滑动效果常用属性详解2-使用scrollX|scrollY和Scroller实现自定义ViewPager
---
使用scrollX,scrollY和Scroller自定义ViewPager
# 原理介绍
废话不多说先上图。

![原理图](http://img-blog.csdn.net/20170102191816574?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ViewPager就是包裹了n个宽高与自己相同的子页面，然后通过滑动内部子页面来达到左右页面切换效果。需要注意的是上图中width，height都是指ViewPager的宽高。

# 继承ViewGroup并实现onLayout方法
在layout方法中我们首先要对子页面进行布局定位。在加上上一次我们实现内容滑动使用的scrollX属性
```java
package cn.hufeifei.scrollertest.view;

import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.ViewGroup;

/**
 * 自定义ViewPager效果
 * Created by Holmofy on 2017/1/2.
 */

public class MyViewPager extends ViewGroup {
    public MyViewPager(Context context) {
        this(context, null);
    }

    public MyViewPager(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyViewPager(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        int width = r - l;
        int height = b - t;
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).layout(width * i, 0, width * (i + 1), height);
        }
    }

    private float lastX;
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = event.getX();
                break;
            case MotionEvent.ACTION_MOVE:
                float currX = event.getX();
                scrollBy((int) (lastX - currX), 0);
                lastX = currX;
                break;
        }
        return true;
    }
}
```
在布局文件中使用我们定义的MyViewPager类
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="cn.hufeifei.scrollertest.MainActivity">

    <cn.hufeifei.scrollertest.view.MyViewPager
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#f99"
            android:gravity="center"
            android:text="0"
            android:textSize="30sp"/>

        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#9f9"
            android:gravity="center"
            android:text="1"
            android:textSize="30sp"/>

        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#99f"
            android:gravity="center"
            android:text="2"
            android:textSize="30sp"/>
    </cn.hufeifei.scrollertest.view.MyViewPager>
</RelativeLayout>
```
运行效果图如下：
![效果图1](http://img-blog.csdn.net/20170102191927012?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
此时我们能发现出现了两个问题：
1.自定义ViewPager无法检测滑动越界问题，
2.当用户停止滑动释放触摸后(也就是MotionEvent.ACTION_UP事件)，没有进行页面的定位。
### 问题解决
#### 1、滑动越界问题
很显然我们在滑动的过程中应该对滑动边界进行判断，也就是在``case MotionEvent.ACTION_MOVE:``中判断滑动的位置有没有超过所有的页面的总宽度。
#### 2、用户停止移动后的定位问题
在case MotionEvent.ACTION_UP:中我们应该判断，如果用户最后停止触摸的位置小于当前所在子页面宽度的一半，应该向左定位到当前子页面；如果用户最后停止触摸的位置大于当前所在子页面宽度的一半，应该定位到下一个子页面。而且在用户释放触摸后，页面应该平滑地滑动到所要定位的子页面，否则会给用户一种突兀感，这里我们可以有很多解决方法，也许你第一个想到的是使用属性动画，其实安卓已经提供给我们一个用来实现View平滑滚动的一个工具类----Scroller。它是专门为自定义view的滑动效果设计的，与之类似的还有一个类--OverScroller，在这里我们使用Scroller来解决我们的问题
### Scroller详解
官方文档是这样解释的：
This class encapsulates scrolling. You can use scrollers (Scroller or OverScroller) to collect the data you need to produce a scrolling animation—for example, in response to a fling gesture. Scrollers track scroll offsets for you over time, but they don't automatically apply those positions to your view. It's your responsibility to get and apply new coordinates at a rate that will make the scrolling animation look smooth.

大致的意思是：这个类封装了滑动效果，我们能使用Scroller或者OverScroller来收集并处理一些数据来达到滑动动画的效果，随着时间推移Scroller会反馈给我们滑动的偏移量，但是它并不能自动将滑动的偏移量应用到View上，需要我们自己来将它计算出的数据应用到View上达到平滑动画的效果。

英语比较渣，大概意思也能看懂。
1、首先我们需要创建一个Scroller对象Scroller提供给我们以下几个构造方法：
```java
public Scroller(Context context);
public Scroller(Context context, Interpolator interpolator);
public Scroller(Context context, Interpolator interpolator, boolean flywheel);
```
Scroller会默认提供给我们一个插值器，我们也不用处理fling手势
2、我们需要重载View的computeScroll方法来将Scroller计算出来的数据应用到View上。

### 详细代码
```java
package cn.hufeifei.scrollertest.view;

import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.ViewGroup;
import android.widget.Scroller;

/**
 * 自定义ViewPager效果
 * Created by Holmofy on 2017/1/2.
 */

public class MyViewPager extends ViewGroup {
    private Scroller mScroller;

    public MyViewPager(Context context) {
        this(context, null);
    }

    public MyViewPager(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyViewPager(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mScroller = new Scroller(getContext());
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        int width = r - l;
        int height = b - t;
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).layout(width * i, 0, width * (i + 1), height);
        }
    }

    private float lastX;

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = event.getX();
                break;
            case MotionEvent.ACTION_MOVE: {
                float currX = event.getX();         //这次用户触摸的x坐标
                float distance = lastX - currX;     //两次触摸的x方向的距离

                int currScrollX = getScrollX();                 //获取当前的滑动值
                int scrollX = (int) (currScrollX + distance);   //计算新滑动值

                //计算右边界
                int right = (getChildCount() - 1) * getWidth();
                if (scrollX < 0) {
                    //有没有越过左边界
                    scrollX = 0;
                } else if (scrollX > right) {
                    //有没有越过右边界
                    scrollX = right;
                }
                scrollTo(scrollX, getScrollY());
                lastX = currX;
            }
            break;
            case MotionEvent.ACTION_UP: {
                int scrollX = getScrollX();
                int childIndex = (int) ((float) scrollX / (float) getWidth() + 0.5);
                //scrollTo(childIndex * getWidth(), getScrollY());//应该平滑地移动到指定的位置
                smoothScrollTo(childIndex * getWidth());
            }
            break;
        }
        return true;
    }

    private void smoothScrollTo(int destX) {
        int scrollX = getScrollX();
        int distanceX = destX - scrollX;
        mScroller.startScroll(scrollX, 0, distanceX, 0, 250);
        invalidate();
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            //滑动没有结束
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
        }
    }
}
```
运行效果图：
![效果图2](http://img-blog.csdn.net/20170102192002948?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 扩展
1、你会发现在这里我们并没有像ViewPager一样自定义一个PagerAdapter适配器，来统一管理各个子页面。ViewPager这样做是为了防止开发者通过xml布局文件自己设置子页面的宽高，上一节中我们讲到开发者通常无法自己去设置left，top，right，height，width，height属性，在这个示例中，你会发现MyViewPager下的子控件宽高无法指定为确定的值，那是因为在xml实例化所有的控件后，MyViewPager会通过``onLayout``方法中重新对各个子控件进行重新布局。如果你有兴趣继续封装自己的ViewPager类你可以实现一个PagerAdapter。
2、这种实现相对于ViewPager来说还是太简单了，如果MyViewPager中有许多嵌套的子控件，还需要考虑滑动、点击等方面的冲突，这一部分需要熟悉[View的事件分发机制(这里也写了一片文章，供大家参考)](http://blog.csdn.net/holmofy/article/details/54092120),所以我们的MyViewPager适合做一些简单的滑动效果(比如第一次启动时的滑动界面，轮播图效果等，使用ViewPager实现就太浪费了)，内部嵌套的控件不能完成一些复杂的操作。
3、同样的如果我们将里面所有的横坐标替换成纵坐标，就能达到上下滑动的效果，把代码也贴出来吧！
```java
package cn.hufeifei.scrollertest.view;

import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.ViewGroup;
import android.widget.Scroller;

/**
 * 自定义ViewPager效果
 * Created by Holmofy on 2017/1/2.
 */

public class MyViewPager extends ViewGroup {
    private Scroller mScroller;

    public MyViewPager(Context context) {
        this(context, null);
    }

    public MyViewPager(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyViewPager(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mScroller = new Scroller(getContext());
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        int width = r - l;
        int height = b - t;
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).layout(0, height * i, width, height * (i + 1));
        }
    }

    private float lastY;

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastY = event.getY();
                break;
            case MotionEvent.ACTION_MOVE: {
                float currY = event.getY();         //这次用户触摸的y坐标
                float distance = lastY - currY;     //两次触摸的y方向的距离

                int currScrollY = getScrollY();                 //获取当前的滑动值
                int scrollY = (int) (currScrollY + distance);   //计算新滑动值

                //计算底部边界
                int bottom = (getChildCount() - 1) * getHeight();
                if (scrollY < 0) {
                    //有没有越过顶部边界
                    scrollY = 0;
                } else if (scrollY > bottom) {
                    //有没有越过底部边界
                    scrollY = bottom;
                }
                scrollTo(getScrollX(), scrollY);
                lastY = currY;
            }
            break;
            case MotionEvent.ACTION_UP: {
                int scrollY = getScrollY();
                int childIndex = (int) ((float) scrollY / (float) getHeight() + 0.5);
                smoothScrollTo(childIndex * getHeight());
            }
            break;
        }
        return true;
    }

    private void smoothScrollTo(int destY) {
        int scrollY = getScrollY();
        int distanceY = destY - scrollY;
        mScroller.startScroll(0, scrollY, 0, distanceY, 250);
        invalidate();
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            //滑动没有结束
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
        }
    }
}
```
效果图
![这里写图片描述](http://img-blog.csdn.net/20170102193343830?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# [代码下载地址](http://download.csdn.net/detail/holmofy/9727064)
