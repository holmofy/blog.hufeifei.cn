---
title: 多态反序列化
date: 2022-04-23
categories: JAVA
keywords:
- Jackson
- Java
---

JSON序列化/反序列化的工具非常多，Google的[Gson](https://github.com/google/gson)、Alibaba号称世界最快的[FastJson](https://github.com/alibaba/fastjson)、实现了Java官方的[JSON Binding API(JSR 367)](https://javaee.github.io/jsonb-spec/)的[Eclipse Yasson](https://github.com/eclipse-ee4j/yasson)和[Apache Johnzon](https://github.com/apache/johnzon)。

这些库始终都不如Jackson好用。不仅仅因为Jackson具有极致的可扩展性，可以无痛对接文本格式[XML](https://github.com/FasterXML/jackson-dataformat-xml)、[csv, properties, yaml](https://github.com/FasterXML/jackson-dataformats-text)和二进制格式的[avro, cbor, ion, protobuf, smile](https://github.com/FasterXML/jackson-dataformats-binary)，还因为他有简单易用的多态反序列化功能。[Gson等库实现多态反序列化](https://ruediste.github.io/java/gson/2020/04/29/polymorphic-json-with-gson.html)做法非常的不优雅。

多态反序列化可以让应用的设计具有极致的可扩展性。这就像schemaless的MongoDB与schemaful的MySQL的区别。

在MySQL中存储一个多态的JSON字段即可实现schemaless，可以极大的简化复杂应用的开发。

## 用法

```java
// @JsonTypeInfo声明类型信息
// use表示用什么方式识别类型信息：
//    * JsonTypeInfo.Id.NAME：用子类的name属性
//    * JsonTypeInfo.Id.CLASS：用子类全类名
//    * JsonTypeInfo.Id.MINIMAL_CLASS：若基类和子类在同一包类，使用类名(忽略包名)作为识别码
//    * JsonTypeInfo.Id.CUSTOM：自定义识别码
// property(可选):制定识别码的属性名称
//    此属性只有当use为
//        JsonTypeInfo.Id.CLASS（若不指定property则默认为@class）
//        JsonTypeInfo.Id.MINIMAL_CLASS(若不指定property则默认为@c)
//        JsonTypeInfo.Id.NAME(若不指定property默认为@type)
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.EXISTING_PROPERTY, property = "type", visible = true)
// @JsonSubTypes声明子类，name用来标记类型名，当use=JsonTypeInfo.Id.NAME时生效。
@JsonSubTypes({
    @JsonSubTypes.Type(value = Vehicle.ElectricVehicle.class, name = "ELECTRIC_VEHICLE"),
    @JsonSubTypes.Type(value = Vehicle.FuelVehicle.class, name = "FUEL_VEHICLE")
})
public class Vehicle {

    public String type;

    // standard setters and getters

    public static class ElectricVehicle extends Vehicle {

        String autonomy;
        String chargingTime;

        // standard setters and getters
    }

    public static class FuelVehicle extends Vehicle {

        String fuelType;
        String transmissionType;

        // standard setters and getters
    }
}
```

测试例子：

```
@Test
public void whenDeserializingPolymorphic_thenCorrect() throws JsonProcessingException {
    String json = "{\"type\":\"ELECTRIC_VEHICLE\",\"autonomy\":\"500\",\"chargingTime\":\"200\"}";

    Vehicle vehicle = new ObjectMapper().readerFor(Vehicle.class).readValue(json);

    assertEquals(Vehicle.ElectricVehicle.class, vehicle.getClass());
}
```

> 参考文档：
> Jackso官方文档：https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization
> Jackson常用注解：https://www.baeldung.com/jackson-annotations#jackson-polymorphic-type-handling-annotations
