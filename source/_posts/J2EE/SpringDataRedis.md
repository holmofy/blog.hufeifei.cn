---
title: SpringDataRedis踩坑记录
date: 2019-2-21 1:12
categories: J2EE
---

这几天做的功能涉及到Redis缓存，踩了不少坑，这里记录下来。

# 1、SpringBoot自动配置的RedisTemplate

在SpringBoot中可以在`properties`配置文件中配置[`spring.redis.*`相关属性](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/api/org/springframework/boot/autoconfigure/data/redis/RedisProperties.html)，SpringBoot就会自动帮你创建相关Redis连接以及RedisTemplate相关对象。

```java
@Configuration
@ConditionalOnClass({ JedisConnection.class, RedisOperations.class, Jedis.class })
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {
    
    // Redis连接的自动配置
  @Configuration
  @ConditionalOnClass(GenericObjectPool.class)
  protected static class RedisConnectionConfiguration { ... }
    
    
  /**
   * RedisTemplate相关配置，SpringBoot会为我们生成两个RedisTemplate
   */
  @Configuration
  protected static class RedisConfiguration {

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<Object, Object> redisTemplate(
        RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
      RedisTemplate<Object, Object> template = new RedisTemplate<Object, Object>();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
    }

    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
        RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
      StringRedisTemplate template = new StringRedisTemplate();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
    }
  }
}
```

大多数情况下使用SpringBoot默认的配置即可。

# 2、StringRedisTemplate与RedisTemplate

SpringBoot默认为我们配置了两个RedisTemplate，其中`StringRedisTemplate`继承自`RedisTemplate<String,String>`。

```java
public class StringRedisTemplate extends RedisTemplate<String, String> {
  public StringRedisTemplate() {
        // StringRedisTemplate默认使用StringRedisSerializer进行序列化
    RedisSerializer<String> stringSerializer = new StringRedisSerializer();
    setKeySerializer(stringSerializer);
    setValueSerializer(stringSerializer);
    setHashKeySerializer(stringSerializer);
    setHashValueSerializer(stringSerializer);
  }
  public StringRedisTemplate(RedisConnectionFactory connectionFactory) {
    this();
    setConnectionFactory(connectionFactory);
    afterPropertiesSet();
  }
  protected RedisConnection preProcessConnection(RedisConnection connection, boolean existingConnection) {
    return new DefaultStringRedisConnection(connection);
  }
}
```



**两者的区别：**

1、StringRedisTemplate使用`StringRedisSerializer`进行序列化，而RedisTemplate默认使用`JdkSerializationRedisSerializer`进行序列化。

2、StringRedisTemplate对`RedisConnection`进行了一层包装。主要是因为RedisConnection的所有操作都是基于字节数组的，`DefaultStringRedisConnection`会把所有的结果转成String，包装了StringRedisSerializer并对批量操作数据进行批量序列化和反序列化，具体可以参考SetConverter，ListConverter，MapConverter的实现。



Spring Data Redis为了适配各种Redis客户端实现，抽象了一个RedisConnection接口。

