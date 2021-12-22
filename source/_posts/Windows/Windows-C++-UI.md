---
title: Windows C++界面库
date: 2017-02-14
tags:
categories: Windows
description: Windows C++界面库
---
记得大一学C语言的时候，觉得黑白窗很无聊，后来在网上找到了[EasyX](http://baike.baidu.com/link?url=UjIZLt5nsju-JH6JC1n4w-1OWR6HxHbWOAJNCA_F7EizdlMUWysdn7Xdai2_R_qptWqWAHZqAnWtiAg1mVBoTa) （一个模仿turbo c的图形库）http://www.easyx.cn ，用它写一些贪吃蛇、扫雷这类有图形界面的游戏来练手。 当时学的时候就很好奇为什么调用这些函数就能绘制图形，后来从网上了解到了Windows编程，于是从淘宝淘了本《Windows程序设计》看了起来，当时看的时候还有点吃力。
趁着大一结束后的那次暑假我看完了王爽老师编写的《汇编语言》，对计算机内存、CPU等底层方面的知识有了更深一层对的了解后，才重新拾起《Windows程序设计》。当时大二也开始学C++了，还记得“亮欧巴”教完谭浩强写的C++，还不能真正理解面向对象的意义，我在直接用Win32API写窗口程序的时候也感觉到要做很多重复工作，写很多模板代码（但当时自己完全不知道怎么用C++去封装Win32API），于是在网上找了些资料，还记得有一位大神出的视频里面讲了MFC的封装原理后，我自己才试着封装了Win32API（当然没使用MFC的消息映射机制，直接用了C++的虚函数多态），之后才明白C++的诞生是计算机工业发展的必然。学完后立马花了2个多月的时间写了个浏览器（为了应付学校的考试，也为了寒假回家过个好年，无奈拖长战线），当然网页显示直接使用MFC封装好的CHtmlView，这其中80%的时间都花在写界面上，当时还不知道开源社区有封装好的MFC控件，也不知道有CBitmapButton这类东西，完全自己封装，最终写出来的界面还贼TM丑，其实这也归结于当时不会PS，搞得后来很多功能都不愿完善了。

![当时写的浏览器](http://img-blog.csdn.net/20170214233258052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

---
最终我在网上了解到[DirectUI](http://baike.baidu.com/link?url=jjY4kgjgDrFjGCEAodKznQ4tjXwz9kCuQ-jEq5DpGk65qX1u6fAXz2TJCCaY4Ze4oAcixng9ssbrrCckMMvXF87Yrm1eScWKAKlgFLYTsIm)这项技术，虽然微软没有为开发者提供技术支持，但网上的总有一大批大牛人物敢于挑战。
下面我以开源与否列举几个知名的。
## 开源界面库
### DuiLib
首先要说的就是大名鼎鼎的DuiLib，国内很多大小公司都在使用该界面库，比如华为网盘，腾讯微信，百度杀毒 and so on。。。这个库是借鉴了国外的大牛Bjarke Viksoe写的[Windowless库](http://www.viksoe.dk/code/windowless1.htm)。据说DuiLib是国内第一个开源的DirectUI界面库，有很多界面库也是基于DuiLib二次开发的。随着时间的洗礼，很多其他界面库都慢慢销声匿迹了，DuiLib算是活的最好的。下面是我以前写过的几个小程序。
![模仿EclipseInstaller写的EclipseSelector](http://img-blog.csdn.net/20170214233906637?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![MediaPlayer](http://img-blog.csdn.net/20170214234045775?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### RingSdk
这是国内的前辈自己写的类库，这里给出前辈对RingSdk的介绍http://blog.csdn.net/ringphone/article/details/2911244
### 金山界面库BKWin
这是金山公司开源的一款界面库，相对个人维护的界面库而言，是更可靠的。
相关链接：http://code.ijinshan.com/index.html

上面三个界面库可以说是国内最知名的开源界面库，其他很多界面库都是来源于这三个界面库（有些库只是将名字改改，就自立一派，( ﹁ ﹁ ) ~→真不想吐槽天朝的盗版能力）

---
## 商业库
### UIPower
老贵的一款商业界面库，但听说产品确实不错，也有很多知名企业使用他们的界面库，比如：招商银行，瑞星杀毒，农业银行，中信证券... 前期华为网盘也是使用这个界面。貌似还能跨平台（用不起，也就无法考证），公司老总还亲自出了一系列相关视频，上个月阙总还到DuiLib交流群宣传他们公司的产品，O(∩_∩)O~~。
公司官网：http://www.uipower.com
### 迅雷Bolt
Bolt界面引擎是迅雷公司从2009年开始开发的第四代界面库。迅雷7是首个采用该引擎成功开发的产品，目前迅雷旗下大部分客户端产品都基于该引擎开发,并稳定运行于超过3.5亿台PC上。
文档方面也比较齐全，唯一的遗憾是闭源。
http://bolt.xunlei.com/
### Skin++
貌似是UIPower之前的产品，最近也没什么动态了。
### LibUIDK
[LibUIDK](http://baike.baidu.com/link?url=fpQw9N6Fe2yOJoYqNXFoCnn5L6QlxAwrkmJgxXtO9pu0RmZdQjboG2HNGovroJ-3h_8efu7ehzDWSjr9xANXRa)是国际上顶尖的专业开发Windows平台下图形用户界面的开发包，也是国内第一款商业的高级界面开发工具。该开发包基于Microsoft的MFC库。使用此开发工具包可轻易把美工制作的精美界面用Visual C++实现，由于LibUIDK采用所见即所得的方式创建产品界面，所以极大的提高了产品的开发速度，并大大增强图形用户界面(GUI)的亲和力。LibUIDK还可以使您的软件轻松具有当今流行的换肤功能，以提高产品的竞争力。
### Flash4UI
Flash4UI 可以让普通的C++应用程序使用flash作为UI，从而使UI开发变的极其轻松。
通过flash的超炫效果，可以使软件提升几个档次。
不过Flash技术日渐甚微，这或许也不是最好的选择。
### clayui
现在支持的系统包括android，windows，wince，linux。clayui的特点是能实现各种2D，3D动画，一些WPF，FLEX才能实现的界面效果，通过clayui可以很方便的实现。
clayui的底层渲染支持纯软件渲染，d3d，opengl es硬件加速渲染，您可以根据自身的需求选择合适的渲染方式，使您界面的用户体验达到最佳效果。
clayui自带的界面编辑系统使您可以很容易的创建界面布局，编辑各种动画效果，彻底实现界面与逻辑的分离，您可以很容易的实现动态换肤，动态换布局，动态更换动画效果。
### DSkinLite
DSkinLite界面库如其名称“lite”一样，是一款轻量级的C++界面库。未使用复杂的Hook操作，仅使用替换窗口过程的方式（SubclassWindow）来处理控件界面绘制。使用XML文件管理GDI资源如颜色，字体，图片，并描述界面构成， 同时UIEASY首次创造性的将界面构成元素抽象为线条元素，矩形元素，图片元素，文本元素，并提供相应规则来使用这些元素“组合” 界面。这极大的提高了界面库产品的灵活性，使得界面库产品可以随意构造出多种多样的控件界面。
官网：http://www.uieasy.cn/
### codejock
国外的一个提供MFC控件，COM组件技术支持的公司，这个公司提供很多界面方面的支持。三星，惠普，eBay，福特等国际公司都和他有过合作。
http://www.codejock.com
### 魔方界面库
不知道跟软媒有什么关系，看软媒魔方的界面像是用了这个库。
http://www.muilib.com/