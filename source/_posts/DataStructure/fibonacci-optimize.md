---
title: 斐波那契数列算法优化问题
date: 2017-08-7
categories: 数据结构与算法
mathjax: true
keywords:
- 斐波那契数列
- 递归算法
---

[斐波那契](https://en.wikipedia.org/wiki/Fibonacci_number)是数学中最值得讨论的一个问题，从12世纪斐波那契提出这个数列后，就有很多数学家研究过这个数列，对斐波那契数列的新发现也越来越多，这些细节我没能力去研究，这篇文章中要讲的是编程中对生成斐波那契数算法的优化。首先要说的就是斐波那契数列的定义，这一切都起源于一个生殖能力超强的兔子：

* 第一个月初有一对刚诞生的兔子
* 第二个月后(第三个月初)他们可以生育
* 每月没对兔子可生育的兔子会诞生下一对新兔子
* 兔子永不死去

几乎每个学计算机的在学编程语言的时候都会遇到这样的习题：计算第N个月兔子的总数

> 点击[这里](https://github.com/holmofy/algorithm/tree/master/Fibonacci)查看完整源代码，建议对着完整的代码调试。

## 简单的递归算法

老师肯定会教的一种方法：

```c
uint64_t fibonacci(unsigned int n) {
	if (n == 0) return 0;
	if (n <= 2) return 1;
	return fibonacci(n - 1) + fibonacci(n - 2);
}
```

> 该方法来自于斐波那契数列的一个递推式：`fib(n) = fib(n-1) + fib(n-2)`
>
> 然后使用递归算法并指定递归出口，即可得出结果。

## 用循环迭代消除递归

递归因为要不断地调用函数自身，调用函数就伴随着参数以及函数局部变量入栈，当递归层数较大容易产生栈溢出，所以通常需要我们使用循环优化递归算法。幸运地是，大多数递归都能修改成循环（使用自定义栈保存变量的方式仍然算递归）。而且上面的算法在效率上存在很大的优化空间：

![斐波那契数列](http://img-blog.csdn.net/20170807193106433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

你会发现fib(5) = fib(4) + fib(3)，而求fib(4)的时候我们已经求过fib(3)，这意味着我们做了很多重复的工作，很明显我们需要把前面做过的工作暂存。

> 递归算法时间呈指数形式增长：O(2^N)；而使用循环迭代时间上呈线性增长：O(N)。在我笔记本上测试时，当n超过40递归算法的时间就开始爆炸了。

```c
uint64_t fibonacci(unsigned int n) {
	if (n == 0) return 0;
	if (n == 1 || n == 2) return 1;
	uint64_t f1 = 1, f2 = 1, fn;
	for (unsigned int i = 3; i <= n; i++) {
		fn = f1 + f2;
		f1 = f2;
		f2 = fn;
	}
	return fn;
}
```

## 阵算法求解

斐波那契数列的递推公式是：fib(n) = fib(n-1) + fib(n-2)；我们可以用矩阵来表示这种关系：
$$
\begin{bmatrix} F_{n} \\ F_{n-1} \end{bmatrix}
 = \begin{bmatrix} F_{n-1}-F_{n-2} \\ F_{n-1} \end{bmatrix}
 = \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix} \times \begin{bmatrix} F_{n-1} \\ F_{n-2} \end{bmatrix}
$$
进一步推到可以得到：
$$
\begin{bmatrix} F_{n} \\ F_{n-1} \end{bmatrix}
= \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}^{n-1} \times \begin{bmatrix} F_1 \\ F_0 \end{bmatrix}
= \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}^{n-1} \times \begin{bmatrix} 1 \\ 0 \end{bmatrix}
$$

从0开始算得到Fn则需要更进一步：
$$
\begin{bmatrix}  F_{n+1} \\  F_{n}  \end{bmatrix}
 = \begin{bmatrix}  1 & 1 \\  1 & 0 \end{bmatrix}^{n} \times \begin{bmatrix} F_1 \\ F_0 \end{bmatrix}
 = \begin{bmatrix}  1 & 1 \\  1 & 0 \end{bmatrix}^{n} \times \begin{bmatrix} 1 \\ 0 \end{bmatrix}
$$

我们要实现一下矩阵运算：
$$
\begin{bmatrix} a_{00} & a_{01} \\ a_{10} & a_{11} \end{bmatrix} \times \begin{bmatrix} b_{00} & b_{01} \\ b_{10} & b_{11} \end{bmatrix}
 = \begin{bmatrix} a_{00}b_{00}+a_{01}b_{10} & a_{00}b_{01}+a_{01}b_{11} \\ a_{10}b_{00}+a_{11}b_{10} & a_{10}b_{01}+a_{11}b_{11} \end{bmatrix}
