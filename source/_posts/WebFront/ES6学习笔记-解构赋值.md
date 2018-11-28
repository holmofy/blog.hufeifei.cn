---
title: ES6语法学习-解构赋值
date: 2018-03-15 18:01
categories: 前端
---

# 解构赋值

解构赋值可以将**数组中的元素**或**对象中的属性**赋值给指定的变量。

### 1. 数组解构

#### 1.1 基本用法

```javascript
var a, b, rest;
// 数组解构
[a, b] = [10, 20];
console.log(a); // 10
console.log(b); // 20

// 变参解构
[a, b, ...rest] = [10, 20, 30, 40, 50];
console.log(a); // 10
console.log(b); // 20
console.log(rest); // [30, 40, 50]
// 变参后不能加逗号
var [a, ...b,] = [1, 2, 3];
// SyntaxError: rest element may not have a trailing comma

// 对象解构
({ a, b } = { a: 10, b: 20 });
console.log(a); // 10
console.log(b); // 20

// 变参解构
({a, b, ...rest} = {a: 10, b: 20, c: 30, d: 40});
console.log(a); // 10
console.log(b); // 20
console.log(rest); //{c: 30, d: 40}
```

上面的例子都是数组字面量或对象字面量的解构，但实际上大部分情况时对变量的解构。

```javascript
var foo = ['one', 'two', 'three'];

// 变量解构
var [one, two, three] = foo;
console.log(one); // "one"
console.log(two); // "two"
console.log(three); // "three"

// 数组长度更长
var arr = [1, 2, 3, 4, 5];
var [x,y] = arr;
console.log(x); // 1
console.log(y); // 2

// 数组长度过端
var arr = [];
var [x,y] = arr;
console.log(x); // undefined
console.log(y); // undefined

// 数组过短,提供默认值
var arr = [];
var [x=5,y=7] = arr;
console.log(x);
console.log(y);
```

#### 1.2 使用解构赋值实现变量交换

```javascript
var a = 1;
var b = 3;

[a, b] = [b, a];
console.log(a); // 3
console.log(b); // 1
```

#### 1.3 解构函数返回值

```javascript
function f() {
  return [1, 2];
}

var a, b;
[a, b] = f();
console.log(a); // 1
console.log(b); // 2
```

#### 1.4 忽略某些返回值

```javascript
function f() {
  return [1, 2, 3];
}

var [a, , b] = f();
console.log(a); // 1
console.log(b); // 3

// 忽略所有返回值
[,,] = f();
```

#### 1.5 数组解构与正则结合使用的实例

```javascript
// 使用正则和赋值解构获取url中的协议、主机、路径等信息
function parseProtocol(url) {
  var parsedURL = /^(\w+)\:\/\/([^\/]+)\/(.*)$/.exec(url);
  if (!parsedURL) {
    return false;
  }
  console.log(parsedURL);
  // ["https://developer.mozilla.org/en-US/Web/JavaScript", "https", "developer.mozilla.org", "en-US/Web/JavaScript"]

  var [, protocol, fullhost, fullpath] = parsedURL;
  return protocol;
}

console.log(parseProtocol('https://developer.mozilla.org/en-US/Web/JavaScript')); // "https"
```

### 2. 对象解构

#### 2.1 基本用法

```javascript
var o = {p: 42, q: true};
var {p, q} = o;

console.log(p); // 42
console.log(q); // true
```

#### 2.2 对象字面量的解构

```javascript
var a, b;

({a, b} = {a: 1, b: 2});
```

> 为了避免对象字面量的大括号`{ }`与代码块的大括号混淆，字面量解构的时候需要在外层包裹一个小括号`( )`。

#### 2.3 为解构的对象属性值定义新的变量名

```javascript
var o = {p: 42, q: true};
var {p: foo, q: bar} = o;

console.log(foo); // 42
console.log(bar); // true
```

#### 2.4 属性默认值

