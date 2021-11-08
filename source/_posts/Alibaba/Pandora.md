---
title: 从Pandora到PandoraBoot
date: 2020-05-28
categories: JAVA
tags: 
- JVM
- JAVA
- ClassLoader
---

看过[Pandora](http://gitlab.alibaba-inc.com/middleware-container/pandora/wikis/home)文档的一些介绍——轻量级的依赖隔离容器，在我脑子里浮现了这几个名词：Tomcat部署多应用、OSGI、Java9模块化。谷歌“依赖隔离”这个关键词出来的结果：类加载机制、蚂蚁开源的[sofa-ark](https://github.com/sofastack/sofa-ark)。所以读了很多文章想看看这些名词之间有什么联系。

# 1、从Tomcat说起

现在SpringBoot推崇内嵌的Servlet容器，直接把Tomcat内嵌到jar包里了。但在早期业务尚不复杂应用还依赖JSP的时候，Tomcat就像一艘航母，承载着若干个轻量级的应用。

其中有个值得思考问题是，整个Tomcat就是一个Java进程，假若Tomcat上有两个应用都依赖spring-framework，但版本不一致，Tomcat如何决定使用哪个版本的依赖呢？

![Tomcat容器](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuSf9JIjHACbNACfCpoXHICaiIaqkoSpFuyhBJqbL2CdFJKuiKQZcKb28TYmeC8o5CenYkMgvG08Akhfs2i45HPbvwSPw1bmWAIGX4w2GGsfU2j0U0000)

答案是两个版本都存在。这个答案会引起更大的疑问，Java是靠全类名来标识两个类的，如果两个应用用到不同版本的`org.springframework.context.annotation.AnnotationConfigApplicationContext`，那这两个类是如何在一个Java进程中共存的呢？

这个问题需要我们对类加载机制有深入的了解才能解答。

# 2、Java类加载机制

当我们把java代码编译后打包成`jar`，我们用到这个jar包的时候只需在`classpath`中加上这个依赖就行。

```java
import java.util.List;
import com.google.common.collect.Lists;

public class ClassPathTest {
    public static void main(String[] args) {
        List<String> list = Lists.newArrayList();
        list.add("a");
        list.add("b");
        list.add("c");
        System.out.println(list);
    }
}
```
比如我们要用到Guava的`Lists`工具类，编译和运行时都需要加上`classpath`。
```bash
javac -classpath ~/.m2/repository/com/google/guava/guava/20.0/guava-20.0.jar ClassPathTest.java
# 默认的classpath是当前目录，如果指定了classpath，需要追加当前目录
java -classpath ~/.m2/repository/com/google/guava/guava/20.0/guava-20.0.jar:. ClassPathTest
```

如果这里直接改用`new ArrayList()`，就不需要指定`classpath`。这是因为JDK里的`ArrayList`和Guava的`Lists`是被不同的类加载器加载的。前者被BootstrapClassLoader加载，后者由SystemClassLoader加载。

### 2.1、类加载器的定义

ClassLoader中定义了JDK默认的类加载机制：

```java
public abstract class ClassLoader {
    /// ...
    // 用于委托的父加载器
    // Note: VM硬编码了这个字段的offset, 所以新字段都需要添加到这个字段的后面
    private final ClassLoader parent;
    //...
    // 默认父加载器为SystemClassLoader
    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先检查类是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // 父类加载器没找到不抛异常
                }

                if (c == null) {
                    // 父类加载器没找到，调用当前类加载器的findClass()
                    c = findClass(name);
										///...
                }
            }
            if (resolve) { // 根据入参决定是否解析该类
                resolveClass(c);
            }
            return c;
        }
    }
    // 由子类实现加载类的逻辑
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
   ///...
}
```

ClassLoader的定义中有一个非常重要的字段就是`parent`，从`loadClass`的代码可以看出Java类加载遵循所谓的“双亲委托机制”——**先看该类是否已经被加载过，如果没有从父类加载器中加载该类，如果父类加载器没找到再调用当前类的加载器中定义的`findClass`去加载**。

### 2.2、Java内置的三个类加载器

Java中默认内置3个类加载器：

![类加载机制](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuOfsoiylAIufIYnmpaaiBlR9Jqn9BOhbYdQjA44LS2n0K-5SMboIduiaPeXDq0YRe9wUNYmNDeiLR7Hr5L2jve9p4IfGtS85vo9KO3gEA5L6-5KXdC_ba9gN0Wm_0000)

* [Bootstrap ClassLoader](http://hg.openjdk.java.net/jdk8u/jdk8u60/hotspot/file/37240c1019fd/src/share/vm/classfile/classLoader.hpp)在虚拟机层，用C++编写。用于加载`rt.jar`等运行时基础类库，也被称作“Root ClassLoader”。

* [ExtClassLoader](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/935758609767/src/share/classes/sun/misc/Launcher.java)是用于加载`JAVA_HOME/lib/ext/*.jar`目录的JDK扩展类库。是JDK2.0引入[标准扩展机制](https://docs.oracle.com/javase/7/docs/technotes/guides/extensions/spec.html)时添加的类加载器。

* [AppClassLoader](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/935758609767/src/share/classes/sun/misc/Launcher.java)是用于加载类路径下的第三方类库，也被称为“SystemClassLoader”，可以通过`ClassLoader.getSystemClassLoader()`获取。

应用也可以自定义类加载器。比如去加载网络服务器上的某个jar包，则可以继承[`ClassLoader`](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html)或[`URLClassLoader`](https://docs.oracle.com/javase/8/docs/api/java/net/URLClassLoader.html)请求网络进行加载。

```java
public class ClassPathTest {
    public static void main(String[] args) {
        // JDK Extendsions目录
        System.out.println(System.getProperty("java.ext.dirs"));
        // 类路径，默认为当前工作目录，可以通过"-classpath"或"-cp"变量修改
        System.out.println(System.getProperty("java.class.path"));
        System.out.println("---");
        // sun.misc.Launcher$AppClassLoader@18b4aac2
        // sun.misc.Launcher$ExtClassLoader@33833882
        printClassLoaderTree(new ClassPathTest());
        // 因为Object是BootstrapClassLoader加载，所以不会打印
        printClassLoaderTree(new Object());
    }

    private static void printClassLoaderTree(Object target) {
        ClassLoader classLoader = target.getClass().getClassLoader();
        while (classLoader != null) {
            System.out.println(classLoader);
            classLoader = classLoader.getParent();
        }
    }
}
```

> **为什么要遵循双亲委派机制?**
>
> * 保证核心类的安全。防止开发者取了和jdk核心类库中一样的包名和类名，委托给父类加载器能保证JDK类库的类优先加载。
> * 保证类的唯一性。先检查是否加载过这个类，避免相同类被多次加载。

# 3、打破双亲委派机制

说回Tomcat的问题。要让一个Java进程同时加载Spring3.0和Spring4.0两个版本的类，按照JDK自带的双亲委派模型是没法解决的。因为`ClassLoader#loaderClass`默认会检查这个类有没有加载过，保证了类在进程中是唯一的。如果我们想加载两个版本的类，需要打破原有的模型：

```java
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class Test {

  public static void main(String[] args) throws ClassNotFoundException {
    ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
    ClassLoader customClassLoader1 = new MyClassLoader();
    ClassLoader customClassLoader2 = new MyClassLoader();

    Class<?> test1 = customClassLoader1.loadClass("Test");
    Class<?> test2 = customClassLoader2.loadClass("Test");
    Class<?> test3 = systemClassLoader.loadClass("Test");

    System.out.println(test1 == test2); // false
    System.out.println(test1 == test3); // false
    System.out.println(test1.getClassLoader()); //Test$MyClassLoader@5c647e05
    System.out.println(test2.getClassLoader()); //Test$MyClassLoader@55f96302
    System.out.println(test3.getClassLoader()); //sun.misc.Launcher$AppClassLoader@2a139a55
  }

  private static class MyClassLoader extends ClassLoader {

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
      // 先检查自己能不能加载这个类
      Class<?> clazz = findClass(name);
      if (clazz == null) {
        // 自己不能加载，就走原来的加载逻辑
        return super.loadClass(name, resolve);
      } else if (resolve) {
        resolveClass(clazz);
      }
      return clazz;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
      String path = name.replace('.', File.separatorChar) + ".class";
      try {
        // 读取类文件
        byte[] bytes = Files.readAllBytes(Paths.get(path));
        return defineClass(name, bytes, 0, bytes.length);
      } catch (IOException e) {
        return null;
      }
    }
  }
}
```

上面这个例子重写了`loadClass`方法，把`findClass`方法放在前面调用，让`Test`类能够被重复加载多次。

### 3.1、Tomcat的类加载器

从代码中可以看出Tomcat会先从war包的`/WEB-INF/classes`目录尝试加载类，如果失败了再委托给parent加载器。而每个[WebApp都会有自己的WebappClassLoader](https://github.com/apache/tomcat/blob/9.0.35/java/org/apache/catalina/loader/WebappLoader.java#LC382)，这样就可以保证每个Webapp的依赖类相互隔离了。

```java
public abstract class WebappClassLoaderBase extends URLClassLoader
        implements Lifecycle, InstrumentableClassLoader, WebappProperties, PermissionCheck {

    private static final String CLASS_FILE_SUFFIX = ".class";
    protected final ClassLoader parent;
    // JavaSE ClassLoader也就是BootstrapClassLoader
    private ClassLoader javaseClassLoader;
    // 缓存这个类加载器已经加载的类
    protected final Map<String, ResourceEntry> resourceEntries =
            new ConcurrentHashMap<>();
  
    protected WebappClassLoaderBase(ClassLoader parent) {

        super(new URL[0]);

        // 没有设置parent，就使用SystemClassLoader作为parent
        ClassLoader p = getParent();
        if (p == null) {
            p = getSystemClassLoader();
        }
        this.parent = p;

        // BootstrapClassLoader
        ClassLoader j = String.class.getClassLoader();
        if (j == null) {
            j = getSystemClassLoader();
            while (j.getParent() != null) {
                j = j.getParent();
            }
        }
        this.javaseClassLoader = j;
        // ...
    }
  
    @Override
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            Class<?> clazz = null;

            // (0) 检查已加载的本地缓存的类
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }

            // (0.1) 检查Native层的类缓存
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }

            // (0.2) 尝试用BootstrapClassLoader加载，防止WebApp重写JavaSE的类
            String resourceName = binaryNameToPath(name, false);

            ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {
                // 用getResource先尝试加载一下，避免触发开销比较大的ClassNotFoundException异常
                tryLoadingFromJavaseLoader = (javaseLoader.getResource(resourceName) != null);
            } catch (Throwable t) {
                // 处理特殊的异常
                // https://bz.apache.org/bugzilla/show_bug.cgi?id=58125
                // https://bz.apache.org/bugzilla/show_bug.cgi?id=61424
                ExceptionUtils.handleThrowable(t);
                tryLoadingFromJavaseLoader = true;
            }

            if (tryLoadingFromJavaseLoader) {
                try {
                    clazz = javaseLoader.loadClass(name);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (0.5) 权限检查
            // ... 省略

            boolean delegateLoad = delegate || filter(name, true);

            // (1) 针对EL表达式、Servlet API、WebSocket API、以及Tomcat内部实现类使用parent去加载
            if (delegateLoad) {
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (2) Search local repositories
            try {
                clazz = findClass(name);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }

            // (3) 前面都没找到该类，就无条件交给parent去加载
            if (!delegateLoad) {
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }
        }

        throw new ClassNotFoundException(name);
    }
    @Override
    public Class<?> findClass(String name) throws ClassNotFoundException {
        // ...
        Class<?> clazz = null;
        try {
            try {
                // 加载war包中/WEB-INF/classes目录下的类
                clazz = findClassInternal(name);
            } catch(AccessControlException ace) {
                throw new ClassNotFoundException(name, ace);
            } catch (RuntimeException e) {
                throw e;
            }
            if (clazz == null) {
                clazz = super.findClass(name);
            }
            if (clazz == null) {
                throw new ClassNotFoundException(name);
            }
        } catch (ClassNotFoundException e) {
            throw e;
        }
        return clazz;

    }
    //...
}
```

# 4、菱形依赖问题

前面说到的Tomcat这种场景需要在一个进程中加载两个不同版本依赖。

推而广之，还有软件开发过程中经常碰到的[菱形依赖问题(Diamond Dependency)](https://en.wikipedia.org/wiki/Dependency_hell)。

![Diamond Dependency](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuSf9JIjHACbNACfCpoXHICaiIaqkoSpFuufsB2Y8LT3LjLE8zibCSen54t022e344IBEiLPbgKN5GBqAXc0v9wnoHbmEgNafG9i1)

[Maven](https://en.wikipedia.org/wiki/Apache_Maven)作为一个Java领域的依赖管理工具，提供了[`exclusion`标签](https://maven.apache.org/guides/introduction/introduction-to-optional-and-excludes-dependencies.html)来排除`LibC`这样的传递依赖，或者直接依赖高版本的库。但这个前提是高版本的依赖需要兼容低版本。向前兼容要保留恶心的祖传代码，这对于有代码洁癖的程序员来说是个极其艰巨的任务，所以除了JDK标准库大多数三方依赖库在升级大版本时会有各种兼容问题，这也是为什么JDK中保留着`Vector`和`StringBuffer`这样的上古代码的原因。

如果真出现了`LibA`和`LibB`依赖的版本差别大无法兼容，`NoClassDefFoundError`、`NoSuchMethodError`等各种错误就会接踵而至，那怎么办呢？

![NoSuchMethodError](http://tva1.sinaimg.cn/large/bda5cd74ly1gf5v05pz5hj20m00sytdk.jpg)

一种方式是直接把别人的代码拷过来，换个包名。这种方式简单粗暴，也许会觉得这个方式很low逼，但其实用的人挺多的，而且不乏业界名流，`spring-framework`就是这样[把`cglib`的代码](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cglib/package-summary.html)拷过来的。但这种方式仅局限于cglib这样没有其他依赖的短小精悍的库。

另一种方式就是之前说的通过打破双亲委派模型的类隔离机制。业界比较知名的就是[OSGI](https://www.baeldung.com/osgi)，Eclipse中的各种插件相互隔离就是靠OSGI实现，而且还支持插件的动态插拔。OSGI联盟野心很大，曾一度想让OSGI成为Java模块化技术的标准，不过[Java9在语法层面提供了JPMS标准](https://www.baeldung.com/java-9-modularity)，直接颠覆了原有的模块化管理方式。

# 5、从OSGI到Pandora

最初，HSF 1.X为了解决与应用的jar冲突问题，使用OSGi来做隔离。当时淘系大部分的应用都运行在JBoss中，`.sar` 作为JBoss支持的一种部署格式（与 `.war`类似），它在JBoss中的默认启动顺序早于`.war`，符合HSF优先于应用启动完成类导出的需求，因此HSF 1.X的部署包被定为`taobao-hsf.sar`。

随着集团的业务发展，内部已经有很多诸如HSF、Notify、MetaQ、Diamond、Tair等各种中间件或客户端产品。这些二方包被各个业务系统使用，为了能解决三方包依赖冲突、方便大规模升级并控制二方包升级成本等问题，从HSF 2.X起，“隔离”的功能被独立地交付给Pandora。这时候的“隔离”不再是“HSF与应用的隔离”，而是“中间件与应用的隔离”以及“中间件之间的隔离”。Pandora容器废弃了OSGI框架，只引入了它的隔离机制，重新实现ClassLoader，形成了全新的轻量级隔离容器。

由于线上大量启动脚本已经写死了`taobao-hsf.sar`，为了降低风险，所以[Pandora](http://gitlab.alibaba-inc.com/middleware-container/pandora/tree/develop)独立成隔离容器后，仍然沿用了原有的名字。

![pandora](http://tva1.sinaimg.cn/large/bda5cd74ly1gf2a9cfdfdj20gw0a3abm.jpg)

和Tomcat类似，每个Pandora Plugin模块都有自己的[ModuleClassLoader](http://gitlab.alibaba-inc.com/middleware-container/pandora/blob/develop/pandora.container/src/main/java/com/taobao/pandora/service/loader/ModuleClassLoader.java)，这样就能保证每个中间件Plugin相互隔离。

# 6、PandoraBoot

受Ruby on Rails“约定大于配置”思想的影响，Pivotal基于Spring3.0的注解配置和Spring4.0的@Conditional Bean，[开发了支持AutoConfiguration的SpringBoot](https://spring.io/blog/2013/12/12/announcing-spring-framework-4-0-ga-release)，大大简化了应用的配置。

而PandoraBoot则将SpringBoot和Pandora进行了整合。让开发可以既享受到SpringBoot简化配置的福利，又能带来Pandora对依赖隔离的功能。

原来的中间件以插件形式加入到`taobao-hsf.sar`中，最后sar包越来越大，而PandoraBoot将sar包Maven化，发布到Maven仓库中，可以在Maven依赖中添加`taobao-hsf.sar`的依赖，并按需添加相应插件的[spring-boot-starter](http://mw.alibaba-inc.com/products/pandoraboot/_book/spring-boot-hsf.html)来整合Pandora Plugin，最终sar包和依赖的插件都可以打包到FatJar中。

> 是否将sar包和插件打包到FatJar是可选的，具体可以参考[Pandora-Boot-Maven-Plugin](http://mw.alibaba-inc.com/products/pandoraboot/_book/maven-plugin.html)
>
> **在日常/线上机器上，都是通过脚本里的`-Dpandora.location`来加载sar包的。** sar包位置是在`/home/admin/$appName/target/taobao-hsf.sar`。
>
> 在pandora-boot 2.1.3版本之后，`taobao-hsf.sar`变成一个空的jar包，它引入了`taobao-hsf.sar-container`和其它的插件。具体可以参考PandoraBoot文档《[对`taobao-hsf.sar`的形式和位置](http://mw.alibaba-inc.com/products/pandoraboot/_book/taobao-hsf-sar.html)》一节。

### 6.1、SpringBoot FatJar

SpringBoot会将应用以及相关的依赖打包成一个[FatJar](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-executable-jar-format.html)，只需要`java -jar`命令即可启动应用，这是因为SpringBoot的maven构建插件会将`MANIFEST.MF`中的`Main-Class`替换成[JarLauncher](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/loader/JarLauncher.html)，SpringBoot定义好针对FatJar的类加载器后，再去调SpringBoot的`Start-Class`的入口方法。

![SpringBoot FatJar](http://tva1.sinaimg.cn/large/bda5cd74ly1gf5yu47atmj20mc0ncwh7.jpg)

但SpringBoot不同的是，PandoraBoot加载的不是简单的jar包，有一些二方包是支持依赖隔离的Pandora插件，这些插件包中包含了自己的依赖jar包。

![Plugin结构](http://tva1.sinaimg.cn/large/bda5cd74ly1gf8cbpg52qj20qg0fy11d.jpg)

这些插件需要和`taobao-hsf.sar`中的插件一样进行依赖隔离。所以PandoraBoot基于SpringBoot的JarLauncher扩展了SarLauncher，再由[SarLoaderUtils](http://gitlab.alibaba-inc.com/middleware-container/pandora-boot/blob/develop/pandora-boot-loader/src/main/java/com/taobao/pandora/boot/loader/SarLoaderUtils.java)加载`sar`包和外部的插件。









**参考链接：**

Wikipedia：https://en.wikipedia.org/wiki/Java_Classloader

Java classes and class loading：https://www.ibm.com/developerworks/java/library/j-dyn0429/

Class Loaders in Java：https://www.baeldung.com/java-classloaders

Find a way out of the ClassLoader maze：https://www.javaworld.com/article/2077344/find-a-way-out-of-the-classloader-maze.html

老大难的 Java ClassLoader 再不理解就老了：https://zhuanlan.zhihu.com/p/51374915

Do You Really Get ClassLoaders：https://www.jrebel.com/blog/how-to-use-java-classloaders

深入浅出ClassLoader(译)：https://www.atatech.org/articles/33671

Tomcat Class Loader How to：http://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html

Pandora VS Hilton：https://www.atatech.org/articles/94481

Pandora Documents：http://gitlab.alibaba-inc.com/middleware-container/pandora/wikis/home

Pandora实现原理：http://gitlab.alibaba-inc.com/middleware-container/pandora/wikis/implementation

Pandora Container 轻量级隔离容器 -- 简介、基本原理、使用：https://www.atatech.org/articles/43952

下一代轻量级容器——Pandora（潘多拉）之隔离原理详解：https://www.atatech.org/articles/2640

Pandora Framework的实现原理：http://gitlab.alibaba-inc.com/middleware-container/pandora-framework/wikis/pandora-framework-whatis

Java中隔离容器的实现：http://codemacro.com/2015/09/05/java-lightweight-container/

SOFAArk介绍：https://www.sofastack.tech/projects/sofa-boot/sofa-ark-readme/

Introduction to OSGi：https://www.baeldung.com/osgi

Java 9, OSGi and the Future of Modularity (Part 1)：https://www.infoq.com/articles/java9-osgi-future-modularity/

Java 9, OSGi and the Future of Modularity (Part 2)：https://www.infoq.com/articles/java9-osgi-future-modularity-part-2/

Java 9, OSGi and the Future of Modularity (Part 1)[中文]：https://www.infoq.cn/article/java9-osgi-future-modularity

Java 9, OSGi and the Future of Modularity (Part 2)[中文]：https://www.infoq.cn/article/java9-osgi-future-modularity-part-2

深入理解OSGI：Java模块化之路：https://www.cnblogs.com/garfieldcgf/p/6378443.html

Classloader-Related Memory Issues：https://www.dynatrace.com/resources/ebooks/javabook/class-loader-issues/

sofa-ark类隔离技术分析调研：https://blog.mythsman.com/post/5d29b12c373f140fc98304a1/

The parent class loader delegation model Detailed：https://codesolu.com/2020/01/07/the-parent-class-loader-delegation-model-detailed/

JVM核心知识体系：https://www.atatech.org/articles/135439