$$

特别地：
$$
\begin{bmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix}^0
 = \begin{bmatrix} 1 & 0 \\ 0 & 1 \end{bmatrix}
$$

```c
uint64_t fibonacci(unsigned int n) {
	uint64_t m[2][2] = { 1,1,1,0 }; // 1次矩阵
	uint64_t result[][2] = { 1,0,0,1 }; // 单位矩阵
	uint64_t temp[2][2];
	// 计算n次矩阵
	for (unsigned int i = 1; i <= n; i++) {
		temp[0][0] = result[0][0] * m[0][0] + result[0][1] * m[1][0];
		temp[0][1] = result[0][0] * m[0][1] + result[0][1] * m[1][1];
		temp[1][0] = result[1][0] * m[0][0] + result[1][1] * m[1][0];
		temp[1][1] = result[1][0] * m[0][1] + result[1][1] * m[1][1];
		memcpy(result, temp, sizeof(uint64_t) * 4);
	}
	// result[1][0] * 1 + result[1][1] * 0;
	return result[1][0] * 1;
}
```

> 该算法在时间上也是按线性增长的：O(N)，由于for循环内指令较多，所以可能会比循环迭代算法更耗时。但是该算法有很多可优化的地方，这里作为引子，方便下面算法的理解。

## 阵快速幂优化矩阵算法

在计算整数的乘法时，计算机底层是通过加法和移位运算实现的，举个例子：

```
十进制：4*13 => 二进制：100b*1101b = 100b*(1000b+100b+00b+1b) = (100b<<3)+(100b<<2)+0+(100b<<1)
```

快速幂：对于幂运算，我们也可以用类似的方式进行优化。

通常我们进行幂运算会直接循环累积，比如：4^13，会循环13次。

但是如果我们使用乘法的结合律就可以将时间复杂度降到O(log(N))：

```
4^13 = (4^8) + (4^4) + (4^1) ;
4^8 = 4^4 * 4^4;
4^4 = 4^2 * 4^2;
4^2 = 4*4;
```

快速幂实现如下：

```c
int quick_pow(int base, int exp) {
	int result = 1;
	while (exp) {
		if (exp & 1)
			result *= base;
		exp >>= 1;
		base *= base; // 2,4,8...次幂
	}
	return result;
}
```

我们可以把快速幂的思想应用到矩阵运算上，从而对上面的矩阵算法进行优化：

```java
uint64_t fibonacci(unsigned int n) {
	uint64_t m[2][2] = { 1,1,1,0 }; // 1次矩阵
	uint64_t result[][2] = { 1,0,0,1 }; // 单位矩阵
	uint64_t temp[2][2];
	while (n) {
		if (n & 1) {
			temp[0][0] = result[0][0] * m[0][0] + result[0][1] * m[1][0];
			temp[0][1] = result[0][0] * m[0][1] + result[0][1] * m[1][1];
			temp[1][0] = result[1][0] * m[0][0] + result[1][1] * m[1][0];
			temp[1][1] = result[1][0] * m[0][1] + result[1][1] * m[1][1];
			memcpy(result, temp, sizeof(uint64_t) * 4);
		}
		// 2、4、8...次幂矩阵
		temp[0][0] = m[0][0] * m[0][0] + m[0][1] * m[1][0];
		temp[0][1] = m[0][0] * m[0][1] + m[0][1] * m[1][1];
		temp[1][0] = m[1][0] * m[0][0] + m[1][1] * m[1][0];
		temp[1][1] = m[1][0] * m[0][1] + m[1][1] * m[1][1];
		memcpy(m, temp, sizeof(uint64_t) * 4);
		n >>= 1;
	}
	// result[1][0] * 1 + result[1][1] * 0;
	return result[1][0] * 1;
}
```

## 用常量表优化幂矩阵运算

为了减少计算2、4、8...次幂矩阵所消耗的时间，我们可以提前把这些矩阵幂求出来并存在常量表中，这样可以减少乘法运算的次数。

先写个程序自动生成常量表。

```c
void power_matrix(uint64_t m[][2], unsigned int exp) {
	uint64_t result[][2] = { 1,0,0,1 }; // 单位矩阵
	uint64_t temp[2][2];
	// 计算n次矩阵
	for (unsigned int i = 1; i <= exp; i++) {
		temp[0][0] = result[0][0] * m[0][0] + result[0][1] * m[1][0];
		temp[0][1] = result[0][0] * m[0][1] + result[0][1] * m[1][1];
		temp[1][0] = result[1][0] * m[0][0] + result[1][1] * m[1][0];
		temp[1][1] = result[1][0] * m[0][1] + result[1][1] * m[1][1];
		memcpy(result, temp, sizeof(uint64_t) * 4);
	}
	memcpy(m, result, sizeof(uint64_t) * 4);
}
void generate_matrix() {
	uint64_t m[2][2] = { 1,1,1,0 }; // 1次矩阵
	uint64_t temp[2][2];
	for (int i = 0; i < 8; i++) {
		memcpy(temp, m, 4 * sizeof(uint64_t));
		printf("{");
		power_matrix(temp, 1 << i);
		for (int j = 0; j < 2; j++) {
			for (int k = 0; k < 2; k++) {
				printf("%llu, ", temp[j][k]);
			}
		}
		printf("},\n");
	}
}
```

