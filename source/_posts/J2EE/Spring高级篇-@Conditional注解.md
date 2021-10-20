---
title: Spring高级篇-@Conditional注解
date: 2018-2-4
categories: J2EE
---

Spring3开始提供的profile机制用起来的确很爽，在Spring4中提供了一种更通用的条件化Bean定义机制。

# @Conditional注解

**使用@Conditional注解**

定义一个Bean，这个Bean只有在满足MagicExistCondition中定义的条件时才会创建。

```java
@Bean
@Conditional(MagicExistCondition.class)
public MagicBean magicBean() {
    return new MagicBean();
}
```

这里的@Conditional注解定义如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

	/**
	 * 我们需要实现Condition接口，并实现他的matches方法
	 */
	Class<? extends Condition>[] value();

}

```

**实现Condition接口**

```java
public class MagicExistCondition implements Condition {

	/**
	 * 如果返回true，Bean将会被创建并注册到Spring容器中；否则不会创建Bean
	 */
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		Environment env = context.getEnvironment();
		// 当定义了magic属性的时候返回true
		return env.containsProperty("magic");
	}

}
```

这里的matches方法有两个参数，分别为ConditionContext，AnnotatedTypeMetadata类型

**ConditionContext**

```java
/**
 * 包含了可能被Condition用到的所有信息
 * @since 4.0
 */
public interface ConditionContext {

	/**
	 * 包含了用于描述Bean实例的BeanDefinition
	 */
	BeanDefinitionRegistry getRegistry();

	/**
	 * Spring容器，可以检查某些Bean是否存在
	 */
	ConfigurableListableBeanFactory getBeanFactory();

	/**
	 * 当前应用包含的环境变量
	 */
	Environment getEnvironment();

	/**
	 * 可以用来加载外部文件
	 */
	ResourceLoader getResourceLoader();

	/**
	 * 类加载器
	 */
	ClassLoader getClassLoader();

}
```

**AnnotatedTypeMetadata**

```java
public interface AnnotatedTypeMetadata {

	/**
	 * 查看@Bean注解的方法是否有annotationName参数指定的注解
	 */
	boolean isAnnotated(String annotationName);

	/**
	 * 如果@Bean注解的方法有annotationName参数指定的注解，
	 * 可以通过这个方法获取这个注解下的所有属性；
	 * 如果没有这个注解，将会返回null
	 */
	Map<String, Object> getAnnotationAttributes(String annotationName);

	/**
	 * classValuesAsString参数为true，将注解中Class类型的属性以String的形式返回。
	 */
	Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);

}
```

# Spring4对@Profile注解进行的重构

从Spring4开始，@Profile注解使用@Conditional注解实现。我们可以以它为例，学习如何使用@Conditional注解。

```java
/**
 * @since 3.1
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 使用@Conditional注解，并定义了ProfileCondition
@Conditional(ProfileCondition.class)
public @interface Profile {
	String[] value();
}
```

可以看以下ProfileCondition的实现

```java
/**
 * @since 4.0
 */
class ProfileCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		if (context.getEnvironment() != null) {
			// 获取Profile注解的所有属性
			MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
			if (attrs != null) {
				// 遍历注解中value属性的所有元素
				for (Object value : attrs.get("value")) {
					// 检查@Profile注解中的是否处于激活状态
					if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
						return true;
					}
				}
				return false;
			}
		}
		return true;
	}

}
```

# SpringBoot自动化配置中的Condition

Spring中的自动化配置基本就是靠这个@Condition注解实现的，所以在springboot-autoconfig中有一大堆实现了Condition接口的类：

![Condition接口实现类](http://img-blog.csdn.net/20180205102900891?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

另外SpringBoot还提供了很多类似于@Profile的注解：

![常见的SpringBootCondition和它相应的注解](http://img-blog.csdn.net/20180205102919302?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 如果有机会再后续的文章中会对SpringBoot的自动化配置进行详细分析。



> 参考：
>
> Spring In Action
>
> Spring Framework Documentation