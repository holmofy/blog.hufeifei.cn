[前一篇文章](https://blog.hufeifei.cn/2020/02/01/J2EE/spring-ioc/spring-ioc/)将Spring IoC容器的整体内容进行了分析，这篇文章就拿Spring框架的另一个核心部分AOP开刀。

# 0、写在前面

AOP是辅助OOP的另一种编程模式，基于Spring AOP的事务、缓存等组件也是Spring的重要组成部分。AOP让代码更加模块化，也更符合OOP的编程理念。

## 0.1、AspectJ vs. Spring AOP

AOP通过代理的设计模式实现，而代理分为"静态代理"和"动态代理"。AspectJ一般都被认为静态代理(严格讲不能叫代理)，而Spring AOP则是动态代理——运行时动态生成代理类。

![代理模式](http://ww1.sinaimg.cn/large/bda5cd74ly1gdfwpemevwj20e405cq2z.jpg)

Spring AOP旨在Spring IOC基础上提供一个简单的AOP实现，以解决一些最常见的问题，它并不是一个完整的AOP实现，而且切面只能应用于Spring容器管理的Bean上。

AspectJ是一套完整的AOP解决方案，更强大，也更复杂，可以将切面应用于所有的对象上。

### 0.1.1、织入方式

AspectJ有三种织入方式，分别为编译时织入，编译后织入，加载时织入。

1. 编译时织入，也就是在编译时将切面织入到原来的正常代码中；

2. 编译后织入，主要是对一些已经编译好的第三方jar包里的类进行织入；

3. 加载时织入，是类加载器将类文件加载进JVM时进行织入。

具体AspectJ的使用可以参考这两篇文章

* [Get Started with AspectJ](http://esus.com/get-started-with-aspectj/)

* [Intro to AspectJ](https://www.baeldung.com/aspectj)

AspectJ实际是通过修改类的字节码将[aspectj语言](https://en.wikipedia.org/wiki/AspectJ)编写的切面代码嵌入到原来的代码中，所以需要[额外的编译器(aspectjweaver)](https://www.eclipse.org/aspectj/doc/next/devguide/printable.html#ajc-ref-top)

Spring AOP使用[JDK的Proxy](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)或[Cglib的Enhancer](https://blog.hufeifei.cn/2020/03/17/Java/cglib/)在运行时对调用的目标方法进行拦截并执行相应的切面方法。

### 0.1.2、JoinPoint

AspectJ和Spring AOP在切入点上也有很大不同：

* AspectJ是直接修改字节码，所以可以对类中的所有结构进行切入。

* Spring AOP因为受限于[JDK的Proxy](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)，所以只能对方法执行进行拦截。

| Joinpoint                    | Spring AOP Supported | AspectJ Supported |
| ---------------------------- | -------------------- | ----------------- |
| Method Call                  | No                   | Yes               |
| Method Execution             | Yes                  | Yes               |
| Constructor Call             | No                   | Yes               |
| Constructor Execution        | No                   | Yes               |
| Static initializer execution | No                   | Yes               |
| Object initialization        | No                   | Yes               |
| Field reference              | No                   | Yes               |
| Field assignment             | No                   | Yes               |
| Handler execution            | No                   | Yes               |
| Advice execution             | No                   | Yes               |

### 0.1.3、Spring AOP与AspectJ整体对比

| Spring AOP                                     | AspectJ                                                      |
| ---------------------------------------------- | ------------------------------------------------------------ |
| 用纯Java语言实现                                 | 使用扩展的AspectJ语言定义切面                                   |
| 无需单独的编译过程                                | 除非设置了LTW(load-time weaving)，否则需要AspectJ编译器（ajc）   |
| 运行时织入                                       | 支持编译时、编译后、类加载时织入                                 |
| 功能不足–仅支持方法级织入                          | 更强大–可以对字段，方法，构造函数，静态代码块，final类/方法等部分织入|
| 只能用在Spring容器管理的bean上                    | 可以用在所有对象上                                             |
| 仅支持方法执行切入点                              | 支持所有切入点                                                 |
| 支持部分AspectJ的指令与AspectJ的注解              | 比Spring AOP复杂得多                                         |
| 代理是针对目标对象创建的，并且切面已应用于这些代理    | 在应用程序执行之前（运行时之前）将切面直接织入到代码中               |
| 运行时生成代理类，比AspectJ慢得多                  | 事先生成好的字节码，具有更好的性能                                |
| 易于学习和应用                                    | 比Spring AOP复杂得多                                         |

## 0.2、动态代理

Spring AOP实现动态代理有两种方式，一种基于[JDK的Proxy](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)，另一种基于[Cglib的Enhancer](https://blog.hufeifei.cn/2020/03/17/Java/cglib/)。

前者只能对实现了接口的类进行代理，可以对接口内定义的方法进行拦截。后者能对任意类进行代理，可以对对象的任意的非final方法进行拦截。

![JdkProxy vs. Cglib](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLSCml22ZAhwYivl9FoafDBb58Joq12sXeMcC8EUSa5XVxv2Ucf1Of92FOG9MrN32356ngzFGKl5AoNIhp4dCpas7YQ0gSqtCoa-1o41v2WQwk0iZklDJYp69KbLmEgNafG3y00000)

我们在AopProxyFactory工厂的实现类看到Spring选择动态代理的逻辑：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}

}
```

# 1、简单回顾Spring AOP的两种配置方式

```java
@Service
public class BusinessService {

    public void doSomething() {
        //... do something
    }

}
```

比如我要对上面业务代码的方法做一个日志切面，可以用XML和@AspectJ注解两种方式。

1、使用xml配置：

```xml
<aop:config>

    <aop:aspect id="myAspect" ref="loggingAspectBean">
        <!-- 声明切入点，切入点是BusinessService的所有方法 -->
        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.BusinessService.*(..))"/>
      
        <!-- 声明通知(这里是前置通知)，切入点调用时，会通知切面的方法 -->
    </aop:aspect>

</aop:config>
<!-- 声明切面Bean -->
<bean id="loggingAspectBean" class="com.xyz.myapp.aspect.LoggingAspect" />


<!-- 也可以用aop:advisor将切面和通知进行绑定 -->
<aop:config>
  <aop:advisor
      pointcut="execution(* com.xyz.myapp.service.*.*(..))"
      advice-ref="tx-advice"/>
</aop:config>

<tx:advice id="tx-advice">
  <tx:attributes>
    <tx:method name="*" propagation="REQUIRED"/>
  </tx:attributes>
</tx:advice>
```

这个时候我们的切面十分简单：

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class LoggingAspect {

    public void log(JoinPoint jp) {
        String methodSignature = jp.getSignature().toShortString();
        log.info("{} be called", methodSignature);
    }

}
```

2、使用全注解的方式配置：

```java
@Slf4j
@Aspect
@Component
public class LoggingAspect {

    @Pointcut("execution(* com.xyz.myapp.service.*.*(..))")
    public void bussiness() {
    }

    /**
     * 除了@Before还有@After、@AfterReturning、@AfterThrowing、@Around等通知
     * 它们分别代表了方法调用过程中的几个位置，其中@Around环绕通知表示控制整个调用过程
     *  try {
     *      // @before
     *      Object result = method.invoke();
     *      // @AfterReturning
     *  } catch (Throwable e) {
     *      // @AfterThrowing
     *  } finally {
     *      // @After
     *  }
     */
    @Before("bussiness()")
    public void log(JoinPoint jp) {
        String methodSignature = jp.getSignature().toShortString();
        log.info("{} be called", methodSignature);
    }
}
```

注解方式要求启用`@EnableAspectJAutoProxy`，或者xml里加上`<aop:aspectj-autoproxy />`

# 2、Spring AOP整体架构

定义一个切面需要指定**切入点**(对哪些方法进行切入)、增强逻辑(对这些方法添加什么逻辑)。这两个分别由`Pointcut`和`Advice`两个接口定义。

## 2.1、Pointcut

`Pointcut`定义了中定义了**`ClassFilter`**和**`MethodMatcher`**用于捕捉系统中相应的`Joinpoint`。

```java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
```

`ClassFilter`和`MethodMatcher`分别用于匹配将被执行织入操作的对象以及相应的方法，Pointcut将这两者分开定义也是为了方便进行各种组合。

`ClassFilter`相对简单，只是定义了类型过滤的规则：

```java
public interface ClassFilter {

	boolean matches(Class<?> clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}
```

![ClassFilter](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLSCv9B2vsoym1yhcua3WADZLwUWf1-VabI8AO2Xppyl9B4aioy_FmAWkfB4WDI2m1yb7KSJcavgK07GG0)

[`MethodMatcher`](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/aop/MethodMatcher.html)的定义相对复杂一点：

```java
public interface MethodMatcher {
  
	boolean matches(Method method, Class<?> targetClass);

	boolean isRuntime();

	boolean matches(Method method, Class<?> targetClass, Object... args);

	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

* 当`isRuntime()`返回`false`时，不会考虑切入方法的参数，也就是只会调用只有两个参数的`matchs`方法，这种类型的`MethodMatcher`称之为`StaticMethodMatcher`。因为不用每次都检查参数，那么对于同一类型的同一方法，每次匹配的返回值都是一样的，那么框架内部可以对匹配结果进行缓存以提高性能。
* 当`isRuntime()`返回`true`时，则会调用三个参数的`matchs`方法，这种类型的`MethodMatcher`称之为`DynamicMethodMatcher`。

基`MethodMatcher`与`ClassFilter`组合出`Pointcut`及其相应的实现类。

![Pointcut](http://www.plantuml.com/plantuml/svg/bP5TQeGm4CVVSufS89wW3qhxLB2KzWHZ73Kq7n97QA67RrVNi677mBuD_d__3qcz44HQdHN2UC8uW4RP8asXRu7qXF7c-tlul_LA0hg58cYIdmHaTCudyUN7W-DLtfhbin4yrLoF3npnUzxXH8dCB9-KagVQhx8uaEAstQFHZ1Cf_Y-FXcBa0IMQJotpIZRUXqjuy1jd_7W2FWStXmKwYmaGpAmTBktbnkqkJYcMPGlB3ycBoLvLyWqSRhaFEr_xBxPJrEZxrTa_)

为了方便进行组合，Spring还为这三个接口提供了相应的工具类，以支持交(intersection)、并(union)逻辑。

![相关工具类](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vL2CW7ifDBIv24d7CIYulTCdE0V2HHtzIIZFmKtyIIv0oWE1TCduADhYxSa68k9BrW6IH-CHUI4L1f916G0Yw7rBmKe0C1)

## 2.2、Advice

Advice经常被国内的程序员译为“通知”，我个人觉得翻译有点生硬。不论Advice被翻译成什么，总之它就是用来定义对切入点添加增强逻辑的。不过这个接口并非Spring框架定义的，而是由AOP联盟(AOP Alliance)定义的。

2003年Spring创始人[Rod Johnson](https://en.wikipedia.org/wiki/Rod_Johnson_%28programmer%29)联合其他几个AOP框架的作者(如[Guice](https://github.com/google/guice/wiki/AOP)的作者[Bob Lee](http://blog.crazybob.org/)，[nanning](https://github.com/codehaus/nanning)的作者[Jon Tirsén](https://tirsen.com/))成立了一个[AOP联盟](http://aopalliance.sourceforge.net/)尝试对AOP的一些接口进行规范，所以Spring AOP的很多接口都是基于`org.aopalliance.aop`中定义的基础接口扩展的。

![Advice](http://www.plantuml.com/plantuml/svg/ZLF1JiCm3BtdAt8iqieNJ6CmBZiW98Iu08SiTRj06a-9sma1_qxQBPAhhYgdEUwpttksjmwaF3Mr5S8u0byg3VAsQ8s69VhgMk51MMkKqz35AuQwWn8zdB0iVV_bL6tqrf77ej5aq9qmybli42qe9qrzi523ex1DTTd6gX2MDoiPMvLNufUrx44Q3eH-JjG3q1wBkO8evN7t0PeYMOkVaAMo5hNg5CTy2eTIDkW2-MWB_Vz9T2hAMFaiMJB3vnnxjcWAkUjoIRQi-v-5U2DvOdnzbyfNON5-Ieele65plY1cXPz16QFHVf_t7e-1fHdu2QZnphXgg5ODlcLdKRvFl2ZRG2zzQ6RMYJXpB7_o0eykbzm4Ypj0FhwB4MHYjPp7GadvyExPWq76Y-DjlAds4SJ7osA6OACUGVAB4-Sq-VHVKLpwmWgdRcCqnMHvkaIimHLDRVy0)

其中最核心的接口就是`MethodInterceptor`：

![MethodInterceptor](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLy4qjoSXFyGJnarCBIlABk31456ngz79Iqqhq59mJapDI2QhLKmWfIimhJamkoSpF8qArOt5bNh9hHMfoAP4Q8CB1G0r5cIML13KKPQPdbC1qXINcPAOaebl4vP2Qbm8C6G00)

最常见的如：缓存、事务、@Async异步方法、JSR-303方法参数校验、Spring Security权限校验等。

除此之外，Spring还提供了三个用于适配BeforeAdvice、AfterAdvice、ThrowsAdvice的MethodInterceptor。

![AdviceImpl](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLS4fDoozATKmfoqnEHH9sJ0EoC4HzKqioybCyGVpar8AI_28kBcJz2ZOrUdfGHSZYo1emZ2164nVKEGXBm091gIMbHNcPUUb4MeEfZR158Hb5-UN5H5Y0NpcNGsfU2Z3e0G00)

> 具体可以参考`DefaultAdvisorAdapterRegistry`对`MethodBeforeAdvice`、`AfterReturningAdvice`、`ThrowsAdvice`这三个Advice的适配

[Spring2.0开始支持的AspectJ表达式](https://docs.spring.io/spring/docs/4.0.x/spring-framework-reference/html/aop.html)，并开始支持AspectJ注解形式定义的切面(@Before、@After、@AfterReturning、@AfterThrowing、@Around)，下面是相应的几个Advice：

![AdviceImpl](http://www.plantuml.com/plantuml/svg/bP4n3i8m34NtdA8NQ4_0qB21n8AuG4WSMgbDAjU1XSCZ90iLkSXbI-pt_VNjTYQ7LCR1z8a0e_DGsN3lFImA9w0kXpt4Z22QDXBWUlKCL33rwVPZuk7zzp1HHkEkCw7pL5b-s7a2JqUMBbogtRCU85AzRlA1caQTnHYtkQummbA1BntcnvAL0tI7hmj8ZMmRiM5fif6K6HG5vf82e_82VC5wFUz_iisqN90kaj4rbOnlMaYXZCCB)

## 2.3、Advisor

定义了切入点(Pointcut)和切入逻辑(Advice)，那这个切面就算基本OK了，接着要做的就是把它们整合到一起，Advisor就是承担这个角色的——Advisor就是Spring中的一个切面。

```java
public interface Advisor {
  
	Advice EMPTY_ADVICE = new Advice() {};
  
  // 定义切入逻辑Advice
	Advice getAdvice();
  
	boolean isPerInstance();

}

// Advisor的子接口PointcutAdvisor中定义了切入点Pointcut
public interface PointcutAdvisor extends Advisor {

	Pointcut getPointcut();

}
```

![Advisor](http://www.plantuml.com/plantuml/svg/fPDTQiCm48JVVGfVm3v17-BI7rf8AKqli97NiS1UcTMe9D335s8LJAsWqBm9QVORxMYryI1568UYi0BMZWoNJVjblTF5pej0NHiCh9FrwRkp0XFmUq9x3oM3SWU2DLj6xzejmVIi5xLDN6G5pooircHrzqpoH0PEJt-rHLoKTz-LsaEFQjN3GZ5mXoePmQ8eYkiphhwcynLm1jJb0tSK1gGCKuxrnUYI-ykU6dyIquWuK7J9rSMqFfN4qtmrGEq-W7SkgE4h65N-YJBrcL78uYCc1fs_HTeeKZxumCTfVkf_r5Vb6urydf4R68sVsE3ryJWfUs-Tv2dfaOYPh7xCXgKDu_03)

## 2.4、TargetSource

切面已经有了，接着就是我们要对哪个具体的对象进行切入，也就是我们要代理的目标对象。

一般情况下我们代理的目标对象是固定的。比如下面这种我们直接用JDK的Proxy生成代理，target在创建`JdkDynamicProxy`的时候就确定了。

```java
public class JdkDynamicProxy implements InvocationHandler {

  private Object target;

  public JdkDynamicProxy(Object target) {
    this.target = target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("before invoke");
    Object result = method.invoke(target, args);
    System.out.println("after invoke");
    return result;
  }

  public static void main(String[] args) {
    Integer target = 0;
    Comparable<Integer> proxied = (Comparable<Integer>) Proxy.newProxyInstance(
      JdkDynamicProxy.class.getClassLoader(), new Class[]{Comparable.class}, new JdkDynamicProxy(target));
    int result = proxied.compareTo(1);
    System.out.println(result < 0);
  }

}
```

但是有时候我们需要代理的对象是动态创建的，比如`prototype`类型的Bean对象。

所以Spring抽象了一个`TargetSource`接口，将需要代理的目标对象做一层简单的封装：

```java
public interface TargetSource extends TargetClassAware {
  
  //getTarget()返回对象的类型
	@Override
	@Nullable
	Class<?> getTargetClass();

  // 每次getTarget()返回的target是否是同一个对象，
  // 是则，不需要调用releaseTarget()释放，且AOP框架可以缓存这个对象。
	boolean isStatic();
  
	@Nullable
	Object getTarget() throws Exception;
  //释放getTarget()拿到的对象
	void releaseTarget(Object target) throws Exception;

}
```

Spring中提供的TargetSource实现类的继承结构如下：

![TargetSource](http://www.plantuml.com/plantuml/svg/dPF1JeH038RlF0LFmC6pXtKt6ZMRQ6HVe3jqOSnCEoab1kF36xB6pBAnH6wG_-lFtvPkUWNH8OQYyAGe9t1O7a1Qr9e7SLZ0iLS1f-NTpyCRdWJx3eu1RN0Fd-DE4DGpsUGMWHx0ASkuXHuRctvb3fxQ1KXOMSU4ruP5_bRUVbNYsqwhyf6r_e2KhZgRyqEgolkOT5oadgdnByTtTBg8rfxQWCfaafMn1gF2cB9LPg-nM8Xojic--bTa4yciCh5sufFEufXzpgSTKy29pDL_MKnpUI9_6SBw_V9MVlLLrOTE49ezEmYhubV6cQmmimxJqCx5EuwHgP_qcyjzRf2Q8OPl)

其中最简单的就是`SingletonTargetSource`、`HotSwappableTargetSource`和`EmptyTargetSource`：

```java
public class SingletonTargetSource implements TargetSource, Serializable {
  
	private static final long serialVersionUID = 9031246629662423738L;

	private final Object target;

	public SingletonTargetSource(Object target) {
		Assert.notNull(target, "Target object must not be null");
		this.target = target;
	}

	@Override
	public Class<?> getTargetClass() {
		return this.target.getClass();
	}

	@Override
	public Object getTarget() {
		return this.target;
	}

	@Override
	public void releaseTarget(Object target) {
		// nothing to do
	}

	@Override
	public boolean isStatic() {
		return true;
	}

  // equals()、hashCode()、toString()方法
  // ...
}

public class HotSwappableTargetSource implements TargetSource, Serializable {
  
	private static final long serialVersionUID = 7497929212653839187L;

	private Object target;
  
	public HotSwappableTargetSource(Object initialTarget) {
		Assert.notNull(initialTarget, "Target object must not be null");
		this.target = initialTarget;
	}
  
	@Override
	public synchronized Class<?> getTargetClass() {
		return this.target.getClass();
	}

	@Override
	public final boolean isStatic() {
		return false;
	}

	@Override
	public synchronized Object getTarget() {
		return this.target;
	}

	@Override
	public void releaseTarget(Object target) {
		// nothing to do
	}
  
  // 动态切换target
	public synchronized Object swap(Object newTarget) throws IllegalArgumentException {
		Assert.notNull(newTarget, "Target object must not be null");
		Object old = this.target;
		this.target = newTarget;
		return old;
	}

  // equals()、hashCode()、toString()方法
  // ...
}

public final class EmptyTargetSource implements TargetSource, Serializable {
  
	private static final long serialVersionUID = 3680494563553489691L;

	public static final EmptyTargetSource INSTANCE = new EmptyTargetSource(null, true);

	public static EmptyTargetSource forClass(@Nullable Class<?> targetClass) {
		return forClass(targetClass, true);
	}
  
	public static EmptyTargetSource forClass(@Nullable Class<?> targetClass, boolean isStatic) {
		return (targetClass == null && isStatic ? INSTANCE : new EmptyTargetSource(targetClass, isStatic));
	}

	private final Class<?> targetClass;

	private final boolean isStatic;
  
	private EmptyTargetSource(@Nullable Class<?> targetClass, boolean isStatic) {
		this.targetClass = targetClass;
		this.isStatic = isStatic;
	}
  
	@Override
	@Nullable
	public Class<?> getTargetClass() {
		return this.targetClass;
	}
  
	@Override
	public boolean isStatic() {
		return this.isStatic;
	}
  
	@Override
	@Nullable
	public Object getTarget() {
		return null;
	}
  
	@Override
	public void releaseTarget(Object target) {
	}
  
	// 重写反序列化保证单例
	private Object readResolve() {
		return (this.targetClass == null && this.isStatic ? INSTANCE : this);
	}
  
  // equals()、hashCode()、toString()方法
  // ...
}
```

## 2.5、Advised与ProxyFactory

万事俱备只欠东风——切面、目标对象都有了，接下来就是执行织入操作了——为目标对象添加切面并生成代理对象。Spring提供了`ProxyFactory`来实现织入操作。`ProxyFactory`继承自`AdvisedSupport`，后者实现了`Advised`接口。

![ProxyFactory](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLS4mfoonEJU62qWesDNfwCC7WqeA2_A8IBbGkH4b0KNv5fNDHQc99VX5C7R8OfcAtn6IWUALJQc8USIfngt8iBaXDBl52KSpba9gN0lGS0000)

`Advised`接口定义了`ProxyFactory`应该包含的配置。

```java
public interface Advised extends TargetClassAware {
  // 配置是否已经被冻结
	boolean isFrozen();
  // 是否是使用Cglib代理目标类(相反，使用JDK的Proxy以接口方式代理)
	boolean isProxyTargetClass();
  // 生成的代理类应该实现的接口
	Class<?>[] getProxiedInterfaces();
  // 判断指定接口是否被代理类实现
	boolean isInterfaceProxied(Class<?> intf);
  // 设置需要代理的目标对象
	void setTargetSource(TargetSource targetSource);
	TargetSource getTargetSource();
  // 是否将AOP代理对象设置到ThreadLocal中
  // 设置后，可以通过AopContext.currentProxy()方法拿到当前线程正在使用的代理对象。
	void setExposeProxy(boolean exposeProxy);
	boolean isExposeProxy();
  // 是否应用Advisor Chain对目标类进行预先过滤
	void setPreFiltered(boolean preFiltered);
	boolean isPreFiltered();
  // ProxyFactory中存了一组Advisor，并提供了一些方法对其进行增删改查
	Advisor[] getAdvisors();
	void addAdvisor(Advisor advisor) throws AopConfigException;
	void addAdvisor(int pos, Advisor advisor) throws AopConfigException;
	boolean removeAdvisor(Advisor advisor);
	void removeAdvisor(int index) throws AopConfigException;
	int indexOf(Advisor advisor);
	boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;
  // 对单独的Advice进行增删，内部会以DefaultPointcutAdvisor将其封装成Advisor
	void addAdvice(Advice advice) throws AopConfigException;
	void addAdvice(int pos, Advice advice) throws AopConfigException;
	boolean removeAdvice(Advice advice);
	int indexOf(Advice advice);
  // toString()方法
	String toProxyConfigString();

}
```

最简单的用法：

```java
public class ProxyTest {
    interface Foo {
    }

    static class FooImpl implements Foo {
    }

    @Test
    public void jdkProxy() {
        Foo foo = new FooImpl();
        ProxyFactory pf = new ProxyFactory();
        pf.setTarget(foo);
        pf.addInterface(Foo.class); // Proxy代理的接口
        Foo proxy = (Foo) pf.getProxy();
        assertTrue(AopUtils.isJdkDynamicProxy(proxy));
        assertTrue(proxy instanceof SpringProxy);
        assertTrue(proxy instanceof Advised);
        assertTrue(proxy instanceof DecoratingProxy);

        Advised advised = (Advised) proxy;
        assertTrue(advised.getTargetSource() instanceof SingletonTargetSource);
        assertEquals(advised.getProxiedInterfaces().length, 1);
        assertEquals(advised.getProxiedInterfaces()[0], Foo.class);
    }

    @Test
    public void cglibProxy() {
        Foo foo = new FooImpl();
        ProxyFactory pf = new ProxyFactory();
        pf.setTarget(foo);
        pf.setProxyTargetClass(true); // 指定使用cglib代理目标类
        Foo proxy = (Foo) pf.getProxy();
        assertTrue(AopUtils.isCglibProxy(proxy));
        assertTrue(proxy instanceof FooImpl);
        assertTrue(proxy instanceof SpringProxy);
        assertTrue(proxy instanceof Advised);
        assertFalse(proxy instanceof DecoratingProxy);

        Advised advised = (Advised) proxy;
        assertTrue(advised.isProxyTargetClass());
        assertEquals(advised.getProxiedInterfaces().length, 0);
    }
}
```

这里例子中没有设置添加切面，只是生成了代理类，这里的直接将`target`设置为自己创建的对象，实际里面会将`target`包装成`SingletonTargetSource`，当我们没有设置`targetSource`时默认会有一个`EmptyTargetSource`。

```java
public class AdvisedSupport extends ProxyConfig implements Advised {

	public static final TargetSource EMPTY_TARGET_SOURCE = EmptyTargetSource.INSTANCE;
  
	TargetSource targetSource = EMPTY_TARGET_SOURCE;
  //...
	public void setTarget(Object target) {
		setTargetSource(new SingletonTargetSource(target));
	}
  //...
}
```

### 2.5.1、Spring代理类实现的三个接口

值得一提的是`ProxyFactory`生成的代理类会自动实现`SpringProxy`、`Advised`、`DecoratingProxy`三个接口(详见`AopProxyUtils#completeProxiedInterfaces`)：

* `SpringProxy`接口：是所有生成的Spring代理的标记接口，可以通过这个接口判断对象是否为Spring生成的代理对象，具体可以参看`AopUtils#isAopProxy`、`AopUtils#isJdkDynamicProxy`、`AopUtils#isCglibProxy`三个方法。
* `Advised`接口：生成的代理类内部会代理`ProxyFactory`中的方法。
* `DecoratingProxy`接口：如果经过多层代理，可以通过`getDecoratedClass()`方法找到最终代理的对象，需要注意的是Jdk的proxy会自动实现改接口，但cglib的代理并不会实现该接口。

### 2.5.2、AdvisorChainFactory切面调用链

基于上面的`ProxyFactory`的例子，继续为我们的代理类提供切面逻辑：

```java
static class NopInterceptor implements MethodInterceptor {

  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.out.println("no interceptor");
    return invocation.proceed();
  }
}

static class CountingBeforeAdvice implements MethodBeforeAdvice {

  int count;

  @Override
  public void before(Method method, Object[] args, Object target) {
    System.out.println("counting");
    count++;
  }
}

@Test
public void proxy() {
  Foo foo = new FooImpl();
  ProxyFactory pf = new ProxyFactory();
  NopInterceptor nop = new NopInterceptor();
  CountingBeforeAdvice counting = new CountingBeforeAdvice();
  Advisor advisor = new DefaultPointcutAdvisor(counting);
  pf.addAdvice(nop);            // 这个切面最终也会被包装成DefaultPointcutAdvisor
  pf.addAdvisor(advisor);       // 添加已经包装好的切面
  pf.setTarget(foo);
  pf.setProxyTargetClass(true); // 指定使用cglib代理目标类
  assertThat(pf.indexOf(nop)).isEqualTo(0);
  assertThat(pf.indexOf(advisor)).isEqualTo(1);

  Foo proxy = (Foo) pf.getProxy();
  proxy.doSomething();
  proxy.doSomething();
  assertEquals(counting.count, 2);
}
```

从打印的结果来看，切面按照添加顺序执行。

![AdvisorChainFactory](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLS4mfoopEBtBEICpCSqjCBialgkJ28gPWKwEdf-2IcfPOcbE2JG-NGsfU2j1i0000)

`AdvisorChainFactory`负责把所有的`Advisor`适配成`Interceptor`，最后封装成`MethodInvocation`。

## 2.6、AspectJProxyFactory

Spring1.0时期，AOP配置仍然非常麻烦，Spring2.0开始增加了@AspectJ的支持：

注解形式的AOP实现方式——将切面的@Pointcut、Advice(@Before、@After、@AfterReturning、@AfterThrowing、@Around)以及织入逻辑统一放到一个Java类中。这种方式更加模块化，更利于切面的管理。

但是需要注意的是，Spring2.0只是借用了AspectJ的注解以及部分表达式的功能，底层的AOP实现逻辑仍然用的是Spring1.0现有的体系。

![AspectJProxyFactory](http://www.plantuml.com/plantuml/svg/bP5H2i8m38RVUufTe3SGsGHz48GnlC1iCrUOpKZIkO67jqWTFWZR7fF_-_8_96UfISAZKwhW7eoSIy9nEjL6aAPCwtrMGTF5m0sGYC8EZf4IenRopusK7CUUWDcXBz5vCK7MsJSMYlCFOK3ztMQxbzRVkWj8Y_O03H8iIhDLD09KWGJopjytUXlnQqluNDCoMuJ1suIh7BoDlti3)

> 同时也在原来的XML配置的形式中，添加了对AspectJ的支持，具体可以参看`ConfigBeanDefinitionParser`这部分代码。

## 2.7、AutoProxyCreator

至此，我们整个织入的过程好像已经完备了。但是还有一个问题需要解决，到目前为止我们的织入都需要我们自己调用ProxyFactory进行操作，但是我们启动Spring应用的时候，所有的@Transactional事务、@Cacheable缓存切面都是自动给织入进去的啊，它是如何做到的呢。这里要介绍AutoProxyCreator，它就是负责将Spring IoC和AOP进行结合的组件——自动为SpringBean创建代理进行织入。

![AutoProxyCreator](http://www.plantuml.com/plantuml/svg/ZL7BIlD05DxFK-G1_lRVIv7Kwg9YfQ0BjwVJIHf8Pihan0jL4F5IHD65TA4BaO8M5qLmPIy-ZMdgobTm6WEADKrsoPplyftpwIw2HC-n2R4uCca0PTe20ruMBfQbeCnrXVmnAtB5u6W1MeBdjq2oMUWrHwcdeKozQBdTJ2QLMo8c4aiV1YekIg2evQEFl8T2ZRTt2f81_eceRbgAEWLCwwaIPhFnVZ63QB7u4CbiQParp8ILhuB3OhXnsg64pGobiCWCyEmON2eTjOXRPnINmnF52N61J9jODiX6QUNzw3mOTdCWwdDKSCCo_y-FMY_-u8BupHk_tmkk3ySdZ_vfvwtV3YwFoHv9uH65uShgkxA8DokF_eKPFNV63-s46AdkKUI618NpA7e95mbxR13_jXVSxO-xhwFMCBZg-CsdNzOVWpFYtn_eInmgDYi0hLdS_4UY_tK9t0KwMRe474pcC8ugBT4PhtgE_Ga0)

前面的[Spring-IOC整体设计与源码分析](https://blog.hufeifei.cn/2020/02/01/J2EE/spring-ioc/spring-ioc/)文章中，已经介绍了`BeanPostProceesor`，这里的`AutoProxyCreator`就是继承自`SmartInstantiationAwareBeanPostProcessor`。我们可以在`AbstractAutoProxyCreator`中看到这样一段代码：

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
  ///...
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}

	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
  ///...
}
```







https://www.baeldung.com/spring-aop

https://www.baeldung.com/spring-aop-vs-aspectj

https://www.baeldung.com/aspectj

http://esus.com/get-started-with-aspectj/

https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop

https://spring.io/blog/2012/05/23/transactions-caching-and-aop-understanding-proxy-usage-in-spring

https://spring.io/blog/2009/03/27/rest-in-spring-3-resttemplate

https://spring.io/blog/2008/10/14/optimising-and-tuning-apache-tomcat-part-2/

https://spring.io/blog/2011/12/08/spring-integration-scripting-support-part-1

https://spring.io/blog/2013/03/04/spring-at-china-scale-alibaba-group-alipay-taobao-and-tmall

https://spring.io/blog/2014/04/30/spring-4-1-s-upcoming-jms-improvements

https://spring.io/blog/2015/03/01/the-portable-cloud-ready-http-session

https://spring.io/blog/2015/04/03/how-spring-achieves-compatibility-with-java-6-7-and-8

https://spring.io/blog/2016/01/12/spring-integration-zip-1-0-0-m1-and-others