调用generate_matrix函数生成0~7次矩阵。

把生成的常量表复制粘贴到代码中：

```c
uint64_t fibonacci6(unsigned int n) {
	const static uint64_t cache[][2][2] = {
		{ 1, 0, 0, 1 },// 0次幂(无用)
		{ 1, 1, 1, 0 },// 1次幂(2^0,1)
		{ 2, 1, 1, 1 },// 2次幂(2^1,2)
		{ 5, 3, 3, 2 },// 4次幂(2^2,3)
		{ 34, 21 ,21, 13 },// 8次幂(2^3,4)
		{ 1597, 987, 987 ,610 },// 16次幂(2^4,5)
		{ 3524578, 2178309, 2178309, 1346269 },// 32次幂(2^5,4)
		{ 17167680177565, 10610209857723, 10610209857723, 6557470319842},//64次幂(2^6,5)
		{ 8102862946581596898, 18154666814248790725, 18154666814248790725, 8394940206042357789}//128次幂(2^7,6)
	};
	uint64_t result[][2] = { 1,0,0,1 }; // 单位矩阵
	uint64_t temp[2][2];
	int bit_pos = 1;
	while (n) {
		if (n & 1) {
			temp[0][0] = result[0][0] * cache[bit_pos][0][0] + result[0][1] * cache[bit_pos][1][0];
			temp[0][1] = result[0][0] * cache[bit_pos][0][1] + result[0][1] * cache[bit_pos][1][1];
			temp[1][0] = result[1][0] * cache[bit_pos][0][0] + result[1][1] * cache[bit_pos][1][0];
			temp[1][1] = result[1][0] * cache[bit_pos][0][1] + result[1][1] * cache[bit_pos][1][1];
			memcpy(result, temp, sizeof(uint64_t) * 4);
		}
		n >>= 1;
		bit_pos++;
	}
	// result[1][0] * 1 + result[1][1] * 0;
	return result[1][0] * 1;
}
```

> 这种方法的缺点是所能求的斐波那契最大项决定于表的大小，上面代码实现中所能求的最大项是255(八位全1的情况)，不过斐波那契数列的第94项就已经超过64位无符号整形了。
>
> (第93项斐波那契数为1220 0160 4151 2187 6738，而64位无符号整形最大值为2^64-1=1844 6744 0737 0955 1615，第94项为1974 0274 2198 6822 3167溢出就成了129 3530 1461 5867 1551)
>
> 如果想要求更大的斐波那契数，则需要自己实现一个类似于Java中的BigInteger类(C++应该会有很多类似的开源库)

## 项式直接求解

求斐波那契数列通项有很多种方法，这里用最容易理解的方法进行求解：初等代数进行数列代换。

这种方法只要有高中数学水平就可以解出来(当时高中解出来的时候，还以为是什么天大的发现^_^)。

* $a_0=0; a_1=1;$
* $a_n=a_{n-1}+a_{n-2};$

①、构造等比数列

​    $a_n+αa_{n-1}=β(a_{n-1}+αa_{n-2})$

​    $a_n=(β-α)a_{n-1}+αβa_{n-2}$

