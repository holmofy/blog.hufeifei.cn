# 大数据抽样：蓄水池抽样算法[^1]

这个算法是做一个这样的需求的时候学会的：创建一个任务从1亿用户中随机筛选100个用户发送红包，要求任务每次执行结果保持绝对随机。

转换成数学语言就是：从**N**个数据中随机抽样**m**个数据。N的数量可能非常大，且未知。

在谷歌上找到一篇文章讲的算比较好的：Reservoir Sampling[^2]

## 	最简单的实现——初等数学证明

在[Wikipedia][^1]上看到最简单的是$O(N)$的的复杂度就能解决这个问题，这个算法是Waterman发明的Algorithm R，在[Donald E. Knuth](https://book.douban.com/search/Donald E. Knuth)的《[计算机程序设计艺术（第2卷）](https://book.douban.com/subject/1231891/)》中有提到。下面是翻译成Java的代码实现：

```java
private class Reservoir<T> {
  private final List<T> values;
  private int m;
  
  Reservoir(int m) {
    values = new ArrayList<>(m);
    this.m = m;
  }

  void sampling(Iterable<T> data) {
    int n = 0;
    for(T item : data){
      if(n < m){
        values.add(item);
      } else {
        int r = RandomUtils.nextInt(0, n);
        if (r < m) {
          values.set(r, data);
        }
      }
    }
  }
}

```

刚开始看到这个算法，对它能否保证绝对随机我是怀疑的，仔细证明后发现算法是OK的，下面是数学证明：

* 当$n>m$时，第n个元素选中的概率是$\frac{m}{n}$；

但后一次被保留的概率=被选中的概率×（后一个元素不被选中的概率+后一个元素被选中*不被替换的概率）。

第$n+1$个元素不被选中的概率是
$$
1-\frac{m}{n+1}=\frac{n+1-m}{n+1}
$$


第$n+1$个元素被选中，但是n个元素不被替换的概率是
$$
\frac{m}{n+1}*(1-\frac{1}{m})=\frac{m}{n+1}*\frac{m-1}{m}=\frac{m-1}{n+1}
$$
所以最终被保留的概率是：
$$
\begin{align}
P(n)&=\frac{m}{n}*(\frac{n+1-m}{n+1}+\frac{m-1}{n+1})*(\frac{n+2-m}{n+2}+\frac{m-1}{n+2})*...*(\frac{N-m}{N}+\frac{m-1}{N})\\
&=\frac{m}{n}*\frac{n}{n+1}*\frac{n+1}{n+2}*...*\frac{N-1}{N}\\
&=\frac{m}{N}
\end{align}
$$

* 当$n<=m$时，元素被选中的概率是$1$；

当第$x$个元素进来的时候$(m \lt x \leqslant N)$，有可能把这个元素给挤掉。

这时元素被保留的概率=第$x$个元素不被选中的概率+第$x$个元素被选中的概率*不被替换的概率）：

第$x$个元素不被选中的概率是
$$
1-\frac{m}{x}=\frac{x-m}{x}
$$


第$x$个元素被选中，但是元素不被替换的概率是
$$
\frac{m}{x}*(1-\frac{1}{m})=\frac{m}{x}*\frac{m-1}{m}=\frac{m-1}{x}
$$


元素不被第$x$元素替换的概率是：
$$
\frac{x-m}{x}+\frac{m-1}{x}=\frac{x-1}{x}
$$


这个元素最终被保留的概率是不被第$m+1$个元素到$N$个元素替换的概率：
$$
\begin{align}
P(n)&=\frac{m}{m+1}*\frac{m+1}{m+2}*...*\frac{N-1}{N}\\
&=\frac{m}{N}
\end{align}
$$
<!--

## 最优化的实现——高等数学证明

还有一个最优化的算法，时间复杂度是$O(k(1+log(\frac{N}{k})))$，主要的原理是**随机跳过中间的值**，避免每次都取一次随机数。

-->







[^1]: https://en.wikipedia.org/wiki/Reservoir_sampling 抽样算法
[^2]: https://steadbytes.com/blog/reservoir-sampling/ Reservoir Sampling
[^3]: https://richardstartin.github.io/posts/reservoir-sampling reservoir sampling
[^4]: https://github.com/gstamatelat/random-sampling github代码实现
