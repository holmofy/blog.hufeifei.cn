---
title: 关系型数据库是如何工作的
date: 2021-03-24 03:25
categories: 数据库
---



> 本文翻译自：http://coding-geek.com/how-databases-work/

当谈及到关系型数据库时，我不禁会以为有些东西被遗忘了。它们无处不在。有各式各样的数据库：从麻雀虽小五脏俱全的 [SQLite](https://www.sqlite.org/index.html) 到强大的[Teradata](https://www.teradata.com/Products/Software/Database)。但是，鲜有文章解释数据库是如何工作的。你可以自己google下“how does a relational database work”看看有多少搜索结果。而且，那些文章还都很简短。如果你现在再找找看最时髦的技术（大数据，NoSQL 或者 JavaScript），你会发现一大堆有深度的文章来解释它们是如何工作的。

难道是关系数据库太老太无聊了，以至于无法在大学课程、研究论文和书籍之外进行讲解？

![database](https://p.pstatp.com/origin/pgc-image/ec1414c44d9443a58d5a3ed8d8e68d5e)

作为一名开发者，我**讨厌**使用那些我不理解原理的技术。并且，数据库已经使用了 40 年了，一定有什么原因才对。多年来，我花费了几百个小时，去真正理解那些我每天都在用的奇怪的黑箱。**关系数据库很有趣，因为它们建立在经久不衰的概念之上**。如果你想了解数据库，但是却没有时间或者没有精力去研究这个宏大的主题，那么，你应该会喜欢这篇文章的。

这篇文章的题目已经明确说明，文章的**目的并不是理解如何使用数据库**。因此，**你应该已经知道如何写出简单的join查询语句以及基本的 CRUD语句**；否则，你可能看不懂这篇文章。这是你开始读这篇文章之前唯一需要知道的事情，其余的内容将由我来解释。

我会从一些计算机科学的概念开始讲起，比如时间复杂性。我知道有些人讨厌这些概念，但是，没有它，你就不能理解数据库内的那些聪明之处。由于这是一个很宏大的主题，我将**重点介绍**我认为必不可少的内容：**数据库处理SQL查询的方式**。我只介绍**数据库背后的基本概念**，在文章结束的时候，你就可以对数据库中幕后做的事儿有一个很好的了解。

这是一篇涉及大量算法和数据结构的冗长的技术性文章，因此请多花一些时间阅读它。有些概念难以理解可以暂时跳过，仍然可以理解整体脉络。

对于已经有一定基础的读者，这篇文章可以大致分成三个部分：

- 底层和高层的数据库组件概述
- 查询优化过程概述
- 事务和缓冲池管理概述

[content]

# 1、回归本源

很久以前（在一个遥远的星系中……），开发人员必须确切地知道他们正在编码的操作数。他们对自己的算法和数据结构了然于胸，因为他们负担不起在他们超慢的计算机上浪费CPU和内存。

在本节，我会提醒你一些概念。这些概念对于理解数据库尤为重要。我还会引入一个术语：**数据库索引（database index）**

## 1.2、O(1) vs $O(n^2)$

现在，很多开发者根本不关心时间复杂度。在一定程度上说，他们是正确的。

但是，当你处理大量数据（我说的不止几千）或者你必须要提高几毫秒时，就不得不理解这个概念了。猜的没错，这两个情形也都是数据库必须考虑的！我不会烦你很久，只要给我点时间来解释这个概念。这将有助于我们将来理解基于成本的优化（cost based optimization）这个概念。

### 1.2.1、概念

时间复杂度用来度量一个算法处理给定数据量的执行时间。为了描述复杂度，计算机科学家设计了大O表示法。大O表示法用一个函数来描述算法对于给定数据量需要多少操作。

举个例子，当我说“这个算法时O(f(n))”时，意思是对于一定数量的数据n，这个算法需要f(n)个操作来完成这项工作。

重点不是数据量，而是当数据量增加时，操作数量时如何增加的。时间复杂度并不会给一个具体的操作数量，这对于描述这个关系不失为一个好方法。

![time complexity](https://p.pstatp.com/origin/pgc-image/643138456eb64682a2a91dc649effcc4)

在上面这张图中，您可以看到不同类型的复杂性的演变。我用对数刻度绘制它。换句话说，数据数量正迅速从1亿增加到10亿。我们可以看到：

* O(1)即常数复杂度一直保持不变。（否则它也不会叫做常数复杂度）
* O(log(n))即使在10亿数据量时也保持在比较低的位置。
* 最糟糕的是$O(n^2)$，它的操作数在飞快地暴增。
* 其他两个复杂度也在快速增长。

### 1.2.2、例子

数据量少时，O(1)和$O(n^2)$之间的差异可以忽略不计。例如，你有一个算法需要处理2000个数据：
* O(1)算法只需要1此操作
* O(log(n))算法需要7次操作
* O(n)需要2000次操作
* O(nlog(n))需要14000次操作
* $O(n^2)$需要4000000次操作

O(1)和$O(n^2)$之间差异似乎很大(400万)，但是最多也就损失2毫秒，只是眨眼的功夫。实际上，当前的处理器每秒可以处理[数亿个操作](https://en.wikipedia.org/wiki/Instructions_per_second)。这也是为什么很多IT项目中性能和优化都不是什么问题。

> 译者注：大多数项目代码写的再烂，只要符合逻辑，性能都不会有特别的影响，大多都是内存操作，数据量再大也不会超过内存的限制。反而是一条SQL写的不好导致数据库性能出现问题，整个系统被拖慢。当然我不是怂恿人写烂代码，相反代码应该要写好，应该让人能容易看懂，不要为了丁点儿性能而影响可读性。我曾经也是个极致的扣性能的开发者，但是软件工程是一个多人合作的工作，对于大多数项目代码容易让人看懂才是好代码。

正如我所说，面对大数据量时，了解这个概念仍然是很重要的。如果这次算法需要处理100 0000个元素(对于数据库来说这都不算大)

* O(1)算法需要1次操作
* O(log(n))算法需要14次操作
* O(n)算法需要100 0000次操作
* O(nlog(n))算法需要1400 0000次操作
* $O(n^2)$算法需要1 0000 0000 0000次操作

我不需要去真正算时间，但我可以这么说：使用$O(n^2)$算法，您有时间去喝一杯咖啡了，如果把数据量再加一个0，你都有时间去睡上一觉了。

### 1.2.3、再深入一些

给你一些场景：

* 好的hash表进行检索需要O(1)时间复杂度
* 平衡树的检索需要O(log(n))时间复杂度
* 数组的检索需要O(n)时间复杂度
* 最好的排序算法需要O(nlog(n))时间复杂度
* 不好的排序算法需要$O(n^2)$时间复杂度

注意：在下一部分，我们将看到这些算法和数据结构。

时间复杂度有多种类型：

- 平均情况
- 最好情况
- 最坏情况

时间复杂度通常按照最坏情况。

我只讨论了时间复杂度，但是复杂度的概念同样适用于：

- 算法的内存消耗
- 算法的磁盘 I/O 消耗

当然，复杂度比$n^2$还差的，例如：

* $n^4$：太糟糕了，我要提到的某些算法就是这个复杂度。
* $3^n$：太烂了，我们将在本文中看到一种算法具有这种复杂度（这种算法在很多数据库中都有在用）。
* $n^n$：如果你的算法是这个复杂度的，你应该问问自己，IT 是不是真的适合你……

注意：我并没有给大 O 记号的准确定义，只给出了一个思想。你可以在[维基百科](https://en.wikipedia.org/wiki/Big_O_notation)阅读真正的定义。

## 1.3、归并排序

如果你要对一个集合进行排序，你要怎么做？什么？你可以调用`sort()`函数……好主意……但是，对于数据库，你必须知道`sort()`函数是怎么工作的。

有很多很好的排序算法，我们会将精力放在最重要的那个上面：**归并排序**。现在你可能并不明白为什么对数据进行排序很有用，但是，当你读完查询优化部分后，你就应该能够回答这个问题。另外，了解合并排序可以帮助我们弄清楚一个常见数据库join操作：**Merge Join**。

### 1.3.1、Merge

和很多有用的算法一样，合并排序也是基于一个小技巧：将两个已经排好序的长度为 N/2 的数组合并为一个排好序的长度为 N 的数组只需要 N 次操作。这个操作成为**合并**。

我们用一个简单的例子来看这是什么意思：

![Merge](https://p.pstatp.com/origin/pgc-image/757c7bce5a8e4e7985042786098d1e5b)

上图最终获得一个排好序的具有 8 个元素的数组，你只需要遍历两个 4 元素的数组一次。由于两个 4 元素的数组都已经排好序了：

- 1) 比较两个数组的当前元素（第一次比较时，current=1）
- 2) 取出二者之间较小的放入 8 元素数组
- 3) 取出上一步较小元素所在的那个数组的下一个元素
- 重复上述三步，直到取出其中一个数组的最后一个元素
- 将另外一个数组的剩余元素全部追加到 8 元素数组中

由于 4 元素数组已经排好序，因此上述步骤可以正常工作。你不需要在这些数组中“回退”。

现在我们已经知道了思路，下面是归并排序的伪代码：

```python
array mergeSort(array a)
   if(length(a)==1)
      return a[0];
   end if
 
   // 递归调用
   [left_array right_array] := split_into_2_equally_sized_arrays(a);
   array new_left_array := mergeSort(left_array);
   array new_right_array := mergeSort(right_array);
 
   // 将两个已经排好序的较短的数组合并到一个较长的数组中
   array result := merge(new_left_array,new_right_array);
   return result;
```

合并排序将一个较大的问题分解成多个较小的问题，找到较小问题的解，从而获得原始问题的解（注意：这种类型的算法被称为“分治法”）。如果你不明白这个算法，不要担心，我第一次看到它也不明白。为了帮助理解，我将这个算法看作一个具有两个步骤的算法：

- 分割阶段：将数组分成较小的数组
- 排序阶段：将小数组放在一起（使用合并）形成大数组

### 1.3.2、分割阶段

![分隔阶段](https://ae04.alicdn.com/kf/U34d4c47b13274af299af91d77e1f86b1M.jpg)

在分割阶段，使用3步，将原始数组分成只有一个元素的单元数组。这一步骤的是 log(N)（本例中，N=8，log(N) = 3）。

我是怎么知道这个的？

~~我是个天才！~~简而言之：数学。这一阶段的思路是，每一步将原数组的长度分成二分之一。所需要的步数就是你能够将原始数组一分为二的次数。这是对数更准确的定义（以 2 为底）。

### 1.3.3、排序阶段

![排序阶段](https://p.pstatp.com/origin/pgc-image/a3bdb11d927b493eac57c1a4fed77b86)

在排序阶段，你从单元数组开始。每一步都要应用多次合并，总的合并是 N=8 次：

- 第一步，你需要 4 次合并，每一次合并需要 2 个操作
- 第二步，你需要 2 次合并，每一次合并需要 4 个操作
- 第三步，你需要 1 次合并，需要 8 个操作

因为一共有 log(N) 步，所以，**总成本是$N * log(N)$次**。

### 1.3.4、归并排序的威力

为什么这个算法这么有用？

原因是：

- 你可以修改该算法，从而减少内存消耗。所需方法是，不要重新创建新的数组，而是直接修改输入的数组。

  注意：这种算法被称为[原地算法（in-place）](https://en.wikipedia.org/wiki/In-place_algorithm)。

- 你可以修改该算法，使用磁盘空间和少量内存，而不会大量的磁盘 I/O 消耗。其思路是，只在内存中加载当前正在处理的部分。当你需要对几个 G 的数据表进行排序，但内存缓存区只有 100M 的时候，这是一种很重要的方法。

  注意：这种算法被称为[外部排序（external sorting）](https://en.wikipedia.org/wiki/External_sorting)。

- 你可以修改该算法，以便在多个处理器/线程/服务器中运行。

  例如，分布式归并排序是 [Hadoop](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/Reducer.html)的核心组件之一（Hadoop 是处理大数据的重要框架之一）。

- 这个算法可以变成真金白银（真事儿！）。

这种排序算法被绝大多数（即便不能说全部）数据库采用，但它也并不是唯一的排序算法。如果你想知道得更多，你可以阅读这篇[研究论文](http://wwwlgis.informatik.uni-kl.de/archiv/wwwdvs.informatik.uni-kl.de/courses/DBSREAL/SS2005/Vorlesungsunterlagen/Implementing_Sorting.pdf)。该论文讨论了数据库中使用的各种通用排序算法的优缺点。

> 译者注：[这里](https://github.com/holmofy/algorithm/tree/master/Sort)有我写过的十来种排序算法的Java实现，[这里](https://github.com/holmofy/algorithm/blob/master/QuickSort/QuickSort.java)是我对快速排序的多种优化策略。

## 1.4、数组，树和哈希表

现在我们已经明白了时间复杂度和排序背后的思想，接下来我要告诉你3种数据结构。这些数据结构都很重要，因为**它们是现代数据库的骨架**。我还会介绍**数据库索引**这一概念。

### 1.4.1、数组

二维数组是最简单的数据结构。一张表可以被当成是一个数组。例如：

![Array](https://p.pstatp.com/origin/pgc-image/7412c07bca194f17ac3a3d62fb3764ee)

这个二维数组就是带有行和列的表：

- 每一行表示一个对象
- 每一列描述对象的特征
- 每一列存储一种特定的数据类型（integer，string，date...）

虽然这种数据结构非常适合存储和数据可视化，但是当你需要查找一个特定值时，这种结构就玩不转了。

例如，**如果你想找到所有在英国工作的人**，你需要遍历每一行，去看看这一行是不是属于英国。**这将花费您N次操作**（N 是行数），这看起来还不算很坏，但是有更快的吗？这就是引入树的目的。

注意：大多数现代数据库为了更高效地存储表提供了更高级的数组，例如堆组织表（heap-organized tables）和索引组织表（index-organized tables）。但是，这些并不能改变快速检索符合特定条件的列组的问题。

### 1.4.2、树和数据库索引

二叉搜索树是带有特殊属性的二叉树，其每个节点的键必须满足：

- 大于所有左子树的key
- 小于所有右子树的key

下面我们来看看这是什么意思。

#### 二叉树的思想

![二叉搜索树](https://p.pstatp.com/origin/pgc-image/de03beab76544fc29e8e2da6313a4a96)

这棵树有 N=15 个元素。假如说我要查找 208：

- 我从根节点开始找起，根节点的键是 136。由于 136<208，那么，我需要查找节点 136 的右子树
- 398>208，那么，我需要查找节点 398 的左子树
- 250>208，那么，我需要查找节点 250 的左子树
- 200<208，那么，我需要查找节点 200 的右子树。但是 200 没有右子树，因此，**该值不存在**（因为如果它存在，它应该在 200 的右子树）

现在，我要找 40：

- 我从根节点开始找起，根节点的键是 136。由于 136>40，那么，我需要查找节点 136 的左子树
- 80>40，那么，我需要查找节点 80 的左子树
- 40=40，**节点存在**。我取出该节点中保存的行号RowID（这一属性没有在上图展示出来）以及RowID对应的表。
- 知道了行号，也就知道了数据在表中的确切位置，因此我就能直接取出。

最后，这两个检索都只耗费了树的层数那么多的操作。如果你认真阅读了合并排序的部分，你应该发现这棵树其实是有 log(N) 层。所以，**检索消耗是 log(N)**，还不错！

#### 回到我们刚才的问题

这些解释都有点抽象，让我们回到我们的问题。换掉刚才简单的整数，我们想想前面的表格中那些代表某人所在国家的字符串。假如你有一棵包含了表格中“国家”一列的树：

- 假设你想知道谁在英国工作
- 查找树，获得表示英国的节点
- 在那些“英国节点”中，你可以找到这些在英国工作的人所在行的位置

这个检索只会消耗 log(N) 个操作，而直接使用数组则需要 N 个操作。这个你想象中的东西就是**数据库索引（database index）。**

你可以为列的任意组合构造树索引（字符串、整型、2 个字符串、一个整型和一个字符串、日期等等），只要你能够有一个可以比较键值（也就是这些列的组合）的函数。该函数可以在**这些键值之间建立顺序**（数据库中的基本类型就是这种情况）。

#### B+树索引

虽然二叉搜索树适合于查找特定值，但是当你需要**获取两个值之间的多个元素**时，会有一个很大的问题。因为你必须检查树中的所有节点，看它是不是落在两个值之间（例如，使用树的中序遍历），二叉搜索树的消耗会达到 O(N)。更要命的是，由于你必须读取整棵树，因此这一操作不是磁盘 I/O 友好的。我们需要找到一个执行**范围查询**的有效方法。为解决这一问题，现代数据库使用了一种改进了的二叉搜索树，被称为 B+ 树。在 B+ 树中：

- 只有最低的节点（叶子节点）**保存信息**（相关表中的行的位置）
- 其它节点仅仅用来**在检索中路由**到正确的节点。

![B+ index](https://p.pstatp.com/origin/pgc-image/f506b01e81f2440f9fb16d4486eada68)

正如上图所示，B+ 树的节点更多（比二叉搜索树多出一倍）。事实上，你有一些额外的节点，这些“决策节点”会帮助你找到正确的节点（也就是保存相关表格行的位置的节点）。然而检索复杂度依然是 O(log(N))（仅仅多了一层）。最大的区别是，**最低层节点与它们的后继节点连接起来。**

使用 B+ 树，假设你正在查找 40 和 100 之间的值：

- 你需要像前面的树那样找到 40（如果 40 不存在的话，就选择大于 40 的最小值）
- 然后通过 40 的后继节点找到 100

加入你找到了 M 个后继节点，而树有 N 个节点。检索特定值就像前面的二叉搜索树一样耗费 log(N)；一旦你找到了这个节点，通过它们的后继节点，你会在 M 个操作中找到 M 个后继。**该检索仅消耗 M + log(N)** 个操作，而前面的二叉搜索树则需要 N 个操作。另外，你不需要一次读取整棵树（只需要 M + log(N) 个节点），这意味着更少的磁盘使用。如果 M 很小（比如 200 行）而 N 很大（1 000 000 行），那就会有很大不同了。

但是，还有一些新的问题（又来！）。如果你向数据库新增或删除一行（因此也必须修改关联的 B+ 树索引）：

- 你必须保持 B+ 树中的节点顺序，否则以后就不能在树中查找节点了
- 你必须尽可能减少 B+ 树的层数，否则时间复杂度就会从 O(log(N)) 变成 O(N)

换句话说，B+ 树必须是自排序的，并且是自平衡的。幸运的是，还真的有智能删除、插入操作。但是这也会导致一个消耗：B+ 树的插入和删除操作都是 O(log(N)) 的。这就是为什么你经常听到这样的话：**过多地使用索引不是一个好主意**。事实上，由于数据库需要更新表索引，而每一个索引的修改都是 O(log(N)) 的，因此，**你正在拖慢原本很快的行插入、更新和删除操作。\**另外，添加索引意味着会给\**事务管理器**带来更多工作负荷（我们会在文章的最后认识这个管理器）。

更多细节可以阅读百科[有关 B+ 树的文章](https://en.wikipedia.org/wiki/B%2B_tree)。如果你想要一个数据库中 B+ 树实现的例子，可以阅读 MySQL 的一位核心开发者的[这篇文章](http://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/)和[这篇文章](http://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)。这两篇文章都关注于 innoDB（MySQL 的引擎之一）的索引处理。

注意：有位读者曾告诉我，考虑到数据库需要进行低层优化，B+ 树必须是完全平衡的。

#### 哈希表

我们最后一个重要的数据结构是哈希表。当你需要快速查找值时，哈希表是非常有帮助的。另外，了解哈希表可以帮助我们理解一个通用的数据库连接操作，被称为**哈希连接（hash join）**。这种数据结构还被用于存储一些内部信息（比如**锁表**或**缓冲池**，我们会在后面见到这些概念）。

哈希表是一种使用键快速查找元素的数据结构。构建哈希表需要定义：

- 元素的**键**。
- 键的**哈希函数**。键通过哈希函数计算出来的哈希值给出了元素位置（称为**桶**）。
- **用于比较键的函数**。一旦找到正确的桶，你需要使用这个比较函数在这个桶中找到你要查找的元素。

##### 一个简单的例子

我们看一个可视化的例子：

![hash table](https://p.pstatp.com/origin/pgc-image/6f31306c0ea545c383326fbebc79fd1a)

这个哈希表有十个桶。因为我很懒，所以我只画出来其中的五个，但是我肯定你能够想像得到另外五个桶。我选择的哈希函数是键值模 10。换句话说，我只用了每个元素的键的最后一位数字去寻找它所在的桶：

- 如果最后一位数字是 0，那么它就会在桶 0
- 如果最后一位数字是 1，那么它就会在桶 1
- 如果最后一位数字是 2，那么它就会在桶 2
- …

我使用的比较函数就是两个整型之间的比较。

假如你需要找到元素 78：

- 哈希表计算出 78 的哈希值是 8；
- 哈希表在桶 8 中寻找，发现它的第一个元素就是 78；
- 哈希表返回元素 78。
- **这个检索只有 2 个操作**（第一个操作是计算哈希值，第二个操作是在桶中寻找元素）。

现在，加入要找到元素 59：

- 哈希表计算出 59 的哈希值是 9；
- 哈希表在桶 9 中寻找，找到的第一个元素是 99。由于 99 != 59，因此 99 不是所需要的元素；
- 使用相同的逻辑，哈希表继续查看第二个元素（9），第三个元素（79），直到最后一个元素（29）；
- 元素不存在。
- **这个检索需要 7 个操作。**

##### 好的哈希函数

从上面的示例可以发现，检索所需要的操作数取决于你所要检索的值。

如果我将哈希函数改成键值模 1000000（也就是取最后 6 个数字），上面所说的第二个检索只会消耗 1 个操作，因为桶 000059 中没有元素。由此可见，**真正的挑战是找到一个好的哈希函数，让每个桶中的元素总数保持在一个很小的值。**

在我的例子中，很容易找到一个好的哈希函数。但这只是一个很简单的例子，当键属于下面的情况时，找到一个好的哈希函数就会复杂得多了：

- 字符串（例如，一个人的姓）
- 两个字符串（例如，一个人的姓和名）
- 两个字符串和一个日期（例如，一个人的姓、名和这个人的出生日期）
- …

**结合一个好的哈希函数。哈希表检索的时间复杂度可以达到O(1)**

##### 数组 vs 哈希表

为什么不使用数组？

这是个好问题。

- 哈希表可以**在内存中只加载一半**，其余的桶依然保存在磁盘。
- 使用数组，你必须使用连续的内存空间。如果你需要加载一个很大的表，是**很难申请到连续内存空间的**。
- 使用哈希表，你可以**选择你想要使用的键**（例如，一个人的国籍和姓）。

更多的信息，你可以阅读 [Java HashMap](http://coding-geek.com/how-does-a-hashmap-work-in-java/)，这篇文章介绍了高性能哈希表的实现。即便不懂 Java，你也能够理解这篇文章中的概念。

# 2、全局概览

前面我们已经介绍过数据库内部的基本组件。现在，我们需要回到最高层。

数据库是一种能够轻松访问和修改的信息集合。但是，简单的文件也可以做相同的事情。事实上，最简单的数据库，比如 SQLite，就是一系列文件。但是，SQLite 是一组被精心组织的文件，允许你：

- 利用事务保证数据安全和一致
- 即使你处理百万级别的数据，它的处理速度依然很快

一般地，数据库可以看作下图：

![database global overview](https://p.pstatp.com/origin/pgc-image/baa81dc1b74240498ee2dd46ab6a8cfc)

在开始写这部分之前，我阅读了很多描述数据库的书和论文，甚至是数据库的源代码。所以，不要太纠结我是怎么组织数据库的，或者我是怎么给它命名的，因为我必须为了本书的主题做出一定的取舍。真正重要的是这些不同的组件。总的思想是，**数据库可以分为多个相互联系的多种组件**。

核心组件

- **进程管理器**：很多数据库都需要管理**进程池或线程池**。而且，为了改进那么几纳秒，一些现代数据库还会使用它们自己的线程，而不是操作系统提供的线程。
- **网络管理器**：网络 I/O 是个大问题，尤其对于分布式数据库。这也就是为什么有些数据库会有它们自己的网络管理器。
- **文件系统管理器**：**磁盘 I/O 是数据库的最大瓶颈**。因此，有一个能够完美地处理操作系统文件系统，甚至取代操作系统文件系统的文件系统管理器就变得非常重要。
- **内存管理器**：为了避免大量磁盘 I/O，大容量内存必不可少。但是，如果你有很大数量的内存，你就需要一个高效的内存管理器。尤其是当你在同一时间有多个查询时。
- **安全管理器**：管理用户的认证和授权。
- **客户端管理器**：管理客户端连接。
- …

工具

- **备份管理器**：保存和恢复数据库。
- **恢复管理器**：在数据库崩溃之后将其重启到一个**一致性状态**。
- **监控管理器**：记录数据库动作日志，提供工具监控数据库。
- **管理管理器**：保存元数据（比如表的名字和结构等），提供工具管理数据库、模式和表空间等。
- …

查询管理器

- **查询处理器**：检查查询语句属否合法。
- **查询重写器**：为查询语句预优化。
- **查询优化器**：优化查询语句。
- **查询执行器**：编译并执行查询语句。

数据管理器

- **事务管理器**：处理事务。
- **缓存管理器**：在使用数据之前或将数据写入磁盘之前，将数据放到内存中。
- **数据访问管理器**：访问磁盘上的数据。

本文剩下的部分，我将会详细阐述数据库是如何通过如下步骤管理一条 SQL 查询：

- 客户端管理器
- 查询管理其
- 数据管理器（这部分也会包括恢复管理器）

## 2.1、客户端管理器

![client manager](https://p.pstatp.com/origin/pgc-image/37cd4124e9fd479eb7680bec51cd2c96)

客户端管理器处理数据库与客户端之间的通讯。所谓客户端，可以是（网络）服务器或最终用户/应用程序。客户端管理器提供了多种不同方法访问数据库，包括一系列著名的 API：JDBC、ODBC、OLE-DB 等。

客户端管理器还提供了数据库访问的专有 API。

当你连接到一个数据库时：

- 管理器首先检查你的**认证**（用户名和密码），然后检查你是不是有使用数据库的**授权**。访问权限由 DBA 设置。
- 然后，管理器检查是不是有一个可用的进程（或线程）可以管理你的查询。
- 管理器还要检查数据库是不是处于过载状态。
- 管理器会等待一段时间，以便获取必要的资源。如果等待超时，它就会管理连接，然后返回一个人可读的错误信息。
- 然后，管理器会**将你的查询发送到查询管理器**，这样，你的查询就被处理了。
- 由于查询处理并不是“要么全部，要么没有”，只要它从查询管理器取得数据，它就会**将这部分数据保存在一个缓冲区，然后开始将数据发送**给你。
- 如果出现了问题，它就会停止连接，返回给你**可读的原因**，然后释放资源。

## 2.2、查询管理器

![Query Manager](https://p.pstatp.com/origin/pgc-image/8d0d4c3292ab4babbf61209b216a60f2)

**这部分是数据库的强大之处。在这部分，一个写得有问题的查询语句会被转换成一个能够快速执行的代码。**然后，代码被执行，结果返回给客户端管理器。这是多步操作：

- 首先，查询语句会被**解析（parse）**，检查是不是合法。
- 然后，查询语句会被**重写（rewrite）**，移除没用的操作，增加一些预先的优化。
- 然后，查询语句会被**优化（optimize）**，改进性能，转换成一个执行和数据访问计划。
- 然后，这个计划会被**编译（compile）**，
- 最后，这个计划会**被执行（execute）**。

在这部分，我不会对最后两点作过多的介绍，因为它们并不是那么重要。

在阅读完本部分时，如果你还想进一步学习，我推荐阅读下面的文章：

- 有关基于成本的优化的原始研究论文（1979）：[Access Path Selection in a Relational Database Management System](http://www.cs.berkeley.edu/~brewer/cs262/3-selinger79.pdf)。这篇论文只有 12 页，只要有计算机科学知识的一般水平就足以理解。
- 一篇关于 DB2 9.X 如何优化查询的非常好并且很有深度的报告：[这里](http://infolab.stanford.edu/~hyunjung/cs346/db2-talk.pdf)。
- 一篇关于 PostgreSQL 如何优化查询的非常好的报告：[这里](http://momjian.us/main/writings/pgsql/optimizer.pdf)。这篇文章是最容易理解的，因为它主要是关于“给我看看在这些情形下，PostgreSQL 给出的查询计划是怎样的”以及“让我们看看 PostgreSQL 使用的算法”。
- 有关优化的 [SQLite 官方文档](https://www.sqlite.org/optoverview.html)。因为 SQLite 使用的规则都很简单，所以这篇文章比较“好懂”。并且，这是真正解释如何工作的唯一官方文档。
- 一篇关于 SQL Server 2005 如何优化查询的非常好的报告：[这里](https://blogs.msdn.com/cfs-filesystemfile.ashx/__key/communityserver-components-postattachments/00-08-50-84-93/QPTalk.pdf)。
- 有关 Oracle 12c 优化的白皮书：[这里](http://www.oracle.com/technetwork/database/bi-datawarehousing/twp-optimizer-with-oracledb-12c-1963236.pdf)。
- 来自《DATABASE SYSTEM CONCEPTS》作者的关于查询优化的 2 个理论课程：[这里](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch12.ppt)以及[这里](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch13.ppt)。着眼于磁盘 I/O 消耗，但是要求一定的计算机科学知识。
- 另外一个[理论课程](https://www.informatik.hu-berlin.de/de/forschung/gebiete/wbi/teaching/archive/sose05/dbs2/slides/09_joins.pdf)更好理解，但是仅着重于连接操作和磁盘 I/O。

### 2.2.1、查询解析器

每一个 SQL 语句都会发送给解析器，后者将检查语句的语法。如果查询语句中有错误，解析器会拒绝该查询。例如，如果你写的是“SLECT …” 而不是“SELECT …”，那么处理就到此为止了。

但是，我们会深入探讨这一点。解析器还会检查关键字使用是不是顺序正确。例如，WHERE 出现在 SELECT 之前是不允许的。

然后，查询中的表和字段都会被分析。解析器使用数据库元数据检查：

- **表格是否存在**
- 表格的**字段**是否存在
- 该字段是否允许**执行该操作**（例如，不允许将整型同字符串进行比较，不允许使用整型调用 substring() 函数）

然后，解析器检查你是不是有**权限**在查询中读取（或写入）数据表。再说一遍，这些表格的访问权限是由你的 DBA 设置的。

在解析期间，SQL 查询会被转换成一种内部的表示形式（通常是一棵树）。

如果所有一切正常，那么这个内部表示形式就会被发送给查询重写器。

### 2.2.2、查询重写器

在这一步，我们有一个查询的内部表示。重写器的目的是：

- 预优化查询
- 避免不必要的操作
- 帮助优化器找到最好的可行方案

重写器会在查询语句上执行一系列已知的规则。如果查询符合某一规则的模式，那么该规则就会被应用到这个查询，查询会被重写。下面是一组简单的（可选）规则列表：

- **视图合并**：如果查询中使用了视图，那么视图会被转换成该视图的 SQL 语句。
- **子查询扁平化**：子查询会明显增加优化的难度，因此，重写器会尝试修改带有子查询的查询，以便去除子查询。

例如，

```
SELECT PERSON.*
  FROM PERSON
  WHERE PERSON.person_key IN
    (SELECT MAILS.person_key
       FROM MAILS
       WHERE MAILS.mail LIKE 'christophe%');
```

可以替换为

```
SELECT PERSON.*
  FROM PERSON, MAILS
  WHERE PERSON.person_key = MAILS.person_key
    and MAILS.mail LIKE 'christophe%';
```

- **移除不必要的操作**：例如，如果你在有 UNIQUE 约束来避免数据出现重复的地方使用了 DISTINCT，那么，DISTINCT 关键字就会被移除。
- **移除冗余连接**：如果你有两个相同的连接，可能是因为在视图中已经暗含了一个连接条件，或者是由于传递律出现了一个没有用的连接，该连接就会被移除。
- **常量算术计算**：如果你的语句需要计算，那么，计算结果就在重写中就获得。例如，WHERE AGE > 10+2 会传换成 WHERE AGE > 12；TODATE(“日期”) 则会直接转换成所需的日期格式。
- **（高级）分区裁剪**：如果你使用的是分区表，重写器会找出使用了哪些分区。
- **（高级）物化视图重写**：如果物化视图符合查询语句中谓词的子集，重写器会检查视图是不是最新的，并且修改查询优先使用物化视图而不是原始表。
- **（高级）自定义规则**：如果有自定义规则修改查询（比如 Oracle 策略），重写器会执行这些规则。
- **（高级）OLAP 变形**：应用分析/窗口函数，star join，rollup 函数等（但是我不确定这部分是由重写器还是优化器完成的，由于这部分处理非常相近，所以很可能是依赖于数据库的实现）。

重写后的查询会发送给查询优化器，这时候，有趣的才真正开始！

### 2.2.3、统计

在我们学习数据库如何优化查询之前，我们需要先讨论下**统计**。因为没有统计的话，数据库是很愚蠢的。如果你不告诉数据库去分析下它们自己的数据，它们是不会主动去做的，会得到一个很糟糕的假设。

但是，数据库需要什么类型的信息？

我需要（简短地）讨论下数据库和操作系统是如何存储数据的。数据存储的最小单位是**页（page）**或者块（block），默认是 4 或 8 KB。这意味着如果你只需要 1 KB，它依然会占用整个页。如果一个页是 8 KB，那么你就会浪费 7 KB。

这意味着如果你只需要 1 KB，它依然会占用整个页。如果一个页是 8 KB，那么你就会浪费 7 KB。

回到统计！当你要求数据库收集统计信息时，它会计算：

- 表格中的行数或页数
- 表格中每一列：
  - 不同的数据值
  - 数据值的长度（最大值、最小值、平均值）
  - 数据范围的信息（最大值、最小值、平均值）
- 表格索引的信息

**这些统计可以帮助优化器估算查询中的磁盘 I/O、CPU 和内存使用。**

每一列的统计都很重要。例如，如果 PERSON 表需要连接 2 列：LAST_NAME 和 FIRST_NAME。通过统计，数据库就会知道 FIRST_NAME  列只有 1000 个不同的值，而 LAST_NAME 则有 1000000 个不同值。因此，数据库会使用 LAST_NAME, FIRST_NAME 连接数据，而不是 FIRST_NAME,LAST_NAME，这会产生更少的比较步骤，因为 LAST_NAME 一般不会相同，有可能只需要比较 2（或 3）个字符就可以了。

但是这些只是基本统计。你可以要求数据库计算高级统计，称为**直方图（histograms）**。直方图代表列内部值的分布。例如：

- 最常见的值
- 分位数
- …

这些额外统计将帮助数据库找到更好的查询计划。这种统计对于相等谓词（例如，WHERE AGE = 18）或范围谓词（例如，WHERE AGE > 10 AND AGE < 40）尤其有效，因为数据库可以获得有关这两种谓词数据行分布的更好信息（注意，这种概念的专业术语是选择性 selectivity）。

统计信息存储在数据库的元数据中。例如，你可以在下面位置看到（非分割）表的统计信息：

- Oracle 数据库在 USER/ALL/DBA_TABLES 和 USER/ALL/DBA_TAB_COLUMNS
- DB2 数据库在 SYSCAT.*TABLES* 和 *SYSCAT.COLUMNS*

**统计信息必须及时更新。最糟糕的是，当数据库表实际有 1000000 行时，它却认为只有 500 行。统计的缺点是，需要花费大量时间进行计算**。这也就是为什么大多数数据库并不会自动计算统计信息。当数据库有百万级别的数据量时，计算统计信息会非常困难。在这种情况下，你可以选择只计算基本统计或计算数据样本库的统计。

例如，我曾经在一个项目工作，该项目中每一个表的数据量达到亿级别。我选择其中 10% 计算统计数据，这节省了大量时间。但是，最终结果表明这是一个错误的决定，因为 Oracle 10G 选择的这 10% 恰恰是一个特殊表的特殊字段，根本不能代表其余 100% 的数据（对于 1 亿的数据量，这并不经常发生）。这个错误的统计使得一个本应只有 30 秒的查询消耗了近 8 小时。查找问题的过程简直就是噩梦。这个故事说明了统计是有多么重要。

注意，每种数据库都会有特殊的高级统计。如果你想要了解更多，可以阅读数据库文档。就像前面说的，当我试图了解统计是如何使用的时候，发现的最好的官方文档就是[来自 PostgreSQL 的一篇](http://www.postgresql.org/docs/9.4/static/row-estimation-examples.html)。

### 2.3.4、 查询优化器

所有现代数据库都使用**基于成本的优化（Cost Based Optimization，CBO）**来优化查询。其思路是，为每一个操作添加一个成本，通过使用成本最低的操作链降低查询成，最终获得结果。

为了弄清楚成本优化器是如何工作的，我觉得最好的方法是“感受”这个任务背后的复杂性。在这部分，我将表述连接两张表的三种通用方法，我们很快就会看到每一个简单的连接查询都是一个优化的噩梦。在此之后，我们将看到真正的优化器是如何进行这个工作的。

对于这些连接，我着眼于它们的时间复杂度，但是**数据库优化器会计算其 CPU 成本、磁盘 I/O 成本以及所需内存**。时间复杂度和 CPU 成本的区别在于，前者是非常粗略的（对于那些像我一样懒的人来说）。对于 CPU 成本，我需要计算每一个操作，比如一次加法，一条“if 语句”，一次乘法，一次便利等。另外：

- 每一条高级代码操作都需要几条低级 CPU 操作。
- 对于 Intel Core i7、Item Pentium 4 或者 AMD 的 CPU 而言，每一个 CPU 操作的成本都不相同（其术语是 CPU 周期）。换句话说，CPU 操作的成本取决于 CPU 架构。

使用时间复杂度更简单（至少对于我来说是这样），使用时间复杂度我们依然能够获得 CBO 的概念。有时我也会提到磁盘 I/O，因为这也是一个重要概念。时刻记住，**瓶颈在磁盘 I/O 所消耗的时间，而不是 CPU 消耗的**。

### 2.3.5、索引

在我们介绍 B+ 树的时候，我们讨论过索引。记住，**索引已经排好序了**。

仅供参考：数据库还有其它类型的索引，比如**位图索引（bitmap indexes）**。它们与 B+ 树索引消耗的 CPU 周期、磁盘 I/O 和内容都不相同。

另外，如果可以改善执行计划的成本，很多现代数据库可以**为当前插件动态创建临时索引**。

#### 访问路径

在应用连接运算符之前，首先你需要获得数据。下面是你能够如何获得数据。

注意：由于访问路径真正的问题在于磁盘 I/O，因此我不会过多讨论时间复杂度。

##### 完整扫描

如果你阅读过执行计划，你一定见过完整扫描（**full scan**，或者就叫 scan）这个词。完整扫描就是数据库读取整张表或索引。**考虑到磁盘 I/O，很明显，对于整张表的完整扫描的成本要远远高于对于索引的完整扫描。**

##### 范围扫描

其它类型的扫描有**索引范围扫描（index range scan）**。当你使用谓词，比如“WHERE AGE > 20 AND AGE <40”时，就会使用索引范围扫描。

当然，为了使用索引范围扫描，你需要在 AGE 字段添加索引。

在前面部分，我们已经了解到，范围查询的时间成本可能会有 log(N) +M，其中，N 是索引数据量，M 是落在这个范围的行数的估计值。**多亏了统计数据，N 和 M 都是已知的**（注意，M 就是谓词 AGE >20 AND AGE<40 的选择率）。另外，对于范围扫描，你不需要只读完整的索引，因此要**比完整扫描的磁盘 I/O 成本小得多**。

##### 唯一扫描

如果你只需要从索引获取一个值，可以使用**唯一扫描（unique scan）**。

##### 通过 row id 访问

大多数时间，如果数据库使用索引，它就得找到关联到索引的那些行。为了达到这一目的，数据库会使用 row id 访问。

例如，如果你的语句如下：

```
SELECT LASTNAME, FIRSTNAME from PERSON WHERE AGE = 28
```

如果 age 列有索引，优化器会使用索引找到所有 28 岁的人，然后会请求表中关联的那些行。这是因为索引只包含了年龄的信息，但是你却想要获取 lastname 和 firstname。

但是，如果你的语句如下：

```
SELECT TYPE_PERSON.CATEGORY from PERSON, TYPE_PERSON
  WHERE PERSON.AGE = TYPE_PERSON.AGE
```

PERSON 的索引用于连接 TYPE_PERSON，但是 PERSON 表却不会使用 row id 访问，因为你并没有请求这个表的数据。

对于少量访问，这是可行的，但这个操作的真正问题在于磁盘 I/O。如果你需要通过 row id 访问很多数据，数据库可能会选择使用完整扫描。

##### 其它路径

我并没有阐述所有访问路径。如果你想了解更多，可以阅读 [Oracle 文档](https://docs.oracle.com/database/121/TGSQL/tgsql_optop.htm)。对于其它数据库，文档的名字可能会有歧义，但是其背后的原理是相通的。

#### 连接运算符（join）

现在我们知道了如何获得数据，然后，让我们将数据连接起来！

我会介绍三种通用连接运算符：归并连接、哈希连接和嵌套循环连接。但在此之前，我需要介绍一个新的术语：**内关系（inner relation） 和 外关系（outer relation）**。一个关系可以是

- 一个表
- 一个索引
- 来自上一步操作的中间结果（例如上一个连接的结果）

当你连接两个关系时，连接算法会分别管理两个关系。在本文剩下的部分，我会假设：

- 外关系是左数据集
- 内连接是右数据集

例如，A JOIN B 是 A 和 B 的连接，其中，A 是外连接，B 是内连接。

大多数情况下，**A JOIN B 的成本与 B JOIN A 并不相同**。

**在这一部分，我同样假设外连接有 N 个元素，内连接有 M 个元素**。时刻记住，真正的优化器可以通过统计知道 N 和 M 的值。

注意，N 和 M 是联系的基数。

##### 嵌套循环连接（nested loop join）

嵌套循环连接是最简单的一个。

![Nested Loop Join](https://p.pstatp.com/origin/pgc-image/ebff4057d93345ac920a53b9adb9ee72)

其思路是：

- 对于外关系的每一行
- 检查内关系中的所有行，看是不是有行能够匹配

伪代码如下：

```python
nested_loop_join(array outer, array inner)
  for each row a in outer
    for each row b in inner
      if (match_join_condition(a, b))
        write_result_in_output(a, b)
      end if
    end for
   end for
```

因为这里有两层循环，所以**时间复杂度是 O(N\*M)**。

从磁盘 I/O 方面，对于外关系 N 行的每一行，内层循环都需要从内关系读取 M 行。这个算法需要从磁盘读取 N + N * M 行。但是，如果内关系足够小，那么就可以把所有关系放在内存，这样就只需要 M +N 次读取。根据这样的修改，**内关系必须是最小的一个**，因为只有这样才有更多机会放入内存。

从时间复杂度方面，这样并没有什么不同，但是从磁盘 I/O 方面，这种方式只需要读取两种关系一次。

当然，内关系可以替换为索引，这样对磁盘 I/O 更有利。

由于这个算法非常简单，对于那些不能完全读入内存的内关系而言，还有一个对磁盘 I/O 更友好的版本。其思路是这样的：

- 不是一行一行读取关系
- 按照批次读取关系，在内存中始终保存两个批次的行（每个关系都保留 2 个批次）
- 在两个批次内部比较行，保留匹配的行
- 然后，从磁盘加载新的批次，再进行比较
- 如此进行下去，知道没有批次可供加载

下面是算法伪代码：

```python
// 改进版本，减少磁盘 I/O。
nested_loop_join_v2(file outer, file inner)
  for each bunch ba in outer
    // 现在 ba 在内存中
    for each bunch bb in inner
      // 现在 bb 在内存中
      for each row a in ba
        for each row b in bb
          if (match_join_condition(a,b))
            write_result_in_output(a,b)
          end if
        end for
      end for
    end for
  end for
```

**使用这个版本，时间复杂度相同，但是磁盘访问次数有所减少：**

- 之前的版本，算法需要 N + N * M 次磁盘访问（每次访问读取一行）
- 新的版本，磁盘访问数变为 number_of_bunches_for(outer) + number_of_ bunches_for(outer) * number_of_ bunches_for(inner)
- 增加每一批次的大小，就能降低磁盘访问次数

注意：虽然每一次磁盘访问都可以加载比上述算法更多的数据，但是由于数据是顺序访问的（机械硬盘真正的问题是获得第一组数据所需时间），因此影响并不会很大。

#### 哈希连接(Hash Join)

哈希连接要比嵌套循环连接复杂得多，但是在很多情形下，它的性能要比后者好得多。

![HashJoin](https://p.pstatp.com/origin/pgc-image/e4dae481e66447528138173ddf10ee5c)

哈希连接的思路是：

1. 从内关系获取所有元素
2. 构建内存中的哈希表
3. 依次获取外关系中的所有元素
4. 计算每一个元素的哈希值（使用哈希表的哈希函数），找到与内关系关联的桶
5. 检查桶中元素和外表中的元素是不是匹配

关于时间复杂度，我需要作一些假定以简化问题：

- 内关系分为 X 个桶
- 对于两个关系，哈希函数都可以均匀地散列哈希值。换句话说，这些桶的大小相同
- 外关系中的一个元素与桶中所有元素的匹配过程消耗只与桶中的元素数目有关

其时间复杂度是 (M/X) * N + cost_to_create_hash_table(M) + cost_of_hash_function*N

如果哈希函数创建足够小的桶，那么，**时间复杂度就是 O(M+N)**。

下面的哈希连接的版本对内存更友好，但是不利于磁盘 I/O。这一次：

1. 同时计算内关系和外关系的哈希表
2. 将计算出的哈希值存入磁盘
3. 然后按照桶依次比较两个关系（现将一个加载到内存，然后依次读取另外一个）

#### 归并连接（Merge Join）

**归并连接是唯一获得排序结果的连接。**

注意：在这个简化的归并连接中，不存在内表或外表；它们的角色是一致的。但是，实际实现是有区别的，例如，当处理复制时。

归并连接可以分为两步：

1. （可选的）排序连接操作：两个输入都按照连接键进行排序
2. 归并连接操作：将排好序的输入合并在一起

##### 排序

我们已经讨论了归并排序，在这个情形下，归并排序是一个不错的算法（但是如果内存不是问题的话，这并不是最好的算法）。

但是，有时数据集已经排好序了，例如：

- 表自然地排序了，例如在连接条件上使用索引组织的表
- 如果在连接条件上，这个关系是一个索引
- 如果这个连接应用在一个中间结果上，而这个中间结果已经在查询处理阶段排好序了

##### 归并连接

![Merge Join](https://p.pstatp.com/origin/pgc-image/d08f01b896c04b578976d18bedd40e1f)

这一步非常像我们之前见过的归并排序的归并操作。但是这一次我们并不是从每个关系中取出每一个元素，而是从每个关系中取出相同元素。其思想是：

1. 比较两个关系中的当前元素（第一次时，当前元素就是第一个元素）
2. 如果相同，将两个元素放进结果集，然后获取两个关系中的下一个元素
3. 如果不相同，从较小元素所在关系中取出下一个元素（因为下一个元素可能会相同）
4. 重复以上 3 步，直到其中一个关系到达最后一个元素

因为两个关系都是排好序的，因此这种操作是可行的。你不需要在这些关系中“后退”。

这个算法是一个简化版本，因为它没有处理两个数组中相同数据出现多次的情况（也就是多次匹配）。真实的版本会更复杂一些，但也不会复杂太多，因此我选择了一个相对简单的版本。

如果两个关系都已经排好序，那么**时间复杂度就是 O(N+M)**。

如果两个关系都需要排序，那么时间复杂度就是两个关系排序的消耗：**O(N \* Log(N) + M \* Log(M))**。

对于计算机专业技术人员，下面是一个能够处理多次匹配的可行算法（虽然我也不是 100% 确保算法正确性）：

```python
mergeJoin(relation a, relation b)
  relation output
  integer a_key:=0;
  integer b_key:=0;
 
  while (a[a_key]!=null or b[b_key]!=null)
    if (a[a_key] < b[b_key])
      a_key++;
    else if (a[a_key] > b[b_key])
      b_key++;
    else //Join predicate satisfied
         //i.e. a[a_key] == b[b_key]
 
      //count the number of duplicates in relation a
      integer nb_dup_in_a = 1:
      while (a[a_key]==a[a_key+nb_dup_in_a])
        nb_dup_in_a++;
        
      //count the number of duplicates in relation b
      integer dup_in_b = 1:
      while (b[b_key]==b[b_key+nb_dup_in_b])
        nb_dup_in_b++;
        
      //write the duplicates in output
       for (int i = 0 ; i< nb_dup_in_a ; i++)
         for (int j = 0 ; i< nb_dup_in_b ; i++)    
           write_result_in_output(a[a_key+i],b[b_key+j])
           
      a_key=a_key + nb_dup_in_a-1;
      b_key=b_key + nb_dup_in_b-1;
 
    end if
  end while
```

#### 哪个最好？

如果有最好的连接方式，就不会有这么多种了。这个问题很难回答，因为有很多因素在里面：

- **可用内存总数**：没有足够了内存，就基本可以跟强大的哈希连接说拜拜了（至少对完全内存中的哈希连接）
- **两个数据集的大小**：例如，如果你有一张大表和一个很小的表，那么，嵌套循环连接要比哈希连接快一些，因为哈希连接需要计算哈希值。如果你的两张表都很大，那么，嵌套循环连接会消耗非常多的 CPU。
- **是不是有索引**：对于归并连接，两个 B+ 树索引绝对是一个好主意。
- 如果**结果需要排序**：即使你的数据集没有排序，你还是可能想使用代价昂贵的归并连接（使用排序），因为最终结果是排好序的，你可以将这个结果交给另外的归并连接（或者是由于查询语句使用 ORDER BY/GROUP BY/DISTINCT 等隐式或显式要求了结果排序）。
- 如果**关系已经排好序**：这种情况下，归并连接是最好的选择。
- 你正在使用的连接类型：是不是**equijoin** （例如 tableA.col1 = tableB.col2）？是不是 **inner join**、**outer join**、**cartesian product** 或者 **self-join**？某些连接不适用与特定情形。
- **数据的分布**：如果连接条件中的数据是**不均衡的（skewed）**，例如，你需要使用人名中的姓氏连接，但是很多人都有相同的姓，使用哈希连接就是一场灾难，因为哈希函数计算出来的桶是非常不均衡的。
- 如果你需要在**多线程或多进程**情形下进行连接。

更多信息，可以阅读 [DB2](https://www-01.ibm.com/support/knowledgecenter/SSEPGG_9.7.0/com.ibm.db2.luw.admin.perf.doc/doc/c0005311.html)、[ORACLE](http://docs.oracle.com/cd/B28359_01/server.111/b28274/optimops.htm#i76330) 或 [SQL Server](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms191426%28v=sql.105%29) 的文档。

#### 简单的例子

我们已经见识到了 3 种类型的连接操作。

现在，假设我们需要连接 5 张表来获取一个人的全部信息。*一个人*有：

- 多个手机（MOBILES）
- 多个邮箱（MAILS）
- 多个地址（ADRESSES）
- 多个银行账户（BANK_ACCOUNTS）

换句话说，我们需要能够快速地找出下面查询的结果：

```
SELECT * from PERSON, MOBILES, MAILS,ADRESSES, BANK_ACCOUNTS
    WHERE
        PERSON.PERSON_ID = MOBILES.PERSON_ID
        AND PERSON.PERSON_ID = MAILS.PERSON_ID
        AND PERSON.PERSON_ID = ADRESSES.PERSON_ID
        AND PERSON.PERSON_ID = BANK_ACCOUNTS.PERSON_ID
```

作为一个查询优化器，我必须找到最好的途径来处理数据。但是，这里有两个问题：

- 针对每一个连接，我需要选择使用哪种类型？

我有 3 种类型可供选择（哈希连接、归并连接和嵌套连接），每种类型可以使用 0 个、1 个或者 2 个索引（并不是说有不同类型的索引）。

- 我应该按照怎样的顺序计算连接？

例如，下面的图显示了 4 张表的 3 个连接的不同计划：

![JOIN](https://p.pstatp.com/origin/pgc-image/176468c0f507439ba74a06e02745a666)

现在我有下面几个选择：

- 暴力实现

利用数据库统计，我**计算得出每种计划的成本**，选择最好的一个。但是这有很多可能性。对于一个给定的连接顺序，每种连接有 3 种可能：哈希连接、归并连接和嵌套连接。那么，对于给定的连接顺序，一共有 34 种可能性。连接顺序是[二叉树上面的排序问题](https://en.wikipedia.org/wiki/Catalan_number)，有 (2*4)!/(4+1)! 种排序。对于这个简单的问题，我们有 34*(2*4)!/(4+1)! 种可能性。

用“人话”来说，一共有 27 216 种可能的计划。现在，如果我增加归并连接可以使用的 B+ 树索引数为 0、1 或 2，可能的计划数变成了 210 000。我记得我说过这个查询是*非常简单*的吗？

- 哭着说不干了

这的确非常解气，但是你没法拿到你要的结果，我也需要花钱去缴我的账单。

- 我只试几个计划，从中选择一个成本最低的

因为我不是超人，我没办法计算每一个计划的成本。所以，我**随机地选择所有可能的计划的一个子集**，计算其成本，给你这个子集中最好的计划。

- 我应用一个非常聪明的**能够减少可能的计划数的规则**

有两种类型的规则：

“逻辑”规则，用于移除没什么用的可能计划，但是并不能过滤掉大部分可能的计划。例如：“嵌套循环连接的内关系必须是最小的数据集”。

我接受不去找到最好的解决方案，而是应用积极的规则来减少可能性的数量。例如，“如果关系很少，使用嵌套循环连接，不使用归并连接和哈希连接”。

在这个简单的例子中，最终我的结果还会有很多种可能性。但是，**在真实的查询中，还有其它的关系运算符**，例如 OUTER JOIN、CROSS JOIN、GROUP BY、ORDER BY、PROJECTION、UNION、INTERSECT、DISTINCT … **这意味着甚至有更多可能性。**

那么，数据库是怎么做的？

### 动态规划、贪心算法和启发式算法

关系数据库会尝试我前面说的各种实现。优化器的真正工作是在有限时间内找出一个好的解决方案。

**大多数时间，优化器并不是找到最好的解决方案，而是一个“好的”解决方案。**

对于小的查询，暴力实现是可能的。但是，还有一种方法可以避免不必要的计算，从而使得一些中级查询也可以使用暴力实现。这种方法叫做动态规划。

#### 动态规划

这个词背后的思想是，大部分执行计划都是相似的。如果你看一下下面的计划：

![](https://p.pstatp.com/origin/pgc-image/9c400c7d3a7d4b1fa23ca1d97ad04062)

这些计划都共享了相同的 (A JOIN B) 子树。因此，不需要在每一个计划都计算这个子树的成本，我们只需要计算一次，将计算所得的成本保存下来，当再次见到这个子树时，复用这个计算结果即可。正式的说法是，我们面对的是一个重叠问题。为避免中间结果的额外计算，我们使用了记忆技术。

利用这一技术，我们的时间复杂度不是 (2*N)!/(N+1)!，“仅仅”是3N。在我们前面四个连接的例子中，这意味着可能性将从 336 降低到 81。如果你的查询更大（比如 **8 个连接的查询**，这其实也算不上大），这意味着可能性**从57 657 600 降低到 6561**。

对于计算机专业人士，我[在前面提到过的正式课程](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch13.ppt)中找到一个算法。我不会解释这个算法，所以，如果你懂动态规划或者对算法有兴趣的话，可以仔细阅读下面的算法（我已经警告过你了！）：

```
procedure findbestplan(S)
if (bestplan[S].cost infinite)
   return bestplan[S]
// else bestplan[S] has not been computed earlier, compute it now
if (S contains only 1 relation)
         set bestplan[S].plan and bestplan[S].cost based on the best way
         of accessing S  /* Using selections on S and indices on S */
     else for each non-empty subset S1 of S such that S1 != S
   P1= findbestplan(S1)
   P2= findbestplan(S - S1)
   A = best algorithm for joining results of P1 and P2
   cost = P1.cost + P2.cost + cost of A
   if cost < bestplan[S].cost
       bestplan[S].cost = cost
      bestplan[S].plan = “execute P1.plan; execute P2.plan;
                 join results of P1 and P2 using A”
return bestplan[S]
```

对于更大的查询，你还是可以继续使用动态规划算法，但是还可以有另外的规则（启发式算法）来移除一些可能性：

- 如果我们分析唯一类型的计划（例如左深树），我们可以做到 n*2n 而不是 3n

![](https://p.pstatp.com/origin/pgc-image/531ac8a17b5249f3b520d77851cab6a0)

- 如果我们添加逻辑规则来避免某些模式的计划（例如，“如果一张表对于给定谓词包含一个索引，那么就要在索引而不是表上应用归并连接”），那么就可以减少可能性的数量而不会影响到最好可能的解决方案。
- 如果我们在工作流添加规则（例如，“在其它关系操作*之前*应用连接”），同样会减少很多可能性。
- …

#### 贪心算法

为了处理非常大的查询，或者为了快速获得答案（针对不是那么快的查询），需要使用另外一种类型的算法——贪心算法。

贪心算法的思路是，利用某种规则（或**启发式**）构建一种渐进的查询计划。根据这种规则，贪心算法每次找到一个问题的最佳解的一个步骤。算法从一个 JOIN 开始查询计划。然后，在之后的每一步种，算法按照相同规则，向查询计划增加一个新的 JOIN。

下面我们看一个简单的例子。假设我们有一个查询，包含五张表（A，B，C，D，E）四个连接。为了简化问题，我们只考虑嵌套连接。我们使用的规则是“使用最低成本的连接”。

- 我们从五张表中任意一个开始（例如，从 A 开始）
- 计算与 A 的每一个连接的成本（A 作为内关系或外关系）
- 发现 A JOIN B 是最低成本
- 然后计算与 A JOIN B 结果关联的每一个连接的成本（A JOIN B 作为内关系或外关系）
- 找到 (A JOIN B) JOIN C 是最低成本
- 然后计算与 (A JOIN B) JOIN C 结果关联的每一个连接的成本…
- ….
- 最后，找到计划 (((A JOIN B) JOIN C) JOIN D) JOIN E)

由于我们是随机从 A 开始，因此也可以为 B、C、D 和 E 应用相同的算法。最后，我们找到了拥有最低成本的计划。

人们给使用这种方法的算法取了一个名字：[最近邻居算法](https://en.wikipedia.org/wiki/Nearest_neighbour_algorithm)。

我不会展开阐述，只说结论，通过良好的建模和 N*log(N) 的排序，该问题可以[很容易解决](http://www.google.fr/url?sa=t&rct=j&q=&esrc=s&source=web&cd=5&cad=rja&uact=8&ved=0CE0QFjAEahUKEwjR8OLUmv3GAhUJuxQKHdU-DAA&url=http%3A%2F%2Fwww.cs.bu.edu%2F~steng%2Fteaching%2FSpring2004%2Flectures%2Flecture3.ppt&ei=hyK3VZGRAYn2UtX9MA&usg=AFQjCNGL41kMNkG5cH)。**这个算法的时间复杂度是 O(N*log(N))，相比而言，完全动态规划算法的时间复杂度是 O(3N)。**如果你有一个 20 个连接的大查询，这意味着 26 比 3 486 784 401，这个差距非常巨大！

这个算法的问题是，我们假设如果能够一直找到两个表直接的最好连接，并且在此基础上增加新的连接，依然能够保持最好的成本。但是：

- 在 A、B、C 三者之间，即便 A JOIN B 给出最好成本
- (A JOIN C) JOIN B 依然可能比 (A JOIN B) JOIN C 还要好

为改进结果，你可以使用不同规则多次运行贪心算法，保留最好的计划。

#### 其它算法

[如果你已经厌倦了算法，直接跳到下一部分就好了。下面我将要介绍的内容与本文其它部分没有太大关系。]

找到最佳解决方案一直是计算机科学家的热门研究方向。他们试图为更多精确的问题或模式找到更好的解决方案。例如：

- 如果查询是星连接（就是多个连接查询的中心位置），有些数据库会使用特殊的算法
- 如果查询是并行的，有些数据库也会使用特殊的算法
- …

另外的算法同样有被研究，以便取代大查询的动态规划。贪心算法属于一个称为**启发式算法**的更大的算法家族。贪心算法遵循一个规则（或启发），持有上一步发现的解决方案，在当前步骤将这个解决方案“追加”以便获得新的解决方案。有些算法则是遵循规则，每一步都会应用规则，但是不会始终保存上一步找到的最佳解决方案。这种算法被称为启发式算法。

例如，**遗传算法**依照规则，但是并不始终保存上一步的最好结果：

- 一个能够表示全查询计划的可能解决方案
- 在每一步不只保存一个，而是保存 P 个解决方案
- 1. P 个查询计划是随机创建的
- 1. 只有最好成本的计划被保留
- 1. 这些最好的计划混合起来，创建 P 个新的计划
- 1. 这些 P 个新计划中的某些被随机改变
- 1. 重复 1，2，3 步 T 次
- 1. 在最后一次循环之后，找到 P 个计划的最佳计划

循环越多，你能获得的计划就会越好。

听起来就像魔法一样？不，这是自然法则：适者生存！

仅供参考，遗传算法在 [PostgreSQL](http://www.postgresql.org/docs/9.4/static/geqo-intro.html) 有实现，但是我不确定是不是默认使用了该算法。

数据库还使用了一些别的启发式算法，例如模拟退火算法、迭代改进算法、两阶段优化… 但是我不确定这些算法是不是已经应用在企业级数据库中，还是仅仅出现在研究型的数据库中。

更多信息可以阅读下面的论文，这篇论文列举了更多算法：[Review of Algorithms for the Join Ordering Problem in Database Query Optimization](http://www.acad.bg/rismim/itc/sub/archiv/Paper6_1_2009.PDF)。

### 真实的优化器

[你可以直接跳到后面的章节，这部分也不是非常重要]

但是，我们啰嗦了半天，都是理论上的。因为我是一个开发人员，不是研究学者，我喜欢**具体的例子**。

下面看 [SQLite 优化器](https://www.sqlite.org/optoverview.html)是如何工作的。这是一个轻量级数据库，使用了基于额外规则的贪心算法的简单优化，用于限定可能数：

- SQLite 从不重新排序 CROSS JOIN 运算符中的表
- **连接使用嵌套连接实现**
- 外连接始终按照出现的顺序
- …
- 在 3.8.0 之前的版本，**SQLite 使用“最近邻居”贪心算法检索最佳查询计划**

等一下… 我们已经见到了这个算法了！真巧！

- 从 3.8.0 （2015 年发布）开始，SQLite 使用“N 个最近邻居”**贪心算法**检索最佳查询计划

我们看看另外的优化器是怎么工作的。IBM DB2 和其它所有企业级数据库一样，我将介绍 DB2，因为这是我转换到大数据之前唯一真正使用过的数据库。

如果我们阅读[官方文档](https://www-01.ibm.com/support/knowledgecenter/SSEPGG_9.7.0/com.ibm.db2.luw.admin.perf.doc/doc/r0005278.html)，我们就会发现 DB2 优化器允许你使用 7 个不同级别的优化：

- 使用贪心算法处理连接
  - 0 – 最小优化，使用索引扫描和嵌套循环连接，避免某些查询重写
  - 1 – 低优化
  - 2 – 全优化
- 使用动态规划处理连接
  - 3 – 中度优化，粗略近似
  - 5 – 全优化，使用基于启发式算法的所有技术
  - 7 – 全优化，与 5 类似，但是不使用启发式算法
  - 9 – 最大优化，不遗余力**考虑所有可能的连接顺序，包括笛卡尔积**。

我们可以发现，**DB2 使用了贪心算法和动态规划**。当然，他们不会告诉你启发式算法的细节，因为查询优化器是一个数据库强大所在。

仅供参考，**默认级别是 5**。优化器默认使用下面的特性：

- **使用全部可用的统计**，包括快速统计值和分位数统计

- **应用所有查询重写规则**（包括物化查询表路由），除了仅适用于少数情况的计算密集型规则

- 使用动态规划算法连接枚举

  ，包括：

  - 限制使用内关系组合
  - 限制针对星模式使用笛卡尔积查询表

- 考虑大范围访问函数，包括列举预取（我们会在后文介绍这个词的含义），索引 AND（针对索引的特殊操作）和物化查询表路由

默认的，**DB2 针对连接排序，使用有限制的启发式动态规划**。

其它条件（GROUP BY, DISTINCT…）则使用简单规则处理。

### 查询计划缓存

由于创建计划要花费一定时间，大部分数据库都会将计划保存到**查询计划缓存**。这是一个复杂的话题，因为数据库需要知道什么时候更新过时的计划。其思路是，使用一个阈值，如果表的统计数据发生的改变超过了阈值，那么有关这个表的查询计划都会从缓存清除。

## 查询执行器

在这一阶段，我们有了经过优化的执行计划。这个计划会被编译成可执行代码。然后，如果有足够的资源（内存、CPU），查询执行器就会执行这个计划。计划中的运算符（JOIN, SORT BY …）可以顺序或并发执行，这取决于执行器。为了获取和写入数据，查询执行器与数据管理器进行交互，下一章我们就将介绍数据管理器。


















