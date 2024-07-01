---
title: Rust入门
date: 2024-07-01
categories: Rust
mathjax: true
tags: 
- Rust
---

近十年来，golang、Swift、Kotlin、Typescript等新兴编程语言异军突起。在系统编程领域也出现了Rust和Zig等语言。

Java这类有GC的语言，除了基本类型和引用在栈上分配，其他所有的对象都在堆内存上分配，堆上分配意味着要CPU无法通过函数执行的栈指针来快速释放内存，因此需要专门的GC算法负责释放申请的堆内存，相当于JVM实现了部分操作系统的内存管理功能。换句话说，把内存管理的功能从操作系统层面迁移到了JVM应用层面，意味着程序员不需要直接跟操作系统交互管理内存，这就降低了程序员的心智负担。Java使用分代GC算法，并在版本迭代中一直对GC算法上不断优化，而且有JIT即时编译期进行逃逸分析和机器码缓存等性能优化。但代价是所有应用都得拖一个臃肿的虚拟机。

系统级编程语言没有GC，变量默认都是在栈上分配，栈上分配内存意味着在函数出栈后内存就会被认为已经回收了，所以这类系统编程语言都有变量生命周期的概念。系统级语言也可以直接调用操作系统的接口申请堆内存，但这也意味着应用需要自己管理从操作系统申请到的内存。

golang虽然丢掉了Java虚拟机，但是仍然是使用标记清除算法的GC语言，仍然会出现内存碎片的问题，本质上和Java换汤不换药。而且Java至少有多种GC算法可供选择，可以在吞吐量和延迟之间进行权衡，golang的GC算法就不像Java有那么多可以选择。

Rust直接对标C/C++等系统编程语言，并在语法层面吸取了C/C++、Java、Javascript、Python、Ruby等语言的优点，同时没有C++的历史包袱，轻装上阵。Rust是这些新兴语言中唯一有实力与C/C++竞争的语言，有必要好好学一下，这篇文章先简单入个门。

