---
title: 安卓常用第三方框架-Gson
date: 2017-02-17
tags:
- Android
- Gson
categories: Android
description: 安卓常用第三方框架-Gson
---
# Gson简介
json因其轻量、高效等特性，而被广泛用作移动开发的信息交互的载体。
我们知道AndroidSDK提供了org.json工具包来解析Json数据，但是仍然避免不了解析过程中的一系列重复工作。所以就出现了许多第三方JSON解析框架。[JSON官方网站](http://www.json.org)也给我们列出了在各种语言中JSON的解析策略:
![这里写图片描述](http://img-blog.csdn.net/20170217205859340?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
可以看到java语言中解析JSON的库有很多，但由于移动设备硬件与软件等各方面的因素，在这些库中比较适合用于移动开发的主要有Gson和FastJson。
今天我们先来介绍以下Gson。

Gson是Google自家开发的，所以Google也极力推荐使用这个库来解析JSON数据。
Google自己是这么说的：已经有一些开源项目可用来将Java对象转换成JSON，然而这里面大多数需要你在Bean类中添加Java注解，很多时候需要你自己去写Bean对象，而且大多数也不支持Java泛型。所以Google开发Gson有以下几个开发目标：
1.提供超简单的toJson()和fromJson()方法来实现Java对象与JSON对象的相互转换。
2.允许不可变对象与JSON之间进行相互转换。
3.支持Java泛型
4.支持任何复杂的对象(深层包含与泛型)
### 添加Gradle依赖
```java
compile 'com.google.code.gson:gson:2.8.0'
```
### 下载地址：
Github：https://github.com/google/gson
MVN：https://mvnrepository.com/artifact/com.google.code.gson/gson
### API文档
http://www.javadoc.io/doc/com.google.code.gson/gson/2.8.0

# GSON的使用
1. 创建Gson对象
  调用构造方法``new Gson()``，或者使用``GsonBuilder``类构造Gson对象
2. 调用toJson()或fromJson()方法
  Gson对象中toJson()方法用来序列化，fromJson()方法用来反序列化

# 示例
### 1、基本类型
```java
// 序列化
Gson gson = new Gson();
gson.toJson(1);            // ==> 1
gson.toJson("abcd");       // ==> "abcd"
gson.toJson(new Long(10)); // ==> 10
int[] values = { 1 };
gson.toJson(values);       // ==> [1]

// 反序列化
int one = gson.fromJson("1", int.class);
Integer one = gson.fromJson("1", Integer.class);
Long one = gson.fromJson("1", Long.class);
Boolean false = gson.fromJson("false", Boolean.class);
String str = gson.fromJson("\"abc\"", String.class);
String[] anotherStr = gson.fromJson("[\"abc\"]", String[].class);
```

### 2、简单Java对象
```java
class BagOfPrimitives {
  private int value1 = 1;
  private String value2 = "abc";
  private transient int value3 = 3;
  BagOfPrimitives() {
    // 无参构造
  }
}

// 序列化
BagOfPrimitives obj = new BagOfPrimitives();
Gson gson = new Gson();
String json = gson.toJson(obj);

// ==> 生成的JSON字符串是   {"value1":1,"value2":"abc"}
// 反序列化
BagOfPrimitives obj2 = gson.fromJson(json, BagOfPrimitives.class);
// ==> obj2与obj内容相同
```

** 注意： **
**1、序列化的Java对象不能包含循环引用(内部类的字段是外部类的引用)，因为这样会导致无限递归**
**2、默认情况下，当前类以及父类的所有字段都将被序列化**
**3、Java对象的字段被标记为transient，那么默认下情况该字段将不被序列化或反序列化**
**4、序列化时，默认会跳过值为null的字段；反序列化时，JSON中没有java对象对应的字段则值为空**
**5、字段是synthetic的，该字段将不被序列化或反序列化**
**6、可以使用@Expose注解来设置是否序列化类中的字段**
**7、可以使用@SerializedName注解设置序列化或反序列化时的key，其中alternate可以设置反序列化时备选的key**
上面所说的默认情况，可以在使用GsonBuilder创建Gson对象之前进行设置。比如
serializeNulls()：用来配置是否序列化值为null的字段
disableInnerClassSerialization()：调用该方法后Gson对象将不会序列化内部类
setDateFormat()：设置Date类型的序列化格式
...

### 3、嵌套对象
Gson可以反序列化静态内部类，但是不能反序列化普通的非静态内部类，因为它们构造对象时需要传入外部类对象的引用。所以反序列化时要么使用静态内部类，要么为非静态内部类定义一个InstanceCreator(Gson中已经提供了该接口)。比如下面的例子：
```java
public class A {
  public String a;

  class B {

    public String b;

    public B() {
      // B的无参构造
    }
  }
}
```

由于B类是内部类所以Gson不能将{"b":"abc"}反序列化成B的实例。需要反序列化B类则需要将B类指定为静态内部类。另一种解决方案是为B类定义一个InstanceCreator。

```java
public class InstanceCreatorForB implements InstanceCreator<A.B> {
	private final A a;
	public InstanceCreatorForB(A a)  {
		this.a = a;
	}
	  public A.B createInstance(Type type) {
	    return a.new B();
	}
}
```

后者虽然可行，但是Gson不推荐使用该方法。

### 4、数组
```java
Gson gson = new Gson();
int[] ints = {1, 2, 3, 4, 5};
String[] strings = {"abc", "def", "ghi"};

// 序列化
gson.toJson(ints);     // ==> [1,2,3,4,5]
gson.toJson(strings);  // ==> ["abc", "def", "ghi"]

// 反序列化
int[] ints2 = gson.fromJson("[1,2,3,4,5]", int[].class);
// ==> ints2 与 ints值相同
```

### 5、集合对象
```java
Gson gson = new Gson();
Collection<Integer> ints = Lists.immutableList(1,2,3,4,5);//Guava Collections库

// 序列化
String json = gson.toJson(ints);  // ==> json is [1,2,3,4,5]

// 反序列化
Type collectionType = new TypeToken<Collection<Integer>>(){}.getType();
Collection<Integer> ints2 = gson.fromJson(json, collectionType);
// ==> ints2 is same as ints
```

### 6、其他特殊情况
**Collections集合的局限性**
Gson能序列化任意的集合对象，但是不能反序列化，因为序列化成字符串不需要指定生成结果的类型，相反，反序列化时，使用哪个集合必须是指定的，泛型也必须是指定的。

#### 1、泛型的序列化与反序列化
当你调用toJson(obj)时，Gson会调用obj.getClass()获取字段信息来序列化。类似地，在fromJson()方法中你可以传入Obj.class来指定你要实例化的对象。但是，如果对象是泛型(比如说Collections集合)，由于泛型对象的Java类型擦除(Java Type Erasure)而丢失类型信息。
```java
class Foo<T> {
  T value;
}
Gson gson = new Gson();
Foo<Integer> foo = new Foo<Integer>();
gson.toJson(foo); // 无法正确序列化value值

gson.fromJson(json, foo.getClass()); // 反序列失败
```

上面的代码无法将value解析成Integer类型，因为Gson调用foo.getClass()来获取类信息返回的是原始的Foo.class。也就是说Gson无法知道这是一个``Foo<Integer>``对象。
我们可以通过指定正确的泛型参数来解决这个问题，Gson提供给我们TypeToken类，我们可以创建一个匿名内部类对象，该类中getType()方法可以获取完整的泛型参数。

```java
Type fooType = new TypeToken<Foo<Bar>>() {}.getType();
gson.toJson(foo, fooType);

gson.fromJson(json, fooType);
```
#### 2、序列化与反序列化混合类型的集合
某些时候可能需要处理包含混合类型的JSON数组，比如：
```[ 'hello', 5, { name:'GREETINGS', source:'guest' } ]```
与之等价的集合对象如下：
```java
class Event {
  private String name;
  private String source;
  private Event(String name, String source) {
    this.name = name;
    this.source = source;
  }
}
Collection collection = new ArrayList();
collection.add("hello");
collection.add(5);
collection.add(new Event("GREETINGS", "guest"));

Gson gson = new Gson();
//序列化，不会出现任何问题
String json = gson.toJson(collection);
//反序列化的结果可能不符合要求
Collection json = gson.fromJson(json,ArrayList.class);
```
上面的collection对象可以不做任何处理使用toJson(collection)来序列化

但是你调用Gson.fromJson()方法反序列化，得到的结果可能不是你想要的。Gson.fromJson会要求你提供一个明确的集合类型。比如你在这里调用``Collection c2 = gson.fromJson(json,ArrayList.class)``，它会将5解析成字符串"5"，将 ``{ name:'GREETINGS', source:'guest' }``解析成含有两个元素的map映射。

这个问题，Gson提供了三种解决方式：

（1）、使用Gson的解析器：底层基于流的解析器JsonReader或基于文档树的解析器JsonParser，然后再使用Gson.fromJson方法解析数组的每一个对象。
(JsonReader和JsonParser类似于XML的SAX和DOM两种解析方式)

这是首选的方法。示例如下：
```java
public class RawCollectionsExample {
	static class Event {
		private String name;
		private String source;
		private Event(String name, String source) {
			this.name = name;
		      	this.source = source;
		}
		@Override
		public String toString() {
			return String.format("(name=%s, source=%s)", name, source);
		}
	}

	@SuppressWarnings({ "unchecked", "rawtypes" })
	public static void main(String[] args) {
		Gson gson = new Gson();
		Collection collection = new ArrayList();
		collection.add("hello");
		collection.add(5);
		collection.add(new Event("GREETINGS", "guest"));
		String json = gson.toJson(collection);
		System.out.println("Using Gson.toJson() on a raw collection: " + json);
		JsonParser parser = new JsonParser();
		JsonArray array = parser.parse(json).getAsJsonArray();	//使用DOM解析器
		String message = gson.fromJson(array.get(0), String.class);
		int number = gson.fromJson(array.get(1), int.class);
		Event event = gson.fromJson(array.get(2), Event.class);
		System.out.printf("Using Gson.fromJson() to get: %s, %d, %s", message, number, event);
	  }
}
```
（2）、在使用GsonBuilder创建Gson时为Collection.class注册一个类型适配器(TypeAdapter)，这个适配器用于将数组的每个成员映射到合适的对象中，这种方法的缺点是，使用该Gson对象解析其他Collection对象也按照该适配器指定的方式进行反序列化。注册方法如下：（其中TypeAdapter需要自己实现）

①使用GsonBuilder创建Gson对象时设置
```Gson gson = new GsonBuilder().registerTypeAdapter(Collection.class,TypeAdapter).builde();```

②也可以使用@JsonAdapter注解对class设置相应的类型适配器
```java
//使用JsonAdapter注解自定义序列化
@JsonAdapter(UserJsonAdapter.class)
public class User {
	public final String firstName, lastName;
	private User(String firstName, String lastName) {
		this.firstName = firstName;
		this.lastName = lastName;
	}
}
public class UserJsonAdapter extends TypeAdapter<User>; {
	@Override public void write(JsonWriter out, User user) throws IOException {
		//实现序列化
		out.beginObject();
		out.name("name");
		out.value(user.firstName + " " + user.lastName);
		out.endObject();
	}
	@Override public User read(JsonReader in) throws IOException {
		// 实现反序列化
		in.beginObject();
		in.nextName();
		String[] nameParts = in.nextString().split(" ");
		in.endObject();
		return new User(nameParts[0], nameParts[1]);
	}
}
```
- 3、与第2中方法类似，只是这里不使用Collection.class注册，而使用其他自定义类来注册，比如MyCollectionMemberType.class。然后再使用fromJson() 与 Collection<MyCollectionMemberType>的TypeToken。这种方法只适用于JSON数组作为顶层元素出现，或者将出现Collection的字段改为Collection<MyCollectionMemberType>。

#### 3、自定义序列化与反序列化
有时候默认的表示方法不是不能满足你的需要，这经常出现在类似于Date，Point等特殊类型上。你可能需要将Data序列化成"YY-MM-DD"这种格式，或者将Point序列化成"(x,y)"这样的字符串，而不是将它们分为多个int存储。这个时候就需要在创建Gson对象时对这些特殊类型进行以下处理：
- 1、JSON序列化器(Json Serializers)：对该类型对象自定义一个序列化方法，用于将该对象的实例转换成JSON字符串。
- 2、JSON反序列化器(Json Deserializers)：为该类型自定义一个反序列化方法，用于从JSON字符串创建该类型的实例。
- 3、实例创建器(Instance Creators)：用于创建JSON对象，如果该对象有无参构造方法或者反序列化器，则实例创建器可以不需要。
```java
GsonBuilder gson = new GsonBuilder();
gson.registerTypeAdapter(MyType.class, new MyTypeAdapter());
gson.registerTypeAdapter(MyType.class, new MySerializer());
gson.registerTypeAdapter(MyType.class, new MyDeserializer());
gson.registerTypeAdapter(MyType.class, new MyInstanceCreator());
```
### 示例
#### 1、序列化器
下面是一个Date的自定义序列化器
```java
private class DateSerializer implements JsonSerializer<Date> {
	public JsonElement serialize(Date date, Type typeOfSrc, JsonSerializationContext context) {
		//将日期序列化成 YY-MM-DD的格式
		return new JsonPrimitive(date.getYear() + "-" + date.getMonth()+ "-" + date.getDay());
	}
}
```

当Gson序列化碰到Date对象时，它就会调用DateSerializer.serialize()方法
#### 2、反序列化器
```java
private class DateDeserializer implements JsonDeserializer<Date> {
	public Date deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context) throws JsonParseException {
		//Date对象会直接将YY-MM-DD格式的字符串解析成对象
		return new Date(json.getAsJsonPrimitive().getAsString());
	}
}
```
当Gson反序列化JSON字符串片段会调用DateDeserializer.deserialize()方法来实例化Date对象。

#### 3、实例创建器
当反序列化对象时，Gson需要创建类的实例，所以这个类最好是有一个无参构造方法(不论是public或private)
显然，实例创建器是为那些没有无参构造方法的类创建对象的。
```java
private class MoneyInstanceCreator implements InstanceCreator<Money> {
	public Money createInstance(Type type) {
		return new Money("1000000", CurrencyCode.USD);
	}
}
```
