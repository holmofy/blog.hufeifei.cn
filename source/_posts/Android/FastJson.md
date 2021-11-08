---
title: 安卓常用第三方框架-FastJson
date: 2017-02-17
tags:
    - Android
    - FastJson
categories: Android
description: 安卓常用第三方框架-FastJson
---
## 简介
上次我们讲到Google的Gson库，作为国际大公司的阿里巴巴也不敢示弱，出了一款号称速度最快的Fastjson，这里有第三方给出的测试结果https://github.com/eishay/jvm-serializers/wiki， 虽然FastJson在Github上戏称Gson的“G”是“龟速”的意思，但FastJson在文档方面确实做得不如Gson(没办法天朝软件行业的通病)。废话不多说下面先给地址。

### 下载地址
Maven：http://central.maven.org/maven2/com/alibaba/fastjson/
Sourceforge : https://sourceforge.net/projects/fastjson/files/
Github：
 标准版https://github.com/alibaba/fastjson
 安卓版https://github.com/alibaba/fastjson/tree/android
### Gradle依赖
Fastjson提供了两种版本：标准版本，Android版本，所以添加Gradle依赖也有所不同
关于两个版本的区别可以查看阿里巴巴写的[文档](https://github.com/alibaba/fastjson/wiki/Android%E7%89%88%E6%9C%AC)
```groovy
##标准版
compile 'com.alibaba:fastjson:1.2.24'
##android版
compile 'com.alibaba:fastjson:1.1.56.android'
```
## FastJson使用
与Gson的fromJson,toJson类似FastJson也有如下方法
```java
package com.alibaba.fastjson;
public abstract class JSON {
      public static final String toJSONString(Object object);
      public static final <T> T parseObject(String text, Class<T> clazz, Feature... features);
}
```
所以序列化的时候也是直接将对象作为toJsonString的参数
### 序列化
```java
String jsonString = JSON.toJSONString(obj);
```
### 反序列化
```java
Type obj = JSON.parseObject("...", Type.class);
```
### 泛型序列化
```java
import com.alibaba.fastjson.TypeReference;

List<Type> list = JSON.parseObject("...", new TypeReference<List<Type>>() {});
```
### 定制序列化的key
使用@JSONField注解能够定制JSON字符串生成的key，不仅如此还可以设置其中的serialize/deserialize来定制该字段是否序列化/反序列化
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER })
public @interface JSONField {
    int ordinal() default 0;//配置序列化的顺序，1.1.42版本之后才支持

    String name() default "";//配置序列化的key

    String format() default "";//配置Date日期格式

    boolean serialize() default true;//能否序列化

    boolean deserialize() default true;//能否反序列化

    SerializerFeature[] serialzeFeatures() default {};//设置序列化配置SerializerFeature是枚举类

    Feature[] parseFeatures() default {};//设置反序列化配置Feature是枚举类
}
```
可以说@JSONField 的一个注解融合了Gson的@Expose和@SerializedName两个注解的功能，

## 示例
JavaBean
```java
public class Group {
    private int id;
    private String name;
    private List<UsersBean> users;

    // 省略getter/setter

    public static class UsersBean {
        private int id;
        private String name;

        // 省略getter/setter
    }
}

//序列化
Group group = new Group();
group.setId(0L);
group.setName("admin");

User guestUser = new User();
guestUser.setId(2L);
guestUser.setName("guest");

User rootUser = new User();
rootUser.setId(3L);
rootUser.setName("root");

group.addUser(guestUser);
group.addUser(rootUser);

String jsonString = JSON.toJSONString(group);

System.out.println(jsonString);

//输出结果
//{"id":0,"name":"admin","users":[{"id":2,"name":"guest"},{"id":3,"name":"root"}]}

//反序列化
Group group2 = JSON.parseObject(jsonString, Group.class);
// ==>group2与group值相同
```

