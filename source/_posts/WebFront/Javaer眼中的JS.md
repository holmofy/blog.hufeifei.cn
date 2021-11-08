---
title: Javar眼中的JS
date: 2018-03-16
categories: 前端
mathjax: true
---

> 把之前写的笔记整理了一下，重新拾起JS

## 本数据类型

| JS                    | Java    |
| --------------------- | ------- |
| number                | double  |
| boolean               | boolean |
| string                | String  |
| null(Object类型的null引用) | null    |
| undefined(未定义类型)      | null    |
| Symbol(ES6标准新增类型)     | enum    |

js中除了Object意外所有类型都是不可变的(immutable)：

1、**JS中的number和Java中的double一样，都是IEEE754 64位双精度浮点型**。JS中所有的数值类型全部都是用双精度浮点型表示，不像其他编程语言拥有丰富的数值类型。和Java中的double一样，JS的number也有对应的包装类Number，所以使用Number构造函数new出来的对象是object类型的：

```javascript
typeof 1; // number

// Number在这里作为一个转换函数,
// 甚至可以将其他类型的值转成number类型
typeof Number(1); // number，类似于java中的Double.parseDouble

// Number在这里作为一个构造函数
typeof new Number(1); // object

// 通常我们应该避免创建包装类对象
// 除非你真的需要执行一些对象操作
// 比如：
var ob = new Number(1);
ob.relatedMessage = "this is number wrapper";
console.log(ob.relatedMessage); // 正常输出

var pb = Number(1);
pb.relatedMessage = "this is primary type number"
console.log(pb.relatedMessage); // undefined
```

和Java中的Double类一样，Number中也定义了64位浮点数的一些常量：

| Number常量                 | 其他语言中的等价物                                | 16进制表示                                   |
| ------------------------ | ---------------------------------------- | ---------------------------------------- |
| Number.NaN               | double NaN = 0.0d / 0.0;                 | 0x7FF8000000000000L<br/>`0x7ff0000000000001L`到   `0x7fffffffffffffffL`以及 `0xfff0000000000001L`到 `0xffffffffffffffffL`的值都是NaN：阶码全1，尾数存在一个或多个1。 |
| Number.POSITIVE_INFINITY | double POSITIVE_INFINITY = 1.0d / 0.0;<br/>任意正数除以0都为正无穷 | 0x7FF0000000000000L<br/>符号位为0阶码全1        |
| Number.NEGATIVE_INFINITY | double NEGATIVE_INFINITY = -1.0d / 0.0;<br/>任意负数除以0都为负无穷 | 0xFFF0000000000000L<br/>符号位为1阶码全1        |
| Number.MAX_VALUE         | double MAX_VALUE = 2.2250738585072014e-308;<br/>Java中的16进制写法：0x1.fffffffffffffp+1023<br/> (即$(2-2^{52})*2^{1023}$) | 0x7FEFFFFFFFFFFFFFL                      |
| Number.MIN_VALUE         | double MIN_VALUE = 4.9e-324;<br/>Java中的16进制科学计数法表示：0x1.0p-1022<br/> (即 $2^{-1022}$) | 0x0000000000000001L                      |
| Number.MAX_SAFE_INTEGER  | double MAX_SAFE_INTEGER = 9007199254740991;<br/>Java中的16进制科学计数法表示：0x1.fffffffffffffp+52<br/> (即$2^{53}-1$) | 0x433FFFFFFFFFFFFF                       |
| Number.MIN_SAFE_INTEGER  | double MAX_SAFE_INTEGER = -9007199254740991;<br/>Java中的16进制科学计数法表示：-0x1.fffffffffffffp+52<br/> (即$-(2^{53}-1)$) | 0xC33FFFFFFFFFFFFF                       |

