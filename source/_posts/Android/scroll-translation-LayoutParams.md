---
title: View滑动效果常用属性详解：scroll、translation、LayoutParams
date: 2017-01-01
tags:
categories: Android
description: View滑动效果常用属性详解：scroll、translation、LayoutParams
---

在自定义View，以及属性动画中常用到以下属性：
scrollX/scrollY,translationX/translationY,x/y,LayoutParams

---
# 1.scrollX/scrollY
对于这两个属性，View中提供了很多公有方法对其进行设置：
```java
1、setScrollX(int value)/setScrollY(int value);
	//对应的也有getScrollX()/getScrollY()方法
2、scrollTo(int x, int y);
3、scrollBy(int x, int y);
```
通过源代码我们发现以上方法最终调用的都是``scrollTo(int x,int y)``方法：

```java
public void setScrollX(int value) {
    scrollTo(value, mScrollY);
}

public void setScrollY(int value) {
   scrollTo(mScrollX, value);
}

public void scrollBy(int x, int y) {
   scrollTo(mScrollX + x, mScrollY + y);
}
```
其中需要注意的是scrollBy与scrollTo的区别在于：
**scrollBy设置的是相对值，scrollTo设置的是绝对值**。

** ScrollTo方法滑动的是View里面的内容 **
在这里我们创建一个类ScrollView继承自TextView并重载里面的onTouchEvent方法，为了便于观察在这里我们只处理x方向的滑动，同时将模拟器的布局边界打开。

```java
public class ScrollView extends TextView {
	...

	//用于保存上一次点击事件的x坐标
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
    ...
}
```
在布局文件中使用ScrollView类

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:gravity="center"
    android:layout_height="match_parent">

    <cn.hufeifei.scrolltest.view.ScrollView
        android:layout_width="160dp"
        android:layout_height="160dp"
        android:background="#fbb"
        android:gravity="center"
        android:text="scroll"
        android:textColor="#00f"
        android:textSize="20sp"/>
</LinearLayout>
```
### 下面是运行效果图：
![scroll效果图](http://img-blog.csdn.net/20161231210519094?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用这个方法我们可以模仿ViewPager自定义一个更加简洁的页面滑动的控件。

---
# 2.translationX/translationY
与translationX/translationY类似的属性有很多，这些属性都存在View的RenderNode成员中。
```java
    /**
     * RenderNode holding View properties, potentially holding a DisplayList of View content.
     * <p>
     * When non-null and valid, this is expected to contain an up-to-date copy
     * of the View content. Its DisplayList content is cleared on temporary detach and reset on
     * cleanup.
     */
    final RenderNode mRenderNode;
```
根据文档注释我们可以知道RenderNode其实是一个专门存放View的属性的容器，这些属性都是与View显示相关的属性，比如：

```java
//RenderNode中的部分方法
boolean setAlpha(float alpha)
float getAlpha()

boolean setElevation(float lift)
float getElevation()

boolean setTranslationX(float translationX)
float getTranslationX()

boolean setTranslationY(float translationY)
float getTranslationY()

boolean setTranslationZ(float translationZ)
float getTranslationZ()

boolean setRotation(float rotation)
float getRotation()

boolean setRotationX(float rotationX)
float getRotationX()

boolean setRotationY(float rotationY)
float getRotationY()

boolean setScaleX(float scaleX)
float getScaleX()

boolean setScaleY(float scaleY)
float getScaleY()

boolean setPivotX(float pivotX)
float getPivotX()

boolean setPivotY(float pivotY)
float getPivotY()
//以上方法对应的属性都是我们制作属性动画常用的属性。

/**
 *对于最后四个方法，由于left，top，right，bottom用于View的定位，
 *这些方法虽然都是公有的，但只允许android的布局系统调用它们来设置这四个定位属性，开发者调用将无效。
 *与之相关的width，height属性也是一样的。
 */
