K-D树在[维基百科](https://en.wikipedia.org/wiki/K-d_tree)上定义是将K维空间中的点进行分割的数据结构，D是*dimensional*(维度)的缩写，K-D树是BSP(Binary Space Partitioning)的一种。

维基百科的解释很正式(看的一头雾水)。简单的说，**K-D树就是二分查找树在K维空间的泛化**。

# 1、K维空间的二分查找树

[之前的一篇文章中有讲过二分查找树(BST)](https://blog.csdn.net/Holmofy/article/details/79692613)这样基础的数据结构，它是基于二分查找的思想实现$O(log_2N)$的插入和查找速度。

如果把BST中的所有元素看成一维线段上的所有点，从树的根节点开始每个节点都会把线段分成两段。

![BST](http://ww1.sinaimg.cn/large/bda5cd74ly1fyh4bu7lfzj20j409bdgg.jpg)

> BST就是一维K-D树(或者叫做1-D树)

现在将树中的元素推广到二维平面上的点，树的每一层按照维度轮流划分。比如，奇数层按x轴划分，偶数层按y轴划分。

![KD-Tree](http://ww1.sinaimg.cn/large/bda5cd74ly1fyh5gpy8hgj221a0vs79r.jpg)

同样我们可以将元素推广到三维空间中的点，那样我们可以得到一棵3d-tree。

![3d-tree](http://ww1.sinaimg.cn/large/bda5cd74ly1fyh6jc4tmpj20f80eh752.jpg)

> 理论上k-d树可以继续到4维，5维… 作为一个三维空间里的生物，就没法用图来展示这样的元素了:relieved:

[这里有一个二维K-D树的模拟程序](http://bl.ocks.org/ludwigschubert/raw/0a300d8a41d9e6b49144fff8b62637f8/)：

![](http://ww1.sinaimg.cn/large/bda5cd74ly1fyq46dzx9lg20jg09s43v.gif)

**优点：**

很明显，K-D树和BST一样不仅有hash一样的精确查找，也更适合做范围查询，但K-D树比BST更强，它能对多个维度进行范围查询。

比如：

Person1(age:18,hight:176)，Person2(age:19,height:181) ... Person10(age:23,height:171)

想要查找年龄在20岁以上并且身高在170到180之间的所有人，用K-D树就能很好的解决。

![k-d-tree range search](http://ww1.sinaimg.cn/large/bda5cd74ly1fz0l8w5prlj208d08d3yg.jpg)

> 由于K-D树的结构要求，上面的例子中要求每个人的年龄和身高数据必须齐全，如果某个人只有年龄或只有身高，就无法使用K-D树索引了。所以实际上K-D树经常用于经纬度定位，或者三维空间定位(某个维度数据缺失，其他维度数据也就没有意义了)

**缺点：**

和二分查找树一样，K-D树也有平衡性问题，这个问题下面会进行详细讨论。

高纬度数据查找效率并不一定好，有时候可能不如最原始的暴力查找。通常，如果维度为k，则k-d树中的点数N应远远大于$2^k$ (即$N \gg 2^k$)。否则查找时，节点中大多数节点都会被遍历到，效率上还不如原始的遍历。

# 2、K-D树的平衡问题

既然K-D树是BST在k维空间的推广，那K-D树自然也有[BST存在的平衡性问题——二叉树退化成链表](https://blog.csdn.net/Holmofy/article/details/79692613)。

![平衡性问题](http://ww1.sinaimg.cn/large/bda5cd74ly1fyq5ity4e2g21hs0r0gq9.gif)

但是由于K-D树按多个维度划分子树，相邻的层级之间不在同一个维度，因此AVL、RB-Tree等平衡二叉搜索树的[旋转技术](https://en.wikipedia.org/wiki/Tree_rotation)不能用于K-D树。

有一种做法是重建K-D树：将已知的所有点作为输入，**每一层按照该层维度排序，并找到中位数的那个点作为子树的根节点**。这个过程用伪代码表示：

```pseudocode
function kdtree (list of points pointList, int depth)
{
    // 该层的维度，2-d树的第1层就是x
    var int axis := depth mod k;
        
    // 按照该层维度选取中位数(这个过程中可能需要排序)
    select median by axis from pointList;
    
    node.location := median; // 构造节点
    // 构造左右子树
    node.leftChild := kdtree(points in pointList before median, depth+1);
    node.rightChild := kdtree(points in pointList after median, depth+1);
    return node;
}
```

> 上面这段为代码摘抄自维基百科，维基百科上有相应的Python实现，在[上面的2-d树模拟程序](http://bl.ocks.org/ludwigschubert/raw/0a300d8a41d9e6b49144fff8b62637f8/)中有相应的JS版实现。

很明显这种方式只适用于输入点已知的静态数据。对于频繁插入删除操作的动态数据，每次重建K-D树成本过大。

# 3、K-D-B树

**[K-D-B树](https://en.wikipedia.org/wiki/K-D-B-tree)是K-D树与B树的结合。**

我[之前有一篇文章](https://blog.csdn.net/Holmofy/article/details/79830773)介绍过B树，它主要通过**增加子节点个数、减少树的层级**解决了平衡二叉树磁盘访问效率的问题，同时B树本身也是一种自平衡树。

K-D-B树汲取了B树的优点：**提供平衡K-D树的搜索效率，同时提供和B树一样面向块的存储以优化外存设备的访问**。

以二维K-D-B树为例，如果最大子节点个数为4的话，数据结构图如下：

![k-d-b tree](http://ww1.sinaimg.cn/large/bda5cd74ly1fz0fck5as8j20dw07iaa7.jpg)



## 3.1、K-D-B树页分裂的效率问题

B树如果节点中的子节点树超过规定的阶数，就会发生节点分裂（数据库中经常称之为“页”分裂）。

![B树的页分裂](http://ww1.sinaimg.cn/large/bda5cd74gy1fqbiffgiumg20e1065asg.gif)

B树属于一维数据（数据只按一种规则排序），而K-D-B树数据有多个维度，所以页分裂的时候不能以B树“一刀切”的方式解决。

以二维K-D-B树为例，一个节点数据过多，需要分裂时，就需要拆分区域。拆分区域的过程中大概率会出现拆分该区域的子区域。



![](http://ww1.sinaimg.cn/large/bda5cd74ly1fz0fd8nkcjj20hg0eqaa1.jpg)

K-D-B树的这个操作修改的节点比较多，由于写入磁盘的速度比较慢，这个操作会相对耗时，这也是K-D-B树最主要的缺点。

# 4、BKD树

BKD树是杜克大学和普渡大学的几位教授在[论文](https://users.cs.duke.edu/~pankaj/publications/papers/bkd-sstd.pdf)中提出的，它主要是为了解决空间利用率和增删操作的速度问题。

BKD树由多个可修改的K-D树组成，并提供了一个独特的插入方法。

BKD树的节点组成了一个完全二叉树，这样的好处是节点不需要存储指向子节点的指针，根据父节点索引即可算出子节点索引。内部节点本身不包含数据，所有数据存储在叶子节点。较小的节点也意味着内存中可以缓存更多的节点。另外内部节点永远不会被修改，而是使用一种策略来添加新的节点。

# 5、K-D树算法的应用



## 最近邻搜索



## Lucene数值类型的索引

org.apache.lucene.util.bkd.BKDWriter



https://www.elastic.co/cn/blog/lucene-points-6.0

https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/KDtree.html

https://medium.com/%40nickgerleman/the-bkd-tree-da19cf9493fb

https://lucene.apache.org/core/7_1_0/core/org/apache/lucene/util/bkd/package-summary.html

https://courses.engr.illinois.edu/cs225/fa2017/mps/5/

https://en.wikipedia.org/wiki/K-D-B-tree

https://en.wikipedia.org/wiki/K-d_tree

https://www.slideshare.net/lucidworks/the-evolution-of-lucene-solr-numerics-from-strings-to-points-presented-by-steve-rowe-lucidworks

https://blog.csdn.net/njpjsoftdev/article/details/54015485

https://www.cs.cmu.edu/~ckingsf/bioinfo-lectures/kdtrees.pdf

https://webcms3.cse.unsw.edu.au/COMP9315/16s1/resources/2348

https://yq.aliyun.com/articles/581877