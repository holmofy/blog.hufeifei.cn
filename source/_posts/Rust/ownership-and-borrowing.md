---
title: 所有权、借用与变量的生命周期
date: 2024-07-15
categories: Rust
mathjax: true
tags: 
- Rust
---

在 Rust 中，所有权（ownership）、借用（borrowing）和生命周期（lifetime）是其内存安全和并发模型的核心概念。它们一起确保了在编译时捕获大部分内存错误，如空指针或悬挂指针。

我们可以通过代码示例和内存模型图来说明这些概念。

## 1. 所有权（Ownership）

所有权规则主要有三条：

* 每个值有一个所有者（owner）。
* 每个值同时只能有一个所有者。
* 当所有者离开作用域时，值会被丢弃。

示例代码：

```rust
fn main() {
    let s1 = String::from("hello"); // s1 是 "hello" 字符串的所有者
    let s2 = s1;                    // s1 的所有权转移给了 s2，s1 不再有效

    // println!("{}", s1);          // 这行代码会编译错误，因为 s1 已经无效
    println!("{}", s2);             // 输出 "hello"
}
```

## 2. 借用（Borrowing）

借用分为可变借用和不可变借用：

* 不可变借用：允许多个借用，但不能修改借用的值。
* 可变借用：一次只能有一个可变借用，且不能同时存在不可变借用。

示例代码：

```rust
fn main() {
    let s1 = String::from("hello");
    
    let s2 = &s1; // 不可变借用
    println!("{}", s2);

    let s3 = &mut s1; // 可变借用，编译器确保没有其他不可变借用
    s3.push_str(", world!");
    println!("{}", s3);
}
```

## 3. 生命周期（Lifetime）

生命周期用于描述引用的作用域，以防止悬挂引用。编译器通过生命周期注解（如 'a）来跟踪引用的生命周期，并确保所有引用在其生命周期内都是有效的。

示例代码：

```rust
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() {
        s1
    } else {
        s2
    }
}

fn main() {
    let string1 = String::from("long string is long");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```
