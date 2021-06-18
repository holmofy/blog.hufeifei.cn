---
title: B树与B+树
date: 2018-04-2 18:31
categories: 数据结构与算法
mathjax: true
tags: 
- B-Tree
- DataStructure
---


# 从二叉搜索树说起

其实[上一篇文章](https://blog.csdn.net/holmofy/article/details/79692613)已经对BST进行过讨论，并对AVL，红黑树这样的自平衡二叉查找树分别解决了什么问题进行了讨论。

![BST](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiag1z25g20iu05nadf.gif)

上面这些数据结构理论上能达到$O(log_2N)$的平均时间复杂度。

这个时间复杂度是基于对内存的操作而计算出来的。倘若我们的数据量十分庞大，内存无法容纳，我们不得不存储在硬盘中。这个时候二叉搜索树还能达到预期的速度吗？

在回答这个问题之前先让我们看看磁盘的存储原理。

# 磁盘的存储原理

我们知道传统的机械硬盘(不考虑SSD)的磁盘通过电磁性来保存数据的，磁头在高速旋转的磁盘上扫描寻道来读取或改变特定位置的磁性，从而实现数据的读写。

![硬盘](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbibmlhotj208c08caa3.jpg)

上图中间的那个原型磁盘被分为被分成若干个扇形区域，这个区域被称为“[扇区(sector)](https://en.wikipedia.org/wiki/Disk_sector)”，**硬盘中每个扇区大小固定位512个字节，新型磁盘一个扇区4KB**。

![扇区](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbicdwfx1j20i40eo7a7.jpg)

为了降低磁头来回寻道的时间，通常我们建议程序员在**执行IO操作时，使用连续顺序读写而不是随机读写**。

> 理解顺序IO与随机IO：https://flashdba.com/2013/04/15/understanding-io-random-vs-sequential/

磁盘的性质总结就是：

1、吞吐量大：以扇区为基本读写单位；

2、速度慢：机械式寻址，寻道依赖机械臂，寻扇区依赖磁盘的旋转；

3、顺序IO显著快于随机IO：不连续的扇区，需要更长的寻址时间。

# 二叉搜索树的磁盘IO

树这种数据结构每个节点空间一般都是临时分配的，也就是说**每个节点存储的物理位置都是随机的**。

![BST磁盘IO问题](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbid9cts2g209107paaj.gif)

**那么每次访问一个节点都可能造成一次随机的磁盘IO**

当树的深度越大，随机的磁盘IO次数越多，这将会严重降低二叉搜索树的效率。

> **为了降低树的深度，B树被发明出来了。**

# B树

B树也是一种自平衡的树状数据结构，它通常用作文件系统、数据库的索引结构。

> B树是Rudolf Bayer和Ed McCreight在1971年发明的，但他们并没有解释字母B所代表的的含义。B可能代表平衡(balanced)、浓密(bushy)，也可能是他的名字Bayer或者他们所在的波音(Boeing)实验室。
>
> 因为国外的文献中把B树写做"[B-tree](https://en.wikipedia.org/wiki/B-tree)"，国内的翻译直接翻译成"B-树"，有人可能会误认为它们是两种数据结构(万恶的翻译)，但实际上它们指的都是同一种数据结构。

**B树中的每个节点可以有两个以上的子节点，每个节点最多可以有m个孩子，其中m被称为B树的阶(order)**

**每个节点最多可以存储m-1个用于比较的数据(key)**

为了保持B树的平衡，B树还有以下几个特性：

1.根结点至少有两个子节点。

2.每个中间节点(根节点和叶子节点除外)都包含k-1个元素和k个孩子节点，其中$\frac{m}{2} <= k <= m$

3.每一个叶子节点都包含k-1个元素，其中$\frac{m}{2} <= k <= m$

## 2-3Tree

当m等于3的时候，每个节点可以有两个或三个子节点，所以也被叫做[2-3Tree](https://en.wikipedia.org/wiki/2%E2%80%933_tree)。

> 2-3Tree是John Hopcroft在1970发明的。B树实际上就是对2-3Tree的泛化。

下图就是2-3Tree的存储结构。

![2-3Tree](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbieoftqnj20cl04s74j.jpg)

插入数据时，当节点中的数据达到3个，就会发生分裂：中间的值将会升级成父节点，比中间值小的将会成为左子节点，比中间值大的将会成为右子节点。

![2-3Tree插入操作](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiffgiumg20e1065asg.gif)

> 同样的当m等于4的时候，就有[2-3-4Tree](https://en.wikipedia.org/wiki/2%E2%80%933%E2%80%934_tree)

## 高阶B树

当m越来越大的时候，B树就成了一个“又胖又矮”的小胖子了。这棵矮胖的树每个节点都存储了很多数据，每次取其中一个节点并使用二分查找，就能立马知道下一个节点的位置了，节点数量以及树的层数的降低，使得I/O次数随之减少。

![高阶B树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbig4c8olj20n306f3yr.jpg)

那是不是B树的阶数越高越好呢？

B树的节点是一个数据块，因为节点块子节点数k满足$\frac{m}{2} <= k <= m$，所以最坏的情况会浪费块内一半的空间，**阶数m越大意味着空间浪费也越大**。这个需要应用程序在空间和时间上进行权衡，一般将块大小设计成磁盘块的大小的整数倍，因为磁盘读写是以扇区作为基本单位（磁盘一个扇区512字节，新型磁盘一个扇区有4K）。比如[InnoDB的默认页面大小](https://dev.mysql.com/doc/refman/5.7/en/innodb-init-startup-configuration.html#innodb-startup-page-size)为16K。

```c
/* 定义key的类型,根据具体需要比较的类型定义 */
# define KEY ...
/* 定义value的类型,根据具体要存储的数据定义 */
# define VALUE ...
/* M为阶数，M大于等于3*/
# define M 10

typedef struct {
    KEY key[M-1];
    VALUE value[M-1];
    /*指向子节点的指针*/
    BTNode* BTptr[M];
} BTNode;
```

对于一个$m=500\approx2^{9}$的4K磁盘块来说，1层树(根节点作为0层)能存储$2^{9}\times4K=2M$的数据，2层树能存储$2^{18}\times4K=1G$的数据，3层树能存储$2^{27}\times4K=512G$的数据，4层树能存储$2^{36}\times4K=256T$的数据。

> 关于B树的阶数该如何选择可以参考https://stackoverflow.com/questions/28677734/how-to-decide-order-of-b-tree

# B树的变体B+树

通常B树的节点中同时存储了Key和Value，更准确的说应该是**Value的引用(存储数据的物理地址)**。

我们比较的时候只需要Key而已，但是却**把Value也取了出来，从而造成了不必要的I/O**。虽然只是取出了value的引用，但是当数据库数据量越来越大的时候，对性能的影响也会随之累积而变得十分可观。

另外，B树进行范围查询时需要[回溯](https://en.wikipedia.org/wiki/Backtracking)，对于硬盘中的数据结构而言，一次回溯意味着一次随机IO。

![回溯](http://tva1.sinaimg.cn/large/bda5cd74ly1fyy0cmozmkj20d30cx74u.jpg)

为了解决这些问题，[B+树](https://en.wikipedia.org/wiki/B%2B_tree)被发明出来了。


B+树中有两个小规则：

**1、内部的非叶子节点只存储Key，用来索引，不保存Value数据，所有的Value数据都保存在叶子节点中**

> 非叶子节点只存储Key，对于同样大小的节点来说，可以**让节点存储更多的Key**，这变相的降低了树的深度，从而让B树索引更多数据。同时**对内存的索引缓存更友好**，内存中就能缓存更多层级的数据了。所有的查找最终都会到叶子节点，从而**保证了查询性能的稳定**。

**2、叶子节点之间用指针串联起来**

> 所有叶子节点形成有序链表，**便于范围查询**。

![B+树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbigvay08j20ns05o0sw.jpg)

因为它的内部节点只存储Key，所以它的分裂方式与B树也略有不同。

![B+树的分裂](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbihhh74hg20j605z17a.gif)

> 由于B+树的这些优秀特性，各大数据库的索引都是基于B+实现的。比如[MySQL](https://dev.mysql.com/doc/refman/5.7/en/create-index.html#create-index-storage-engine-index-types)，[Oracle](https://docs.oracle.com/cloud/latest/db112/CNCPT/indexiot.htm#CNCPT1170)等。



参考链接：

https://www.cs.cornell.edu/courses/cs3110/2012sp/recitations/rec25-B-trees/rec25.html

http://cis.stvincent.edu/html/tutorials/swd/btree/btree.html

https://cstack.github.io/db_tutorial/parts/part7.html

https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/
