---
title: 【译】Cglib缺失的文档
date: 2020-03-17
categories: JAVA
---

> 本文翻译自：https://dzone.com/articles/cglib-missing-manual

作为[字节码](https://en.wikipedia.org/wiki/Java_bytecode)库，cglib是许多著名的java框架（[Hibernate](https://hibernate.org/)、[Spring](https://spring.io/)等）比较流行的选择。字节码库允许在java应用的编译阶段之后，操作或者动态创建新的class。由于Java类在运行时动态链接，因此可以向正在运行的Java程序中添加新类。比如，Hibernate就会将cglib用于动态代理的生成。Hibernate将返回存储类的检测版本，该版本仅在需要时才从数据库延迟加载某些值，而不是返回存储在数据库中的完整对象。又比如，Spring用cglib给你的方法调用添加安全性规则。

[Spring Security](https://spring.io/projects/spring-security)在调用方法时，会首先检查指定的安全性检验是否通过，仅当校验通过后才调用到具体的方法，而不是直接就去调用你的方法。cglib另外一个更普及的应用场景是在mock框架中，比如mockito。那些mocks只不过是仪式化的类，类中的方法被替换成空的实现（同时会添加一些跟踪逻辑）。

除了[ASM](http://asm.ow2.org/)之外——另外一个字节码库，cglib基于[ASM](https://asm.ow2.io/)提供一些高级别的字节码操作功能——cglib提供了底层的字节码转换，可以让用户在不了解任何Java类编译细节的下使用。不幸的是，cglib的文档很短，甚至可以说基本上没有。除了一篇[2005年写的介绍Enhancer类的博客](http://jnb.ociweb.com/jnb/jnbNov2005.html)之外，别无他物。这篇博客将尝试着演示cglib和它那些不幸的很少使用的API。

## nhancer

让我们从`Enhancer`类开始讲解，该类可能是cglib库中最常用的类了。`Enhancer`可以为一个不实现任何接口的类创建代理。可以将`Enhancer`与标准库中Java 1.3引入的[`Proxy`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)类进行比较。`Enhancer`会动态地为指定类创建子类，但该子类的所有方法调用都会被拦截。与`Proxy`不一样，它对类和接口类型均适用。

后续的例子会基于以下的POJO进行演示：

```java
public class SampleClass {
  public String test(String input) {
    return "Hello world!";
  }
}
```

有了cglib，可以使用`Enhancer`和`FixedValue`回调轻松将`test(String)`方法的返回值替换为另一个值：

```java
@Test
public void testFixedValue() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
      return "Hello cglib!";
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
}
```

在上面的示例中，`enhancer`将会返回一个的`SampleClass`的仪式化子类的实例，该实例所有方法调用都会返回一个固定值，该值是由上面实现的`FixedValue`匿名类生成。这个对象由`Enhancer#create(Object...)`创建的，该方法可传入多个参数用于决定调用被增强类的哪个构造方法。（尽管在Java字节码层面，构造方法也只是个方法，但`Enhancer`不能插入构造函数。同时`Enhancer`也不能插入`static`或者`final`类。）如果你只想创建一个增强类，而非其实例，用`Enhancer#createClass`方法可以创建`Class`实例，用它就可以动态地创建实例对象了。在这个动态生成的类中所有的构造函数都被委托给被增强类的托构造函数。

另一个结论是**final方法不会被拦截**，比如，当`Object#getClass`被调用时将返回一个类似于`SampleClass$$EnhancerByCGLIB$$e277c63c`的东西。这个类名是cglib为了避免类名冲突随机生成的。当你想要在程序代码中使用显式类型时，请注意每次执行生成增强实例的类不同。不过，cglib生成的类与被增强的类在同一个包下（因此可以覆盖package-private方法）。与final方法类似，通过生成子类实现增强的方案**不能增强final类**。正因此，像Hibernate这样的框架不能持久化final类。

接下来，让我们来看一个更强大的callback。`InvocationHandler`也能和`Enhancer`一起使用：

```java
@Test
public void testInvocationHandler() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) 
        throws Throwable {
      if(method.getDeclaringClass() != Object.class 
          && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        throw new RuntimeException("Do not know what to do.");
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
}
```

这个callback，可以让我们根据被调的方法进行回答。不过需要注意的是，在`InvocationHandler#invoke`中调用代理对象上的方法要格外小心。代理对象上的所有方法调用，都会分发给同一个`InvocationHandler`，从而可能导致无限循环。为了避免这种情况，我们可以使用另一个回调——MethodInterceptor：

```java
@Test
public void testMethodInterceptor() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
        throws Throwable {
      if(method.getDeclaringClass() != Object.class 
          && method.getReturnType() == String.class) {
        return "Hello cglib!";
      } else {
        proxy.invokeSuper(obj, args);
      }
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode();// Does not throw an exception or result in an endless loop.
}
```

`MethodInterceptor`允许我们完全控制被拦截的方法，并提供了一些工具用于调用被增强类的原始方法。但是既然有了`MethodInterceptor`为什么仍然要使用其他方法？因为其他方法效率更高，而cglib通常还会用于效率至关重要的边缘案例框架。比如`MethodInterceptor`的创建和链接需要生成不同类型的字节码并创建一些`InvocationHandler`不需要的运行时对象。因此，还有其他可以和`Enhancer`一起使用的类：

-  **LazyLoader**：尽管`LazyLoader`仅有的一个方法与`FixedValue`有相同的方法签名，但是`LazyLoader`与`FixedValue`还是有本质上的区别的。`LazyLoader`其返回的是增强子类的实例，这个实例仅在第一次访问其方法时才返回，然后缓存该实例并用于后续调用。如果你的对象创建比较费事儿而且又不知道该对象是否会被使用，那适合用它。需要注意的是，不管是proxy对象还是懒加载对象，都**只能使用被增强类的构造方法来创建对象**。因此，请确保被增强类拥有一个不耗时的构造方法（可能是protected的），或者将接口类型用作代理。你可以通过香`Enhancer#create(Object...)`提供参数来选择被调的构造方法。
-  **Dispatcher**：`Dispatcher`与`LazyLoader`相似，但是`Dispatcher`会在每个方法调用时都被调用而不存储已加载的对象。这使得在不改变引用对象的情况下，切换其类的实现。同样需要注意的是，为了代理和生成对象构造函数必须被调用。
-  **ProxyRefDispatcher**：在这个类的方法签名中携带了一个指向代理对象的引用。这就允许将一个方法的调用代理到另外一个方法上去。需要注意这种使用方式很容易导致无限循环，特别是在`ProxyRefDispatcher#loadObject(Object)`方法中始终调用同一个方式时必然会导致无限循环。
-  **NoOp**：不像`NoOp`类的名字所暗示的那样什么都不做，而是直接将方法的调用委托给被增强类的方法实现。

现在看来，最后两个拦截器可能不会引起你的注意。总是将方法调用委派给被增强类，为什么还要去增强一个类呢？你是对的，这些拦截器只会与CallbackFilter结合起来使用，下面是示例代码：

```java
@Test
public void testCallbackFilter() throws Exception {
  Enhancer enhancer = new Enhancer();
  CallbackHelper callbackHelper = new CallbackHelper(SampleClass.class, new Class[0]) {
    @Override
    protected Object getCallback(Method method) {
      if(method.getDeclaringClass() != Object.class 
          && method.getReturnType() == String.class) {
        return new FixedValue() {
          @Override
          public Object loadObject() throws Exception {
            return "Hello cglib!";
          };
        }
      } else {
        return NoOp.INSTANCE; // A singleton provided by NoOp.
      }
    }
  };
  enhancer.setSuperclass(MyClass.class);
  enhancer.setCallbackFilter(callbackHelper);
  enhancer.setCallbacks(callbackHelper.getCallbacks());
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
  assertNotEquals("Hello cglib!", proxy.toString());
  proxy.hashCode(); // Does not throw an exception or result in an endless loop.
}
```

`Enhancer`的方法`Enhancer#setCallbackFilter(CallbackFilter)`接收一个`CallbackFilter`参数，这个方法期望被增强类的方法调用都被映射到`Callback`实例数组的数组索引。当调用proxy的方法时，`Enhancer`会选择相应的拦截器，并在相应的`Callback`上转发相应的方法(这个`Callback`是目前引入的所有拦截器的标记接口)。为了让`CallbackFilter`的创建不那么费劲，cglib提供了一个`CallbackHelper`。该类代表了`CallbackFilter`，同时回为你创建一组`Callback`。上面事例中的增强对象功能上等同于`MethodInterceptor`示例的对象，但是CallbackFilter允许你将编写特定的Callback逻辑与分发逻辑分开编写(解耦)。

**How does it work?**

当Enhancer创建一个增强类时，它会为每一个Callback创建一个private static变量，且该操作是在被代理类创建之后执行的。这就意味着，cglib创建的类不能被复用，因为注册的callback不会成为类定义的一部分，而只是cglib在JVM加载类之后手动添加的。同时，从技术层面来说由cglib创建的类在初始化后是还未达到ready状态的，比如该类不能通过网络发送到另一台机器，因为在目标机器上，该类可能并不存在。
 对于不同的Callback类，cglib可能会注册不同的额外变量。比如，MethodInterceptor就会注册两个private static变量(一个用于保存Method的反射，另一个是MethodProxy的反射)到代理的每个方法中。需要注意的是，MethodProxy会过渡使用FastClass，而FastClass的创建会触发额外的类的创建，后面将会详细介绍FastClass。
 由于以上所有原因，请在使用Enhancer的时候多多注意。在注册callback的时候需要格外小心，比如MethodInterceptor会额外创建一些类，同时还会在增强类中注册额外的static变量。这在将callback保存为静态变量的时，尤为危险：这可能会隐式的导致从不会对增强类进行垃圾回收(除非是它的Classloader被回收)。另外一种危险的情况是，使用匿名类时，会使用到其外层类的引用。回想一下上面的事例：

```java
@Test
public void testFixedValue() throws Exception {
  Enhancer enhancer = new Enhancer();
  enhancer.setSuperclass(SampleClass.class);
  enhancer.setCallback(new FixedValue() {
    @Override
    public Object loadObject() throws Exception {
      return "Hello cglib!";
    }
  });
  SampleClass proxy = (SampleClass) enhancer.create();
  assertEquals("Hello cglib!", proxy.test(null));
}
```

这个FixedValue的匿名子类，将会变得很难从被增强类SampleClass中进行引用，因此这个匿名的子类以及包含这个@Test放到的类将永远不会被垃圾回收，这将会导致严重的内存泄露。因此，**不要在cglib中使用非静态类**（我在这篇博客中使用匿名类仅仅是为了让事例更短小些）。
 最后，**千万不要拦截Object#finalize()方法！**由于cglib是通过子类方式实现的代理，所以finalize方法会被覆盖，但是覆盖finalize[通常不是个好主意](https://howtodoinjava.com/java/basics/why-not-to-use-finalize-method-in-java/)。这些拦截了finalize方法的增强类事例不会被垃圾回收器特别对待，同样会被放入JVM的finalization队里。如果你不小心在callback中硬编码应用了被增强类，那么你就创建了一个永远不被回收的实例。以上问题通常是你不希望发生的。庆幸的是，cglib不会代理所有的final方法，因此Object#wait, Object#notify和Object#notifyAll方法不会遇到这些问题。需要注意的是Object#clone是会被代理的，这通常是你不希望发生的。

## mmutableBean

cglib库`ImmutableBean`允许创建一个不可变的包装器，这个类似于`Collections#immutableSet`。底层bean的所有修改操作都会被阻止，并抛出`IllegalStateException`异常(不是用java API建议的`UnsupportedOperationException`)。让我们来看一些Bean：

```java
public class SampleBean {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

我们可以让这个bean变成immutable的：

```java
@Test(expected = IllegalStateException.class)
public void testImmutableBean() throws Exception {
  SampleBean bean = new SampleBean();
  bean.setValue("Hello world!");
  SampleBean immutableBean = (SampleBean) ImmutableBean.create(bean);
  assertEquals("Hello world!", immutableBean.getValue());
  bean.setValue("Hello world, again!");
  assertEquals("Hello world, again!", immutableBean.getValue());
  immutableBean.setValue("Hello cglib!"); // Causes exception.
}
```

从示例看显而易见，ImmutableBean阻止了对bean的所有状态修改，并且会抛出`IllegalStateException`异常。然而，可以通过原始对象来修改Bean的状态，并且所有更改将会反映到`ImmutableBean`上。

## eanGenerator

`BeanGenerator`是cglib的另一个Bean工具类，它可以在运行时动态地创建bean对象：

```java
@Test
public void testBeanGenerator() throws Exception {
  BeanGenerator beanGenerator = new BeanGenerator();
  beanGenerator.addProperty("value", String.class);
  Object myBean = beanGenerator.create();
  
  Method setter = myBean.getClass().getMethod("setValue", String.class);
  setter.invoke(myBean, "Hello cglib!");
  Method getter = myBean.getClass().getMethod("getValue");
  assertEquals("Hello cglib!", getter.invoke(myBean));
}
```

从示例看显而易见，`BeanGenerator`会首先通过`addProperty`方法设置属性名和属性类型键值对。在创建时，`BeanGenerator`会为属性创建getter和setter的访问方法：

- `<type> get<name>()`
- `void set<name>(<type>)`

有一些依赖于Cglib的第三方库期望通过反射决定这些bean，在运行时不知道这些bean时，这种情况下BeanGenerator可能很有用。(这种场景的一个应用实例是[Apache Wicket](http://wicket.apache.org/))。

## eanCopier

`BeanCopier`是另一个Bean工具类，它可以复制bean的属性值。假设现在有一个与`SampleBean`拥有相同属性的bean：

```java
public class OtherSampleBean {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

现在，你就可以将属性从一个Bean拷贝到另一个Bean了：

```java
@Test
public void testBeanCopier() throws Exception {
  BeanCopier copier = BeanCopier.create(SampleBean.class, OtherSampleBean.class, false);
  SampleBean bean = new SampleBean();
  myBean.setValue("Hello cglib!");
  OtherSampleBean otherBean = new OtherSampleBean();
  copier.copy(bean, otherBean, null);
  assertEquals("Hello cglib!", otherBean.getValue());  
}
```

这种拷贝不受特定类型限制。`BeanCopier#copy`方法可以传递一个可选的`Converter`参数，这个参数允许在复制过程中对每个属性做更多细致的操作。如果`BeanCopier`在通过create方法创建时，第三个参数传了false。拷贝时会忽略第三个`Converter`参数，所以这里的例子直接传了个`null`。

## ulkBean

`BulkBean`可以通过数组指定Bean的一组访问方法来对属性进行批量操作。

```java
@Test
public void testBulkBean() throws Exception {
  BulkBean bulkBean = BulkBean.create(SampleBean.class, 
      new String[]{"getValue"},   // 传一组getter
      new String[]{"setValue"},   // 传一组setter
      new Class[]{String.class}); // 指定这组属性相应的类型
  SampleBean bean = new SampleBean();
  bean.setValue("Hello world!");
  // getPropertyValues会返回这一组属性
  assertEquals(1, bulkBean.getPropertyValues(bean).length);
  assertEquals("Hello world!", bulkBean.getPropertyValues(bean)[0]);
  bulkBean.setPropertyValues(bean, new Object[] {"Hello cglib!"});
  assertEquals("Hello cglib!", bean.getValue());
}
```

`BulkBean`需要一个getter数组、一个setter数组和一个相应属性类型的数组来作为构造参数。然后可以通过`BulkBean#getPropertyValues(Object)`方法提取出一个属性数组。相应地，可以通过`BulkBean#setPropertyValues(Object, Object[])`方法设置一组属性。

## eanMap

这是cglib库中的最后一个bean的工具类，`BeanMap`会将bean的所有属性转换为`String`到`Object`的键值对映射`Map`：

```java
@Test
public void testBeanGenerator() throws Exception {
  SampleBean bean = new SampleBean();
  BeanMap map = BeanMap.create(bean);
  bean.setValue("Hello cglib!");
  assertEquals("Hello cglib", map.get("value"));
}
```

另外，`BeanMap#newInstance(Object)`方法可以重用相同`Class`来为其他Bean创建BeanMap。

## eyFactory

`KeyFactory`允许创建由多个值组成的Key，这些Key可以在`Map`等实现中使用。为了达到这个目的，`KeyFactory`需要一个接口，该接口用于定义组成key需要的值。这个接口需要有一个方法，方法名必须为`newInstance`，返回值必须为Object实例，比如：

```java
public interface SampleKeyFactory {
  Object newInstance(String first, int second);
}
```

有了接口后，就可以创建key的实例了：

```java
@Test
public void testKeyFactory() throws Exception {
  SampleKeyFactory keyFactory = (SampleKeyFactory) KeyFactory.create(SampleKeyFactory.class);
  Object key = keyFactory.newInstance("foo", 42);
  Map<Object, String> map = new HashMap<Object, String>();
  map.put(key, "Hello cglib!");
  assertEquals("Hello cglib!", map.get(keyFactory.newInstance("foo", 42)));
}
```

`KeyFactory`会确保正确实现`Object#equals(Object)`和`Object#hashCode`方法，因此生成的key可以直接用于`Map`和`Set`中。在cglib库内部，`KeyFactory`也是被频繁使用的。

## ixin

有些人可能已经从其他编程语言（如Ruby或Scala）中了解了`Mixin`类的概念，cglib的`Mixin`s允许将多个对象组合成一个对象。但是，为了实现这个功能，这些对象必须要实现接口：

```java
public interface Interface1 {
  String first();
}

public interface Interface2 {
  String second();
}

public class Class1 implements Interface1 {
  @Override 
  public String first() {
    return "first";
  }
}

public class Class2 implements Interface2 {
  @Override 
  public String second() {
    return "second";
  }
}
```

现在可以通过附加接口将类`Class1`和`Class2`组合到单个类中：

```java
public interface MixinInterface extends Interface1, Interface2 { 
/* empty */
}

@Test
public void testMixin() throws Exception {
  Mixin mixin = Mixin.create(
      new Class[]{Interface1.class, Interface2.class, MixinInterface.class}, 
      new Object[]{new Class1(), new Class2()}
  );
  MixinInterface mixinDelegate = (MixinInterface) mixin;
  assertEquals("first", mixinDelegate.first());
  assertEquals("second", mixinDelegate.second());
}
```

诚然，`Mixin`API相当笨拙，因为需要额外定义接口，这个问题可以通过非检测(non-instrumented)的Java来解决。

## tringSwitcher

`StringSwitcher`可以模拟一个`String`到int的键值对映射`Map`：

```java
@Test
public void testStringSwitcher() throws Exception {
  String[] strings = new String[]{"one", "two"};
  int[] values = new int[]{10, 20};
  StringSwitcher stringSwitcher = StringSwitcher.create(strings, values, true);
  assertEquals(10, stringSwitcher.intValue("one"));
  assertEquals(20, stringSwitcher.intValue("two"));
  assertEquals(-1, stringSwitcher.intValue("three"));
}
```

`StringSwitcher`可以模拟`String`类型的`switch`的分支逻辑，就像java7及更高版本已经内建的`swtich`一样。如果在Java 6或更低版本中使用`StringSwitcher`可能真的会给代码带来一些好处，但这是值得怀疑的，我个人是不建议使用的。

## nterfaceMaker

`InterfaceMaker`顾名思义，它可以动态地创建新Interface：

```java
@Test
public void testInterfaceMaker() throws Exception {
  Signature signature = new Signature("foo", Type.DOUBLE_TYPE, new Type[]{Type.INT_TYPE});
  InterfaceMaker interfaceMaker = new InterfaceMaker();
  interfaceMaker.add(signature, new Type[0]);
  Class iface = interfaceMaker.create();
  assertEquals(1, iface.getMethods().length);
  assertEquals("foo", iface.getMethods()[0].getName());
  assertEquals(double.class, iface.getMethods()[0].getReturnType());
}
```

与cglib库的其他API不同的是，InterfaceMarker依赖于ASM的类型。在一个正在运行的程序中创建interface几乎没有意义，因为一个interface仅仅代表着一个类型，一般是在编译器的进行类型检测时使用。当然，如果是你是要将生成的代码用于后续的开发，还是有一些用处的。

## ethodDelegate

MethodDelegate允许通过将方法调用绑定到某个接口来模拟类似于[`c#`的方法委托](http://msdn.microsoft.com/de-de/library/900fyy8e%28v=vs.90%29.aspx)，例如，下面的代码将`SampleBean#getValue`方法绑定到委托:

```java
public interface BeanDelegate {
  String getValueFromDelegate();
}

@Test
public void testMethodDelegate() throws Exception {
  SampleBean bean = new SampleBean();
  bean.setValue("Hello cglib!");
  BeanDelegate delegate = (BeanDelegate) MethodDelegate.create(
      bean, "getValue", BeanDelegate.class);
  assertEquals("Hello world!", delegate.getValueFromDelegate());
}
```

然而，有一些事情需要注意：

- 工厂方法`MethodDelegate#create`会明确接收一个方法名称作为第二个参数，这个参数是`MethodDelegate`将代理的方法；
- `MethodDelegate#create`的第一个参数，必须是一个包含无参方法的对象实例，由此可以看出`MethodDelegate`并没有达到应有的强大程度；
- 第三个参数必须是个只包含一个方法的接口。`MethodDelegate`会实现这个接口，并将对象强转为该接口。当这个接口的方法被调用时，他会调用由第一个参数指定的对象的代理方法。

此外，还应该考虑下它的这些缺点：

- cglib会为每个代理创建一个新的类，这最终浪费永久代的空间；
- 不能代理带参的方法；
- 如果你的接口方法带参，Method Deletegate根本无法正常工作，而且它不会抛出任何异常信息(方法的返回值永远都是null)。如果你的接口方法的返回类型与被代理的方法不一致(即便是被代理方法返回对象的父类)，你将会收到一个`IllegalArgumentException`异常。

## ulticastDelegate

尽管`MulticastDelegate`与`MethodDelegate`的目标功能是相似的，但是二者之间的工作方式还是会有一些区别的。为了使用`MulticastDelegate`，我们的对象需要实现一个接口：

```java
public interface DelegatationProvider {
  void setValue(String value);
}

public class SimpleMulticastBean implements DelegatationProvider {
  private String value;
  public String getValue() {
    return value;
  }
  public void setValue(String value) {
    this.value = value;
  }
}
```

基于这个实现了DelegateProvider接口的Bean，我们就可以创建一个`DelegateProvider`。这个`DelegateProvider`会将所有调用`setValue(String)`方法的请求，分发到多个实现了`DelegateProvider`接口的类：

```java
@Test
public void testMulticastDelegate() throws Exception {
  MulticastDelegate multicastDelegate = MulticastDelegate.create(
      DelegatationProvider.class);
  SimpleMulticastBean first = new SimpleMulticastBean();
  SimpleMulticastBean second = new SimpleMulticastBean();
  multicastDelegate = multicastDelegate.add(first);
  multicastDelegate = multicastDelegate.add(second);
  DelegatationProvider provider = (DelegatationProvider)multicastDelegate;
  provider.setValue("Hello world!");
  assertEquals("Hello world!", first.getValue());
  assertEquals("Hello world!", second.getValue());
}
```

同样，它也有一些缺点：

- 所有的对象都需要实现一个包含单个方法的接口，这对于第三方库来说很糟糕，当你想要使用CGlib实现一些隐藏性的操作时，是不可行的，实现这些操作的代码就会暴露到正常的代码中。其实，你可以自己轻松的实现这种代理方式（即使不通过字节码，但是我猜你自己实现会更好）。
- 当被代理方法需要返回数据时，你只能拿到最后一个对象返回的数据，其他对象返回的数据都会丢失（但是可以在某些点被multicast delegate检测到）。

## onstructorDelegate

`ConstructorDelegate`允许创建一个以字节为单位的[工厂方法](http://en.wikipedia.org/wiki/Factory_method_pattern)。要使用它，首先需要一个接口，这个接口必须包含一个名称为`newInstance`的方法，该方法的返回值必须为`Object`，这个方法可以包含任意多个参数（参数个数和类型与需要代理的构造方法相同）。例如，为了为`SampleBean`创建`ConstructorDelegate`，我们需要以下内容来调用SampleBean的默认（无参）构造函数：

```java
public interface SampleBeanConstructorDelegate {
  Object newInstance();
}

@Test
public void testConstructorDelegate() throws Exception {
  SampleBeanConstructorDelegate constructorDelegate = (SampleBeanConstructorDelegate) ConstructorDelegate.create(
    SampleBean.class, SampleBeanConstructorDelegate.class);
  SampleBean bean = (SampleBean) constructorDelegate.newInstance();
  assertTrue(SampleBean.class.isAssignableFrom(bean.getClass()));
}
```

## arallelSorter

在二维数组排序时，`ParallelSorter`声称是Java标准库的数组排序的更快替代品：

```java
@Test
public void testParallelSorter() throws Exception {
  Integer[][] value = {
    {4, 3, 9, 0},
    {2, 1, 6, 0}
  };
  ParallelSorter.create(value).mergeSort(0);
  for(Integer[] row : value) {
    int former = -1;
    for(int val : row) {
      assertTrue(former < val);
      former = val;
    }
  }
}
```

`ParallelSorter`创建时使用的是二维数组，然后就可以对第二级数组(子数组)进行归并排序或快速排序。使用时请务必小心：

- 当对原始类型进行排序时，你必须使用mergeSort方法的重载方法来手动指定排序范围(比如：e.g. 上面例子中调用`ParallelSorter.create(value).mergeSort(0, 0, 4)`排序， 其中4表示排序数组的长度)，否则`ParallelSorter`出现一个明显的bug，因为`ParallelSorter`会将基本类型的数组强制转换为Object数组，导致ClassCastException异常；
- 如果被排序的数组的长度不一致，mergeSort的第一个参数会决定采用哪一行的长度作为参考长度。长度不一致的行将导致`ArrayIndexOutOfBoundException`。

就我个人而言，我怀疑ParallelSorter是否真的在排序时间上有优势。诚然，我还没有尝试对它进行基准测试。如果你尝试过，我很乐意在评论中听到你的回复。

## astClass and FastMembers

`FastClass`承诺要提供比[Java reflection API](http://docs.oracle.com/javase/tutorial/reflect/)更快的方法调用，它包装一个Class，并提供与Java reflection API相同API：

```java
@Test
public void testFastClass() throws Exception {
  FastClass fastClass = FastClass.create(SampleBean.class);
  FastMethod fastMethod = fastClass.getMethod(SampleBean.class.getMethod("getValue"));
  MyBean myBean = new MyBean();
  myBean.setValue("Hello cglib!");
  assertTrue("Hello cglib!", fastMethod.invoke(myBean, new Object[0]));
}
```

除了`FastMethod`外，`FastClass`还可以创建`FastConstructor`，但是不能创建fast field。但是，`FastClass`怎么就比正常的反射API更快呢？Java reflection是通过JNI（Java Native Interface）执行的，JNI会调用C语言的代码执行反射方法，而`FastClass`是通过生成一些字节码直接在JVM中调用的。然而，新版本的HotSpot JVM（或许还有其他的现代JVM）会有一个叫做inflation的概念，当使用JNI调用超过一定次数后会[翻译反射方法调用](http://www.docjar.com/html/api/sun/reflect/ReflectionFactory.java.html)生成一个[本地版本](http://www.docjar.com/html/api/sun/reflect/NativeMethodAccessorImpl.java.html)的`FastClass`字节码。你通过属性`sun.reflect.inflationThreshold`设置这个次数（默认为15次），以控制jvm的inflation行为（至少在HotSpot JVM中）。这个属性决定了在执行多少次JNI调用后会在本地生成字节码。我建议在现代的JVM中不再使用`FastClass`，但是在老版本的JVM中可以用来优化性能。

## glib Proxy

就像本文开头个所说，cglib `Proxy`是Java `Proxy`的重新实现版本。开发这个库的本意是想要在Java 1.3之前的版本中使用Java库的proxy，当时的cglib Proxy与Java Proxy仅有很少的细节上有区别。在Java Standard库的javadoc中有关于[Java Proxy很好的文档](http://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Proxy.html)，为此我将省略对其的细节讨论。

## 后的警告

在概述了cglib的功能后，我想说最后一句警告。cglib生成字节码的类，会导致这些额外的类保存在JVM的一块特殊内存中：所谓的**perm space**。就像他的名字一样，这块永久的内存空间适用于保存不需要垃圾回收的永久对象的。然而，这也不是完全正确的：一旦`Class`被加载(load)后，如果加载它的`ClassLoader`没有准备好进行垃圾回收，它就不会被卸载(unload)。`ClassLoader`被回收的唯一场景是，这个`ClassLoader`不是JVM系统的`ClassLoader`而是自定义的(程序创建的)`ClassLoader`。这种`ClassLoader`如果自己准备好，且它加载的所有类以及这些类的实例都准备好回收了，垃圾回收器才会真正的回收。这就意味着，如果你在Java程序中创建越来越多的类，并且不认证考虑移除内存中的这些类，你迟早会将perm space耗尽，最终程序死于`OutOfMemoryError`之手。因此，请谨慎使用cglib。但是，如果你明知且谨慎的使用cglib，你将可以做很多纯Java程序不能做的奇妙的事情。
最后，当你创建依赖于cglib的项目的时候，你需要注意到一个事实：cglib项目没有得到应有的管理和开发积极性（考虑到它的流行程度来说）。缺少文档就是这么说的第一个证据，第二个就是它经常出现的凌乱的public接口。发布到Maven中心仓库也存在不好的地方，邮件列表就像是垃圾邮件一样，它的版本迭代也相当的不稳定。因此你可能需要了解一下[javassist](http://www.csg.ci.i.u-tokyo.ac.jp/~chiba/javassist/)，一个真正可以替代cglib的库（但是功能较cglib弱）。Javassist附带了一个伪Java编译器，这样就可以在不清楚Java字节码的情况下创建不可思议的字节码增强了。如果你想要亲力亲为，你可能更喜欢ASM（cglib就是基于这个构建的），ASM不管是库还是Java代码又或者是字节码，都有强大的文档支持。
需要注意的是，本文中的所有示例都只能运行在cglib 2.2.2中，与新的3.x的版本不兼容。不幸的是，我体验了cglib的最新版本，但是它偶尔会生成一些无效的字节码，所以我现在在生产环境中使用的还是老版本。另外一个需要注意的问题是，大多数使用cglig的项目都将cglib转移到了他们自己的namespace下，以防止与其他依赖包的版本冲突，比如[Spring project](http://docs.spring.io/spring/docs/3.2.5.RELEASE/javadoc-api/org/springframework/cglib/package-summary.html)。建议你在使用cglib的时候也这么做，很多工具可以帮助你自动完成这一最佳实践，比如[jarjar](https://code.google.com/p/jarjar/)。



> Read More：
>
> https://dzone.com/articles/cglib-missing-manual
>
> https://www.baeldung.com/byte-buddy
>
> https://www.baeldung.com/cglib
>
> https://www.baeldung.com/javassist
>
> https://www.baeldung.com/java-asm
>
> http://www.javassist.org/tutorial/tutorial.html
>
> https://taodaling.github.io/blog/2018/09/27/javassist/
>
> https://notes.diguage.com/byte-buddy-tutorial/


