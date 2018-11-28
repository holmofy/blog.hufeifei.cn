---
title: Android使用MultiDex处理64K限制
date: 2017-07-09 14:07
tags:
categories: Android
---

随着Android平台的不断发展，Android应用的规模也越来越大。当你的程序以及程序所使用的库达到一定大小，build时可能会产生错误，这表示你的程序已经达到Android应用架构的极限。


老版本构建工具报错：

```shell
Conversion to Dalvik format failed:Unable to execute dex: method ID notin[0,0xffff]:65536
```

新版本构建工具报错：

```
trouble writing output : Too many field references : 131000 ;max is 65536. You may try using -- multi - dex option .
```

这两个错误都提到`65535`，这个数字表示Dalvik字节码文件(也就是DEX文件)中可包含的方法的总数，因为Dalvik虚拟机中以`short`类型来索引DEX文件中的方法。

如果你的应用出现了这个错误，首先恭喜你，你的应用肯定有很大的代码量。文章接下来就介绍如何解决该问题，让你的应用拥有更大的代码量和更丰富的功能。

# 在Android5.0之前平台中使用MultiDex

因为Android5.0之前的系统是使用[Dalvik虚拟机](http://baike.baidu.com/item/Dalvik)来执行应用程序。默认情况下，Dalvik虚拟机将每个APK限制在单个`classes.dex`字节码文件中。我们可以使用MultiDex支持库来解决单个Dex文件的限制。

**MultiDex的原理**：将MultiDex支持库的代码放在主Dex文件中，把超额的方法放在附加Dex文件中，MultiDex支持库负责管理附加Dex文件中的方法，主Dex的函数通过MultiDex支持库去间接调用附加Dex文件中的方法。

![MultiDex原理图](http://img.blog.csdn.net/20170709135951416?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# Android5.0及以上版本的MultiDex

Androd5.0之后使用了名为[ART(Android Runtime)](http://baike.baidu.com/item/Android%20runtime)的运行时环境，它本身就支持“APK中包含多个dex文件”。

ART在应用安装时执行预编译，将Dalvik字节码翻译成本机二进制机器码从而加快应用执行速度：在预编译过程中会扫描dex文件，并将多个dex文件(dalvik字节码)编译成单个`.oat`文件(二进制机器码)。关于ART的更多信息，可以参考Android官方文档。

# 解决65K限制的方法。

## 1. 避免64K问题——减少方法数

* 在程序中应尽量避免使用重量型的库，比如Google的Guava，Apache的Commons等。大多数JavaEE中的库都不适合在Android中使用。
* 使用ProGuard删除未使用的代码，保证应用的发布版本体积尽可能小巧。ProGuard的使用会在后面的文章中提到。

## 2. 配置MultiDex构建多dex应用

首先确保你使用的Build Tools是21.1及以上版本，因为这样才可以使用Gradle的安卓插件来支持multidex。

步骤如下：

* 更改Gradle配置以启用MultiDex，示例如下：

  ```groovy
  android {
      compileSdkVersion 21
      buildToolsVersion "21.1.0"

      defaultConfig {
          ...
          minSdkVersion 14
          targetSdkVersion 21
          ...

          // 构建时生成多个dex
          multiDexEnabled true
      }
      ...
  }

  dependencies {
    // 添加MultiDex支持库的依赖
    compile 'com.android.support:multidex:1.0.0'
  }
  ```

  > `multiDexEnabled`属性可以设置在`defaultConfig`，`buildType`或者`productFlavor`中。

* 修改应用清单以使用MultiDexApplication类

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      package="com.example.android.multidex.myapplication">
      <application
          ...
          android:name="android.support.multidex.MultiDexApplication">
          ...
      </application>
  </manifest>
  ```

  > 如果你的应用需要在Application中进行初始化，可以继承MultiDexApplication类。示例代码如下：

  ```java
  package cn.hff.App;
  import android.support.multidex.MultiDexApplication;

  // 集成MultiDexApplication类
  public class App extends MultiDexApplication{
      @Override
      public void onCreate() {
      	//initialize
      	...
      }
  }
  ```
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      package="com.example.android.multidex.myapplication">
      <application
          ...
          android:name="cn.hff.App">
          ...
      </application>
  </manifest>
  ```

事实上MultiDexApplication源码很简单：

 ```java
 public class MultiDexApplication extends Application {
     public MultiDexApplication() {
     }

     protected void attachBaseContext(Context base) {
         super.attachBaseContext(base);
         MultiDex.install(this);
     }
 }
 ```

 所以对于上面的`App`我们还可以这么写：

 ```java
 package cn.hff.App;
 import android.app.Application;
 // 直接集成Application类
 public class App extends Application{
     @Override
     public void onCreate() {
     	//initialize
     	...
     }
     protected void attachBaseContext(Context base) {
         super.attachBaseContext(base);
 		// 在应用中安装MultiDex支持库
         MultiDex.install(this);
     }
 }
 ```

完成以上配置后，Android构建工具会根据需要将字节码文件打包成一个主Dex文件(classes.dex)和多个附加Dex文件(classes2.dex、classes3.dex...)。

# 开发过程中对MultiDex程序构建的优化

配置MultiDex后应用构建的时间明显增长，因为构建系统需要判断主Dex文件中应该包含哪些类、辅助Dex文件中应该包含哪些类，这个过程非常复杂。构建MultiDex应用于构建常规应用通常需要更多的时间，这可能会降低你的开发速度。

为了减短MultiDex应用的构建时间，我们可以在Gradle的`productFlavors`属性中配置两种输出模式：开发模式和发行模式。

开发模式中我们可以将`minSdkVersion`设置为21(也就是Android5.0)，这样应用可以使用ART的格式，从而更快的生成MultiDex的Apk。

发行模式中我们可以将`minSdkVersion`设置为应用实际需要兼容的最低SDK版本，这个设置可以兼容更低版本的Android平台，但相应地需要更长的构建时间。

一下是Gradle文件中相应的配置。

```groovy
android {
    productFlavors {
		// 定义开发模式和发行模式
        dev {
			// 开发模式设置 minSDKVersion = 21 从而允许Android Gradle插件预
			// 先将每个模块打包成dex并生成一个可以在Android5.0上测试的APK文件，
			// 这样不需要在Dex合并上耗费过多的时间
            minSdkVersion 21
        }
        prod {
            // 应用实际需要兼容的最低SDK版本
            minSdkVersion 14
        }
    }
          ...
    buildTypes {
        release {
			// 发行版运行ProGuard对方法数进行压缩
            runProguard true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                                                 'proguard-rules.pro'
        }
    }
}
dependencies {
  compile 'com.android.support:multidex:1.0.0'
}
```



参考链接：

* https://developer.android.com/studio/build/index.html	"Gradle插件用户指南"
* https://developer.android.com/studio/build/multidex.html