对于不存在的属性，默认解构得到的属性为`undefined`，但是可以设置默认值

```javascript
var {a, b} = {a: 3};

console.log(a); // 3
console.log(b); // undefined

// 提供默认值
var {a = 10, b = 5} = {a: 3};

console.log(a); // 3
console.log(b); // 5
```

#### 2.5 同时提供默认值并取新变量名

```javascript
var {a:aa = 10, b:bb = 5} = {a: 3};

console.log(aa); // 3
console.log(bb); // 5
```

#### 2.6 对象解构与函数默认参数结合使用的实例

```javascript
// ES5版本：函数参数默认值的设置
function drawES5Chart(options) {
  options = options === undefined ? {} : options;
  var size = options.size === undefined ? 'big' : options.size;
  var cords = options.cords === undefined ? {x: 0, y: 0} : options.cords;
  var radius = options.radius === undefined ? 25 : options.radius;
  console.log(size, cords, radius);
  // now finally do some chart drawing
}

// 调用ES5函数
drawES5Chart({
  cords: {x: 18, y: 30},
  radius: 30
});

// ES6版本：函数参数默认值的设置
function drawES2015Chart({size = 'big', cords = {x: 0, y: 0}, radius = 25} = {}) {
  console.log(size, cords, radius);
  // do some chart drawing
}

drawES2015Chart({
  cords: {x: 18, y: 30},
  radius: 30
});
```

#### 2.7 对象解构和数组解构嵌套使用

```javascript
var metadata = {
    title: 'Scratchpad',
    translations: [
       {
          locale: 'de',
          localization_tags: [],
          last_edit: '2014-04-14T08:43:37',
          url: '/de/docs/Tools/Scratchpad',
          title: 'JavaScript-Umgebung'
       }
    ],
    url: '/en-US/docs/Tools/Scratchpad'
};

var {title: englishTitle, translations: [{title: localeTitle}]} = metadata;

console.log(englishTitle); // "Scratchpad"
console.log(localeTitle);  // "JavaScript-Umgebung"
```

#### 2.8 循环迭代中使用解构赋值

```javascript
var people = [
  {
    name: 'Mike Smith',
    family: {
      mother: 'Jane Smith',
      father: 'Harry Smith',
      sister: 'Samantha Smith'
    },
    age: 35
  },
  {
    name: 'Tom Jones',
    family: {
      mother: 'Norah Jones',
      father: 'Richard Jones',
      brother: 'Howard Jones'
    },
    age: 25
  }
];

// 迭代解构赋值
for (var {name: n, family: {father: f}} of people) {
  console.log('Name: ' + n + ', Father: ' + f);
}

// "Name: Mike Smith, Father: Harry Smith"
// "Name: Tom Jones, Father: Richard Jones"
```

#### 2.9 通过对象解构直接从函数参数中获取对象字段

```javascript
function userId({id}) {
  return id;
}

function whois({displayName, fullName: {firstName: name}}) {
  console.log(displayName + ' is ' + name);
}

var user = {
  id: 42,
  displayName: 'jdoe',
  fullName: {
      firstName: 'John',
      lastName: 'Doe'
  }
};

console.log('userId: ' + userId(user)); // "userId: 42"
whois(user); // "jdoe is John"
```

#### 2.10 动态解构对象字段

```javascript
let key = 'z';
// key是变量
let {[key]: value} = {x: 'X', y: 'Y', z: 'Z'};

console.log(value); // "Z"
```

#### 2.11 变参解构切割对象

```javascript
let {a, b, ...rest} = {a: 10, b: 20, c: 30, d: 40}
a; // 10
b; // 20
rest; // { c: 30, d: 40 }
```

#### 2.12 解构特殊属性

```javascript
// fizz-buzz属性不能作为变量名
const foo = { 'fizz-buzz': true };
// 这里就必须取别名
const { 'fizz-buzz': fizzBuzz } = foo;

console.log(fizzBuzz); // "true"
```





参考：

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment