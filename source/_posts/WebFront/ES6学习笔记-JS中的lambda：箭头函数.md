---
title: ES6语法学习-JS中的lambda:箭头函数
date: 2018-03-15
categories: 前端
---

##### 1. 最基本的写法

使用`=>`操作符，简化匿名函数的定义

```javascript
(param1,param2,...,paramN) => {
  // 函数体
}

// 参数列表与箭头符号不能换行
var func = ()
           => 1;
// SyntaxError: expected expression, got '=>'

// 注意解析的优先级
let callback;
callback = callback || function() {}; // ok
callback = callback || () => {};
// SyntaxError: invalid arrow-function arguments
// 需要加括号提高解析优先级
callback = callback || (() => {});    // ok
```

![最基础的lambda](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjespwloj208n020t8m.jpg)

##### 2. 如果函数体只有一条语句，可以省略函数体的大括号

```javascript
(param1,param2,...,paramN) => expression
// 等价于
(param1,param2,...,paramN) => {
  return expression;
}
```

![省略函数体的大括号](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjfbv04fj209h04oq2z.jpg)

##### 3. 如果参数只有一个，可以省略参数列表的小括号

```javascript
singleParam => { statements }
// 等价于
(singleParam) => { statements }
```

##### 4. 如果函数没有参数，参数列表的小括号不能省略

```javascript
// 参数列表的小括号不能省略
() => { statements }
```

##### 5. 当返回值是对象字面量时，为了避免函数体的大括号与对象字面量的大括号冲突，必须在字面量上加一对小括号

```javascript
params => ({foo: bar})
```

![返回对象字面量](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjfyvhfrj207702zweg.jpg)

##### 6. 使用可变参数

```javascript
(param1, param2, ...rest) => { statements }
```

![lambda可变参数](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjgnglifj20a8023t8n.jpg)

##### 7. 参数默认值

默认情况下参数默认值为`undefined`，可以使用`param=defaultValue`的方式指定参数的默认值。

```javascript
(param1 = defaultValue1,
 param2 = defaultValue2, …, paramN = defaultValueN) => {
	statements
}
```

##### 8. 在参数列表中使用解构赋值

```javascript
// a+b+c = a+b+a+b
var f = ([a=5, b=6] = [1, 2], {x: c} = {x: a + b}) => a + b + c;

f(); // 6
f([3,4]); // 14
f([3]); // 18
f([3,4],{x:5}); // 12
```

##### 9. 对象解构赋值

```javascript
var materials = [
  'Hydrogen',
  'Helium',
  'Lithium',
  'Beryllium'
];

materials.map(function(material) {
  return material.length;
}); // [8, 6, 7, 9]
/* 等价于 */
materials.map((material) => {
  return material.length;
}); // [8, 6, 7, 9]
/* 等价于 */
/* 在参数列表中直接解构数组对象的length属性 */
materials.map(({length}) => length); // [8, 6, 7, 9]
```

##### 10. 箭头函数没有绑定`this`变量

this作用域问题

```javascript
function Person() {
  // 构造函数中的this表示对象本身
  this.age = 0;

  setInterval(function growUp() {
    // 在non-strict模式中, growUp()函数中的this是全局对象
    // growUp是全局函数，不是对象的方法
    this.age++;
    // 浏览器环境中全局对象为window
    // window.age=undefined
    // undefined++ -> NAN
    // NAN++ -> NAN
    // 一直都是NAN
  }, 1000);
}

var p = new Person();
```

ES3/5中，可以定义一个变量指向外部对象

```javascript
function Person() {
  var that = this;
  that.age = 0;

  setInterval(function growUp() {
    // that暂存了对象的引用
    that.age++;
  }, 1000);
}

// 还有一种方式直接绑定函数的this对象
function Person() {
  this.age = 0;

  setInterval(function growUp() {
    // 这时的this就是Person对象
    this.age++;
  }.bind(this), 1000);
}
```

ES6有了箭头函数，就不用这么麻烦了，因为箭头函数中没有this变量：

```javascript
function Person(){
  this.age = 0;

  setInterval(() => {
    // 这里的this对象就是Person对象
    this.age++;
  }, 1000);
}

var p = new Person();
```

##### 11. 箭头函数使用`call`和`apply`方法

```javascript
var func = (a,b) => a+b;
func.call(null,1,2); // 3
func.apply(null,[1,2]); // 3
```

##### 12. 箭头函数没有绑定`arguments`变量

箭头函数没有自己的arguments变量。

```javascript
var arr = () => arguments[0];
arr(); // ReferenceError: arguments is not defined

var arguments = [1, 2, 3];
// 因为箭头函数没有自己的arguments
// 所以访问的是父作用域中的arguments
var arr = () => arguments[0];
arr(); // 1

// 使用可变参数作为arguments使用
var f = (...args) => args[0] + n;

// 普通函数有自己的arguments变量
function foo(n) {
  var f = () => arguments[0] + n;
  return f();
}

foo(1); // 2
```

##### 13. 箭头函数作为对象的方法，注意this的问题

```javascript
var obj = {
  i: 10,
  // 箭头函数没有this,所以访问的是全局this
  b: () => console.log(this.i, this),
  c: function() {
    console.log(this.i, this);
  }
}

obj.b(); // undefined, Window {...} (or the global object)
obj.c(); // 10, Object {...}
```

##### 14. 箭头函数不能作为构造函数

```javascript
var Foo = () => { this.age = 18 };
var foo = new Foo(); // TypeError: Foo is not a constructor
```

##### 15. 箭头函数没有`prototype` 属性

```javascript
var Foo = () => {};
console.log(Foo.prototype); // undefined
```


参考：

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions