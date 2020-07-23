# 0、写在前面

去年刚进公司的时候，在我们CRM组已经分享过B树和MySql索引的内容。而且之前也有几位同事分享过这些内容，所以这篇文章就不再赘述，这里仅以几句话做个总结：

[二分查找树(BST)](https://en.wikipedia.org/wiki/Binary_search_tree)解决了顺序表进行二分查找需要预先排序的问题——规定左子树值小于父节点，右子树值大于父节点。

[自平衡二叉树](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree)解决了二分查找树可能退化成链表的问题——平衡左右子树的深度。

[AVL树](https://en.wikipedia.org/wiki/AVL_tree)是最早出现的平衡二叉树，AVL树要求左右子树**严格平衡**（左右子树深度差不大于1），可能需要一次或多次旋转，从而导致插入效率较低。

而[红黑树](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)保证了**大致平衡**，插入查找效率整体性能较好——从根到最远的叶子节点的路径不超过从根到最近叶子节点路径的两倍。

[B树](https://en.wikipedia.org/wiki/B-tree)是一个面向磁盘的数据结构，它通过降低树的深度来减少磁盘的访问次数——因为磁盘是以扇区块的单位读写的，B树一个节点可以存储多个键值对，更贴合磁盘速度慢吞吐量大的读写模式。

[B+树](https://en.wikipedia.org/wiki/B%2B_tree)对B树做了两个优化。非叶子节点只存储Key，完整的Key-Value存储在叶子节点，可以让节点存储更多的Key，变相的降低了树的深度，从而索引更多数据，同时对内存的索引缓存更友好；叶子节点之间以链表的形式串联起来，解决了B树范围查找时需要回溯节点导致随机I/O次数增加的问题。

B+树是MySQL，Oracle等大多数关系型数据库的存储结构。

> 如果对我的理解感兴趣，具体可以参考[二叉搜索树BST,AVL,红黑树,伸展树](https://blog.hufeifei.cn/2018/03/28/DataStructure/二叉搜索树BST,AVL,红黑树,伸展树/)和[B树与B+树](https://blog.hufeifei.cn/2018/04/02/DataStructure/B树与B+树/)两篇文章。

# 1、B+树的缺点与不足

B+树在插入操作时，超过节点允许的最大分支因子后会导致节点分裂（数据库中常称为页分裂）：

![节点分裂](http://ww1.sinaimg.cn/large/bda5cd74gy1fqbihhh74hg20j605z17a.gif)

这通常是一个相对较大的操作，如果数据库在操作中由于任何原因而崩溃(比如断电)，这将是一个非常严重的问题。

为了能解决数据库崩溃的问题，B+树通常还会结合[预写日志(Write-Ahead Log)](https://en.wikipedia.org/wiki/Write-ahead_logging)提供原子性和持久性——数据在写入B+树前，首先会将修改写入日志。

> [MySQL的InnoDBy引擎中的WAL](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design/)实现被称为[redo-log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)。

想象一下，MySQL正在执行某些操作时，数据库服务器断电了。重新启动后，MySQL可能需要知道其执行的操作是成功，部分成功还是失败了。如果使用了预写日志，MySQL可以检查该日志，并将意外断电的情况与实际操作进行比较。根据比较，程序可以决定撤消已开始的操作，继续完成已开始的操作或保持原样。

另外对于频繁的写入操作，**B+树的节点分裂会造成更多的随机读写**，明显会影响写入的吞吐量。

这篇文章要介绍的LSM-Tree算法就是为了解决B+树的这些不足之处。

> 下文中，B树就指代优化过的B+树

# 2、LSM-Tree

[Log-Structured_Merge Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)是数据库专家[Patrick O'Neil](https://en.wikipedia.org/wiki/Patrick_O'Neil)教授在1996年的[一篇论文](https://www.cs.umb.edu/~poneil/lsmtree.pdf)中提出来的。但直到被谷歌三篇著名论文中的[BigTable](http://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)引用(另外两篇是[GFS](http://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)、[MapReduce](http://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf))，LSM-Tree才正式从理论进入实践。

> Facebook为MySQL开发的[MyRocks存储引擎](https://myrocks.io/)就是基于LSM-Tree，[Mongo3.0的wiredTiger引擎](https://docs.mongodb.com/manual/core/wiredtiger/)也是基于LSM-Tree，ElasticSearch和Solr的底层Lucene使用的分段策略也是LSM-Tree，[Bigtable](https://en.wikipedia.org/wiki/Bigtable)、[HBase](https://en.wikipedia.org/wiki/HBase)、 [LevelDB](https://en.wikipedia.org/wiki/LevelDB)、[Apache Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra)、[RocksDB](https://en.wikipedia.org/wiki/RocksDB)、[ScyllaDB](https://en.wikipedia.org/wiki/Scylla_(database))、[InfluxDB](https://en.wikipedia.org/wiki/InfluxDB)等各种NoSQL数据库都是基于LSM-Tree数据结构。

![Database](http://ww1.sinaimg.cn/large/bda5cd74ly1g8w7jdaywnj214q0jndk1.jpg)

它的名字可能会对人造成误解，认为它是一棵具体的树结构，更准确地说它只是一个组织树结构的一种算法。正如它的名字一样，它的灵感来自于永远都是顺序写入的日志结构。

LSM-Tree的核心思想就是**将写入推迟(Defer)并转换为批量(Batch)写**：首先将大量写入缓存在内存，当积攒到一定程度后，将他们批量写入文件中，这样一次I/O可以进行多条数据的写入，充分利用每一次I/O。

## 2.1、两部件的LSM-Tree算法

![LSM-Tree](http://ww1.sinaimg.cn/large/bda5cd74ly1g7xypbgj96j20nz07ldfv.jpg)

基础的LSM-Tree拥有两部分：

- 一个较小的位于内存的部件，就是上图中的$C_0$ tree，它可以是AVL-Tree，RB-Tree等内存数据结构。
- 一个较大的位于磁盘的部件，就是上图中的$C_1$ tree，它可以是类似B树的面向磁盘的数据结构。尽管$C_1$tree常驻磁盘，但它常被访问的磁盘页也会被缓存在内存中。

> 在BigTable一文中，位于内存的部分被称为Memtable(MemoryTable)，位于磁盘的部分被称为SSTable(SortStringTable)

每生成一行新记录流程如下：

1. 首先向WAL中写一条用于恢复这次插入行为的日志记录。
2. 该行数据会被索引到常驻内存的 $C_0$ 树中。
3. $C_0$树上的数据会适时地迁移到磁盘上的$C_1$树中。意味着如果$C_0$未存入磁盘时数据库崩溃，还需要恢复$C_0$​的数据。

$C_0$树位于内存，它的插入操作没有任何I/O消耗，但是这也限制了它的大小。所以我们需要一个高效的方式将数据迁移到$C_1$树。为此，每当$C_0$达到最大阈值时，会使用类似于归并排序的算法将$C_0$的数据并入$C_1$中，在论文中这种算法被称为滚动合并(Rolling Merge)。

到这里，有的人也许会说，目前许多关系型数据库也都对写入做了优化。

> 比如MySQL一个InnoDB表会有一个聚簇索引和若干个二级索引。每个索引都是一个B树，将记录插到数据表时，会先将记录插入到聚簇索引，然后再插入到每个二级索引。聚簇索引主键一般是单调递增的，而二级索引相对更随机，因而二级索引上的插入在磁盘I/O上也更随机，为了解决过多的随机I/O，InnoDB引擎提供了一个称为[Change Buffer](https://dev.mysql.com/doc/refman/5.7/en/innodb-change-buffer.html)的特殊结构。具体可以参考https://mysqlserverteam.com/the-innodb-change-buffer/

但是如果LSM-Tree只有这么简单，那谷歌研究这个内容就没什么价值了。因为两部件的LSM-Tree太简单，事实上更常见的是多部件LSM-Tree。不过在介绍多部件LSM-Tree之前，先比较一下LSM-Tree和B-Tree插入的代价。

## 2.2、LSM-Tree与B-Tree的插入代价比较

B-tree执行插入操作，需要首先找到记录要插入的位置，也就是进行一次B-tree查找。假设插入的位置是随机的，那么缓存无法重复使用了，由此定义B-tree插入操作进行随机访问的次数等于B-tree的深度：
$$
\large D_e = B树深度
$$
为了执行一个插入操作，需要进行$D_e$次I/O操作，以在B-tree叶结点中找到插入的节点，插入数据之后，再进行1次I/O操作将新的节点数据写回。相对少见的节点拆分对我们的分析影响不大，因此可以忽略它们。那么B-tree的插入代价就是：
$$
\large COST_{B-ins}=COST_{P}(D_{e}+1)
$$
LSM-tree的插入操作首先是直接作用在常驻内存的C0上的，然后是批量地从C0将条目滚动合并到C1中。滚动合并是以多页块作为单元进行合并的，合并时读取一页所需的I/O代价为$COST_π$。为了评价滚动合并的延迟插入的效果，定义：
$$
\large M=从C_0合并到C_1的平均条目数 \\
\large S_e=条目的字节大小 \\
\large S_p=硬盘页的字节大小 \\
\large S_0=C_0的叶子层的大小（以兆字节为单位） \\
\large S_1=C_1的叶子层的大小（以兆字节为单位） \\
$$
一个硬盘页（通常就是一个结点的大小）中所含的条目的数目约为$\frac{S_p}{S_e}$，那么滚动合并时一页中来自于C0的条目约为$\frac{S_0}{S_0 + S_1}$。那么M就可以估计为：
$$
\large M=\frac{S_p}{S_e}\cdot \frac{S_0}{S_0+S_1}
$$
当C0相对于C1的大小越大时，参数M也就越大。对于一次插入来说，LSM-tree在插入C0之后，滚动合并时需要读进C1的叶子结点，再写回C1。由于滚动合并是批量插入C0的条目的，所以一次插入的代价是滚动合并代价的1/M。所以LSM-tree的插入代价：
$$
\large COST_{LSM-ins}=2\cdot COST_\pi /M
$$
将B-tree与LSM-tree的插入代价相比，即可得到：
$$
\large \frac{COST_{LSM-ins}}{COST_{B-ins}}=\frac{2}{M(D_e+1)}\cdot \frac{COST_\pi }{COST_P}=\frac{K_1}{M}\cdot \frac{COST_\pi }{COST_P} \\
\large K_1=\frac{2}{D_e+1}
$$
在实际应用中$D_e$基本等于一个常数，即$K_1$等于常数，所以LSM-tree与B-tree之比取决于M和$\frac{COST_π}{COST_P}$。由于$C_1$的多页块在硬盘中是连续存储的，$COST_π$一般会比$COST_P$来得小，故公式的大小其实取决于M。当一页中包含的条目数较多，$C_0$的规模与$C_1$差距不大时，LSM-tree在数据插入上是优于B-tree的。

一个解决方法就是在日益增大的$C_1$和有空间上限的$C_0$之间加入一个C做为$C_0$和$C_1$的缓冲。这时LSM-tree就由三个部分（$C_1$、C和$C_0$）组成。这种LSM-tree称为多部件的LSM-tree。

## 2.3、多部件的LSM-Tree算法

在数据不断增加的情况下，即使在$C_1$和$C_0$之间增加上一个C，C的规模也会不断增长。当C像过去$C_1$那么大时，就需要在C和$C_0$之间再增加一个C。以此类推，硬盘中的C树将会越来越多。如下图所示：

![multi-component LSM-Tree](http://ww1.sinaimg.cn/large/bda5cd74ly1g8o64h1kaaj20hh058t8z.jpg)

通常，一个多部件的LSM-tree由大小依次递增的$C_0$，$C_1$，$C_2$，...，$C_{K-1}$和$C_K$组成，$C_0$常驻内存之中，可以是键值索引的数据结构。而$C_1$~$C_K$则存储于硬盘之中，但其经常访问的页会被缓存于内存之中，它们的结构都是索引树。在数据不断插入的过程中，当较小的$C_{i-1}$的规模超过某一阈值时，相邻的两个部件$C_{i-1}$和$C_i$会进行滚动合并，从较小的$C_{i-1}$转移条目至较大的$C_i$中。各个相邻部件$C_{i-1}$和$C_i$的滚动合并是异步的。也就是说，一个条目会插入到$C_0$中，之后经过不断的异步滚动合并过程，最终合并至$C_K$中。

## 2.4、LSM-Tree的查找过程

在LSM-tree树进行查找时，为了保证LSM-tree上的所有条目都被检查。首先要搜索$C_0$，再搜索$C_1$，进而搜索$C_2$，...，$C_{K-1}$和$C_K$。即使硬盘中的部件$C_1$，$C_2$，...，$C_{K-1}$和$C_K$的结构都是B-tree，这也将耗费一些时间。

相比B-Tree，LSM-Tree会导致读放大，[这篇文章](https://tikv.org/docs/deep-dive/key-value-engine/b-tree-vs-lsm/)对B-Tree和LSM-Tree的读写性能进行了比较。

一种常见的优化是对每个部件在内存中构建[Bloom Filter](https://en.wikipedia.org/wiki/Bloom_filter)以减少对I/O操作。



https://www.cs.umb.edu/~poneil/lsmtree.pdf

https://blog.acolyer.org/2014/11/26/the-log-structured-merge-tree-lsm-tree/

https://dash.harvard.edu/bitstream/handle/1/37736786/RUTA-DOCUMENT-2017.pdf

https://www.cs.princeton.edu/courses/archive/spr19/cos518/docs/L7-indexing.pdf

http://www.eecs.harvard.edu/~margo/cs165/papers/gp-lsm.pdf

https://kunigami.blog/2018/07/19/log-structured-merge-trees/

http://www.benstopford.com/2015/02/14/log-structured-merge-trees/

https://medium.com/@balrajasubbiah/blsm-a-general-purpose-log-structured-merge-tree-e69d23ad0cd0

https://www.cnblogs.com/haippy/archive/2012/01/14/2322207.html

https://my.oschina.net/u/4064459/blog/2999407

https://www.cnblogs.com/siegfang/archive/2013/01/12/lsm-tree.html

https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/

https://www.slideshare.net/ssuser7e134a/log-structured-merge-tree

https://github.com/wiredtiger/wiredtiger/wiki/Btree-vs-LSM

https://rkenmi.com/posts/b-trees-vs-lsm-trees

https://queue.acm.org/detail.cfm?id=3220266

https://tikv.org/docs/deep-dive/key-value-engine/b-tree-vs-lsm/

https://zhuanlan.zhihu.com/p/65557081

《数据密集型应用设计》https://book.douban.com/subject/30329536/