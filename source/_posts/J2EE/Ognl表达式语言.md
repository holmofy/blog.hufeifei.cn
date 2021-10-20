---
title: Ognl表达式语言
date: 2017-08-5
categories: J2EE
tags: 
- OGNL
---

Ognl全称Object Graph Navigation Language，[是Apache Commons下的一个子项目](http://commons.apache.org/proper/commons-ognl/)。和JSP中的EL表达式一样，通常作为View层访问数据的一种方式。但是OGNL的功能比EL表达式功能强大的多(看完这篇文章后你会觉得OGNL能把EL表达式秒成渣)

# jar包下载

使用ognl最有名的项目就是Struts2和MyBatis了，关于OGNL在Struts2中的使用以及原理，后面会有一篇文章详细说明。

这里的例子不会依赖Struts2的运行环境。

**官网下载地址：**

http://commons.apache.org/proper/commons-ognl/download_ognl.cgi

**Maven依赖：**

```xml
<!-- https://mvnrepository.com/artifact/ognl/ognl -->
<dependency>
    <groupId>ognl</groupId>
    <artifactId>ognl</artifactId>
    <version>3.2.3</version>
</dependency>
```

# 基本语法

最基本的语法很简单：通过属性(property)来访问Bean的数据，所以OGNL要求类对象具有getter/setter方法(也就是符合标准JavaBean的规则)。

下面的例子中将会使用User类作为测试的JavaBean

```java
public class User {
	private String username;
	private String password;

	public User(String username, String password) {
		this.username = username;
		this.password = password;
	}

	public User() { }

	// 省略getter/setter

	// 省略toString
}
```

下面的例子中会经常用到Ognl类的两个方法(暂时不了解这两个方法有什么作用不要紧，后面用到的时候自然就明白)：

```java
/**
 * @param expression 将要解析的OGNL表达式
 * @param root OGNL表达式的root元素对象
 */
public static Object getValue(String expression, Object root)

/**
 * @param expression 将要解析的OGNL表达式
 * @param context 用于取值的命名上下文
 * @param root OGNL表达式的root元素对象
 */
public static Object getValue(String expression, Map context, Object root)
```

## root对象

使用ognl表达式访问root元素对象中的属性不需要任何前缀。

**root对象属性访问：**

```java
@Test
public void test() throws OgnlException {
    User user = new User("apache", "123456");

    Object value = Ognl.getValue("username", user); // user对象作为root元素
    System.out.println(value); // apache

    // 注意这里的username需要加引号
    Object value = Ognl.getValue("['username']", user); // user对象作为root元素
    System.out.println(value); // apache
}
```

> 对象的访问方式和Map的一样

**对象方法调用：**

```java
@Test
public void test() throws OgnlException {
    User user = new User("apache", "123456");

    // 调用root元素的toString方法
    Object value = Ognl.getValue("toString()", user);
    System.out.println(value); //User [username=apache, password=123456]
}
```

**数组索引：**

```java
@Test
public void test() throws OgnlException {
    User u1 = new User("apache", "123456");
    User u2 = new User("git", "234");
    User u3 = new User("google", "567");
    User[] users = new User[] { u1, u2, u3 };
    // 访问root元素下的索引为1的元素
    Object value = Ognl.getValue("[1].toString()", users);
    System.out.println(value); // User [username=git, password=234]
}
```

## context上下文

使用OGNL表达式访问context中的元素需要添加`#`符号作为前缀，而且context需要是Map接口的实现类。同时要求添加一个root对象，此时root对象会以`"root"`字符串做为Key添加到context中。

```java
@Test
public void test() throws OgnlException {
    User u1 = new User("apache", "123456");
    User u2 = new User("git", "234");
    User u3 = new User("google", "567");
    User[] users = new User[] { u1, u2, u3 };

    Map<String, Object> context = new HashMap<>();
    context.put("names", new String[] { "ali", "huawei", "baidu" });

    // 使用#前缀访问context下的元素
    Object value = Ognl.getValue("#names[1]", context, users); // users作为root元素
    System.out.println(value); //huawei

  	// 使用#root访问root元素
    Object v2 = Ognl.getValue("#root[0]", context, users);
    System.out.println(v2); // User [username=apache, password=123456]
}
```

> 事实上使用`Object value = Ognl.getValue("username", user);`方式，不指定context，它会默认帮你创建一个OgnlContext，OgnlContext也实现了Map接口。而使用`Object value = Ognl.getValue("#names[1]", context, users);`方式，指定的map类型的context，最终也会被转换成OgnlContext。

通过查看OgnlContext源码，会发现有几个关键点：

```java
public class OgnlContext extends Object implements Map
{
	// root元素的key
    public static final String ROOT_CONTEXT_KEY = "root";
    // this会在后面提到
    public static final String THIS_CONTEXT_KEY = "this";
    ...

    private static Map RESERVED_KEYS = new HashMap(11);

    private Object _root; // root元素
    private Object _currentObject; // this元素
	...

    private final Map _values; // context内实际保存对象引用的Map

	static {
        String s;

        RESERVED_KEYS.put(ROOT_CONTEXT_KEY, null);
        RESERVED_KEYS.put(THIS_CONTEXT_KEY, null);
        ...
    }

    public Object get(Object key)
    {
        Object result;

        if (RESERVED_KEYS.containsKey(key)) {
            // this元素
            if (key.equals(OgnlContext.THIS_CONTEXT_KEY)) {
                  result = getCurrentObject(); // currentObject
            } else if (key.equals(OgnlContext.ROOT_CONTEXT_KEY)) {
                  // root元素
                  result = getRoot();
            } else if (key.equals(OgnlContext.TRACE_EVALUATIONS_CONTEXT_KEY)) {
                  ...
                  ...
            } else {
            	throw new IllegalArgumentException("unknown reserved key '" + key + "'");
            }
        } else {
            // 从map中取值
            result = _values.get(key);
        }
        return result;
    }
    ...
}
```

# OGNL中的常量

**1.**字符串常量，需要用引号引起来(可以是单引号也可以是双引号，单引号可以作为内嵌脚本使用)。

**2.**字符常量，需要用单引号引起来

**3.**数值常量，支持Java的int，long，float，double。同时可以使用"b" 或 "B"作为后缀表示BigDecimal类型的数据，使用"h" 或 "H"作为后缀表示BigInteger类型的数据(这里的h表示huge，这个不会干扰16进制的数字)

**4.**布尔类型，true和false

**5.**空指针，null

# 集合元素

**访问List元素**

```java
@Test
public void testOgnl() throws OgnlException {
    User u1 = new User("apache", "123456");
    User u2 = new User("git", "234");
    User u3 = new User("google", "567");

    List<User> users = new ArrayList<>();
    users.add(u1);
    users.add(u2);
    users.add(u3);

    Map<String, Object> ctx = new HashMap<>();
    ctx.put("users", users); // 同时将users以"users"为key添加到context中

    // 使用#前缀访问context下的属性
    Object v1 = Ognl.getValue("#users[1].password", ctx, users); // 以user作为root元素
    System.out.println(v1); // 234

    // 访问root元素下的属性(List也可以使用[]索引方式访问)
    Object v2 = Ognl.getValue("[1].password", ctx, users);
    System.out.println(v2); //234

    // 也可以调用List.get方法访问
    Object v3 = Ognl.getValue("get(1).password", ctx, users);
    System.out.println(v3); // 234

    // 因为root元素会以“root”为key添加到context中
    Object v4 = Ognl.getValue("#root[1].password", users);
    System.out.println(v4); // 234
}
```

**访问Map元素**

```java
@Test
public void test() throws OgnlException {
    Object root = new Object();
    Map<String, Object> ctx = new HashMap<>();

    Map<String, Object> users = new HashMap<>();
    users.put("apache", "123456");
    users.put("git", "234");
    users.put("google", "567");

    ctx.put("users", users);

    // 使用[key]进行访问
    Object v1 = Ognl.getValue("#users['apache']", ctx, root);
    System.out.println(v1);

  	// 使用.运算符进行访问
    Object v2 = Ognl.getValue("#users.apache", ctx, root);
    System.out.println(v2);
}
```

Map和对象的访问方式一样：

```java
@Test
public void test() throws OgnlException {
    Object root = new Object();
    Map<String, Object> ctx = new HashMap<>();

    User user = new User("apache", "123456");

    ctx.put("user", user);

    Object v1 = Ognl.getValue("#user['username']", ctx, root);
    System.out.println(v1);

    Object v2 = Ognl.getValue("#user.username", ctx, root);
    System.out.println(v2);
}
```

# 集合的伪属性

```java
@Test
public void test() throws OgnlException {
    Object root = new Object();
    Map<String, Object> ctx = new HashMap<>();
    ctx.put("usernames", Arrays.asList("ali", "huawei", "baidu"));

    // 调用size方法获取集合元素个数
    Object v1 = Ognl.getValue("#usernames.size()", ctx, root);
    System.out.println(v1);

  	// 使用size伪属性获取集合元素个数
    Object v2 = Ognl.getValue("#usernames.size", ctx, root);
    System.out.println(v2);
}
```

OGNL中为集合相关的类提供了以下的伪属性：

| 集合                   | 特殊属性                                     |
| -------------------- | ---------------------------------------- |
| `Map`, `List`  `Set` | `size`: 集合中元素的个数<br>`isEmpty`: 集合是否为空    |
| List                 | `iterator`: 获取`list`的迭代器                 |
| Map                  | `keys`: 等价于keys()方法<br>`values`: 等价于values()方法。<br>**注意：** 这两个属性以及 `size` and `isEmpty`, 和`map['size']`这种访问方式不同，`map['size']`会访问key为`'size'`的元素。 |
| Set                  | `iterator`: 获取Set的迭代器                    |
| Iterator             | `next`: 获取迭代器的下一个元素<br>`hasNext`: 迭代器是否有可用元素。 |
| Enumeration          | `next`: 获取迭代器的下一个元素<br>`hasNext`迭代器是否有可用元素。 |

# 构造集合

**构造原生数组**

```java
@Test
public void test() throws OgnlException {
	// 使用OGNL表达式构造list
	Object v = Ognl.getValue("new int[] { 1, 2, 3 }", null);
	System.out.println(v.getClass()); // class [I
	System.out.println(Arrays.toString((int[]) v)); // [1, 2, 3]
}
```

**构造List元素**

```java
@Test
public void test() throws OgnlException {
      // 使用OGNL表达式构造list
      Object v = Ognl.getValue("new int[] { 1, 2, 3 }", null);
      System.out.println(v.getClass()); // java.util.ArrayList
      System.out.println(v); // [abc, def, hff, git]
}
```

**构造Map集合**

```java
@Test
public void test() throws OgnlException {
      // 构造map
      Object v = Ognl.getValue("#{'foo':'ffff', 'bar':'barvalue'}", null);
      System.out.println(v.getClass()); // class java.util.LinkedHashMap
      System.out.println(v); // {foo=ffff, bar=barvalue}
}
```

**使用指定的类构造Map**

```java
@Test
public void test() throws OgnlException {
      // 指定使用HashMap构造map
      Object v = Ognl.getValue("#@java.util.HashMap@{'foo':'ffff', 'bar':'barvalue'}", null);
      System.out.println(v.getClass());
      System.out.println(v);
}
```

**构造复杂map**

```java
@Test
public void test() throws OgnlException {
    // 构造复杂map
    Object v = Ognl.getValue("#{'foo': {1, 2, 3, 4}, 'bar':'barvalue'}", null);
    System.out.println(v); // {foo=[1, 2, 3, 4], bar=barvalue}
}
```

**使用变量构造map、list或数组**

```java
@Test
public void test() throws OgnlException {
	Map<String, Object> ctx = new HashMap<>();

	ctx.put("usernames", Arrays.asList("ali", "git", "apache", "chrome"));

	Object v1 = Ognl.getValue("#{'u1':#usernames[0], 'u2':#usernames[2]}", ctx, new Object());
	System.out.println(v1); // {u1=ali, u2=apache}

	Object v2 = Ognl.getValue("{#usernames[1], #usernames[3]}", ctx, new Object());
	System.out.println(v2); // [git, chrome]
}
```

# [not] in操作

```java
@Test
public void test() throws OgnlException {
    Map<String, Object> ctx = new HashMap<>();

    ctx.put("usernames", Arrays.asList("ali", "git", "apache", "chrome"));

    Object v5 = Ognl.getValue("'git' in #usernames", ctx, new Object());
    System.out.println(v5); // true

    Object v6 = Ognl.getValue("'holmofy' not in #usernames", ctx, new Object());
    System.out.println(v6); // true
}
```

# ^、$和?操作符进行元素过滤

> 也有人把这些操作叫做投影(projection)

```java
@Test
public void test() throws OgnlException {

	User u1 = new User("root", "1");
	User u2 = new User("git", "2");
	User u3 = new User("apache", "3");
	User u4 = new User("chrome", "4");
  	User u5 = new User("root", "5");
	User u6 = new User("git", "6");
	User u7 = new User("apache", "7");
	User u8 = new User("chrome", "8");

	List<User> users = new ArrayList<>();
	users.add(u1);
	users.add(u2);
	users.add(u3);
	users.add(u4);
  	users.add(u5);
	users.add(u6);
	users.add(u7);
	users.add(u8);

	Map<String, Object> ctx = new HashMap<>();

	ctx.put("users", users);

	// 第一个元素
	Object v1 = Ognl.getValue("#users.{^ true}", ctx, new Object());
    System.out.println(v5.getClass());  // java.util.ArrayList
	System.out.println(v1);  // [User [username=root, password=1]]

	// 第一个username == 'git'的元素
	Object v2 = Ognl.getValue("#users.{^ #this.username=='git'}", ctx, new Object());
    System.out.println(v5.getClass());  // java.util.ArrayList
	System.out.println(v2);  // [User [username=git, password=2]]

	// 最后一个元素
	Object v3 = Ognl.getValue("#users.{$ true}", ctx, new Object());
    System.out.println(v5.getClass());  // java.util.ArrayList
	System.out.println(v3);  // [User [username=chrome, password=8]]

	// 从后往前第一个username == 'apache'的元素
	Object v4 = Ognl.getValue("#users.{$ #this.username=='apache'}", ctx, new Object());
    System.out.println(v5.getClass());  // java.util.ArrayList
	System.out.println(v4); // [User [username=apache, password=7]]

    // 使用?号进行条件过滤
    Object v5 = Ognl.getValue("#users.{? #this.username in {'git','apache'} }", ctx, new Object());
    System.out.println(v5.getClass());  // java.util.ArrayList
    System.out.println(v5); // [User [username=git, password=2], User [username=apache, password=3], User [username=git, password=6], User [username=apache, password=7]]
}
```

# 三目运算符

```java
@Test
public void test() throws OgnlException {
    Map<String, Object> ctx = new HashMap<>();

    ctx.put("usernames", Arrays.asList("root", "git", "apache", "chrome"));

    Object value = Ognl.getValue("#usernames.size.(#this>2?2:#this)", ctx, new Object());
    System.out.println(value); // 2
}
```

# 类的静态方法调用

```java
@Test
public void test() throws OgnlException {
    Map<String, Object> ctx = new HashMap<>();

    ctx.put("usernames", new String[] {"root", "git", "apache", "chrome"});

    Object value = Ognl.getValue("@java.util.Arrays@toString(#usernames)", ctx, new Object());
    System.out.println(value); // [root, git, apache, chrome]
}
```

# 多条语句的执行

```java
@Test
public void test() throws OgnlException {
    Map<String, Object> ctx = new HashMap<>();

    int[] arr = new int[] { 4, 5, 2, 6 };

    ctx.put("arr", arr);

    Object value = Ognl.getValue("@java.util.Arrays@sort(#arr),#arr", ctx, new Object());
    System.out.println(Arrays.toString((int[]) value)); // [2, 4, 5, 6]
    System.out.println(Arrays.toString(arr)); // [2, 4, 5, 6]
}
```

# Lambda表达式

```java
@Test
public void test() throws OgnlException {
    Object value = Ognl.getValue("#fib =:[#this==0 ? 0 : #this==1 ? 1 : #fib(#this-2)+#fib(#this-1)], #fib(8)",
    null);
    System.out.println(value);
}
```





# 附录：OGNL语法摘要表

| Operator                                 | `getValue()` Notes                       | `setValue()` Notes                       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| *e1*`,` *e2*<br>Sequence operator        | Both `e1` and `e2` are evaluated with the same source object, and the result of `e2`is returned. | `getValue` is called on `e1`, and then `setValue` is called on `e2`. |
| *e1* `=` *e2*<br>Assignment operator     | `getValue` is called on `e2`, and then `setValue` is called on `e1` with the result of `e2` as the target object. | Cannot be the top-level expression for `setValue`. |
| *e1* `?` *e2* `:` *e3*<br>Conditional operator | `getValue` is called on `e1` and the result is interpreted as a boolean. `getValue` is then called on either `e2` or `e3`, depending on whether the result of `e1` was `true` or `false` respectively, and the result is returned. | `getValue` is called on `e1`, and then `setValue` is called on either `e2` or `e3`. |
| *e1* `||` *e2*, e1 `or` *e2*<br>Logical `or`operator | `getValue` is called on `e1` and the result is interpreted as a boolean. If `true`, that result is returned; if `false`, `getValue` is called on `e2` and its value is returned. | `getValue` is called on `e1`; if `false`, `setValue` is called on `e2`. Note that `e1`being `true` prevents any further setting from taking place. |
| *e1* `&&` *e2*, *e1* `and` *e2*<br>Logical `and`operator | `getValue` is called on `e1` and the result is interpreted as a boolean. If `false`, that result is returned; if true, `getValue` is called on e2 and its value is returned. | `getValue` is called on `e1`; if `true`, `setValue` is called on `e2`. Note that `e1` being `false` prevents any further setting from taking place. |
| *e1* `|` *e2*, *e1* `bor` *e2*<br>Bitwise `or`operator | `e1` and `e2` are interpreted as integers and the result is an integer. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `^` *e2*, *e1* `xor` *e2*<br>Bitwise exclusive-or operator | `e1` and `e2` are interpreted as integers and the result is an integer. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `&` *e2*, *e1* `band` *e2*<br>Bitwise and operator | `e1` and `e2` are interpreted as integers and the result is an integer. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `==` *e2*, *e1* `eq` *e2*<br>Equality test<br><br>*e1* `!=` *e2*, *e1* `neq` *e2*Inequality test | Equality is tested for as follows. If either value is `null`, they are equal if and only if both are `null`. If they are the same object or the `equals()` method says they are equal, they are equal. If they are both `Number`s, they are equal if their values as double-precision floating point numbers are equal. Otherwise, they are not equal. These rules make numbers compare equal more readily than they would normally, if just using the equals method. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `<` *e2*, *e1* `lt` *e2*<br>Less than comparison<br><br>*e1* `<=` *e2*, *e1* `lte` *e2*<br>Less than or equals comparison<br><br>*e1* `>` *e2*, *e1* `gt` *e2*<br>Greater than comparison<br><br>*e1* `>=` *e2*, *e1* `gte` *e2<br>*Greater than or equals comparison<br><br>*e1* `in` *e2*<br>List membership comparison<br><br>*e1* `not in` *e2*<br>List non-membership comparison | The ordering operators compare with `compareTo()` if their arguments are non-numeric and implement `Comparable`; otherwise, the arguments are interpreted as numbers and compared numerically. The in operator is not from Java; it tests for inclusion of e1 in e2, where e2 is interpreted as a collection. This test is not efficient: it iterates the collection. However, it uses the standard OGNL equality test. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `<<` *e2*, *e1* `shl` *e2*<br>Bit shift left<br><br>*e1* `>>` *e2*, *e1* `shr` *e2*<br>Bit shift right<br><br>*e1* `>>>` *e2*, *e1* `ushr` *e2*<br>Logical shift right | `e1` and `e2` are interpreted as integers and the result is an integer. | Cannot be the top-level expression passed to `setValue`. |
| *e1* `+` *e2*<br>Addition<br><br>*e1* `-` *e2*<br>Subtraction | The plus operator concatenates strings if its arguments are non-numeric; otherwise it interprets its arguments as numbers and adds them. The minus operator always works on numbers. | Cannot be the top-level expression passed to `setValue`. |
| *e1*`*` *e2*<br>Multiplication<br><br>*e1* `/` *e2*<br>Division<br><br>*e1* `%` *e2*<br>Remainder | Multiplication, division, which interpret their arguments as numbers, and remainder, which interprets its arguments as integers. | Cannot be the top-level expression passed to `setValue`. |
| `+` *e*<br>Unary plus<br><br>`-` *e*<br>Unary minus<br><br>`!` *e*, `not` *e*<br>Logical not<br><br>`~` *e*<br>Bitwise not<br><br>*e* `instanceof` *class*Class <br>membership | Unary plus is a no-op, it simply returns the value of its argument. Unary minus interprets its argument as a number. Logical not interprets its argument as a boolean. Bitwise not interprets its argument as an integer. The *class* argument to instanceof is the fully qualified name of a Java class. | Cannot be the top-level expression passed to `setValue`. |
| *e*`.`*method*`(`*args*`)`<br>Method call<br><br>*e*`.`<br>*property*Property<br><br>*e1*`[` *e2* `]`<br>Index<br><br>*e1*`.{` *e2* `}`<br>Projection<br><br>*e1*`.{?` *e2* `}`<br>Selection<br><br>*e1*`.(`*e2*`)`<br>Subexpression evaluation<br><br>*e1*`(`*e2*`)`<br>Expression evaluation | Generally speaking, navigation chains are evaluated by evaluating the first expression, then evaluating the second one with the result of the first as the source object. | Some of these forms can be passed as top-level expressions to `setValue`and others cannot. Only those chains that end in property references (e.property), indexes (`e1[e2]`), and subexpressions (`e1.(e2)`) can be; and expression evaluations can be as well. For the chains, `getValue` is called on the left-hand expression (`e` or `e1`), and then `setValue` is called on the rest with the result as the target object. |
| *constant*Constant<br><br>`(` *e* `)`<br>Parenthesized expression<br><br>*method*`(`*args*`)`<br>Method call<br><br>*property*<br>Property reference<br><br>`[` *e* `]`<br>Index reference<br><br>`{` *e*`,` ... `}`<br>List creation<br><br>`#`*variable*<br>Context variable reference<br><br>`@`*class*`@`*method*`(`*args*`)`<br>Static method reference<br><br>`@`*class*`@`*field*<br>Static field reference<br><br>`new` *class*`(`*args*`)`<br>Constructor call<br><br>`new` *array-component-class*`[] {` *e*`,` ... `}`<br>Array creation<br><br>`#{` *e1* `:` *e2*`,` ... `}`<br>Map creation<br><br>`#@`*classname*`@{` *e1* `:` *e2*`,`... `}`<br>Map creation with specific subclass<br><br>`:[` *e* `]`Lambda expression definition | Basic expressions                        | Only property references (`property`), indexes (`[e]`), and variable references (`#variable`) can be passed as top-level expressions to `setValue`. For indexes, `getValue` is called on `e`, and then the result is used as the property "name" (which might be a `String` or any other kind of object) to set in the current target object. Variable and property references are set more directly. |



> 参考文章：
>
> OGNL官方文档：http://commons.apache.org/proper/commons-ognl/language-guide.html
