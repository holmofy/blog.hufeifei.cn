---
title: ES6语法学习-OOP面向对象编程
date: 2018-03-15
categories: 前端
---

## ES5中使用构造函数定义类

ES6之前定义一个类，都是通过定义构造函数实现：

```javascript
function Rectangle(x,y){
  this.x = x;
  this.y = y;
}
Rectangle.prototype.area = function(){
  return x * y;
}
console.log(new Rectangle(1, 2));
// Rectangle {x: 1, y: 2}
```

**为了和普通函数区分开来，构造函数首字母一般大写。**

如果把构造函数当做普通函数调用了，没有new指令分配内存，函数中的`this`就会指向全局变量：

```javascript
var r = Rectangle(1, 2);
console.log(r); // undefined
console.log(window.x) // 1
console.log(window.y) // 2
```

![OOP](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj7lgif8j208h05jaa2.jpg)

### 1. 防止构造函数被当成普通函数调用

有两种方式可以防止构造函数被当成普通函数调用。

1. 在构造函数中使用严格模式，构造函数第一行加上`use strict`

   ```javascript
   function Rectangle(x,y) {
     'use strict'
     this.x = x;
     this.y = y;
   }
   // 当成普通函数调用时会报错
   Rectangle();
   // TypeError: Cannot set property 'x' of undefined
   ```

   ​

2. 在构造函数中判断是否用了`new`指令，如果发现没有`new`，则内部创建一个对象返回

   ```javascript
   function Rectangle(x,y) {
     if(!this instanceof Rectangle) {
       // this引用的不是Rectangle对象，说明没有使用new指令
       // 手动加上new指令
       return new Rectangle(x,y);
     }
     this.x = x;
     this.y = y;
   }
   ```

### 2. 构造函数中的return

```javascript
function Rectangle(x,y) {
  this.x = x;
  this.y = y;
  return {};
}
console.log(new Rectangle(1,2)); // {}


function Rectangle(x,y) {
  this.x = x;
  this.y = y;
  return null;
}
console.log(new Rectangle(1,2)); // Rectangle {x: 1, y: 2}

function Rectangle(x,y) {
  this.x = x;
  this.y = y;
  return undefined;
}
console.log(new Rectangle(1,2)); // Rectangle {x: 1, y: 2}

function Rectangle(x,y) {
  this.x = x;
  this.y = y;
  return 10;
}
console.log(new Rectangle(1,2)); // Rectangle {x: 1, y: 2}

function Rectangle(x,y) {
  this.x = x;
  this.y = y;
  return 'Rectangle';
}
console.log(new Rectangle(1,2)); // Rectangle {x: 1, y: 2}
```

如果构造函数中返回的是对象类型，那么`new`指令会返回这个对象；如果是基本数据类型、`null`或`undefined`则返回new出来的对象。

### 3. new指令的执行原理

执行new指令，具体可以分为以下几步：

1. 创建空对象initObj
2. 将initObj的原型指向构造函数的prototype
3. 将initObj绑定到构造函数的this变量中
4. 开始执行构造函数

用代码表示上面的流程：

```javascript
function _new_( _constructor_, ...params) {
  // 创建一个空对象，继承构造函数的 prototype 属性
  var initObj = Object.create(_constructor_.prototype);
  // 将initObj绑定到构造函数this变量中，传入参数执行构造函数
  var result = _constructor_.apply(initObj, params);
  // 如果返回结果是对象，就直接返回，否则返回 context 对象
  return (typeof result === 'object' && result != null) ? result : initObj;
}
console.log(_new_(Rectangle,1,2)); // Rectangle {x: 1, y: 2}

function _new_( _constructor_, ...params) {
  var initObj = {};
  initObj.__proto__ = _constructor_.prototype;
  var result = _constructor_.apply(initObj, params);
  return (typeof result === 'object' && result != null) ? result : initObj;
}
console.log(_new_(Rectangle,1,2)); // Rectangle {x: 1, y: 2}
```

测试prototype：

```javascript
function Rectangle(x,y){
  'use strict'
  this.x = x;
  this.y = y;
}
var r = new Rectangle(1,2);
r.__proto__ == Rectangle.prototype; // true
r.__proto__ == Object.prototype; // false
var obj = new Object();
obj.__proto__ == Object.prototype; // true
```

## ES6中使用class定义类

```javascript
class Rectangle {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
}
console.log(new Rectangle()); // Rectangle {height: 1, width: 2}
```

### 1. class定义的类没有变量提升

```javascript
// class必须先声明在使用
var p = new Rectangle(); // ReferenceError

class Rectangle {}

//////////////////////////////////
// 构造函数会变量提升
new Rectangle();

function Rectangle() {}
```

### 2. 匿名类

```javascript
// unnamed
var Rectangle = class {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
};
console.log(new Rectangle(1,2));
// Rectangle {height: 1, width: 2}

///////////////////////////////////
// named
var Rectangle = class Rectangle {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
};
console.log(new Rectangle(1,2));
// Rectangle {height: 1, width: 2}
```

### 3. 一个类不允许有多个构造函数

```javascript
class Rectangle {
    // constructor
    constructor() {
        this.height = 1;
        this.width = 2;
    }
    // constructor
    constructor(height, width) {
        this.height = height;
        this.width = width;
    }
}
// SyntaxError: A class may only have one constructor
```

可以使用默认参数的方式提供默认值：

```javascript
class Rectangle {
    // constructor
    constructor(height=1, width=2) {
        this.height = height;
        this.width = width;
    }
}
```

### 4. 类中定义方法

```javascript
class Rectangle {
    // constructor
    constructor(height, width) {
        this.height = height;
        this.width = width;
    }
    area(){
		return this.width * this.height;
    }
}
```

### 5. 定义getter/setter方法

```javascript
class Rectangle {
    // constructor
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
  // 语法糖而已
  get area(){
    return this.width * this.height;
  }
}
var r = new Rectangle(1,2);
console.log(r.area);
```

### 6. 定义类静态方法

```javascript
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }

  move(deltaX,deltaY){
    this.x = this.x + deltaX;
    this.y = this.y + deltaY;
  }

  // 静态方法不能使用this访问类实例
  static distance(a, b) {
    const dx = a.x - b.x;
    const dy = a.y - b.y;

    return Math.hypot(dx, dy);
  }
}

const p1 = new Point(5, 5);
const p2 = new Point(10, 10);

console.log(Point.distance(p1, p2)); // 7.0710678118654755

// 相当于
Point.prototype.move = function(deltaX,deltaY){
    this.x = this.x + deltaX;
    this.y = this.y + deltaY;
}
Point.distance = function() {
    const dx = a.x - b.x;
    const dy = a.y - b.y;

    return Math.hypot(dx, dy);
  }
```

### 7. 类的继承

和原型链继承本质上是一样的

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(this.name + ' makes a noise.');
  }
}

class Dog extends Animal {
  speak() {
    console.log(this.name + ' barks.');
  }
}

var d = new Dog('Mitzie');
d.speak(); // Mitzie barks.
```

### 8. 使用`super`关键字调用父类方法

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(this.name + ' makes a noise.');
  }
}

class Dog extends Animal {
  speak() {
    super.speak();
    console.log(this.name + ' barks.');
  }
}

var d = new Dog('Mitzie');
d.speak();
// Mitzie makes a noise.
// Mitzie barks.
```



参考：

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get

http://javascript.ruanyifeng.com/oop/basic.html