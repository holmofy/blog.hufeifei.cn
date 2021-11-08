---
title: ES6语法学习-let,var和const
date: 2018-03-15
categories: 前端
---

##var,let和const

ES6新增了`let`关键字**用于声明变量**，用法上和`var`类似，这里列举一些两者的区别。

### let与var区别

**`let`声明的变量只在它所在的代码块内有效**：

![let变量只在所在的代码块中有效](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj8wrhx0j20ah04mmx4.jpg)

因为上面的特性，所以`let`很适合在for循环中做计数器：

![在for循环中使用let，循环体外变量就无效](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbj9fjl6nj209c09xt8x.jpg)

**`var`声明的变量会被挂在到全局的window上，而`let`并不会**：

![let-in-loop](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbja31ne9j207h0f2wex.jpg)

> **let是function scope，而let是block scope**

**`let`不允许在同一个作用域内对同一个变量重复声明**：

![let-already](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjalfqyaj209k05m3yi.jpg)

**`let`不存在变量提升问题，必须先声明再使用**：

![var](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjb1bcjlj208r03umx5.jpg)

**`let`的作用域屏蔽造成临时性死区**：

![临时性死区](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjbiq2d1j2095030aa0.jpg)

### const定义常引用

**常引用不允许修改**

![常量不可变](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjbz9ji7j208y03t74a.jpg)

**`const`和`let`一样都有作用域**

![const作用域](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjcfb2gpj208u02paa0.jpg)

**引用的对象可以改变**

常引用而非常变量

![const对象可变](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbjcw9gf2j207f04a3yh.jpg)

