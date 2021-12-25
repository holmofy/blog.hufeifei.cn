---
title: 找零钱问题
date: 2021-12-23
categories: 算法
mathjax: true
tags:
- 算法
keywords: 公考,算法,换零钱,数学
---

>（2019年江西）小王在商店消费了90元，口袋里只有1张50元、4张20元、8张10元的钞票，他共有几种付款方式，可以是店家不用找零钱？
> A. 5    B. 6    C. 7    D. 8

这是2019年江西的公务员考试题，但是从这题目中我嗅到了程序的味道。

如果题目改成90元钱换零钱，用50元、20元、10元有多少种换法？看上去就熟悉多了，也就是经典的“换零钱问题”。

## 常规思维

按照初等代数列个方程:

$$
50x+20y+10z=90
$$

然后看x、y、z的组合有多少种：
$$
\begin{align}
50\times1+20\times2+10\times0&=90 \newline
50\times1+20\times1+10\times2&=90 \newline
50\times1+20\times0+10\times4&=90 \newline
50\times0+20\times4+10\times1&=90 \newline
50\times0+20\times3+10\times3&=90 \newline
50\times0+20\times2+10\times5&=90 \newline
50\times0+20\times1+10\times7&=90 \newline
50\times0+20\times0+10\times9&=90 \newline
\end{align}
$$

因为10块钱最多只有8张，所以最后一种不可能，也就是说总共7种可能，题目答案选C。

如果只是把题目做出来就太没意思了，来看看“找零钱问题”中包含的算法思维。

## 递归思维

这是SICP（《计算机程序的构造和解释》）中提到的一个有意思的程序题。书中是这么说的：

> 已知有50美分、25美分、10美分、5美分和1美分共5种零钱，将1美元（100美分）换成零钱一共有多少种方式？

把这类题目泛化一下：把`m`块钱换成`coins`数组中的`n`种零钱(如【10,20,50】)有多少种换法？

假设零钱`coins`数组已经按照某种顺序排列了，将`m`元现金换成`n`种零钱的方式应该等于下面两项的总和：

* 将`m`元现金换成不包含第`n`种零钱的方式
* 将`m-coins[n]`换成n种零钱的方式

如此，不断递归将会把问题收敛为更少金额、更少零钱种类的情况。仔细分析以上规则，极端收敛后的情况为（递归出口）：

* 如果现金`m<0`或零钱种类`n=0`，应该算作有`0`种换零钱方式
* 如果现金`m=0`，应该算作有1种换零钱方式

我们用上面的90块钱换【50块、20块、10块】零钱来做演示：

$$
\begin{align}
f(90,[10,20,50])&=f(90,[10,20])+f(40,[10,20,{\color{Red}50}]) \newline
&=f(90,[10])+f(70,[10,20])+f(40,[10])+f(20,[10,20]) \newline
&=1+f(70,[10])+f(50,[10,20])+1+f(20,[10])+f(0,[10,20]) \newline
&=1+1+f(50,[10])+f(30,[10,20])+1+1+1 \newline
&=1+1+1+f(30,[10])+f(10,[10,20])+1+1+1 \newline
&=1+1+1+1+1+1+1+1 \newline
&=8
\end{align}
$$

## 写成代码

```java
public int changeCoins(int money, int[] coins, int n) {
    if (money < 0 || n <= 0) {
        return 0;
    }
    if (money == 0) {
        return 1;
    }
    return changeCoins(money - coins[n - 1], coins, n) + 
           changeCoins(money, coins, n - 1);
}

@Test
public void testChangeCoins() {
    Assert.assertEquals(changeCoins(90, new int[]{10, 20, 50}, 3), 8);
    Assert.assertEquals(changeCoins(90, new int[]{50, 20, 10}, 3), 8);
    Assert.assertEquals(changeCoins(190, new int[]{10, 20, 50}, 3), 26);
    Assert.assertEquals(changeCoins(100, new int[]{1, 2, 5, 10}, 4), 2156);
}
```