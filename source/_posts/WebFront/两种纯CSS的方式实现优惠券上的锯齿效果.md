---
title: 两种纯CSS的方式实现优惠券上的锯齿效果
date: 2018-03-21
categories: 前端
---

昨天有个模块分到我手里了，有个优惠券的组件要封装，公司做前端的好像对CSS都不是很熟(其实就是一群Javaer兼职干前端)，估计是用多了React、Bootstrap这种现成的框架(就只知道写组件了)，没写过啥基础的CSS。然后有个优惠券的模块分到了我的头上，正好总结总结。



优惠券最主要就是这个锯齿的问题。其实用图片做也完全可以，反正最后那些小图片都会被webpack编码成Base64的[DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs)

> 关于DataURL的内容可以参考[RFC2397](https://tools.ietf.org/html/rfc2397)

不过用图片方式就没有啥挑战性了，那我也没必要写这篇文章记录这个过程。

我们的目的是用纯CSS实现锯齿

## 、使用before和after伪元素的border实现

先看一张效果图

![效果图](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiuzdjmfj20dd06cwec.jpg)

先定义一下最基本的html：

> 这里我们主要关注怎么实现锯齿，内容什么的我就没写，背景颜色也直接放在html里面了。

```html
<div class="sawtooth" style="background:#e24141;width:400px;height:170px;"></div>
```

## 1. 用dotted圆点边框覆盖div

我们的主要思路就是用两排圆点覆盖在div上：

```css
.sawtooth {
    /* 相对定位，方便让before和after伪元素绝对定位偏移 */
    position: relative;
}

.sawtooth:before, .sawtooth:after {
    content: ' ';
    width: 0;
    height: 100%;
    /* 绝对定位进行偏移 */
    position: absolute;
    top: 0;
}

.sawtooth:before {
    /* 圆点型的border */
    border-right: 10px dotted white;
    /* 偏移一个半径，让圆点的一半覆盖div */
    left: -5px;
}

.sawtooth:after {
    /* 圆点型的border */
    border-left: 10px dotted white;
    /* 偏移一个半径，让圆点的一半覆盖div */
    right: -5px;
}
```

得到下面这样的效果：

![1](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbivveosnj20c405m0sl.jpg)

## 2. 微调边框位置

但是由于圆点是顶着div开始的，总感觉不怎么好看，我们把before和after两个伪元素往下移动一点：

```css
.sawtooth:before, .sawtooth:after {
    ...
    /* 下移一个圆点直径的距离，让最后一个圆点超出div */
    top: 10px;
}
```

![效果图](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiwlea3bj20dd06cwec.jpg)


这样虽然已经出效果了，但是在实际应用中，由于背景色的不同导致，我们border圆点露馅了：

![3](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbix3cohyj20d906s746.jpg)

## 3. 隐藏多余的部分

为了解决这个问题，我们把需要把超出div的部分剪切掉：

```css
.sawtooth {
    /* 相对定位，方便让before和after伪元素绝对定位偏移 */
    position: relative;
    /* 把超出div的部分隐藏起来 */
    overflow: hidden;
}
```

![4](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbixo93urj20ch05n0sl.jpg)

因为border的颜色(白色)和背景色(青绿色)不同，导致优惠券看上去很突兀。

对于这个问题，最开始相到的解决办法是让把border设置为`transparent`(透明)，可惜透明的结果是显示后面div的红色。所以暂时只能用最笨的方法：把border的颜色设置成背景色。

![5](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiyf1904j20cb060jr8.jpg)

**整理以下CSS**

```css
.sawtooth {
    /* 相对定位，方便让before和after伪元素绝对定位偏移 */
    position: relative;
    /* 把超出div的部分隐藏起来 */
    overflow: hidden;
}

.sawtooth:before, .sawtooth:after {
    content: ' ';
    width: 0;
    height: 100%;
    /* 绝对定位进行偏移 */
    position: absolute;
    top: 10px;
}

.sawtooth:before {
    /* 圆点型的border */
    border-right: 10px dotted white;
    /* 偏移一个半径，让圆点的一半覆盖div */
    left: -5px;
}

.sawtooth:after {
    /* 圆点型的border */
    border-left: 10px dotted white;
    /* 偏移一个半径，让圆点的一半覆盖div */
    right: -5px;
}
```



## 、使用透明背景实现

上面的方法的缺点是，我们需要让border的颜色和背景色一致，才能让我们的优惠券不“露馅”。

所以这次我们就直接用透明的背景。透明的背景当然不是指全透明，全透明那就是没背景了。看一下下面这张图你就知道我们要怎么干了。

![1](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiywxmi9j20d705wglg.jpg)

哇塞，这么多洞！密集恐惧症患者，看了估计有点闹心。

我们的主要思路是：在这张有这么多洞的网上面盖一张div，把多余的洞遮住，只保留两边有锯齿的洞。

> 不过使用这种方式我们先要把html中的背景色干掉，只保留宽高：

```html
<div class="sawtooth" style="width:420px;height:170px;"></div>
```

##  1. 用css画出这张网

为了画出这么多的洞，我们要用到`radial-gradient`这个渐变函数，不过我们并不需要用到它的渐变功能，只要用它来画透明的圆点就行。

> radial-gradient要求IE10及以上的版本的浏览器。
>
> 关于radial-gradient的更多详细内容，请参考[MDN的文档](https://developer.mozilla.org/en-US/docs/Web/CSS/radial-gradient)

```css
.sawtooth{
    /* 画出一个半径为5px的透明的圆，透明圆以外都是#e24141颜色 */
    background-image: radial-gradient(transparent 0, transparent 5px, #e24141 5px);
    /* 截取上面生成的渐变图的一部分，相当于截取15px的正方形中有一个直径10px的透明圆点 */
    background-size: 15px 15px;
    /* 根据优惠券div大小进行微调 */
    background-position: 8px 3px;
}
```

上面简单的三句话就生成了这张网：

![1](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiywxmi9j20d705wglg.jpg)

也许你看到样式上面的注释会很纳闷，这样就生成了一张网？你把sawtooth类改成下面的样式看一下，估计你就会明白了：

```css
.sawtooth{
    background-image: radial-gradient(transparent 0, transparent 5px, #e24141 5px);
    background-size: 100px 100px;
    background-repeat: no-repeat;
}
```

![2](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj0dtoeej207j05da9u.jpg)

## 2. 找一个元素把多余的洞覆盖掉

**为了不影响html的整洁性**，我们就**使用before或after这两个伪元素中的一个**来覆盖多余的洞吧。

```css
.sawtooth {
    background-image: radial-gradient(transparent 0, transparent 5px, #e24141 5px);
    background-size: 15px 15px;
    background-position: 8px 3px;
    /* 相对定位，让before伪元素方便定位 */
    position: relative;
}

.sawtooth:before {
    content: ' ';
    display: block;
    /* 用相同的颜色覆盖 */
    background-color: #e24141;
    /* 绝对定位，遮住中间所有的洞，只保留边角的锯齿 */
    position: absolute;
    top: 0;
    bottom: 0;
    /* 为锯齿保留的距离 */
    left: 10px;
    right: 10px;
}
```

完成的效果图如下：

![3](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj10ocfnj20d505zq2r.jpg)

## 3. 让遮罩层下移，让它成为背景

上面的样式还有一点点问题。

当我们网div中加内容的时候：

```html
<div class="sawtooth" style="width:420px;height:170px;">
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
  内容内容内容内容内容内容内容内容内容内容
</div>
```

发现它不仅把洞给遮住了，还把我们的内容给遮住了：

![4](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj1oa891j20dh068jr9.jpg)

出现上面这种现象的原因就是：绝对定位脱离文档流后，会浮在文档的上面。

> “我们的内容在地表上，遮罩层却在一楼”

为了解决这个问题，我们还需要加一个样式，让这个遮罩层不再是“遮罩层”，而是在文档下面当做背景。

```css
.sawtooth:before {
    ...
    z-index: -1;
}
```

> 让“遮罩层”到负一楼去。

![5](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj2836ysj20d705v3yf.jpg)

OK，我们想要的效果有了。

而且不管背景换成什么颜色，我们的“优惠券”都不会“露馅”。

![6](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj2p9r0tj20di062q2r.jpg)

## 4. 双层背景实现遮罩

使用伪元素实现遮罩除了要使用`z-index`改变层级外，代码也显得冗长，我们还可以用双层背景来实现遮罩。为了让两个背景大小不一样我们还要用到[background-clip](https://developer.mozilla.org/en-US/docs/Web/CSS/background-clip)来设置背景的裁剪区域，`background-clip`支持四种属性值(text兼容性不好)：

![](http://tva1.sinaimg.cn/large/bda5cd74gy1frdjsv0d2lj216s0a0q6e.jpg)

下面用将网装背景剪裁区域设置为`border-box`，纯色背景裁剪区域设置为`padding-box`用来遮罩。

```css
.sawtooth {
    /* 注意：radial-gradient是作为background-image属性设置的，而纯色#e24141是作为background-color属性设置的 */
    background: border-box radial-gradient(transparent 0, transparent 5px, #e24141 5px), padding-box #e24141;
    background-size: 15px 15px;
    background-position: 8px 3px;
    border-left: 10px solid transparent;
    border-right: 10px solid transparent;
    position: relative;
}
```

> 感谢评论区[xlf-summer](https://blog.csdn.net/qq_34212101)给出解决思路。他最初的思路是用`linear-gradient`实现遮罩背景：
>
> `background: border-box radial-gradient(transparent 0, transparent 5px, #e24141 5px), padding-box linear-gradient(#e24141, #e24141);`
>
> 不过我在实践的时候，发现直接用纯色背景也可以实现效果。

## 附加：改变形状，生成别样的锯齿

我们可以通过适当修改background的几个参数，将圆形压扁，生成其他形状的锯齿：

```css
.sawtooth {
    background-image: radial-gradient(transparent 0, transparent 4px, #e24141 4px);
    background-size: 12px 8px;
    background-position: -5px 10px;
    position: relative;
}

.sawtooth:before {
    content: ' ';
    display: block;
    background-color: #e24141;
    position: absolute;
    top: 0;
    bottom: 0;
    left: 6px;
    right: 6px;
    z-index: -1;
}
```

![7](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj3cwgvxj20cr063q2q.jpg)