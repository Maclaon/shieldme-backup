---
title: 算法导论研读与分析（四）
date: 2013-07-29 10:00:00
tags: 堆排序，优先级队列
author: maclaon
comments: false
---
# 堆
使用一种“堆”的数据结构来进行信息管理，除此之外还可以构造有效的优先级队列。最大（小）堆中除了根节点以外的所有节点都要满足$$A[parent(i)]\geq A[i], A[parent(i)]\leq A[i]$$其中**最大堆常用于堆排序中，最小堆用于优先级队列**。
> 特性：最大（小）堆的特性是位于根节点的元素始终是该树结构中最大(小)的元素。

## 堆排序
堆排序在满足堆特性的数据结构中选择根元素逐步加入到新排序的数据结构中去(可以是原来的数组，比如讲最大堆的数组放在末尾)

### 建堆的代码片段
为了维护堆的性质，必须要新加入堆的$$A[i]$$的值在最大堆中“逐级下降”，使得以下标$$i$$为根节点的子树重新遵循最大堆的性质。建堆的代码可以根据原始的数组进行子模块的划分，比如我们使用数组的元素，那么就可以书写子函数获取当前$$i$$所对应的左边元素和右边元素。
<!--more-->

建堆的过程满足分治算法的思想：
+ 分解初始堆中每个问题都是左右子节点和父节点进行比较然后将最大值与父节点进行比较的过程
+ 比较左右子节点和父节点三者中哪个是最大的值，然后交换他们的位置，每次对调整产生的位置都需要继续调整
+ 合并所有结果集

程序每一步从$$A[i]、A[LEFT(i)]、A[RIGHT(i)]$$中选择最大的，并将其下标存储与$$maxIndex$$中；如果$$A[i]$$是最大的，那么以$$i$$为根节点的子树已经是最大堆，程序结束。否则，交换$$A[i]$$和$$A[maxIndex]$$的值，保证以i为根节点的左右子树保持最大堆的性质，但是交换后，下标$$maxIndex$$的节点值是原来的$$A[i]$$，可能有违背最大堆的性质，需要对该子树递归调用。**整个数组中的自底向上每个元素都需要调用上述的方法来对其进行调整，这样才能达到整个数组是堆的性质**。

``` java
public List<Integer> MaxHeapify(List<Integer> array, int insertIndexOfArray){
	int maxVauleIndex = 0;
	int leftValue = array.get(insertIndexOfArray/2);
	int rightValue = arrary.get(insertIndexOfArray/2 + 1);
	if(leftValue >= array.get(insertIndexOfArray)){
		maxValueIndex = insertIndexOfArray/2;
	}else{
		maxValueIndex = insertIndexOfArray;
	}
	
	if(rightValue >= array.get(maxValueIndex)){
		maxValueIndex = insertIndexOfArray/2 + 1;
	}
	
	if(maxValueIndex != insertIndexOfArray){
		int temp = array.get(insertIndexOfArray);
		array.get(insertIndexOfArray) = array.get(maxValueIndex);
		array.get(maxValueIndex) = temp;
		
		MaxHeapify(array, maxValueIndex);
	}
	return array;
}
```

上述是维护堆特性的代码，那么需要对堆中二叉树非叶子节点的层级数$$\log_2(array.size())$$开始，逐步的递减到1，去进行调整。其时间复杂度为$$\Theta(n)$$
### 堆排序
以数组为例，每次拿出最大堆的根节点放在数组的末端，交换末端的数组到根节点，删除堆中交换的节点，调整最大堆的顺序后，继续剔除根节点的，这样每次拿出的数据都已经是当前数组中最大的，最后得出一个有序的数组。时间负责度为$$\Theta(n\lg n)$$

![示例图](http://img.blog.csdn.net/20150606133355179)
## 优先级队列


