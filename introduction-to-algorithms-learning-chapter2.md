---
title: 算法导论研读与分析（二）
date: 2013-06-23 10:00:00
tags: 算法导论，算法分析，最大子数组，分治，动态规划
author: maclaon
comments: false
---
# 最大子数组
## 整体性学习引入
股票价格波动中，每天价格都会上涨下跌，那么如何在有效的时间内买入和卖出获取最大的收益？关键点是确定买入与卖出的时间点(脑洞一下可以利用该模型结合机器学习的理论计算股市中改区间的存在)，在时间间隔内其收益是最大的。数学模型转化为:
> 在给定的一个整形数组中，数组的值有正负值，如何在其中找出一个区段，使得这个区段中所有数组的值相加达到最大。

<!--more-->
## 算法设计
### 暴力破解
暴力破解主要计算数组中每个子范围数组中的和

$$\sum_{i=j}^{k}a[i],0\leq j \leq k \leq n$$
并记录索引位置，如果当前的子数组的和比记录的值要大，那么替换对应的值，并重新标记数组。那么可以得出上述数组的时间组合有
$$C_2^n\approx\Theta(n^2)$$

### 分治策略
假设数组$$A[low...high]$$
+ 将数组划分成$$2$$个规模相等的子数组，找到子数组的中央$$mid$$
+ 然后考虑如下两个子数组$$A[1...mid], A[mid+1...high]$$

分治算法很容易放在数组的存储结构中，所以**存储结构(可以扩展)**是其一个特点，那么上述的步骤就是可以循环的执行分治策略的算法，因为$$A[low...high]$$的任何子数组的问题的解必然存在于如下三个子数组中：
+ 完全位于子数组$$A[low...mid]$$中，$$low\leq i \leq j \leq mid$$
+ 完全位于子数组$$A[mid+1...high]$$中，$$mid+1 \leq i \leq j \leq high$$
+ 跨越中点，$$low\leq i \leq mid \leq j \leq high$$

可以得出结论，最大连续子数组的和必然是上述三个位置中最大的一个，分解完后合并各个子问题的解即可。
其实最主要的是要建立上述**分解模型，如果模型不够简单的或者覆盖不到位的话，得出的结果集也是错误的**。

其中**跨越中点**这个**问题并非原问题规模更小的实例**，因为它有一个限制--求出的子数组必须跨越中点的最大子数组。这里不能划分的原因是它不是原问题规模更小额实例，所以我们**要实现它**。

> 此时我们要解决的问题的焦点就是: 最大连续子数组的值，必然位于跨越中点的场景，其起点必然是再$$mid$$的左边，终点也必然是$$mid$$的右边。这一点很重要，请看[分治算法总结](http://shieldme.cn/2013/08/11/introduction-to-algorithms-learning-chapter3/)，不然会陷入思维循环中去不能自拔。

#### 跨越中间节点的代码片段
``` Java
public int findMaxCrossingArraySum(List<Integer> A, int left, int mid ,int right){
	int leftMaxSum = -1;
	int leftMaxIndex = 0;
	int leftTempMaxSum = 0;
	for(int leftIndex = mid; leftIndex > left; leftIndex--){
		leftTempMaxSum += A.get(leftIndex);
		if(leftTempMaxSum > leftMaxSum){
			leftMaxSum = leftTempMaxSum;
			leftMaxIndex = leftIndex;
		}
	}
	
	int rightMaxSum = -1;
	int rightMaxIndex = 0;
	int rightTempMaxSum = 0;
	for(int rightIndex = mid + 1; rightIndex < right; rightIndex++){
		rigthTempMaxSum += A[rightIndex]
		if(rightTempMaxSum > rightMaxSum){
			rightMaxSum = rightTempMaxSum;
			rightMaxIndex = rightIndex;
		}
	}
	
	return leftMaxSum + rightMaxSum;
}
```

#### 跨越中间节点的代码片段时间复杂度分析
按照[算法分析](http://shieldme.cn/2013/05/12/introduction-to-algorithms-learning-chapter1/)中对代码执行次数进行标记。
其中跨越中间节点的代码片段时间复杂度分析如下：
两个`for`循环的执行次数为$$(mid - low + 1) + (right - (mid + 1) + 1) = high - low + 1 = n$$，可以得出其时间复杂度为$$\Theta(n)$$

#### 最大连续子数组和代码片段
``` Java
public int findMaxArraySum(List<Integer> A, int low, int high){
	if(low == high){
		return A.get(low);
	}
	
	int mid = (low + high) / 2;
	int leftMaxSum = findMaxArraySum(A, low, mid);
	int rightMaxSum = findMaxArraySum(A, mid, right);
	int cossingMaxSum = findMaxCrossingArraySum(A,low,mid,high);
	
	if(leftMaxSum >= rightMaxSum && lefMaxSum >= cossingMaxSum){
		return leftMaxSum;
	}else if(rightMaxSum >= leftMaxSum && rightMaxSum >= cossingMaxSum){
		return rightMaxSum;
	}else{
		return cossingMaxSum;
	}
}
```

#### 最大连续子数组和代码片段时间复杂度分析
按照[算法分析](http://shieldme.cn/2013/05/12/introduction-to-algorithms-learning-chapter1/)中对未知时间复杂度的可以设置数学符号，后续再进行分析，那么这里假设子数组的时间复杂度为$$T(n)$$，从**整个算法的组成部分来**看，真个时间复杂度为:

$$T(n)=2T(n/2)+\Theta(n)$$

对于数学递归式子的求解请看[数学递归求解]()。

### 动态规划
稍后继续!