![RedisConnection](http://www.plantuml.com/plantuml/svg/ZP0n3i8m34Ltdo8NI4_0KCGMCS49WckaIAcBR6VZy2m8a6cZZjz-NxRUg9R5sbm12Xl9FIE52qr5JmipePM50V9DJJ9Qm9fLm_4TFUToE3o7OHE6xyAtOWp9qLtuJ6ODQIz-bSTUD8d_sbkgJOsaAo76BHP-vrvSMkzqAXyH_uT6ugdDzGK0)

事实上如果直接使用Jedis客户端，其实更方便，Jedis已经对String类型做了编解码处理。

![Jedis](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuUBAIKqhKIZ9LoZAJCyeKKZ9B4fDBidCp-DooinBBAfqpibCpIjHiAdHrLM0iBbWGa0HY1glr9JCOYuaDaGvH9ZB8JKl1MWH0000)

```java
package redis.clients.jedis;
public class Client extends BinaryClient implements Commands {
  ...
  public void hset(final String key, final String field, final String value) {
    hset(SafeEncoder.encode(key), SafeEncoder.encode(field), SafeEncoder.encode(value));
  }

  public void hget(final String key, final String field) {
    hget(SafeEncoder.encode(key), SafeEncoder.encode(field));
  }
  ...
}
//
package redis.clients.util;
public final class SafeEncoder {
  private SafeEncoder(){
    throw new InstantiationError( "Must not instantiate this class" );
  }

  public static byte[][] encodeMany(final String... strs) {
    byte[][] many = new byte[strs.length][];
    for (int i = 0; i < strs.length; i++) {
      many[i] = encode(strs[i]);
    }
    return many;
  }

  public static byte[] encode(final String str) {
    try {
      if (str == null) {
        throw new JedisDataException("value sent to redis cannot be null");
      }
      return str.getBytes(Protocol.CHARSET);
    } catch (UnsupportedEncodingException e) {
      throw new JedisException(e);
    }
  }

  public static String encode(final byte[] data) {
    try {
      return new String(data, Protocol.CHARSET);
    } catch (UnsupportedEncodingException e) {
      throw new JedisException(e);
    }
  }
}
```

# 3、使用RedisTemplate

使用RedisTemplate很简单，因为SpringBoot已经为我们创建了RedisTemplate和StringRedisTemplate，所以我们直接在需要使用的Bean里面注入就行：

```java
@Component
public class Example {

    // 因为StringRedisTemplate继承自RedisTemplate<String, String>
    // 那么问题来了：
    // 这个地方注入的是StringRedisTemplate还是普通的RedisTemplate呢
    @Autowired
    private RedisTemplate<String, String> template;

    @PostConstruct
    public void init() {
        // 答案是：StringRedisTemplate
        System.out.println(template.getClass());
    }
    
    // 只有明确声明RedisTemplate<Object, Object>才会注入普通的RedisTemplate
    @Autowired
    private RedisTemplate<Object, Object> redisTemplate;

}
```

# 4、使用RedisTemplate的操作视图

RedisTemplate按照Redis的命令分组为我们提供了相应的操作视图：

![RedisTemplate操作视图](http://tva1.sinaimg.cn/large/bda5cd74ly1g0c5ifo75aj218a17ugt9.jpg)

```java
@Component
public class Example {

    @Autowired
    private StringRedisTemplate template;

    public void doSomething() {
        template.opsForList().leftPush("my-list", "value");
        template.opsForSet().add("my-set", "member1", "member2");
        ...
        BoundHashOperations<String, Object, Object> hashOps = template.boundHashOps("my-hash");
        hashOps.put("name", "holmofy");
        hashOps.put("age", "23");
        hashOps.put("gender", "male");
    }

}
```

这种随用随调方式的弊端是每次调用opsForXxx()都会创建一个新的视图。

[SpringDataRedis可以直接注入视图](https://docs.spring.io/spring-data/redis/docs/2.1.5.RELEASE/reference/html/#redis:template)：

```java
@Component
public class Example {
  
  // 只能用jsr250的@Resource注解注入
  @Resource(name="redisTemplate")
  private ListOperations<String, String> listOps;

  public void addLink(String userId, URL url) {
    listOps.leftPush(userId, url.toExternalForm());
  }
}
```

> 这个功能得益于PropertyEditorSupport，具体可参考[该链接](https://stackoverflow.com/questions/43006197/why-a-redistemplate-can-convert-to-a-listoperations)

# 5、RedisSerilizer

因为StringRedisTemplate和RedisTemplate默认使用的序列化不一样，所以在使用视图操作时要注意一些序列化方面的细节：

```java
@Component
public class Example {

    @Resource(name = "redisTemplate")
    private ValueOperations<String, Object> jdkSerializerValueOps;

    @Resource(name = "stringRedisTemplate")
    private ValueOperations<String, String> stringSerializerValueOps;

    @PostConstruct
    public void doSomething() {
        jdkSerializerValueOps.set("jdkNumber", 1);
        jdkSerializerValueOps.set("jdkString", "1");
        stringSerializerValueOps.set("string", "1");

        try {
            jdkSerializerValueOps.increment("jdkNumber"); //失败
        } catch (Exception ignore) { }
        try {
            jdkSerializerValueOps.increment("jdkString"); //失败
        } catch (Exception ignore) { }
        try {
            stringSerializerValueOps.increment("string"); //成功
        } catch (Exception ignore) { }
    }
}
```

![Redis](http://tva1.sinaimg.cn/large/bda5cd74ly1g0dbwd5xfqj21em0e8n6f.jpg)

经过不同的序列化器保存到Redis中的内容是不一样的，StringRedisTemplate直接转成字符串保存到Redis里面，但RedisTemplate默认使用JdkSerializer会将对象信息存储到Redis中。

**JdkSerializer优缺点**

优点：序列化存储了类型信息，所以反序列化能直接生成相应对象。

缺点：

1、Redis中存储的内容包括对象头信息，存储了过多的无用内容，浪费Redis内存。

2、Redis中的一些操作不能使用，比如自增自减。

**StringRedisSerializer优缺点**

优点：

1、使用方便，所有的操作都以字符串形式保存到Redis

2、占用Redis更小

缺点：所有操作只能以字符串形式执行。StringRedisTemplate的key，value等参数都必须是String类型，因为StringRedisSerializer只负责把String转换成byte[]。存储对象时，需要我们手动序列化成字符串；相应地，取对象需要反序列化。

# 6、其他序列化

目前最新的SpringDataRedis 2.1.5版默认提供了6种序列化方案。

![RedisSerializer](http://www.plantuml.com/plantuml/svg/ZP0ngiCm341tdK8doF3dFsGhP2aqlO2mbLJ4aK5MePGUlePIAAPGBneUUkYXiJYPN_S4a7Xnz8mcwyKnYd5moGeWwcmB1SOJHoapcr2IEnj00_3_CGnuOAqaJ1IsalLlggCLltgzGdled6StqVLds6kjhoLkRqGkdJt7s_xPCBB6-jad)

**GenericJackson2JsonRedisSerializer**

底层使用Jackson进行序列化并存入Redis。对于普通类型(如数值类型，字符串)可以正常反序列化回相应对象。

但如果存入对象时由于没有存入类信息，则无法反序列化。

不过GenericJackson2JsonRedisSerializer默认为我们开启了Jackson的类型信息的存储：

```java
public GenericJackson2JsonRedisSerializer(String classPropertyTypeName) {

    this(new ObjectMapper());

    // 使用Jackson的类型功能嵌入反序列化所需的类型信息
    // the type hint embedded for deserialization using the default typing feature.
    mapper.registerModule(new SimpleModule().addSerializer(new NullValueSerializer(classPropertyTypeName)));

    if (StringUtils.hasText(classPropertyTypeName)) {
        mapper.enableDefaultTypingAsProperty(DefaultTyping.NON_FINAL, classPropertyTypeName);
    } else {
        mapper.enableDefaultTyping(DefaultTyping.NON_FINAL, As.PROPERTY);
    }
}
```

所以当我存入一个对象时，它会把对象的类型信息也序列化存入Redis：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
private class Person {

    private String name;
    private int age;

}

//{\"@class\":\"com.example.demo.Person\",\"name\":\"Tom\",\"age\":10}
jacksonSerializerValueOps.set("jsonObject", new Person("Tom", 10));
Object obj = jacksonSerializerValueOps.get("jsonObject");
System.out.println(obj.getClass()); // com.example.demo.Person
```

具体可以参考[Jackson相关文档](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization)

**Jackson2JsonRedisSerializer与GenericToStringSerializer**

这两种序列化器是针对特定对象类型，前者用的是Jackson，后者用Spring的ConversionService。

# 7、JedisConnection的选择db问题

SpringBoot使用Jedis作为Redis的默认Client，可是`1.8.11.RELEASE`之前的版本中，发现如果redis的database不是0的话，JedisConnection每次创建的时候执行`select n`，并在关闭的时候执行重置`select 0`。

下面是测试代码：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = RedisConnectionTest.Config.class)
public class RedisConnectionTest {

    @Configuration
    public static class Config {
        @Bean
        public RedisConnectionFactory db0ConnectionFactory() {
            JedisConnectionFactory connectionFactory = new JedisConnectionFactory();
            connectionFactory.setHostName("localhost");
            connectionFactory.setPort(6379);
            connectionFactory.setDatabase(0);
            return connectionFactory;
        }

        @Bean
        public StringRedisTemplate redis0Template(RedisConnectionFactory db0ConnectionFactory) {
            return new StringRedisTemplate(db0ConnectionFactory);
        }

        @Bean
        public RedisConnectionFactory db1ConnectionFactory() {
            JedisConnectionFactory connectionFactory = new JedisConnectionFactory();
            connectionFactory.setHostName("localhost");
            connectionFactory.setPort(6379);
            connectionFactory.setDatabase(1);
            return connectionFactory;
        }

        @Bean
        public StringRedisTemplate redis1Template(RedisConnectionFactory db1ConnectionFactory) {
            return new StringRedisTemplate(db1ConnectionFactory);
        }

    }

    @Autowired
    private StringRedisTemplate redis0Template;

    @Autowired
    private StringRedisTemplate redis1Template;

    @Test
    public void test() {
        redis0Template.opsForValue().set("0", "0");

        redis1Template.opsForValue().set("1", "1");
    }

}
```

以下是执行过程中，redis monitor监控到的redis请求。

![image.png](http://tva1.sinaimg.cn/large/bda5cd74ly1g7an1irry0j20re08caf4.jpg)

社区已经向[Spring官方](https://jira.spring.io/browse/DATAREDIS-714)提出了这个bug，并在[新版本](https://github.com/spring-projects/spring-data-redis/pull/318)中解决。