> 如果对浮点算法不熟悉可参考[wikipedia](https://en.wikipedia.org/wiki/Floating-point_arithmetic)

2、 **boolean和Java中的boolean一样，只有两个取值：true和false**。 同样地，boolean也有对应的包装类Boolean

```javascript
typeof true;  // boolean
typeof Boolean(true); // boolean,类似于java中的Boolean.parseBoolean
typeof new Boolean(true); // object
```

3、**JS中的字符串和Java中的字符串一样都是不可变的**：所有对字符串的操作都会返回一个新字符串，原始字符串对象并没有改变。和其他C系列编程语言一样字符串索引都是从0开始。和前面两种基本数据类型一样string也有自己的包装类String。

4、Null和Undefined本质上都是指“空”。它们都只有一个取值：Null只有一个取值null，Undefined也只有一个取值undefined。

**但是在JS中null并不等同于undefined。**

[ES规范](https://www.ecma-international.org/ecma-262/6.0/#sec-terms-and-definitions-null-type)中定义：

* **null表示Object类型的引用为空(已确定是对象类型的引用)；**
* **undefined表示任意变量未赋值(尚未确定类型的变量)；**

从代码上比较主要有以下区别：

```javascript
typeof null        // object (因为一些历史原因而不是'null')
typeof undefined   // undefined
null === undefined // false
null  == undefined // true
null === null // true
null  == null // true
!null //true
isNaN(1 + null) // false
isNaN(1 + undefined) // true
```


## 象类型--[Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)

| JS     | Java |
| ------ | ---- |
| Object | Map  |

JS中的Object与Java中的Object不同，而与Java中的Map类型类似：

一个 Javascript 对象就是键和值之间的映射，键是字符串类型(也可以是Symbol类型)，值可以是任意类型。JS中的Object类型非常符合[哈希表](https://en.wikipedia.org/wiki/Hash_table)的数据结构。

```javascript
var o = new Object();
o.name = "C";  // 使用“.”运算符读写对象属性
o["age"] = 18; // 使用“[]”运算符读写对象属性
console.log(o);
```

输出结果：

```javascript
{name: "C", age: 18}
```

还有一种方式构造字面量对象：

```javascript
var p = {
    name: "C",
    age: 18
};

// 因为Key-Value对的值可以是任何类型，所以：
var p = {
    name: "C",
    age: 18,
    /* value可以是另一个对象 */
    father: {
        name: "B",
        age: 34
    },
    /* value可以是一个对象数组 */
    friends: [
        {name: "J", age: 19},
        {name: "G", age: 18}
    ]
};
```

> JSON数据格式就是从Javascript语言中出来的。

## 他内置对象

除了前面介绍的Object以及基本数据类型的包装类型JS还提供了一些其他的内置对象。

下面介绍的对象本质上都是基于JS中的Object，所以和Object一样都是键值对映射。

```javascript
typeof Math;      // object
typeof /^.*$/;    // object
typeof [6, 7, 8]; // object
```

## 1. [Math](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math)

和绝大多数语言一样，JS也提供了数学计算的API——Math对象，Math对象有常见的数学常量和函数。但是和下面介绍的其他内置对象不同，Math对象不是function对象，也就是说Math不是构造函数，所以你不能使用`new Math()`或`Math()`的方式来创建Math对象。JS中的Math对象可以看做java中的Math.class。

```javascript
Math {
  E: 2.718281828459045
  LN2: 0.6931471805599453
  LN10: 2.302585092994046
  LOG2E: 1.4426950408889634
  LOG10E: 0.4342944819032518
  PI: 3.141592653589793
  SQRT1_2: 0.7071067811865476
  SQRT2: 1.4142135623730951
  abs: ƒunction abs()
  acos: ƒunction acos()
  acosh: ƒunction acosh()
  asin: ƒunction asin()
  asinh: ƒunction asinh()
  atan: ƒunction atan()
  atan2: ƒunction atan2()
  atanh: ƒunction atanh()
  cbrt: ƒunction cbrt()
  ceil: ƒunction ceil()
  clz32: ƒunction clz32()
  cos: ƒunction cos()
  cosh: ƒunction cosh()
  exp: ƒunction exp()
  expm1: ƒunction expm1()
  floor: ƒunction floor()
  fround: ƒunction fround()
  hypot: ƒunction hypot()
  imul: ƒunction imul()
  log: ƒunction log()
  log1p: ƒunction log1p()
  log2: ƒunction log2()
  log10: ƒunction log10()
  max: ƒunction max()
  min: ƒunction min()
  pow: ƒunction pow()
  random: ƒunction random()
  round: ƒunction round()
  sign: ƒunction sign()
  sin: ƒunction sin()
  sinh: ƒunction sinh()
  sqrt: ƒunction sqrt()
  tan: ƒunction tan()
  tanh: ƒunction tanh()
  trunc: ƒunction trunc()
  Symbol(Symbol.toStringTag): "Math"
  __proto__: Object
}
```

## 2. [Date](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date)

和大多数编程语言一样，JS中也有基于计算机纪元(1970年1月1日)的日期类Date。

构造函数如下：

```javascript
new Date();
new Date(value);
new Date(dateString);
new Date(year, month[, date[, hours[, minutes[, seconds[, milliseconds]]]]]);
```

示例：

```javascript
var d = new Date();
console.log(d);
```

运行结果：

```javascript
VM546:1 Mon Sep 11 2017 19:42:48 GMT+0800 (中国标准时间)
```

## 3. [RegExp](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp)

尽管Java也有Pattern和Matcher两个类来提供正则表达式的支持，但相对于JS和Python这类脚本语言，Java的正则用起来相当麻烦。JS中提供的正则很强大，使用也相当便捷。

构造正则的方法：

```javascript
/pattern/flags
// JS中对正则表达式提供了字面上的支持，直接在两个“/”之间定义正则
// 正则字面量会编译成常量，所以在循环迭代中不会被重新编译。
// 示例：/ab+c/i;

new RegExp(pattern[, flags])
// 也可以使用构造方法创建，这种方式是运行时编译
// 适合动态生成正则表达式的情况。
// 例子：new RegExp('ab+c', 'i');
//      new RegExp(/ab+c/, 'i');
// 使用字符串构造正则，需要对特殊字符进行转义
// 如：new RegExp("\\w+"); // 等价于/\w+/

RegExp(pattern[, flags])
// 工厂函数
```

其中flags用于指定正则表达式匹配的一些选项：

| 标志位  | 对应的RegExp对象属性         | 解释                                       |
| ---- | --------------------- | ---------------------------------------- |
| `g`  | `regexObj.global`     | 全局匹配，找到所有匹配项，而不是在第一个匹配后停止                |
| `i`  | `regexObj.ignoreCase` | 忽略大小写                                    |
| `m`  | `regexObj.multiline`  | 匹配多行，将^和$作为多行文本的开始于结束。                   |
| `u`  | `regexObj.unicode`    | 将正则表达式Unicode码点                          |
| `y`  | `regexObj.sticky`     | 粘性匹配，只从lastIndex属性指定的位置开始匹配(并且不再尝试后续的匹配)。 |

RegExp主要有两个常用方法：

```javascript
// exec() 对指定字符串进行匹配
// 返回一个结果数组 或 null
regexObj.exec(str);
// test() 检查指定字符串是否符合该正则
// 返回 true 或 false。
regexObj.test(str)
```

示例：

```javascript
var regex = /\d+/;    // 一个或以上的数字
regex.test("abc");    // false
regex.test("abc123"); // true
regex.test("123");    // true

var regex = /^\d+$/;  // 开头到结尾只能是数字
regex.test("abc");    // false
regex.test("abc123"); // false
regex.test("123");    // true

var regex = /\d+/g; // 一个或以上的数字
var arrRe;
while(null !== (arrRe = regex.exec("12C19D88at221B99"))){
    console.log(arrRe[0] + " end index: " + (regex.lastIndex-1));
}
// 输出：
12 end index: 1
19 end index: 4
88 end index: 7
221 end index: 12
99 end index: 15
```

> 有关正则表达式的基础内容可以参考[这篇文章](http://blog.csdn.net/holmofy/article/details/74884785)

## 4. [Array数组](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Array)

Array数组是一种使用整数作为键同时具有长度属性(length)的常规对象，数组对象还继承了Array.prototype的一些函数用于操作数组对象。

示例：

```javascript
var arr = [6,7,8];
console.log(arr);  // 执行结果：[6, 7, 8]
arr[3] = "Hello";  // 可以在Array中存储任意类型的数据
arr["4"] = "JS";
console.log(arr);  // 执行结果：[6, 7, 8, "Hello", "JS"]
```

上面例子中的Array数组底层是以这种方式存储的：

```javascript
0: 6
1: 7
2: 8
3: "Hello"
4: "JS"
length: 5
__proto__: Array.prototype
```

所以如果出现：

```javascript
var arr = [6, 7, 8];
arr[5] = 11;      // arr[3]和arr[4]还未赋值
console.log(arr); // 运行结果：[6, 7, 8, undefined × 2, 11]
```

此时的arr数组底层存储情况如下：

```javascript
0: 6
1: 7
2: 8
5: 11
length: 6  // 注意数组长度为6
__proto__: Array.prototype
```

### push|pop 和 unshift|shift

Array.prototype中提供了很多函数用于操作，比如这两对：unshift|shift 和 push|pop。从这两对函数上看JS中的数组倒有点像双端队列

![Array](//img-blog.csdn.net/20180316231839713?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

push/pop示例：

```javascript
var arr = [6,7,8];
arr.push(9); // 在Array末尾添加数据。返回添加元素后Array的长度：4
arr.push(10,11); // 可以同时添加多个数据，同时返回添加元素后Array的长度：6
console.log(arr); // 输出：[6, 7, 8, 9, 10, 11]
arr.pop(); // 删除并返回最后一个元素：11
console.log(arr); // 输出：[6, 7, 8, 9, 10]
```

unshift/shift示例：

```javascript
var arr = Array.of(6,7,8); // ES6中定义的另一种创建数组的方式(注意兼容性)
arr.unshift(5); // 将之前的数据后移，然后在Array头部添加数据。并返回添加元素后Array的长度：4
arr.unshift(1,2,3,4); // 同时添加多个数据，同时返回添加元素后Array的长度：8
console.log(arr); // 输出： [1, 2, 3, 4, 5, 6, 7, 8]
arr.shift(); // 删除并返回最前面的一个元素：1
console.log(arr); // 输出： [2, 3, 4, 5, 6, 7, 8]
```

### keys(), values(), entries()

这三个方法和Java中Map类型的三个类似方法功能一样，都是用于遍历Array对象的。这也从另一方面验证了前面所说的：JS中的Array本质上仍然是一个Map。但是需要注意这三个函数都是ES6中新定义的，所以还存在很多兼容性问题。

### 函数式编程

JS是一门函数式语言，Java8中提供的[Stream API](http://blog.csdn.net/holmofy/article/details/77481304)功能也是学习了这一类语言。而且ES6开始支持[Lambda表达式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)，这使得JS的功能更强大。但是和Java8中的Stream不同的是：JS中的这些操作都不是惰性的，不会进行优化，也就是说任何一个方法都会立即返回一个新生成的数组。作为解释性的脚本语言，JS也没必要进行优化，毕竟JS设计这些API和Java的Stream目的性不同。

> Lambda表达式在JS中被叫做Arrow Functions，可能是因为运算符是“=>”的原因。
>
> 这里有篇文章介绍了JS中的Lambda表达式：https://hacks.mozilla.org/2015/06/es6-in-depth-arrow-functions/

如果把JS和Java类比的话，也可以将Array提供的函数划分为以下几类：

<table><tbody><tr><td colspan="3" align="center" border="0">Array函数分类</td></tr><tr><td rowspan="2" border="1">中间操作(Intermediate operations)</td><td>无状态(Stateless)</td><td>filter(), map()</td></tr><tr><td>有状态(Stateful)</td><td>sort(), reverse()<br/>slice()</td></tr><tr><td rowspan="2" border="1">终断操作(Terminal operations)</td><td>非短路操作</td><td>forEach()<br/>reduce(), reduceRight()</td></tr><tr><td>短路操作(short-circuiting)</td><td>every(), some()<br/>find(), findIndex()</td></tr></tbody></table>

### map()

![map](//img-blog.csdn.net/20180316232612261?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```javascript
// 提供一个函数将源Array中的元素按函数定义的规则映射成新的元素，最终返回新元素的数组。
var new_array = arr.map(function callback(currentValue, index, array) {
    // Return element for new_array
}[, thisArg])

// 示例：
Array.of(1,2,3,4).map(function(a){
    return a*a;
});
// 使用ES6新增Lambda表达式的写法：
Array.of(1,2,3,4).map(a => a*a);
// 返回数组：1,4,9,16
```

### filter()

![filter操作](//img-blog.csdn.net/20180316232633719?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```javascript
var newArray = arr.filter(function callback(element,index,array){
    // 返回true表示保留，false表示过滤不要
}[, thisArg]);

// 示例：
Array.of(1,2,3,4).filter(function(a){
    return a%2 == 0;
});
// 使用ES6新增Lambda表达式的写法：
Array.of(1,2,3,4).filter(a => a%2==0);
// 返回数组：1,4,9,16
```

### sort()

![sort操作](//img-blog.csdn.net/20180316232647468?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```javascript
// sort方法使用的排序算法不一定稳定，
// sort的性能以及稳定性要看JS引擎的具体实现。
var new_array = arr.sort([function compareFunction(a, b){
  if (a is less than b by some ordering criterion) {
    return -1;
  }
  if (a is greater than b by the ordering criterion) {
    return 1;
  }
  // a must be equal to b
  return 0;
}]);

// 示例：
Array.of(3,2,4,1).sort(); // 使用默认规则排序：1,2,3,4
Array.of(3,2,4,1).sort((a,b)=>b-a); // 自定义排序规则:4,3,2,1
```

### reverse()

![reverse](//img-blog.csdn.net/20180316232706903?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
数组逆序。这个很简单，而且也没什么参数。

### slice()

![slice](//img-blog.csdn.net/20180316232726335?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```javascript
// slice用于截取数组的一部分，包含begin位置，不包含end位置
arr.slice([begin[, end]])

// 示例：
Array.of(1,2,3,4).slice(1,3);
// 包含索引为1的“2”,不包含索引为3的“4”,最终结果为：2,3
```

### forEach()

```javascript
// forEach用于遍历数组中的元素
arr.forEach(function callback(currentValue, index, array) {
    //your iterator
}[, thisArg]);

// 示例：
Array.of(1,2,3,4).forEach((value,index)=>console.log("Array["+index+"] = "+value));
// 输出结果：
Array[0] = 1
Array[1] = 2
Array[2] = 3
Array[3] = 4
```

### reduce()与reduceRight()

![reduce操作](//img-blog.csdn.net/20180316232826592?watermark/2/text/Ly9ibG9nLmNzZG4ubmV0L0hvbG1vZnk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```javascript
// 提供一个函数将源Array中的元素按函数定义的规则进行累积。
reduce(function callback(accumulator, currentValue, currentIndex, array){
    // Accumulate
}[, initialValue])

// 示例：
Array.of(1,2,3,4).reduce((acc,v) => acc+v);
Array.of(1,2,3,4).reduce((acc,v) => acc+v, 0);// 这里的初始值默认为0
// 1+2+3+4 = 10

// reduceRight就是从后往前进行累积
arr.reduceRight(function callback(accumulator, currentValue, currentIndex, array){
    // Accumulate
}[, initialValue])

Array.of(1,2,3,4).reduce((acc,v)=>acc.concat(v), ""); // 1234
Array.of(1,2,3,4).reduceRight((acc,v)=>acc.concat(v), ""); // 4321
```

### every(),some()

```javascript
// every函数表示数组中的每个元素都符合callback函数中指定的条件
arr.every(function callback(currentValue,index,array){
    // test current value
}[, thisArg])

// some函数表示数组中存在一个或多个元素符合callback函数中指定的条件
arr.some(function callback(currentValue,index,array){
    // test current value
}[, thisArg])

Array.of(1,2,3,4).map(function(v){
    console.log(v);
    return v;
}).every(v=>v%2==1);
// 1,2,3,4都会输出，说明map操作并不像Java的Stream那样是惰性的。

Array.of(1,2,3,4).every(v=>v%2==1); // false
Array.of(1,2,3,4).some(v=>v%2==1); // true
```

### find(),findIndex()

```javascript
// find函数用于从数组中查找符合callback函数中指定条件的元素
// 如果未找到则返回undefined
arr.find(function callback(element,index,array){
    // test element
}[, thisArg]);

// findIndex函数用于从数组中查找符合callback函数中指定条件的元素
// 如果未找到则返回-1
arr.find(function callback(element,index,array){
    // test element
}[, thisArg]);

// 示例：
function isPrime(element) {
    if(element%2===0) return false;
    for(var i=3;i*i<=element;i+=2){
        if(element%i===0) return false;
    }
    return true;
}
Array.of(10,11,12,13,14,15)
     .find(isPrime);
// 找到第一个出现的素数就会返回: 11

Array.of(10,11,12,13,14,15)
     .findIndex(isPrime);
// 找到第一个出现的素数就会返回: 1
```

## 5. ArrayBuffer,DataView

如果说JS中的Array是Java中的List（容量可变），那么JS中的ArrayBuffer才是Java中真正的数组（容量不可变）。

随着浏览器技术的发展，JS不再甘心于操作BOM和DOM了，JS标准的制定者把目标定的更远。H5中[2D(Canvas)](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API)和[3D(WebGL)](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API)绘图以及[音视频](https://developer.mozilla.org/en-US/docs/Web/Media)的API也呼之欲出，这些API避免不了要直接操作内存，所以ArrayBuffer也随之诞生。

也就是说JS中的ArrayBuffer就是用来申请一段内存的，但是JS不允许我们直接使用ArrayBuffer这段内存，而是把对内存的操作封装成了视图，也就是`DataView`。

```java
// 申请16字节的内存
var buffer = new ArrayBuffer(16);
// ArrayBuffer的byteLength属性存储了内存的字节数
console.log(buffer.byteLength);
ArrayBuffer.isView(buffer); // false

// 创建几个视图来操作这段内存(一段内存可以被多个视图操作)
var view1 = new DataView(buffer);
var view2 = new DataView(buffer,12,4); // 操作12-16这四个字节
// ArrayBuffer.isView方法可以查看对象是否是视图类型
ArrayBuffer.isView(view1); // true
view1.setInt8(12, 42); // 把一个8位int型的整数放到第12个字节中

console.log(view2.getInt8(0)); // 使用视图二获取第12个字节的内容
// 输出值: 42
```

DataView类提供了下面16种操作内存的方法，如果把getter/setter看成一个整体，应该算8种。

8种操作分别对应8种数据类型，这倒和[标准C中提供的数据支持](http://en.cppreference.com/w/c/types)有点类似。

1. [`DataView.prototype.getFloat32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getFloat32)
2. [`DataView.prototype.getFloat64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getFloat64)
3. [`DataView.prototype.getInt16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt16)
4. [`DataView.prototype.getInt32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt32)
5. [`DataView.prototype.getInt8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt8)
6. [`DataView.prototype.getUint16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint16)
7. [`DataView.prototype.getUint32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint32)
8. [`DataView.prototype.getUint8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint8)
9. [`DataView.prototype.setFloat32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setFloat32)
10. [`DataView.prototype.setFloat64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setFloat64)
11. [`DataView.prototype.setInt16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt16)
12. [`DataView.prototype.setInt32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt32)
13. [`DataView.prototype.setInt8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt8)
14. [`DataView.prototype.setUint16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint16)
15. [`DataView.prototype.setUint32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint32)
16. [`DataView.prototype.setUint8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint8)

## 6. TypedArray

尽管DataView能操作各种数据类型，但是如果我在一段内存中只操作一种数据类型，那DataView就显得有些多余。所以JS还提供了用于操作单一数据类型的视图类 —— TypedArray。TypedArray主要有以下几种类型，它们的在使用上都是一样的。

| 类                                        | 单位元素的字节数 | 描述               | Web IDL类型             | C语言等价类型    |
| ---------------------------------------- | -------- | ---------------- | --------------------- | ---------- |
| [`Int8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array) | 1        | 8位有符号整形          | `byte`                | `int8_t`   |
| [`Uint8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) | 1        | 8位无符号整形          | `octet`               | `uint8_t`  |
| [`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray) | 1        | 8位无符号整形(clamped) | `octet`               | `uint8_t`  |
| [`Int16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int16Array) | 2        | 16位有符号整形         | `short`               | `int16_t`  |
| [`Uint16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint16Array) | 2        | 16位无符号整形         | `unsigned short`      | `uint16_t` |
| [`Int32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int32Array) | 4        | 32位有符号整形         | `long`                | `int32_t`  |
| [`Uint32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array) | 4        | 32位无符号整形         | `unsigned long`       | `uint32_t` |
| [`Float32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array) | 4        | 32位单精度浮点型        | `unrestricted float`  | `float`    |
| [`Float64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float64Array) | 8        | 64位双精度浮点型        | `unrestricted double` | `double`   |



> Mozilla的rhino引擎(Java实现)：https://github.com/mozilla/rhino
>
> 微软Edge浏览器中的ChakraCore引擎(C++实现)：https://github.com/Microsoft/ChakraCore
>
> 谷歌Chrome浏览器中的v8引擎(C++实现)：https://github.com/v8/v8



参考：

https://developer.mozilla.org/en-US/docs/Web/JavaScript