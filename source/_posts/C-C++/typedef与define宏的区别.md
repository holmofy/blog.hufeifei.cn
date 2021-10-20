---
title: typedef与#define的区别
date: 2017-07-25
categories: C&C++
tags: 
- C
---

`typedef`关键字用于为类型数据创建别名，通常的用法如下：

```c
typedef char* PCHAR;

typedef struct list_node{
    int value;
    list_node *next;
} Node;

typedef void (*PFUNC)(int);
```

为了程序跨平台，我们可能会对一些类型取一个特殊的名字，换一个平台我们只需要修改这些类型的定义即可，比如`stdint.h`头文件中对各种长度的整形进行了定义：

```c
typedef signed char        int8_t;
typedef short              int16_t;
typedef int                int32_t;
typedef long long          int64_t;
typedef unsigned char      uint8_t;
typedef unsigned short     uint16_t;
typedef unsigned int       uint32_t;
typedef unsigned long long uint64_t;

typedef signed char        int_least8_t;
typedef short              int_least16_t;
typedef int                int_least32_t;
typedef long long          int_least64_t;
typedef unsigned char      uint_least8_t;
typedef unsigned short     uint_least16_t;
typedef unsigned int       uint_least32_t;
typedef unsigned long long uint_least64_t;

typedef signed char        int_fast8_t;
typedef short              int_fast16_t;
typedef int                int_fast32_t;
typedef long long          int_fast64_t;
typedef unsigned char      uint_fast8_t;
typedef unsigned short     uint_fast16_t;
typedef unsigned int       uint_fast32_t;
typedef unsigned long long uint_fast64_t;

typedef long long          intmax_t;
typedef unsigned long long uintmax_t;
```

> 各个平台编译器中`stdint.h`定义可能不一样。

有时候，我们也会使用宏定义的方式来定义宏类型名，比如：

```c
#define PCHAR char *
```

`typedef char* PCHAR;`和`#define PCHAR char *`都可以表示字符指针，但这两者还是有明显的区别：

区别一：

```c
typedef char* PCHAR;
unsigned PCHAR pc; //语法错误
```

```c
#define PCHAR char *
unsigned PCAHR pc; // 编译通过
// 因为宏定义在预处理时全部被替换，所以unsigned PCAHR pc;会被替换成unsigned char *pc;
```

区别二：

```c
typedef char* PCHAR;
PCHAR pa,pb;
// pa,pb都是字符指针类型
```

```c
#define PCHAR char *
PCHAR pa,pb;
// PCHAR pa,pb;会被翻译成char *pa,pb;
// 这导致pa是字符指针类型，而pb是字符类型
```



所以定义变量类型时建议还是使用`typedef`关键字，预处理宏适合定义常量。
