---
title: Rust入门
date: 2024-07-01
categories: Rust
mathjax: true
tags: 
- Rust
---

近十年来，golang、Swift、Kotlin、Typescript等新兴编程语言异军突起。在系统编程领域也出现了Rust和Zig等语言。

Rust吸取了C/C++、Java、Javascript、Python、Ruby等语言的优点，没有C++的历史包袱，轻装上阵，是这些新兴语言中唯一有实力与C/C++竞争的语言，有必要好好学一下，这篇文章先简单入个门。

## 变量

Rust的变量声明吸取了Javascript和Typescript的语法特性，使用`let`关键字。但是和Javascript不同的是，Rust是编译型语言，所以变量类型都是编译期确认，不过Rust的编译器可以根据初始化的变量进行类型推断，所以大部分情况下不需要显示声明类型。

### `let`关键字

```rust
let x; // 声明变量 "x"
x = 42; // 将42赋值给"x"
```

也可以写成一行，直接赋值

```rust
let x = 42;
```

### 类型声明

可以使用`:`显示声明变量的类型

```rust
let x: i32; // `i32` 是有符号32位整数
x = 42;

// 共有 i8, i16, i32, i64, i128 几个有符号整数类型
// 还有 u8, u16, u32, u64, u128 几个无符号整数类型
// 和其他语言一样，有 f32, f64    两种浮点类型
```

也可以写成一行，直接赋值

```rust
let x: i32 = 42;
```

> 点击[官方文档 data-types 章节](https://doc.rust-lang.org/book/ch03-02-data-types.html#scalar-types)查看更多的数据类型。

### 未初始化的变量

在C/C++中是允许使用未初始化的变量的，但是由于变量内存在栈中分配，这块内存有可能之前被使用过，所以为初始化的变量只是随机的。在Java、C#等新语言中，未初始化的变量会赋予默认值，比如`int`会赋值为`0`。

在rust中则会在编译阶段检查出这类错误。

```rust
let x;
foobar(x); // error: borrow of possibly-uninitialized variable: `x`
x = 42;
```

同时rust会根据第一次使用该变量的地方，推断出变量类型。

```rust
let x;
x = 42;
foobar(x); // 将从这里推断出“x”的类型
```

### 弃用变量

下划线`_`是一个特殊的变量名，或者更确切地说，是“没有名称”。它意味着变量值被扔掉不管了。

```rust
// 这里啥事儿没干， 因为42是个常量
let _ = 42;

// 这里调用了get_thing()，但是抛弃了它的返回值
let _ = get_thing();
```

下划线`_`打头的是常规变量，只是**编译器不会警告它们未使用**

```rust
// 我们最终可能会使用'x'，但我们的代码仍在编写中。
// 现在我们只想去掉编译器告警。
let _x = 42;
```

### 变量遮蔽(shadowing)

Rust 允许声明相同的变量名，在后面声明的变量会遮蔽掉前面声明的

```rust
fn main() {
    let x = 5;
    // 在main函数的作用域内对之前的x进行遮蔽
    let x = x + 1;

    {
        // 在当前的花括号作用域内，对之前的x进行遮蔽
        let x = x * 2;
        println!("The value of x in the inner scope is: {}", x);
    }

    println!("The value of x is: {}", x);
}
```

## 元组

大多数编程语言中都提供了数组类型，**数组类型是相同类型值的固定长度的集合**。而**元组类型是不同类型值的固定长度的集合**。但是元组类型只有少数语言提供了支持。

在Python语言中就有支持元组，这样可以让一个函数返回多个数据。

在原生Java中没有元组类型，但是[Apache Commons](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/tuple/package-summary.html)等第三方库中，使用`Pair`、`Triple`等类来实现类似元组的功能。

在[C++11标准](https://en.cppreference.com/w/cpp/utility/tuple)中也通过模版类的方式提供了元组的支持。

Rust的元组借鉴了Python的语法，在语言层面就支持了元组功能。

```rust
let pair = ('a', 17);
pair.0; // 这个值是 'a'
pair.1; // 这个值是 17
```

也可以显式的标记元组的类型

```rust
let pair: (char, i32) = ('a', 17);
```

> 点击[官方文档 data-types 章节](https://doc.rust-lang.org/book/ch03-02-data-types.html#compound-types)查看Rust的组合数据类型。

### 解构

解构赋值是专门针对元组、数组、结构体等复合类型的现代编程语言语法，我最早是在Javascript的ES6标准中接触到的。[C++17标准](https://en.cppreference.com/w/cpp/language/structured_binding)中也有提供解构复制的功能。

Rust的解构基本借鉴了Javascript的[ES6标准](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)。初学者刚学这个功能的时候会觉得Rust语法噪音太多了。

```rust
let (some_char, some_int) = ('a', 17);
// 现在, 变量`some_char`的值为 'a', `some_int`的值为 17
```

特别是当一个函数返回元组类型时，解构赋值非常有用：

```rust
let (left, right) = slice.split_at(middle);
```

当然，在解构元组的时候，也可以抛掉某些不需要的数据：

```rust
let (_, right) = slice.split_at(middle);
```

除了可以解构元组，还可以解构数组：

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
let [first, second, ..] = a;             // .. 表示忽略掉数组后面的值
println!("数组第一个值是: {}", first);     // 输出 1
println!("数组第二个值是: {}", second);    // 输出 2

let [.., last] = a;                      // .. 表示忽略掉数组前面的值
println!("数组最后一个值是: {}", last);     // 输出 5
```

甚至可以解构结构体：

```rust
struct A {
    int_value: i32,
    bigint_value: i64,
    float_value: f64,
}

let a = A {
    int_value: 1,
    bigint_value: 2,
    float_value: 1.2
};

let A { int_value, bigint_value, .. } = a;
println!(int_value);                      // 输出 1
println!(bigint_value);                   // 输出 2

let A { float_value, .. } = a;
println!("float value is: {}", float_value);  // 输出 1.2
```

> 详细可以参考[rust文档的destructuring一章](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring.html)。



* https://doc.rust-lang.org/stable/book/
* https://doc.rust-lang.org/rust-by-example/index.html
* https://course.rs/into-rust.html
* https://fasterthanli.me/articles/a-half-hour-to-learn-rust
* https://rust-book.junmajinlong.com/about.html
