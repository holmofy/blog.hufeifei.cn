---
title: 单轴快排（SinglePivotQuickSort）和双轴快排（DualPivotQuickSort）及其JAVA实现
date: 2017-05-04
tags:
	- JAVA
	- 算法
keywords:
	- 排序算法
	- 快速排序
	- 单轴快排
	- 双轴快排
	- SinglePivotQuickSort
	- DualPivotQuickSort
categories: 数据结构与算法
---


快速排序使用的是分治思想，将原问题分成若干个子问题进行递归解决。通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比另外一部分的所有数据都要小，然后再按此方法对这两部分数据分别进行快速排序，整个排序过程可以递归进行，以此达到整个数据变成有序序列。

##单轴快排（SinglePivotQuickSort）

单轴快速排序是快速排序最简单的实现。

步骤如下：

1. 如果待排序的数组项数为0或1，直接返回。(递归出口)
2. 在待排序的数组中任选一个元素，作为中心点(pivot)。
3. 将小于pivot的元素，大于pivot的元素划分为开来。也就是将小于中心点的元素放在中心点前面，大于中心点的元素放在中心点后面。
4. 对前面小于pivot的元素进行快速排序，对大于pivot的元素进行快速排序。

![快速排序的基本原理](http://img-blog.csdn.net/20170419214340080?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

根据上面的步骤可以看出，**如何将大于pivot和小于pivot的元素进行划分**是实现快速排序的关键

## 元素划分的方式

###  两端扫描交换方式

![两端扫描交换](http://img-blog.csdn.net/20170504125640821?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**注意** ：i 与 j 必须交错，如果两者相遇之后就停止比较，那相遇点所在的元素就没有和中心点进行比较。

实现代码：

```java
/**
 * 双端扫描交换 Double-End Scan and Swap
 *
 * @param items
 *            待排序数组
 */
public void deScanSwapSort(int[] items) {
	deScanSwapSort(items, 0, items.length - 1);
}

public void deScanSwapSort(int[] items, int start, int end) {
	if (start < end) {
		int pivot = items[start];

		int i = start + 1, j = end;
		while (i <= j) {
			while (i <= j && items[i] < pivot)
				i++;
			while (i <= j && items[j] >= pivot)
				j--;
			if (i <= j) {
				swap(items, i, j);
			}
		}
		swap(items, start, j);// 将中心点交换到中间。
		deScanSwapSort(items, start, j - 1);// 中心点左半部分递归
		deScanSwapSort(items, j + 1, end);// 中心点右半部分递归
	}
}

private void swap(int[] items, int i, int j) {
	int tmp = items[i];
	items[i] = items[j];
	items[j] = tmp;
}
```
### 赋值填充方式 ---- 一端挖坑一端填充

![赋值填充过程图](http://img-blog.csdn.net/20170504125809679?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**注意**：最后 i 和 j 相遇，所在的位置是个坑。

实现代码：

```java
/**
 * 赋值填充方式
 * 一端挖坑一端填充
 *
 * @param items
 *            待排序数组
 */
public void fillSort(int[] items) {
	fillSort(items, 0, items.length - 1);
}

public void fillSort(int[] items, int start, int end) {
	if (start < end) {
		int pivot = items[start];
		int i = start, j = end;
		while (i < j) {
			while (i < j && items[j] > pivot)
				j--;
			items[i] = items[j];
			while (i < j && items[i] <= pivot)
				i++;
			items[j] = items[i];
		}
		// 相遇后i == j，此处是个坑
		items[i] = pivot;
		fillSort(items, start, i - 1);
		fillSort(items, i + 1, end);
	}
}
```
### 单向扫描划分方式

前面的i，j标记都是相向而行，i标记负责找比pivot大的元素，j标记负责比pivot小的元素。下面要说的这种实现方式中思想与前两者不太一样：

1.  初始时，i=start，j=start+1；j 负责扫描整个序列。

  ![初始状态](http://img-blog.csdn.net/20170504125908840?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2.  扫描过程中始终保持：序列中start+1~ i 是小于pivot；i+1~ j 是大于pivot的。

    ![排序原则](http://img-blog.csdn.net/20170504130104962?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3.  为了保持2的特性，j扫描时遇到小于pivot的元素，i++，并将i元素与j元素进行交换，然后扫描下一个元素；

    ![扫描过程](http://img-blog.csdn.net/20170504130204447?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

    遇到大于pivot的元素，直接扫描下一个元素。

    ![扫描过程](http://img-blog.csdn.net/20170504130246682?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4.  整个序列扫描完成后，将第一个元素pivot与小于pivot的元素的最后一个进行交换。

    ![排序结束](http://img-blog.csdn.net/20170504130316557?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

示例过程图如下：

![单向扫描交换方式](http://img-blog.csdn.net/20170504130404807?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

实现代码：

```java
/**
 * 单向扫描划分方式
 *
 * @param items
 *            待排序数组
 */
public void forwardScanSort(int[] items) {
	forwardScanSort(items, 0, items.length - 1);
}

public void forwardScanSort(int[] items, int start, int end) {
	if (start < end) {
		int pivot = items[start];
		int i = start, j = start + 1;
		while (j <= end) {
			if (items[j] < pivot) {
				i++;
				swap(items, i, j);
			}
			j++;
		}
		swap(items, start, i);
		forwardScanSort(items, start, i - 1);
		forwardScanSort(items, i + 1, end);
	}
}
```
### 单轴快排的一种优化方式----三分单向扫描

先来看一个例子：

对于这样一个序列``2,2,2,2,3,1``，我们使用上面提到的单轴快排中最简单的实现对其进行排序：选择第一个元素2作为pivot中心点，划分后得到两段子序列分别为：``1``和``2,2,3,2``，接着继续递归对子序列进行排序，对于``2,2,3,2``子序列又是将2作为pivot中心点... 你会发现对于这种大量元素等于pivot的序列，单轴快排并没有起到很好的划分作用。如果我们将等于pivot的元素也作为一个划分区段，则可以将序列划分为3段：小于pivot的元素，等于pivot的元素，大于pivot的元素。看下图：

![单轴快排的优化](http://img-blog.csdn.net/20170504130434676?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

很明显的看出这种处理方式，会大大节省递归次数。

如何实现该算法呢，很显然我们不能使用像上面单轴快排中前两种相向扫描的方式，我们要结合上面的几种实现方式----单向扫描,双向靠拢，看下面的很容易理解。不过为了将序列划分为三个区段我们需要三个变量i，j，k。大致过程如下：

1. 初始化时，i=start，j=end，k=start+1。k负责扫描。

   ![三分单轴快排初始化状态](http://img-blog.csdn.net/20170504130519224?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2. 扫描过程中始终保持：start~i是小于pivot的元素，i~k是等于pivot的元素，j~end是大于pivot的元素

   ![扫描过程](http://img-blog.csdn.net/20170504130605834?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3. 扫描过程中，遇到小于pivot的元素，i与k元素进行交换，i++，然后k扫描下一个元素；

   ![扫描过程遇到小于pivot的元素](http://img-blog.csdn.net/20170504130652210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

   遇到大于pivot的元素，k与j交换，j--，k不需加一，继续扫描k处元素。

   ![扫描过程中遇到大于pivot的元素](http://img-blog.csdn.net/20170504130723055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

   扫描过程遇到等于pivot的元素，直接扫描下一个元素。

   ![扫描过程遇到等于pivot元素](http://img-blog.csdn.net/20170504130753884?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

   4. pivot已经包含在等于pivot的分段中，无需交换。最后k>j的时候停止扫描。

实现代码：

```java
/**
 * 三分单向扫描
 */
public void div3ScanSort(int[] items) {
	div3ScanSort(items, 0, items.length - 1);
}

public void div3ScanSort(int[] items, int start, int end) {
	if (start < end) {
		int pivot = items[start];
		int i = start, j = end, k = start + 1;
		while (k <= j) {
			if (items[k] < pivot) {
				swap(items, i, k);
				i++;
				k++;
			} else if (items[k] > pivot) {
				swap(items, j, k);
				j--;
			} else {
				k++;
			}
		}
		div3ScanSort(items, start, i - 1);
		div3ScanSort(items, j + 1, end);
	}
}
```

### 另一种优化----三分双向扫描

在上面的实现中，扫描到大于pivot的元素，将最后一个未扫描的元素(j所在的位置)与当前元素(k所在的位置)进行交换。那如果这个未扫描的元素正好是比pivot大的元素呢，这无疑增加了交换的次数。

所以j索引应当扫描到一个不比pivot大的元素，再做判断，如果==pivot，将k与j进行交换，如果<pivot，将k与j进行交换，然后k与i进行交换。

![三分双端扫描](http://img-blog.csdn.net/20170504130832088?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

实现代码：

```java
/**
 * 双端扫描三分排序
 */
public void div3DeScanSort(int[] items) {
	div3DeScanSort(items, 0, items.length - 1);
}

public void div3DeScanSort(int[] items, int start, int end) {
	if (start < end) {
		int pivot = items[start];
		int i = start, j = end, k = start + 1;

		OUT_LOOP: while (k <= j) {
			if (items[k] < pivot) {
				swap(items, i, k);
				i++;
				k++;
			} else if (items[k] == pivot) {
				k++;
			} else {
				// j向左扫描，直到一个不大于pivot的元素
				while (items[j] > pivot) {
					j--;
					if (k > j) {
						// 后面的待排元素全大于pivot，直接结束排序
						break OUT_LOOP;
					}
				}
				if (items[j] < pivot) {
					swap(items, j, k);
					swap(items, i, k);
					i++;
				} else {
					swap(items, j, k);
				}
				k++;
				j--;
			}
		}
		div3DeScanSort(items, start, i - 1);
		div3DeScanSort(items, j + 1, end);
	}
}
```

##双轴快排（DualPivotQuickSort）

## 双轴快排思想

理解了前面的三分单向扫描和三分双向扫描，双轴快速排序就很好理解了。

双轴快速排序，顾名思义，取两个中心点pivot1，pivot2，且pivot≤pivot2，可将序列分成三段：`x<pivot1、pivot1≤x≤pivot2，x<pivot2`，然后分别对三段进行递归。基本过程如下图：

![双轴快排基本原理](http://img-blog.csdn.net/20170504130901919?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

既然要两个中心点，我们一般将第一个元素和最后一个元素作为两个中心点。实现大致过程如下：

1. 初始化时，i=start，j=end，k=start+1，k负责扫描。序列第一个值大于序列最后一个值，需要进行交换。然后pivot1=items[start]，pivot2=items[end]。

2. 扫描过程中保持：1~i是小于pivot1的元素，i~k是大于等于pivot1、小于等于pivot2的元素，j~end-1是大于pivot2的元素。

   ![双轴快排扫描过程](http://img-blog.csdn.net/20170504130926371?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

3. 扫描过程与前面的三分双向扫描类似。

4. 最后扫描完成，将pivot1与pivot2移到中间（这和之前讲的都差不多，就不在这进行过多的解释了）。

   ![扫描完成](http://img-blog.csdn.net/20170504130948590?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

实现代码：

```java
/**
 * 双轴快排
 *
 * @param items
 */
public void dualPivotQuickSort(int[] items) {
	dualPivotQuickSort(items, 0, items.length - 1);
}

public void dualPivotQuickSort(int[] items, int start, int end) {
	if (start < end) {
		if (items[start] > items[end]) {
			swap(items, start, end);
		}
		int pivot1 = items[start], pivot2 = items[end];
		int i = start, j = end, k = start + 1;
		OUT_LOOP: while (k < j) {
			if (items[k] < pivot1) {
				swap(items, ++i, k++);
			} else if (items[k] <= pivot2) {
				k++;
			} else {
				while (items[--j] > pivot2) {
					if (j <= k) {
						// 扫描终止
						break OUT_LOOP;
					}
				}

				if (items[j] < pivot1) {
					swap(items, j, k);
					swap(items, ++i, k);
				} else {
					swap(items, j, k);
				}
				k++;
			}
		}
		swap(items, start, i);
		swap(items, end, j);

		dualPivotQuickSort(items, start, i - 1);
		dualPivotQuickSort(items, i + 1, j - 1);
		dualPivotQuickSort(items, j + 1, end);
	}
}
```



##各种实现速度大比拼

我们测试的序列长度为10000

```java
private final int[] testItems = new int[10000];
```

为了更精确的测量耗时，我们使用的单位是纳秒

```java
start = System.nanoTime();
xxxSort(tmp);
end = System.nanoTime();
```

## 对多重复元素的序列进行排序

```java
for (int i = 0; i < testItems.length; i++) {
	testItems[i] = (int) (Math.random() * 100);
}
```

这里我们随机生成100以内的数字，这个序列中肯定有大量重复的数字，这里取5次测试

```java
forwardScanSort:   5478840  5378672  5097875  3898327  5543293   平均：5077401
fillSort :         6060962  5830248  6263760  5737880  6264992   平均：6031568
deScanSwapSort:    8733057  5184906  9776196  6462043  8275323   平均：7686305
div3ScanSort:      3749307  4663541  4904929  4225924  4642604   平均：4437261
div3DeScanSort:    3935273  4457049  4770688  3951695  4396291   平均：4302199
dualPivotQuickSort:8891518  5160274  5430809  4968971  5393451   平均：5969004
Arrays.sort:       3513666  3856864  3460299  3684855  3888063   平均：3680749
```

## 对稀疏序列进行排序

这次生成的随机数是10万以内的。

```java
for (int i = 0; i < testItems.length; i++) {
	testItems[i] = (int) (Math.random() * 100000);
}
```

测试结果

```java
forwardScanSort:   3088775 3257911 3402414 2415518 3579351   平均：3148793
fillSort :         3606444 3495603 3665970 2871609 6503916   平均：4028708
deScanSwapSort:    9766343 9425198 4219766 3890937 7017070   平均：6863862
div3ScanSort:      5138927 4565016 6069582 4714857 5053538   平均：5108384
div3DeScanSort:    4303513 4722246 5923026 3613423 4158188   平均：4544079
dualPivotQuickSort:5681229 3971810 4823645 3376963 4739078   平均：4518545
arrays.sort:       5820806 5953815 6402928 7119290 7506824   平均：6560732
```

> Arrays.sort底层使用的也是DualPivotQuickSort，这个类对双轴快排在策略上进行了一个改动（不仅仅是双轴快排，还是用到了其他的排序，如直接插入排序，对于byte，char，short基本类型还用到了计数排序）。关于其他排序算法的实现可以参考[这篇文章](http://blog.csdn.net/holmofy/article/details/70245895)。

代码地址：https://github.com/holmofy/algorithm/tree/master/QuickSort


>  **参考文章：**
>
>  * Wiki：https://en.wikipedia.org/wiki/Quicksort#Variants
>  * 3-WAY AND DUAL PIVOT：http://rerun.me/2013/06/13/quicksorting-3-way-and-dual-pivot/
>  * 双轴快排：http://www.cnblogs.com/nullzx/p/5880191.html