boolean setLeft(int left)
boolean setTop(int top)
boolean setRight(int right)
boolean setBottom(int bottom)
```
在View中与这些属性相关的get/set方法大都直接或间接的调用了RenderNode的相关get/set方法。

### 在这里为了实现滑动效果，同时为了便于观察，我们只修改translationX属性
同样的我们定义一个类TranslationView继承自TextView

```java

public class TranslationView extends TextView {
	...

    private float lastX;
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = event.getX();
                break;
            case MotionEvent.ACTION_MOVE:
                float currX = event.getX();
                float distanceX = currX - lastX;
                setTranslationX(getTranslationX() + distanceX);
                lastX = currX;
                break;
        }
        return true;
    }
}
```
在布局中使用TranslationView类

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center">

    <cn.hufeifei.scrolltest.view.TranslationView
        android:layout_width="160dp"
        android:layout_height="160dp"
        android:background="#aaf"
        android:gravity="center"
        android:text="translation"
        android:textColor="#f00"
        android:textSize="20sp"/>
</LinearLayout>
```
![translation效果图](http://img-blog.csdn.net/20161231215657155?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**我们可以看到平移属性只是将View的绘制进行了平移，View的定位并没有发生改变**

在View中还有几个属性与方法与translationX/translationY有着密不可分的关系

```java
public float getX() {
    return mLeft + getTranslationX();
}
public void setX(float x) {
    setTranslationX(x - mLeft);
}
public float getY() {
    return mTop + getTranslationY();
}
public float getY() {
    return mTop + getTranslationY();
}
```
在前面的我们说到left,top,right,bottom几个属性都是android系统用来进行布局定位的，开发者一般无法进行修改。这四个属性在view中对应着：mLeft,mTop,mRight,mBottom字段。

**这几个属性之间有以下关系：**
**1、   x = left + translationX**
**2、   y = top + translationY**

### 将上面的效果图进行了一些处理，得到以下关系图：
![关系图](http://img-blog.csdn.net/20161231231836055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 需要注意的是这里的left/top属性与x/y属性都是相对于父容器得到的。

# LayoutParams的margin属性
看到这个属性，很容易就联想到前端CSS中的盒子模型概念。安卓中margin，padding的概念与前端CSS类似。

同样的我们以marginLeft作为操作对象，来创建一个TextView的子类LayoutView。

```java

public class LayoutView extends TextView {
	...

    private float lastX;
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = event.getX();
                break;
            case MotionEvent.ACTION_MOVE:
                float currX = event.getX();
                LinearLayout.LayoutParams params = (LinearLayout.LayoutParams) getLayoutParams();
                params.leftMargin += currX - lastX;
                requestLayout();//必须调用该方法通知布局系统对View重新布局
                lastX = currX;
                break;
        }
        return true;
    }
}
```
并将其使用在布局文件中

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center">

    <cn.hufeifei.scrolltest.view.LayoutView
        android:layout_width="160dp"
        android:layout_height="160dp"
        android:background="#cfc"
        android:gravity="center"
        android:text="layout"
        android:textColor="#00f"
        android:textSize="20sp"/>
</LinearLayout>
```

运行得到效果图如下：
![marginLeft效果图](http://img-blog.csdn.net/20161231233705795?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到我点击鼠标往做移动，marginLeft减小成负数，使得控件往也左移动。在前端CSS中我们为了实现特殊的定位要求也经常用到类似的方法。


# 三种滑动方式的对比
1、通过修改scrollX/scrollY属性达到滑动效果：它可以方便地实现滑动效果而且不影响内部元素的单击事件。但是它只能滑动View的内容，并不能滑动View本身。
2、View本身定位没有发生改变，只是绘制的位置发生了改变
3、从布局上改变了View本身的定位。

# [代码下载地址](http://download.csdn.net/detail/holmofy/9726419)

刚开始写博客，不正确的地方请多多指正