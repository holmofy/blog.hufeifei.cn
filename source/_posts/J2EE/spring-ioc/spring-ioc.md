---
title: Spring-IOC整体设计与源码分析
date: 2020-2-01
categories: J2EE
---

最近读完《[Spring技术内幕](https://book.douban.com/subject/10470970/)》一书，虽然此书评价貌似不高，但边看书边读源码，感觉还是有点收获，至少为阅读Spring源码提供了思路。然后这篇文章就记录一下这几天看Spring IOC这块的源码以及整体思路。

# 1、 BeanFactory与ApplicationContext

在Spring的IOC容器设计中，主要由两个容器系列：

* 实现BeanFactory接口的简单容器，提供了完整IoC容器服务支持，默认延迟初始化(lazy-load)——只有访问托管对象的时候，才会对对象进行初始化和依赖注入操作。
* 实现ApplicationContext接口的集成容器，在BeanFactory的基础上增加了更多复杂的企业级功能，容器启动时，就默认把所有的单例对象实例化完成。

![BeanFactory](http://www.plantuml.com/plantuml/svg/bLBFJXD17BxFK-mBy0AD1nGn7ZmG6XCFlGpBj4wodMbcfrMY9aL0kmPi4ui4QgA4493ORXeJtQLf-p3Ep2vluInBo02xJRZjpE_xytspltcNGyRhLGS0Zhc32jOZ1CaJQ7FArgBHYVAciZS101EEM1dQo9m3GZcoEAqLh6ADOL99Pd8GoltJA-hmlVfr61rigXzXYIY_BOAp17EnCLRVlFZZcVmxb5rV14sDaPqTsTypd1xMENs56Lg0DRZYe3l6AvHpMYrOSd0WGa-SVqX3N2RKUl7HriNMJdooALlxXkepxFAPSghT4PEUbfEjFH5CqZuYw2klgKDVYQkleVgzMoAoxOlHSJJw8ZijG_4vnuuhnjEeUsfOBr1InfKwcs5llG6twgJ-sb9RwJxHU91yV_eURup1TLH3ROcwV8bH6xakPV-Qwq0VQeZnjSLLhEVBg89Tpk3bAA4jlgunZSMKB2ENEWsK37JI1cB9PH6n1hOHYFgU-dmisqUwhfXCfH-cmU9fBpxSJlCexwSSxe9tHkMd66al-oMsePFxuY8uDxjUjpANg4HKrrVRwr7hZ-ntKc3EqsRzOM0Sh0TpSxEMwpRKx30Bb-cRGqu8kVi0yiCPlFoovlo-tFJk_ZnmmpLJvQsMUOpAiExmVESCJf53ZkrCqrovDFIBgdC3Fe_8Qhtg_0S0)

# 2、 最简单的容器StaticListableBeanFactory

整个Spring容器中最简单的实现就是StaticListableBeanFactory，从它开始分析最合适不过了。

```java
public class StaticListableBeanFactory implements ListableBeanFactory {

	/** <BeanName, BeanInstance>的映射 */
	private final Map<String, Object> beans;
  
  // 默认构造会创建一个空的Map，自行调用addBean方法添加到容器中
	public StaticListableBeanFactory() {
		this.beans = new LinkedHashMap<String, Object>();
	}

  // 由外部预初始化一个Map
	public StaticListableBeanFactory(Map<String, Object> beans) {
		Assert.notNull(beans, "Beans Map must not be null");
		this.beans = beans;
	}

  // addBean就是简单的将name和instance加入到Map中
  public void addBean(String name, Object bean) {
		this.beans.put(name, bean);
	}
  
  // 根据beanName从容器中获取Bean对象
	@Override
	public Object getBean(String name) throws BeansException {
    // 处理"&"开头的BeanName（dereference解引用）
		String beanName = BeanFactoryUtils.transformedBeanName(name);
		Object bean = this.beans.get(beanName);

		if (bean == null) {
      // Bean不存在
			throw new NoSuchBeanDefinitionException(beanName,
					"Defined beans are [" + StringUtils.collectionToCommaDelimitedString(this.beans.keySet()) + "]");
		}

		if (BeanFactoryUtils.isFactoryDereference(name) && !(bean instanceof FactoryBean)) {
		  // BeanName以"&"开头，但Bean不是FactoryBean
			throw new BeanIsNotAFactoryException(beanName, bean.getClass());
		}

		if (bean instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
      // Bean是FactoryBean，但BeanName不是"&"开头，则由FactoryBean创建Bean对象
			try {
				return ((FactoryBean<?>) bean).getObject();
			}
			catch (Exception ex) {
				throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
			}
		}
		else {
      // 其他情况直接返回Bean
			return bean;
		}
	}
  
  // 其余getBean方法(byName,byType)最终都是调用上面的getBean方法
  ...

  // 判断bean是否为共享单例，也就是说调用getBean方法会始终返回同一个对象
  // 注意：返回false，不一定说明是prototype的Bean，应该调isPrototype去显示检查是否是原型对象
	@Override
	public boolean isSingleton(String name) throws NoSuchBeanDefinitionException {
		Object bean = getBean(name);
		return (bean instanceof FactoryBean && ((FactoryBean<?>) bean).isSingleton());
	}

  // 判断bean是否为原型实例，也就是说调用getBean方法会始终返回一个独立的新实例
  // 注意：返回false，不应定说明是singleton的Bean，应该调isSingleton去显示检查是否是单例对象
	@Override
	public boolean isPrototype(String name) throws NoSuchBeanDefinitionException {
		Object bean = getBean(name);
		return ((bean instanceof SmartFactoryBean && ((SmartFactoryBean<?>) bean).isPrototype()) ||
				(bean instanceof FactoryBean && !((FactoryBean<?>) bean).isSingleton()));
	}

  // 获取Bean的类型
	@Override
	public Class<?> getType(String name) throws NoSuchBeanDefinitionException {
		String beanName = BeanFactoryUtils.transformedBeanName(name);

		Object bean = this.beans.get(beanName);
		if (bean == null) {
			throw new NoSuchBeanDefinitionException(beanName,
					"Defined beans are [" + StringUtils.collectionToCommaDelimitedString(this.beans.keySet()) + "]");
		}

		if (bean instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
			// 如果是FactoryBean, 去看Factory创建的是什么类型的Bean.
			return ((FactoryBean<?>) bean).getObjectType();
		}
		return bean.getClass();
	}
  ...
}
```

# 3、 特殊的Bean对象——FactoryBean

Spring容器中有两种Bean：普通Bean和工厂Bean。Spring直接使用前者，后者以工厂模式生产Bean对象，并由Spring管理。

Spring设计[FactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/FactoryBean.html)的目的是为了将复杂对象的相关构造逻辑封装在类中，比如常见的ProxyFactoryBean。

另外一个目的是为了让我们能将依赖注入到一些第三方库的对象中，比如`LocalContainerEntityManagerFactoryBean`负责JPA的`EntityManagerFactory`的创建，`ThreadPoolExecutorFactoryBean`负责线程池的创建，`ForkJoinPoolFactoryBean`负责ForkJoinPool线程池的创建，`ScheduledExecutorFactoryBean`负责调度线程池的创建...

但是Spring3.0基于Java注解的配置开始流行之后，这些FactoryBean基本都用不到了，这些对象我们可以通过@Bean方法的形式配置。

> 我个人大胆的猜测，FactoryBean是为了早期以XML配置对象的方式而设计的。

通过在BeanName前加上`&`前缀我们能拿到FactoryBean工厂本身。比如：

```java
public class MyBeanFactoryBean implements FactoryBean<MyBean> {
 
    public MyBean getObject() throws Exception {
        return new MyBean();
    }
 
    public Class<?> getObjectType() {
        return MyBean.class;
    }
}

// 直接通过beanName获取的是工厂创建的MyBean对象
beanFactory.getBean("myBean");
// 加&前缀获取的是MyBeanFactoryBean这个工厂本身
beanFactory.getBean("&myBean");
```

经常被问的一个问题是BeanFactory与FactoryBean的区别，用一句话总结起来：

**BeanFactory代表Spring容器，而FactoryBean表示工厂类，其创建的对象被获取后作为容器中的Bean注册**。

# 4、 核心容器DefaultListableBeanFactory

Spring IOC体系结构中最核心的容器实现类就是**DefaultListableBeanFactory**，它实现了[ConfigurableListableBeanFactory](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/ConfigurableListableBeanFactory.html)和[BeanDefinitionRegistry](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/support/BeanDefinitionRegistry.html)两个接口的功能。

![DefaultListableBeanFactory](http://www.plantuml.com/plantuml/svg/VP51hiCW34Jtd88Bv0P_aVnKNNNLdi19JMg900AZgb8FNo9LLHGPjiERCMFtYI5oNgrIv1YZWHdraDa_AU880IQB_mZk33Fx-Df1etU6bXoFH0MvKE9wsAQUq90Z9k-kk1HwovejfAI_V85-JxSSWe-iLFrD_tKvT7gOYbQW_M11Ez2D5TIHGrOf1Dcor5p9solETnTfUR3yRtcaP6nN40uZXZKxo3VRl1PDswfwTFUysWy0)

### 4.1 、BeanDefinition、BeanDefinitionRegistry、BeanDefinitionReader

**BeanDefinition**用于描述一个Bean实例的scope、是否为懒加载、生命周期方法(init、destroy)、属性值、构造参数值以及组件依赖等信息。

![BeanDefinition](http://www.plantuml.com/plantuml/svg/bOyn3i8m34Ntd28No0qOMa1YXnEOrAMMKWSbxiRXMH01IrPWTFBUj_zG1OfiQtAEMB3C4D7l4VY8Cp49PVxu69cpWE2a2BXMAH35nmIr-l4rAifzptxt28LkYmGpLmjXkmShlJqRIqx8M2Z-h0L_pbd-m0yB9TsWBTLuAwsOldc9m6nvxfshrPvfYzrZtO0yRMjw0W00)

**BeanDefinitionRegistry**就是BeanDefinition的注册表，提供了`registerBeanDefinition(beanName, beanDefinition)`和`removeBeanDefinition(beanName)`等相关方法。

![BeanDefinitionRegistry](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLSCp9J2mEIatFB2ufgaGITqfDp7D9JSlCoop9pC-3A-12KQzWKwERab-UfqkiSjtI0bs5uCpSWfnKL8inz6Dgm645H1AhimZeH1N6r0oKIopDAV412YScGKnnIqmkoKUrb2HzN0wfUId06000)

**BeanDefinitionReader**负责从Properties、Xml、Groovy等配置文件中读取BeanDefinition，并将其注册到BeanDefinitionRegistry中。

![BeanDefinitionReader](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLS4fDp7D9JSlCoop9pCyBIarCIItYuafCAYufIamkKKZEIImkLd24Sh4hnYQgO5EZfuTV7qmIXtPTNOM0elo2rAAIpDHYCWrmByhFBwiaKtD4RWvs_pgavgK0Gn40)

### 4.2、基于Java注解的配置

![AnnotationConfigRegistry](http://www.plantuml.com/plantuml/svg/TSvD3i9020NWFQUOPTtq18sswGMCdW2dT4AiJ0DeDEhTxK_ScBZayH5UROxgryi0mEAaFKOAZKXsTCxIPkav7IYnkJwUSClS1Lr6qg8TqApQRSko3BZUKBU4P9lLMaGfZguiQLOdDDfZF6EQnHl-VGhLQzA_ssOS1uxVmEdk03L9DzN_0000)

Spring3.0之前只支持@Component、@Controller、@Service、@Repository这几个组件级的注解。

Spring3.0开始支持Java注解配置Bean对象，也就是通过@Configuration配置类中方法上的@Bean注解来定义Bean对象。

其中Bean被加入到Spring容器有两种方式：

1、register()方法直接以组件类的方式注册AnnotatedGenericBeanDefinition到容器中，具体实现在`AnnotatedBeanDefinitionReader`中；

2、scan()方法通过扫描包下的所有组件类以批量的方式注册若干个ScannedGenericBeanDefinition到容器中，具体实现在`ClassPathBeanDefinitionScanner`中；

> 如果注册的是@Configuration注解的类，则在ConfigurationClassPostProcessor处理器中将所有@Bean注解方法的Bean注册到容器中，这部分内容后面会讲BeanFactoryPostProcessor的时候再详细解析。

# 5、可配置的ConfigurableListableBeanFactory

![ConfigurableListableBeanFactory](http://www.plantuml.com/plantuml/svg/jLPVRnj547_EVOfzF4jkFa7ird5IWTIGgh7W0V6mlXqd2-VTOtVErFoHUC1Gse9e1NqWa0fLq8Y78X61abIKBvEppRVWrklRk_CbRJ7mOkt-vZU_dPcRdVKLZLHXt0yzZmi4rQC1a6iyHRiXh0CLLsc0KWq_yBfIXkcU158WvK8RumRqkE38fV1tK76nIxef-XhjGyt8aLt0CgqjOu5-pRFiDz-gCeophZ8iVbMgpZ02_mPe6GvCY6PBCFrvWKSxf5glNMxEkLiqMhFyEkaqCm-ztIkGK_pvmYqX958JT2PFu2Q2O9hafYQXRjsfdBtTVHi2p0DuW-FUah8jafQGvOnjaIfMlakTMCrMLU2ZGWUqOfJlSGLj6bKQeT58RkqdJq-J-tUdd_sSt3wSFXhzDVh2S98d1sVFxao-_UJi-EEpq_raT9mSt1a_E1aVVTDeU1qIAA0uKyjhR2ARRMhUG796g3tQicnzqlovnZGDWzKy2vf6xF7T-69cdLIQCetzbnDZctpzCBdtHtBlSFpTm-cV-zCVxawUVZhvzNvo-jFuoyV9hT_J92QVduoVt7RXsnEgELmkv50dNA1BOKiog9FiuZ28G30GmR2z1y4xBe-C_M_bAjMxhcG42ZdeywgpX7OKXSE2yF1r6iQWNxBqSzRicjEGlszB-8zyym2anZI80BIMIgz3JofuDHCsBVtV2BTw26ePNLil1XgL75xSo8t6zF6ZyS5NuwF3PFJZsRFlfxyzC7stVvoCZbpV6KLc75wH8GDbxeoQpJzmblmsANWtvWgOlTBsr8o-uRwgdoytG0UAHoWLOMxfCZ9ou28sv_qledLlPMLb1t0-5vkkBPcRjtLYKcfBCG25e0WMTwQKQQLMLrTBKwmzdn2B8yn-7WrudIeGM33vXJM95gqrvxYUtT1haZ9GVc5DkcLRxjI1VdIH4rfRrQbDWxrPy5k0b56aldk85otby3PlHWgqvbBOrnAKVpwSVSd2eRQmXhG3Qi03y2i82HQHR4embis7JSPHgXkmopKgglmBSAAobPMOxq6rOusoRzbkEfPQl4uMdyZo6KqIbVOpTfvdgQNyvpMOossptmN6WCrcdwkixTiIHWR5NIvG6JD-1I7T1MInC3k1Z21xshNPsBTzXblW9Qx4EN_pFEySaAnzU2cEBhtGj_YdOJa5Pr_EtRi2WiNzy6y0)

ConfigurableListableBeanFactory结合了ListableBeanFactory、AutowireCapableBeanFactory、ConfigurableBeanFactory三个接口的功能。这三个接口也分别代表了Spring容器提供的三大基本功能：

ListableBeanFactory：根据类型或注解查找Bean

AutowireCapableBeanFactory：解析Bean的依赖关系并进行自动装配

ConfigurableBeanFactory：为容器提供了配置接口，以拓展容器的功能

我们想要拓展Spring IOC容器的功能主要就是通过ConfigurableBeanFactory开出的几个接口实现的：

### 5.1、BeanExpressionResolver

ConfigurableBeanFactory有两个关于Spring EL表达式的方法：

```java
	void setBeanExpressionResolver(@Nullable BeanExpressionResolver resolver);
	BeanExpressionResolver getBeanExpressionResolver();
```

在Spring中BeanExpressionResolver的实现是StandardBeanExpressionResolver

![BeanExpressionResolver](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAp2j9BKfBJ4vLS4fDp7EjA2XABIxEpCyBIYtEpobBBUB2BgnWKwDNb9cUKQAd45oIc9UIM9I2Gp-NGsfU2j0g0000)

```java
public class StandardBeanExpressionResolver implements BeanExpressionResolver {
  /// ...
  /** 该方法用于解析Spring EL表达式 **/
  public Object evaluate(@Nullable String value, BeanExpressionContext evalContext) throws BeansException {
		if (!StringUtils.hasLength(value)) {
			return value;
		}
		try {
			Expression expr = this.expressionCache.get(value);
			if (expr == null) {
				expr = this.expressionParser.parseExpression(value, this.beanExpressionParserContext);
				this.expressionCache.put(value, expr);
			}
			StandardEvaluationContext sec = this.evaluationCache.get(evalContext);
			if (sec == null) {
				sec = new StandardEvaluationContext(evalContext);
        ///...做了一堆的配置
				customizeEvaluationContext(sec);
				this.evaluationCache.put(evalContext, sec);
			}
			return expr.getValue(sec);
		}
		catch (Throwable ex) {
			throw new BeanExpressionException("Expression parsing failed", ex);
		}
	}

  // StandardBeanExpressionResolver提供了一个方法允许我们进行重载
	protected void customizeEvaluationContext(StandardEvaluationContext evalContext) {
	}
}
```

StandardBeanExpressionResolver提供了一个`customizeEvaluationContext`方法允许我们重载，我们唯一能做的就是对StandardEvaluationContext进行一些自定义配置。

### 5.2、Scope

Spring容器为我们提供了`SCOPE_SINGLETON`和`SCOPE_PROTOTYPE`两种基本的Scope，它还允许我们注册自己的Scope。比如Web应用中会用到的`request`和`session`两个Scope，另外Spring还提供了很多基础的Scope给我们，如线程级别`SimpleThreadScope`、事务级别的`SimpleTransactionScope`。

![Spring提供的Scope](http://www.plantuml.com/plantuml/svg/ZOr12i8m44NtEKKkq9x0HOjx1UC5qdHg0sbIPtx4XOSNJ2ceA5qbyDwRtmWi8qz1AHz1F5X7shWqarAlH-yUTQ01jJwaurp82cfjY6-1i4yHT4V1jXEmTT3jyZdHDPEW1TXt6IHVPxQRnazpeFF8PFjVa6qKw-1J_33ONqOKpP38AgW_-oMjAhsmKcm9tYSQYdsUmXC0)

通常我们往容器中注册Scope只需调用`beanFactory.registerScope`方法即可，另外Spring提供了一个BeanFactoryPostProcessor给我们——CustomScopeConfigurer。

> 关于BeanFactoryPostProcessor后面有一部分会介绍。


### 5.3、ConversionService

ConversionService，顾名思义，就是Spring提供给我们的类型转换服务。它主要有几种用途，解析配置文件中字符串，解析BeanDefinition中的property并绑定到对象中([DataBinder](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/DataBinder.html))，解析Web应用中的请求参数并绑定到Controller的入参对象中([WebDataBinder](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/WebDataBinder.html))，解析Spring EL表达式中的字面量。

![ConversionService](http://www.plantuml.com/plantuml/svg/fL9TJeGm47xlAVh89hit82w96ttJHBb0AGFMb7OpdPAGrRkxX8h1CQ3nZNu_NuTlMWHIIB6pGXX7W8tI86-zwm63yUuPi3SQBgBote9oKYitiPGL3z5QLTchtgeGykED33wYXd8um_uB98Kjq0ZkrcD6oGc2HdCcZukm9RM8BALcIO_LAsYQ4kPHokKeIRDf_kzyxwnO0do2rWJ2uI9wRsgfwdgcilahF-xbjJ_zUTxz9Fv5zGdyI-mzE42ZPs1LVQSqUHooxq2wg9bEoOZG-HwIr5GMrdzXhSh6j177pR3tAXWYyxV3OSF4jZEQqBGljELaBBywDSgzZ5ZOJj2eQ8dRH06kZftz0000)

除了ConversionService，ConfigurableBeanFactory还提供了[PropertyEditorRegistrar](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/PropertyEditorRegistrar.html)、 [TypeConverter](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/TypeConverter.html)、 [StringValueResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/StringValueResolver.html)等接口用于处理Bean对象和类型转换。

> 关于Spring Converter的内容可以参考这几篇教程:
>
> https://www.baeldung.com/spring-type-conversions
>
> https://www.baeldung.com/spring-mvc-custom-data-binder
>
> https://www.baeldung.com/spring-mvc-custom-property-editor

### 5.4、BeanPostProcessor

这个接口是用于扩展Spring功能的核心接口，事实上Spring容器本身的很多功能特性就是通过这个接口丰富的。

BeanPostProcessor有两个方法：

```java
public interface BeanPostProcessor {
  // 初始化方法之前调用
  Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
  // 初始化方法之后调用
  Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

很明显我们只要找到SpringBean什么时候初始化，就知道这两个方法什么时候被调用，Spring Bean的初始化分为四步：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
  /// ...
  /// 这里是整个初始化过程的代码
  protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    // 第一步：调用Aware方法
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

    // 第二步：调用BeanPostProcessor的postProcessBeforeInitialization方法
		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

    // 第三步：调用初始化方法
		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
    // 第四步：调用BeanPostProcessor的postProcessAfterInitialization方法
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
  ///...
}
```

1. **第一步：调用Aware方法**

   ```java
   	private void invokeAwareMethods(final String beanName, final Object bean) {
       /// 如果bean对象实现了相应的Aware接口，那这里会调用三种Aware方法：
       /// BeanNameAware、BeanClassLoaderAware、BeanFactoryAware
   		if (bean instanceof Aware) {
   			if (bean instanceof BeanNameAware) {
   				((BeanNameAware) bean).setBeanName(beanName);
   			}
   			if (bean instanceof BeanClassLoaderAware) {
   				ClassLoader bcl = getBeanClassLoader();
   				if (bcl != null) {
   					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
   				}
   			}
   			if (bean instanceof BeanFactoryAware) {
   				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
   			}
   		}
   	}
   ```

   

2. **第二步：调用BeanPostProcessor.postProcessBeforeInitialization方法**

   ```java
   	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
   			throws BeansException {
       /// 这个很简单：就是拿到BeanFactory里的所有BeanPostProcessor
       /// 然后依次调用postProcessBeforeInitialization方法
   		Object result = existingBean;
   		for (BeanPostProcessor processor : getBeanPostProcessors()) {
   			Object current = processor.postProcessBeforeInitialization(result, beanName);
   			if (current == null) {
   				return result;
   			}
   			result = current;
   		}
   		return result;
   	}
   ```

   

3. **第三步：调用bean对象的init方法**

   ```java
   	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
   			throws Throwable {
   
       /// 先看有没有实现Spring定义的InitializingBean接口
       /// 如果有的话就调用InitializingBean的afterPropertiesSet方法
   		boolean isInitializingBean = (bean instanceof InitializingBean);
   		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
   			if (logger.isTraceEnabled()) {
   				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
   			}
   			if (System.getSecurityManager() != null) {
   				try {
   					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
   						((InitializingBean) bean).afterPropertiesSet();
   						return null;
   					}, getAccessControlContext());
   				}
   				catch (PrivilegedActionException pae) {
   					throw pae.getException();
   				}
   			}
   			else {
   				((InitializingBean) bean).afterPropertiesSet();
   			}
   		}
   
       /// 然后调用自定义的init方法
       /// 这个init方法是xml或@Bean注解配置的init-method属性
   		if (mbd != null && bean.getClass() != NullBean.class) {
   			String initMethodName = mbd.getInitMethodName();
   			if (StringUtils.hasLength(initMethodName) &&
   					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
   					!mbd.isExternallyManagedInitMethod(initMethodName)) {
   				invokeCustomInitMethod(beanName, bean, mbd);
   			}
   		}
   	}
   
   ```

   这里也许会有人想到使用[JSR-250](https://en.wikipedia.org/wiki/JSR_250)规范中定义的@PostConstruct注解进行初始化。

   Spring提供了[CommonAnnotationBeanPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html)和[InitDestroyAnnotationBeanPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/annotation/InitDestroyAnnotationBeanPostProcessor.html)来处理JavaEE标准中的注解，如@PostConstruct、@PreDestroy、@Resource。

   在InitDestroyAnnotationBeanPostProcessor中可以看到它是通过实现BeanPostProcessor#postProcessBeforeInitialization方法并在其中调用Bean的@PostConstruct注解方法的：

   ```java
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
      try {
         /// 这里的initMethod就是通过@PostConstruct注解解析出来的方法
         metadata.invokeInitMethods(bean, beanName);
      }
      catch (InvocationTargetException ex) {
         throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
      }
      catch (Throwable ex) {
         throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
      }
      return bean;
   }
   ```

   所以关于**初始化方法的调用顺序**是：

   **@PostConstruct==>InitializingBean.afterPropertiesSet==>Xml或@Bean中定义的init-method**

   而且Bean中@PostConstruct方法可以定义多个，但InitializingBean因为是接口所以afterPropertiesSet只有一个方法，init-method也只能有一个。

   > 关于Spring初始化的过程可以参考[这篇教程](https://www.baeldung.com/running-setup-logic-on-startup-in-spring)

4. **第四步：调用BeanPostProcessor.postProcessAfterInitialization方法**

   ```java
   	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
   			throws BeansException {
       /// 这个很简单：就是拿到BeanFactory里的所有BeanPostProcessor
       /// 然后依次调用postProcessBeforeInitialization方法
   		Object result = existingBean;
   		for (BeanPostProcessor processor : getBeanPostProcessors()) {
   			Object current = processor.postProcessAfterInitialization(result, beanName);
   			if (current == null) {
   				return result;
   			}
   			result = current;
   		}
   		return result;
   	}
   ```

用一张图总结一下：

![initializeBean](http://tva1.sinaimg.cn/large/bda5cd74gy1gbdyhiplcxj21bc0bewgp.jpg)

### 5.5、BeanPostProceesor的常见实现

![BeanPostProceesorImpl](http://www.plantuml.com/plantuml/svg/fP31IWCn48RlUOgyGFS9F3IAdXGMB7eUaxzj83jPPgPh2pwyxg8KiU2MFIM7x_iI_hKQYHswubncWsmfmj-2kArghTG8rIgEtjI4eldmVGbfo9fvDmCTaOUliyefl9FWH_sjkJybV_FH56ojyQ7lIuvakV9TPSFHfj1PlkWs_Xao5DXLpcEbjCaTNa43P8uZURUtPiOo_n5ZRRMwShVPzFc19zY-fXSgEKsRBWu6FN4CpDMctcWkRhOGpMhWYFjZH3-6DqAivSAVtHgS3btv1000)

Spring内部实现各种Aware的ApplicationContextAwareProcessor；

Web应用中装配ServletContext的ServletContextAwareProcessor；

类加载时编织AspectJ切面的LoadTimeWeaver被LoadTimeWeaverAwareProcessor装配；

> 关于加载时编织参考[Spring官方文档](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-aj-ltw)和[AspectJ官方文档](https://www.eclipse.org/aspectj/doc/released/devguide/ltw.html)

另外还有几个基于AOP实现的BeanPostProceesor，最常用的就是@Async注解实现异步方法，而这个就是AsyncAnnotationBeanPostProcessor处理器实现的。

> 关于Spring异步方法的使用可以参考[这篇教程](https://www.baeldung.com/spring-async)

### 5.6、BeanPostProcessor的子接口

![BeanPostProcessor](http://www.plantuml.com/plantuml/svg/bP7DIiKm44RtUOei5RnluEAsTDE5Ml09fkcNCj9EP395yEVTjKMmM8NSPS8v7mVcd8tKbdboZiMWaG9y3P8kPUiq1UISzCqzz4y8vfz_Vcl4f6Y5ZMdYLp9ESlMDzI2vyO-cBEFskASPrt-CLD6W5sryx3fRoKPYl7dL2oaEvJkwGJPTGX5x1nqnh4I3o6jV-iMwW-vltq-degP_r6DWcLXIUuOCNrV-1000)

BeanPostProcessor有三个字接口：

1、InstantiationAwareBeanPostProcessor：添加了实例化对象的回调方法，以及属性值被自动装配时的回调；

2、DestructionAwareBeanPostProcessor：添加了销毁对象前的回调方法；

3、MergedBeanDefinitionPostProcessor：合并BeanDefinition后调用；



既然InstantiationAwareBeanPostProcessor是实例化和属性自动装配的回调，让我们看一下这块代码：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
  ///...
  // Spring创建Bean对象的整个过程
  protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   RootBeanDefinition mbdToUse = mbd;

   /// 解析并加载BeanDefinition中定义的Class
   /// ... 代码省略

      /// ...省略异常处理代码
      // 第一步：
      // 调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation方法
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         // Processor的方法可能返回一个代理对象，
         // 比如Spring AOP的自动代理(@EnableAspectJAutoProxy)。
         // 这种情况比较少，可以不考虑
         return bean;
      }
    
      /// ...省略异常处理代码
      /// 根据BeanDefinition创建对象
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      return beanInstance;
  }
    
  /// 调用内部的创建方法  
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// 第二步： 实例化Bean对象
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// 调用MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition
    // 让Processor对BeanDefinition最后再进行一些处理
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
        /// ...省略异常处理代码
			  applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

		// 将Bean对象缓存起来以解决循环引用的问题
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		Object exposedObject = bean;
    /// ...省略异常处理代码
    // 填充Bean的属性
	  populateBean(beanName, mbd, instanceWrapper);
    // 初始化Bean对象
	  exposedObject = initializeBean(beanName, exposedObject, mbd);

		///...

    /// ...省略异常处理代码
		// 注册Disposable接口和DestructionAwareBeanPostProcessor的DisposableBeanAdapter
	  registerDisposableBeanIfNecessary(beanName, bean, mbd);

		return exposedObject;
	}
  
  protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		///...

		// 调用InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation方法
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 自动装配对象
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// 根据名称自动装配
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// 根据类型自动装配
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
        /// 调用InstantiationAwareBeanPostProcessor.postProcessPropertyValues
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		if (pvs != null) {
      // 将依赖的Bean对象注入到新创建的Bean中
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
  ///...
}
```

至此，创建Bean的整个过程可以总结为下图：

![CreateBean](http://tva1.sinaimg.cn/large/bda5cd74gy1gbdz16c5wuj21gk0tu7b3.jpg)



# 6. 功能更强大的容器——ApplicationContext

![ApplicationContext继承图](http://www.plantuml.com/plantuml/svg/ZPBVRjCm5CRl_HHvWUu141Shqu0BaARn16vmMqkk7TbE1G8kM0i8A4r1tGHKuWzZR3UZSC76KbPU9dQwjy1PaF0Kc-qkEVvypk_xZfU5X5p67GA0n9AWIq4zYWWEeSIChZ0gqHsPptRrqzzgCWS0cm9lmX05Ln2aLs4e6RzhbsyY8M0BtM8n3n6WJA90iYYu1_BnNfOE5xlR-jse8rhvstw-lvLlxkZXZagsWJewqEEf7ZpK-v_pxxDIeP8DlVbD3RRKgu6Q79-yUMI-mGDht0ri-1i4MJJMwNMCEEHXszWXJeLjThMBg5oB6mIBDk8bUeD9oJg6NYUZR3x9qiSgUQb-zhBqUJOxA0YVlL_qZiyWhT8kvensJBSL61cuVp7OVTAnIIGVJjMddafUdpn9JBV5y50bPmsk8t4QvHgKfaIBNzAjudquqY--4bOw2Q9IN8Qz-D7NwGZewLXzsQXJ6PXMRRttKVqw7NauLVE-6MdDhBu0wz1KchWLyyAOGmYid8FBzsirvueg8j-cWCT_SOkWfgovOAiRAEHEgVrS_Ig71Q_MuyIhzxp0qYB7hRRR8wZMoX7dxGSsXEI6AGW952Ae__rrI3tATTJaIBGz_S-_L0AwqDpebhsbfV_n-e_AP2vcmA_oD_IRkp3SDKGHnw4h5h2CwEsuG_u0)

ApplicationContext在BeanFactory容器的基础上继承了另外四个接口：

1. [EnvironmentCapable接口](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/EnvironmentCapable.html)：暴露了应用当前的环境配置信息的[Environment](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/Environment.html)接口。Environment提供了*profiles*和*properties*两方面的配置信息：

   profiles指的是一组Bean的集合，这些Bean可以在xml中配置profile或着使用[@Profile](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Profile.html)注解。Environment对于Profile来说，就是用于决定当前哪个profile被激活了，那个profile应该默认被激活。

   > Spring Profile的相关内容可以参考[这篇教程](https://www.baeldung.com/spring-profiles)

   properties在应用中扮演着重要角色，主要来源形式有：properties文件、JVM系统参数、系统环境变量、JDNI、Servlet容器参数等。

2. [MessageSource接口](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/MessageSource.html)：提供了消息的参数化与国际化，避免了开发人员编写大量额外的代码处理各种复杂的情况。

   > 在SpringBoot中使用MessageSource处理校验报错信息，可以参考[这篇教程](https://www.baeldung.com/spring-custom-validation-message-source)

3. [ApplicationEventPublisher接口](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationEventPublisher.html)：为Spring容器提供了事件发布功能。ApplicationContext容器通过代理[ApplicationEventMulticaster](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/event/ApplicationEventMulticaster.html)实现事件发布功能（具体可以参看AbstractApplicationContext的相关代码）。Spring容器本身也会发布各种事件，如*ContextRefreshedEvent，ContextStartedEvent，RequestHandledEvent*等。

   > Spring Events的相关内容可以参考[官方教程](https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2)或[这篇教程](https://www.baeldung.com/spring-events)，Spring容器内置事件可参考[这篇教程](https://www.baeldung.com/spring-context-events)

4. [ResourcePatternResolver接口](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/support/ResourcePatternResolver.html)：提供了解析资源路径的功能。如`/WEB-INF/*-context.xml`、`classpath*:context.xml`这样的路径。

其次就是中间过渡的ConfigurableApplicationContext接口：

ConfigurableApplicationContext在ApplicationContext的基础上提供了一些配置接口，和ConfigurableBeanFactory类似，ConfigurableApplicationContext将配置和生命周期方法封装在此主要是为了避免对客户端代码可见，ConfigurableApplicationContext接口的方法应该只在应用启动和关闭的时候调用。这里面有这么几个方法比较重要：

```java
// 启动容器加载配置并实例化所有的单例Bean
// 所有ApplicationContext容器必须refresh()了才能使用
void refresh();
// 关闭容器释放所有相关资源，包括销毁所有缓存的单例Bean
void close();
```

### 6.1、AbstractApplicationContext

先让我们看一下refresh()方法。

```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备ApplicationContext以进行refresh
			prepareRefresh();

			// 通知子类去刷新内部的BeanFactory
      // AbstractApplicationContext提供了三个抽象方法由子类去实现：
      // void refreshBeanFactory()
      // void closeBeanFactory()
      // ConfigurableListableBeanFactory getBeanFactory()
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 对这个BeanFactory进行配置
			prepareBeanFactory(beanFactory);

			try {
				// 让子类再对BeanFactory进行相应的配置
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactoryPostProcessor
        // 并找到beanFactory容器中实现了BeanFactoryPostProcessor接口的bean，一起调用
        // 主要这个这个接口和BeanPostProcessor有所区别
				invokeBeanFactoryPostProcessors(beanFactory);

				// 找出beanFactory中实现了BeanPostProcessor的Bean
        // 并将它们以BeanPostProcessor的角色注册到beanFactory中
				registerBeanPostProcessors(beanFactory);

				// 初始化用于i18n的MessageSource
				initMessageSource();

				// 初始化用于事件分发的ApplicationEventMulticaster
				initApplicationEventMulticaster();

				// 让子类做一些其他的初始化，比如web应用中要初始化ThemeSource
				onRefresh();

				// 找到容器中所有的ApplicationListener并将其注册到ApplicationEventMulticaster中
				registerListeners();

				// 对BeanFactory进行最后的初始化，然后冻结配置，预初始化所有的单例对象
				finishBeanFactoryInitialization(beanFactory);

				// 最后一步：完成最后的刷新工作并发布ContextRefreshedEvent事件
				finishRefresh();
			}

			///...
		}
	}
```

AbstractApplicationContext提供了很多模版方法给子类去实现，接下来我们看一下最主要的两个子类。

### 6.2、GenericApplicationContext与AbstractRefreshableApplicationContext

![Spring容器的两个体系](http://www.plantuml.com/plantuml/svg/dLAxJW915EttA_O7-05Z0P98OZGnqb3G3Ej5DfcT8RCN8YKnMWXeOf36niI7MqaX92hmD-o3ln1ogPJTMIbtpZdtddlEPbra2XiEDmnR8AWgiy3CIr6rpngALJZawdLkMmnjAPRF2ETei8gBYbbeMfovhfbRVwPdda1LWLkZyLk8o5zwQSdX6yX6yfdcRYQJT5iiHCe2252szNzXkf1qh9XvclnISscs9k0abFJxDeTYoqLmjfsGNnLzpB0Mqt5S3QGk8iED7Gc9O5OaedHspFda9PmEGgyJ3BxywMtmuVbNtJrWeYpcdxsmx_dNZz5ivZyF5XVUuh8NpjvNo2HwRI_1-VTDCEv4mtkDbpLAT-Zyb8uEoGoj8pF1i9ySHrduTrrDvi73A6oFWz4adTs2ahBPIk6OYBzDQvRaX918h_GRwlSSi2PRDXIzFVtPPMH1d3OS_WC0)

[**GenericApplicationContext**](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/support/GenericApplicationContext.html)和[**RefreshableApplicationContext**](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/support/AbstractRefreshableApplicationContext.html)两个体系内的容器实现本质上都是代理了**DefaultListableBeanFactory**提供的功能。

两者的区别在于GenericApplicationContext只能refresh()一次，而RefreshableApplicationContext允许refresh()多次。

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {

	private final DefaultListableBeanFactory beanFactory;

  /// ...
	public GenericApplicationContext() {
    // 在构造的时候就创建了beanFactory
    // 之后的refresh()等操作都是对这个beanFactory进行操作
		this.beanFactory = new DefaultListableBeanFactory();
	}
  ///...
}

public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
  ///...
  
	@Nullable
	private DefaultListableBeanFactory beanFactory;
  
  public AbstractRefreshableApplicationContext() {
	}
  
  // 实现AbstractApplicationContext提供的模版方法
  // 每次refresh()都会重新创建新的BeanFactory
  protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
      // 销毁已有的容器
			destroyBeans();
			closeBeanFactory();
		}
		try {
      // 创建新的容器
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
  
	protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
  ///...
}
```

### 6.3、BeanFactoryPostProcessor

不要将[BeanFactoryPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanFactoryPostProcessor.html)和前面的[BeanPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html)搞混，这个接口是BeanFactory初始化后的回调接口。

BeanFactoryPostProcessor可以与BeanDefinition进行交互并进行修改，但不能与Bean实例交互。这样做可能会导致bean过早实例化，从而违反了容器规则并造成了意外的副作用。如果需要与bean实例交互，请考虑实现[BeanPostProcessor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html)。

> 常见的BeanFactoryPostProcessor有CustomScopeConfigurer(自定义Scope)、CustomAutowireConfigurer(自定义@Qualifier)、CustomEditorConfigurer(自定义PropertyEditor)。它们都是用来扩展BeanFactory功能的。还有一个EventListenerMethodProcessor是用来处理@EventListener注解的。

BeanFactoryPostProcessor另一个核心的实现类就是ConfigurationClassPostProcessor，它是用来解析@Configuration注解类的处理器。

![BeanFactoryPostProcessor](http://www.plantuml.com/plantuml/svg/ZP112i9034NtSugvG7i25yMgAuNY3S8qCmtK199K43oyTIS3bkxVUo6_CmVrvJ67GEoe6HB68m9V8BdeQn3pGIMXcMY5d30JavFm7GkPrtJuruc7TzFiIMmLMgKoUHlFJsI_hYuowWrzal7NtxnHfNhXJ6LH-CBz36RLGntok6xr0G00)

前面提到过Spring3.0开始支持@Configuration的配置，而它是由AnnotationConfigApplicationContext透出的，所以我们从这个类开发分析。

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {

	private final AnnotatedBeanDefinitionReader reader;

	private final ClassPathBeanDefinitionScanner scanner;
  
	public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
  ///...
  
 	@Override
	public void register(Class<?>... componentClasses) {
		Assert.notEmpty(componentClasses, "At least one component class must be specified");
		this.reader.register(componentClasses);
	}
  
	@Override
	public void scan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		this.scanner.scan(basePackages);
	}

  ///...
}
```

这里的AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner都是为了解析@Component注解并注册BeanDefinition。

当然这里的@Component还包括@Repository、@Service、@Controller、@Configuration这几个复合注解。

Annotate和Class在读取和扫描过程中都会去调用`AnnotationConfigUtils`的`registerAnnotationConfigProcessors`方法注册Spring的注解处理器，其中包括：

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, Object source) {

   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
      }
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

  // 注册@Configuration、@Bean、@Import、@ComponentScan等注解的ConfigurationClassPostProcessor
   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

  // 注册@Autowired、@Value等注解的AutowiredAnnotationBeanPostProcessor
   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

  // 注册处理@Required注解的RequiredAnnotationBeanPostProcessor
   if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

  // 检查是否有JSR-250的依赖
  // 有则注册处理@PostConstruct、@PreDestroy、@Resource等注解的
  // CommonAnnotationBeanPostProcessor.
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // 检查JPA的依赖, 有则注册处理@PersistenceUnit、@PersistenceContext注解的
   // PersistenceAnnotationBeanPostProcessor.
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

  // 注册处理@EventListener注解的EventListenerMethodProcessor
   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }
  // 为@EventListener注解的方法提供一个事件类型适配器
   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

这里我们主要关注ConfigurationClassPostProcessor，因为它实现了BeanDefinitionRegistryPostProcessor接口，所以它会回调两次：

1、先回调BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry方法，由ConfigurationClassParser解析出所有的ConfigurationClass，再由ConfigurationClassBeanDefinitionReader将所有的ConfigurationClass读取并将BeanDefinition注册到容器中。

2、然后回调BeanFactoryPostProcessor.postProcessBeanFactory方法，其中ConfigurationClassEnhancer使用Cglib对@Configuration类进行动态代理增强，主要是为了调用@Bean注解方法时由BeanFactory负责创建对象并将其放入容器中。

```java
@Configuration
public class DemoConfiguration {
    
    @Bean
    public ObjectA a(){
        return new ObjectA();
    }
    
    @Bean
    public ObjectB b(){
      // 正因为ConfigurationClassEnhancer
      // 所以调用a()方法时会从BeanFactory中取出bean，而不是每次创建一个新对象
        return new ObjectB(a());
    }
}
```

> 对于ConfigurationClassPostProcessor更详细的源码分析请参考这两篇文章：
>
> [ConfigurationClassPostProcessor源码分析](https://xuanjian1992.top/2019/08/02/ConfigurationClassPostProcessor源码分析/)
>
> [Spring Boot源码分析](https://atbug.com/spring-boot-configuration-annotation/)

其中比较重要的是@Import注解的处理，因为Spring中很多@EnableXxx注解都是基于这个实现的。

@Import注解内可以填三种类型：

1、@Configuration注解类，2、ImportSelector接口实现类，3、ImportBeanDefinitionRegistrar接口实现类

@EnableXxx  => @Import(XxxConfiguration ==> @Configuration) => XxxBeanPostProcessor

​                              eg. @EnableScheduling

​                        => @Import(XxxSelector ==> ImportSelector)

​                             eg. @EnableAsync、@EnableDubboConfig

​                        => @Import(XxxRegistrar ==> ImportBeanDefinitionRegistrar)

​                              eg. @EnableAspectJAutoProxy、@EnableJpaAuditing、@EnableJpaRepositories

看一下这块的代码：

```java
	private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {

		if (importCandidates.isEmpty()) {
			return;
		}

		if (checkForCircularImports && isChainedImportOnStack(configClass)) {
			this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
		}
		else {
			this.importStack.push(configClass);
			try {
				for (SourceClass candidate : importCandidates) {
          // 处理ImportSelector实现类
					if (candidate.isAssignable(ImportSelector.class)) {
						Class<?> candidateClass = candidate.loadClass();
						ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
						ParserStrategyUtils.invokeAwareMethods(
								selector, this.environment, this.resourceLoader, this.registry);
						if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
              // 如果实现的是DeferredImportSelector接口则延迟处理
							this.deferredImportSelectors.add(
									new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
						}
						else {
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
              // 直接拿到import的类，递归处理一下
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
							processImports(configClass, currentSourceClass, importSourceClasses, false);
						}
					}
          // 处理ImportBeanDefinitionRegistrar实现类
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
						Class<?> candidateClass = candidate.loadClass();
						ImportBeanDefinitionRegistrar registrar =
								BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
						ParserStrategyUtils.invokeAwareMethods(
								registrar, this.environment, this.resourceLoader, this.registry);
            // 注解将ImportBeanDefinitionRegistrar放到configClass中
            // 最后由ConfigurationClassBeanDefinitionReader处理
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					else {
						// 否则当作@Configuration注解的类处理
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
						processConfigurationClass(candidate.asConfigClass(configClass));
					}
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]", ex);
			}
			finally {
				this.importStack.pop();
			}
		}
	}
```

至此已将Spring IOC的大部分核心内容的整体结构理清楚了，更多细节上的知识还需要查看源码去理解。



参考资料：

https://www.baeldung.com/spring-factorybean

https://www.baeldung.com/spring-profiles

https://spring.io/blog/2007/06/05/more-on-java-configuration

https://spring.io/blog/2014/11/04/a-quality-qualifier

https://spring.io/blog/2011/08/09/what-s-a-factorybean

https://spring.io/blog/2015/02/11/better-application-events-in-spring-framework-4-2

https://spring.io/blog/2010/01/05/task-scheduling-simplifications-in-spring-3-0

https://spring.io/blog/2011/02/17/spring-3-1-m1-introducing-featurespecification-support

https://spring.io/blog/2016/03/04/core-container-refinements-in-spring-framework-4-3