​    可得到系数关系：
$$
\left\{\begin{matrix} β-α=1 \\ αβ=1 \end{matrix}\right.
$$
​    解得：
$$
\left\{\begin{matrix} α=\frac{\sqrt{5}-1}{2} \\ β=\frac{\sqrt{5}+1}{2} \end{matrix}\right.
或
\left\{\begin{matrix} α=\frac{-\sqrt{5}-1}{2} \\ β=\frac{-\sqrt{5}+1}{2} \end{matrix}\right.
$$
​    因为$a_n+αa_{n-1}=β(a_{n-1}+αa_{n-2})$，所以{ $a_n+αa_{n-1}$ }是公比为β的等比数列，首项为$a_1+αa_0=1$

​    求等比数列通项：
$$
a_n+αa_{n-1}=(a_1+αa_0)β^{n-1}=β^{n-1}
$$
②、再次构造等比数列

​    上一步得到$a_n+αa_{n-1}=β^{n-1}$，等式两边同时除以$β^n$，得到：
$$
\frac{a_n}{β^n}+\frac{α}{β}*\frac{a_{n-1}}{β^{n-1}}=\frac{1}{β}
$$
​    不妨设$c_n=\frac{a_n}{β^n}$，则有：
$$
c_1=\frac{a_1}{β}=\frac{1}{β}
$$

$$
c_n+\frac{α}{β}*c_{n-1}=\frac{1}{β}
$$

​    继续构造：
$$
c_n+λ=-\frac{α}{β}*(c_{n-1}+λ)
$$
​    有等式：
$$
-λ-\frac{α}{β}λ=\frac{1}{β}
$$

$$
λ=\frac{-1}{α+β}
$$
​    求得等比通项：
$$
c_n+λ=(-\frac{α}{β})^{n-1}*(c_1+λ)=(-\frac{α}{β})^{n-1}(\frac{1}{β}-\frac{1}{α+β})=-\frac{(-α)^n}{(α+β)*β^n}
$$
​    即：
$$
\frac{a_n}{β^n}=\frac{1}{α+β}-\frac{(-α)^n}{(α+β)*β^n}
$$

​    得到：
$$
a_n=\frac{β^n-(-a)^n}{α+β}
$$

​    对上一步的解进行分类讨论：

* $当α, β>0, α+β=\sqrt{5}, -α=\frac{1-\sqrt{5}}{2}), a_n=a_n=\frac{1}{\sqrt{5}}((\frac{\sqrt{5}+1}{2})^n-(\frac{1-\sqrt{5}}{2})^n)$

* $当α, β<0, α+β=-\sqrt{5}, -α=\frac{\sqrt{5}+1}{2}, a_n=\frac{1}{\sqrt{5}}((\frac{\sqrt{5}+1}{2})^n-(\frac{1-\sqrt{5}}{2})^n)$



综上所述：

$$
斐波那契数列通项：a_n=\frac{1}{\sqrt{5}}((\frac{\sqrt{5}+1}{2})^n-(\frac{1-\sqrt{5}}{2})^n)
$$

有了公式，代码就简单了。

```c
/* 通项公式直接求解 */
uint64_t fibonacci6(unsigned int n) {
	const double sqrt5 = 2.2360679774997896964091736687313;
	const double a = (sqrt5 + 1) / 2;
	const double b = (1 - sqrt5) / 2;
	const double sqrt1_5 = 1 / sqrt5;

	return (uint64_t)((pow(a, n) - pow(b, n))*sqrt1_5);
}
```

> 该方法依赖于pow函数的复杂度，由于是浮点数的幂运算，所以不能像之前那样优化运算。
>
> 因为[double位64位双精度浮点数](https://en.wikipedia.org/wiki/IEEE_754-1985)，只有52位保证数据精度，所以斐波那契数列项数越大，精度越低，同样的这种方式也会发生溢出。有一个解决方案是使用类似于Java中的BigDecimal的类来代替double。
>
> ![双精度浮点数](http://img-blog.csdn.net/20170807193148407?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 实现方法比较测试

第一种递归算法就不拉进来测试了，笔记本要炸( ╯□╰ )：

```c
/* 计算时间间隔 */
double duration(struct timespec *end, struct timespec *start) {
	double d_sec = difftime(end->tv_sec, start->tv_sec);
	long d_nsec = end->tv_nsec - start->tv_nsec;
	return (d_sec*10e9 + d_nsec);
}

/* 算法测试 */
void compare_and_test() {
	typedef uint64_t(*PFUNC)(unsigned int n);
	PFUNC pFuncs[] = { fibonacci2 ,fibonacci3, fibonacci4, fibonacci5, fibonacci6 };
	struct timespec start, end;
	for (int j = 0; j < sizeof(pFuncs) / sizeof(PFUNC); j++) {
		timespec_get(&start, TIME_UTC);
		// 93项后会发生溢出，这里测试计算时间，不关心溢出问题
		for (int i = 0; i < 95; i++) {
#			ifdef NDEBUG
			(*pFuncs[j])(i);
#			else
			printf("%llu ", (*pFuncs[j])(i));
#			endif
		}
		timespec_get(&end, TIME_UTC);
		printf("\t duration: %lf nanosecond\n", duration(&end, &start));
	}
}
```

运行结果（单位纳秒）：

```shell
duration: 165800.000000 nanosecond
duration: 1653500.000000 nanosecond
duration: 147000.000000 nanosecond
duration: 76000.000000 nanosecond
duration: 134700.000000 nanosecond
```

> 项数越大，矩阵快速幂算法的优势越明显。




> **参考链接：**
>
> 维基百科：https://en.wikipedia.org/wiki/Fibonacci_number