---
title: Gradle的那些坑
date: 2017-03-25
tags:
- Android
- Grade
categories: Android
description: SurfaceView、SurfaceHolder、Surface之间的关系
---

Gradle是Android Studio使用的构建系统，类似于Eclipse上的Maven（貌似新版的Eclipse也有支持Gradle的插件了），有了Gradle就可以很好的帮我们解决库依赖的问题了，而且也不要我们自己手动从网上下载。Gradle支持JCenter，Maven等中央仓库，关于JCenter与Maven的区别这里[有篇文章](http://blog.sina.com.cn/s/blog_72ef7bea0102vvqg.html)。Gradle对下载下来的jar包进行统一管理，避免了一个jar包到处拷贝，搞得每个项目里面都有。

# 修改Gradle的默认路径
Gradle默认将所有的jar包下载到C盘用户目录下的**.Gradle**下
![修改Gradle默认路径](http://img-blog.csdn.net/20170325223333076?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
对于我这种C盘用了小小的固态的人来说，简直就是逼着我重装系统。
但是由于Android Studio一启动就会build整个工程，导致你没办法把文件夹剪切到另外一个目录下，而且AS运行的时候又关联了这个文件夹下的内容，从而没法拷贝更别说剪切。所以最好的方式在AS关闭的时候将.gradle文件夹拷贝到你想放到的路径下，然后再运行AS对gradle路径进行设置。
![修改Gradle默认路径](http://img-blog.csdn.net/20170325223445468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# Gradle 与 Gradle Plugin for Android Studio 的版本问题
Gradle是一个独立运行的程序，不但可以与AndroidStudio协同工作还可以和Eclipse等IDE配合使用。
但由于Gradle发展速度比较快，导致Gradle版本不一，比如我AS中可用的版本就有四种
![Gradle](http://img-blog.csdn.net/20170325223523188?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
而Gradle Plugin是为了让Android Studio使用Gradle来进行构建的一个插件。
Gradle 和 Gradle Plugin插件经常由于版本冲突问题导致Gradle无法build。
比如说，有些朋友更新了Andorid Studio后原来的很多工程build的时候都报错了
![Error:A problem occurred configuring project ':app'.](http://img-blog.csdn.net/20170325223630705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这种问题就是由于版本不匹配造成的。
# 修复方式
下面有一个表是Gradle与Gradle Plugin插件之间的匹配关系
<table><tbody><tr><th>Plugin version</th><th>Required Gradle version</th></tr><tr><td>1.0.0 - 1.1.3</td><td>2.2.1 - 2.3</td></tr><tr><td>1.2.0 - 1.3.1</td><td>2.2.1 - 2.9</td></tr><tr><td>1.5.0</td><td>2.2.1 - 2.13</td></tr><tr><td>2.0.0 - 2.1.2</td><td>2.10 - 2.13</td></tr><tr><td>2.1.3 - 2.2.3</td><td>2.14.1+</td></tr><tr><td>2.3.0+</td><td>3.3+</td></tr></tbody></table>
那怎么在项目中配置Gradle与Gradle插件呢，看下图
![Gradle与Gradle插件的匹配](http://img-blog.csdn.net/20170325223815725?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
文件1是用来配置Gradle的
![Gradle与Gradle Plugin的匹配](http://img-blog.csdn.net/20170325223913522?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
文件2是用来配置Gradle Plugin的
![Gradle与Gradle Plugin的匹配](http://img-blog.csdn.net/20170325224004288?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