rust官方提供了[在线运行环境](https://play.rust-lang.org/)可以进行练习，先熟悉熟悉Rust基本的语法特性。

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

甚至可以解构结构体。这个在下面的结构体部分会提到。

> 详细可以参考[rust文档的destructuring一章](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring.html)。

## 函数

Rust使用`fn`关键字声明一个函数。Rust中的关键字都极其简短的缩略字，我猜测应该是和Rust大量使用的编译时过程宏有关，为了尽可能快的提高编译速度。

所以如果没有多门编程语言的经验，上来就学Rust可能会一脸懵逼。

```rust
fn greet() {
    println!("Hi there!");
}
```

使用`->`声明一个有返回值的函数：

```rust
fn fair_dice_roll() -> i32 {
    4
}
```

这里`4`是一个表达式，是一个隐式的返回值。也可以使用`return`关键字显式声明返回值。

```rust
fn fair_dice_roll() -> i32 {
    return 4;
}
```

Rust还提供了`async`关键字来支持异步函数，不过Rust没有提供原生的异步运行时，而是由第三方实现，目前有[`tokio`](https://tokio.rs/)和[`async-std`](https://async.rs/)等几个主流的异步运行时。

```rust
async fn request() -> Response {
    let res = send_request().await;
    return res;
}
```

## 闭包

Rust也支持函数式编程。在Java中最早是不支持函数式编程的，都是通过匿名类的方式实现回调等功能；到了Java8提供了[`java.util.function`](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)标准函数式接口，并在语法层面支持lambda表达式。

Rust的函数式编程语法借鉴了Ruby的闭包语法，这里的闭包就等价于java的函数式接口。

```rust
fn for_each_planet<F>(f: F)
    where F: Fn(&'static str)
{
    f("Earth");
    f("Mars");
    f("Jupiter");
}

fn main() {
    for_each_planet(|planet| println!("Hello, {}", planet));
}
```

Rust使用两根竖杠`|`来标明闭包的参数。

## 结构体

和C语言一样，Rust使用`struct`关键字定义结构体：

```rust
struct Vec2 {
    x: f64,
    y: f64,
}
```

结构体变量的初始化如下：

```rust
// 和C语言一样，结构体变量默认在栈中分配内存
let v1 = Vec2 { x: 1.0, y: 3.0 };
let v2 = Vec2 { y: 2.0, x: 4.0 };
```

Rust的结构体支持和[Javascript的Object更新的Spread语法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)。

这个在Rust中称为[struct update syntax](https://rust-lang.github.io/rfcs/2528-type-changing-struct-update-syntax.html)。

```rust
let v3 = Vec2 {
    x: 14.0,
    ..v2
};
```

结构体也支持解构：

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

## 在条件语句中进行解构

这个语法是Rust的特色。

### 在`if let`中进行解构

`let`变量声明可以在`if`条件语句中使用。

```rust
struct Number {
    odd: bool,
    value: i32,
}

fn main() {
    let one = Number { odd: true, value: 1 };
    let two = Number { odd: false, value: 2 };
    print_number(one);
    print_number(two);
}

fn print_number(n: Number) {
    if let Number { odd: true, value } = n {
        println!("Odd number: {}", value);
    } else if let Number { odd: false, value } = n {
        println!("Even number: {}", value);
    }
}
```

### 在`match`语句中进行解构

Rust使用`match`语句实现类似于C/C++、Java等语言的`switch`分支判断功能。区别在于**`match`必须匹配所有可能的结果**，而`switch`使用`default`分支来覆盖未匹配的分支，而且`switch`没有严格要求必须有`default`分支。

解构语句也可以在`match`判断条件中使用：

```rust
fn print_number(n: Number) {
    match n {
        Number { value: 1, .. } => println!("One"),
        Number { value: 2, .. } => println!("Two"),
        Number { value, .. } => println!("{}", value),
        // 如果最后一个分支不存在，将会编译错误
    }
}
```

可以使用`_`来实现类似于`switch`的`default`分支的功能：

```rust
fn print_number(n: Number) {
    match n.value {
        1 => println!("One"),
        2 => println!("Two"),
        _ => println!("{}", n.value),
    }
}
```

## 结构体的方法

C语言是过程式语言，没有针对struct绑定的方法。C++的面向对象因为有多继承的问题，导致了很多复杂的问题。Rust在C和C++之间取了折中的方案：首先不支持继承，可复用的特性使用trait定义。关于trait特性下面会提及。

可以针对struct声明相应的方法：

```rust
struct Number {
    odd: bool,
    value: i32,
}

impl Number {
    fn is_strictly_positive(self) -> bool {
        self.value > 0
    }
}
```

这里的`self`类似于Java的`this`，并且这个**`self`必须是方法的第一个参数**，这一点上和python很类似。

和其他大多数语言一样，可以用`.`调用结构体的方法：

```rust
fn main() {
    let minus_two = Number {
        odd: false,
        value: -2,
    };
    println!("positive? {}", minus_two.is_strictly_positive());
    // this prints "positive? false"
}
```

## 可变与不可变

在Rust中，变量默认是不可变的，变量内部的字段也不可修改：

```rust
fn main() {
    let n = Number {
        odd: true,
        value: 17,
    };
    // 下面的修改赋值编译时将会报错
    n.odd = false; // error: cannot assign to `n.odd`,
                   // as `n` is not declared to be mutable
}
```

变量也不能重新赋值：

```rust
fn main() {
    let i = 3;
    // 编译报错
    i = 4; // error: cannot assign twice to immutable variable `i`

    let n = Number {
        odd: true,
        value: 17,
    };
    // 编译报错
    n = Number {
        odd: false,
        value: 22,
    }; // error: cannot assign twice to immutable variable `n`
}
```

`mut`关键字可以将变量声明为可变变量：

```rust
fn main() {
    let mut n = Number {
        odd: true,
        value: 17,
    }
    n.value = 19; // ok
}
```

## Trait

在Java、TypeScript等语言中都提供了`interface`的功能，用来定义一类对象的共同特性。在Rust中这个功能被`trait`关键字定义。

```rust
trait Signed {
    fn is_strictly_negative(self) -> bool;
}
```

定义trait后，可以在任意类型上去实现这个trait。一个类型也可以实现多个trait。

```rust
impl Signed for Number {
    fn is_strictly_negative(self) -> bool {
        self.value < 0
    }
}

fn main() {
    let n = Number { odd: false, value: -44 };
    println!("{}", n.is_strictly_negative()); // prints "true"
}
```

甚至可以在基本类型上实现trait：

```rust
impl Signed for i32 {
    fn is_strictly_negative(self) -> bool {
        self < 0
    }
}

fn main() {
    let n: i32 = -44;
    println!("{}", n.is_strictly_negative()); // prints "true"
}
```

## 数据范围(Range)

Range这个功能在原生Java中没有提供支持，但是在[Apache Commons](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/Range.html)提供了支持。

Rust在语法层面就支持了Range，语法借鉴自Python：

```rust
fn main() {
    // [0, +∞)
    println!("{:?}", (0..).contains(&0)); // true
    println!("{:?}", (0..).contains(&100)); // true
    println!("{:?}", (0..).contains(&-1)); // false
    println!("{:?}", (0..).typeid); // true
    // (-∞, 20)
    println!("{:?}", (..20).contains(&20)); // false
    println!("{:?}", (..20).contains(&0));  // true
    println!("{:?}", (..20).contains(&-20));  // true
    // (-∞, 20]
    println!("{:?}", (..=20).contains(&20)); // true
    println!("{:?}", (..=20).contains(&-20)); // true
    println!("{:?}", (..=20).contains(&21)); // false
    // [3, 6)
    println!("{:?}", (3..6).contains(&4)); // true
    println!("{:?}", (3..6).contains(&6)); // false
    println!("{:?}", (3..6).sum::<i32>()); // 3+4+5 = 12
    println!("{:?}", (3..6).last()); // Some(5)

    // [0.0, 1.0)
    println!("{:?}", (0.0..1.0).contains(&0.5)); // true
    // 浮点型范围不可遍历，没有sum()、last()等方法
}
```


* https://doc.rust-lang.org/stable/book/
* https://doc.rust-lang.org/rust-by-example/index.html
* https://google.github.io/comprehensive-rust/index.html
* https://course.rs/into-rust.html
* https://fasterthanli.me/articles/a-half-hour-to-learn-rust
* https://rust-book.junmajinlong.com/about.html
