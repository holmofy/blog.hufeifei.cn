---
title: BST,AVL,Red-BlackTree,SplayTree
date: 2018-03-28 18:31
categories: 数据结构与算法
mathjax: true
tags: 
- DataStructure
- BST
- AVL
- RB-Tree
---

# 从线性查找和二分查找说起

**线性查找**是最基础(野蛮)的查找算法，最坏的情况从头遍历到位，最好的情况比较一次，**平均时间复杂度为$\frac{N}{2}$**。

![线性查找](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiiyfojcg20c60503yp.gif)

**二分查找能达到$O(log_2N)$的时间复杂度，但是前提是列表中的数据必须是有序的**。

不管是基于数组实现的列表(ArrayList)还是基于链表实现的列表(LinkedList)，想要在插入新元素的同时保证列表元素的有序性，插入操作仍然需要大量的时间：

* 数组插入时保证元素有序性，时间复杂度为N（一部分用于查找元素插入的位置，一部分用于数组后移）。
* 链表插入时保证元素有序性，平均时间复杂度为$\frac{N}{2}$，(最好的情况比较一次即可插入，最坏的情况遍历全表)

> 用大O表示法，两种存储结构都为O(N)。
>
> 那有没有什么方法能实现插入和查找的时间复杂度都能低于O(N)呢。二叉搜索树就是来解决这个问题的。

# 二叉搜索树(BST)

> 二叉搜索树(Binary Search Tree)也被称为二分查找树。

二叉树是叶子节点不超过两个的特殊树结构，**二叉搜索树则是在二叉树的基础上加了一些规则**：

* 左子树的值都小于父节点
* 右子树的值都大于父节点

插入的时候和父节点比较一下，就知道应该插入左子树还是右子树了：

![BST](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbijpr9vhg20iu05nadf.gif)

查找的时候和父节点比较一下，就知道应该从左子树里面找还是从右子树里面找了。

插入和查找都能达到$O(log_2N)$级别的时间复杂度。

二叉搜索树看似无懈可击，但是仍有致命的**缺点**：**可能退化成链表**

![BST退化成链表](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbikmqc8fg20g109udmr.gif)

即使没有退化成严格意义上的链表，左右子树不平衡，也仍然难以达到时间复杂度为$O(log_2N)$的理想状态。

# 自平衡二叉搜索树

为了防止二叉搜索树退化，我们需要保证左右子树的平衡。

根据不同程度的平衡需求，有不同的平衡算法实现，如AVL树，红黑树(Red-Black Tree)，伸展树(Splay Tree)等。

而这些实现了左右子树平衡的也被称为**自平衡二叉搜索树(Self-balancing binary search tree)**

> 我们先从最古老的AVL树开始。

## 1. AVL树

AVL树是**A**delson - **V**elsky和**L**andis在1962年的论文中发表的。

AVL树在二叉搜索树的基础上再增加了一个限制：**左右子树的高度差不能大于一**。

这个左右子树的高度差被定义为**平衡因子(Balance Factor)**：

对于任意节点N要求：
$$
balanceFactor(N)=height(rightSubtree(N)) - height(leftSubtree(N))；
$$

$$
其中balanceFactor(N) \in \left \{ -1,0,1 \right \}
$$

当左右子树不平衡时，就会发生左旋转(Left Rotate)、右旋转(Right Rotate)或先左旋再右旋(rotateLeftThenRight)、先右旋再左旋(rotateRightThenLeft)，具体是左是右就需要看平衡因子的正负了。

> 下图是一张AVL树左右旋转的例子

![AVL树旋转的例子](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbilbbimyg208w050dm1.gif)



> 我这里也录了两张稍微慢一点的动画

![AVL树旋转](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbilvjygog20j809i47q.gif)

![AVL树旋转](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbimdctf8g20j809i47q.gif)

> [这个链接](http://rosettacode.org/wiki/AVL_tree)中有AVL树在各语言中的实现

**AVL的缺点**：因为要保证左右子树的严格平衡，插入和删除操作可能需要经过一个或多个旋转。

**在插入删除操作频繁的时候，效率相对比较低下**；查找操作密集的场景下，AVL树就比较适用。

## 2. 红黑树

红黑树是AVL树的“晚辈”，它的整体性能也比AVL树要好。因为红黑树并不像AVL树那样要求树在整体严格平衡，红黑树自己制定了一个规则能实现树的**大致平衡**：

* 每个节点要么是红色要么是黑色
* 根节点是黑色
* 所有叶子(NULL LEAF,空节点)都是黑色的
* 如果一个节点是红色的，那么它的两个子节点都是黑色的
* 从给定节点到其后代任何一个NIL节点的路径都包含相同数量的黑色节点

这些特性强化了红黑树的一个关键属性：**从根到最远的叶子节点的路径不超过从根到最近叶子节点路径的两倍**，结果是树**大致上高度平衡**。

> 对于一颗红黑树Tree，根据第五条特性设B为路径中黑色节点的数量且为最短路径，根据第四条特性不可能有多个连续的红色节点，也就是最长路径只能由红色和黑色交替出现，即最长路径为2B。

![红黑树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbin2lperj20mb0armxz.jpg)

红黑树通过左旋、右旋实现平衡：

![左旋右旋](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbinjj5olg206y06ywlt.gif)

这里有两张红黑树插入过程中进行旋转整个过程的动态图：

![带叶子节点的红黑树示例](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbinym1hwg20nu090tqq.gif)

![不带叶子节点的红黑树示例](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbioc68s0g20nu090qeu.gif)

> 因为红黑树在整体上性能较佳，所以在各大编程语言中都有它的身影，如C++[标准模板库中的`std::map`](http://en.cppreference.com/w/cpp/container/map)，Java[集合中的`java.util.TreeMap`](https://docs.oracle.com/javase/8/docs/api/java/util/TreeMap.html)。

## 3. 伸展树

伸展树（Splay Tree）是一种特殊的自平衡二叉搜索树，它基于一种简单的认识：**最近访问的元素很快再次访问**。

所以它在插入或访问一个元素后，会**把这个最近使用的元素通过一次或多次旋转最终放到根节点的位置**，以便下次访问。

![伸展树](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbiovv6jag20o909vtpc.gif)

> 在某些特性上SplayTree和[LruCache](https://en.wikipedia.org/wiki/Cache_replacement_policies#LRU)有点相似。猜的没错，SplayTree比较适合用来做缓存或GC。

伸展树的优点就是不需要存储任何附加信息(AVL需要存储平衡因子，红黑树需要存储节点的红黑属性)，它的平均性能也能达到$O(log_2N)$。但是它的缺点也很明显，顺序插入或访问元素将会导致二叉树退化成链表。

![伸展树顺序插入元素](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbipdxb61g20k90cm7cl.gif)

![伸展树顺序查找元素](http://tva1.sinaimg.cn/large/bda5cd74gy1fqbipuofdug20n90cw1ca.gif)

> [这个链接](http://www.link.cs.cmu.edu/link/ftp-site/splaying/)是CMU大学CS专业提供的伸展树实现。



参考链接：

https://en.wikipedia.org/wiki/Red–black_tree

https://en.wikipedia.org/wiki/Splay_tree
