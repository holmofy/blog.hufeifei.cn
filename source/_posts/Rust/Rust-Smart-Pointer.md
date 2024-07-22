---
title: Rust智能指针类型
date: 2024-07-16
categories: Rust
tags: 
- Rust
---

|引用类型|Rust|C++|
|--|--|--|
|普通引用|`&T`|  |
|独享引用|`Box`|`std::unique_ptr`|
|共享引用(引用计数)|`Rc`|`std::shared_ptr<const T>`|
|共享可变引用|`Rc<RefCell<T>>`|`std::shared_ptr<T>`|
|多线程共享引用|`Arc<T>`||
|弱引用|`Weak<T>`|`std::weak_ptr<T>`|
