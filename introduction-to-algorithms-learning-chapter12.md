---
title: 算法导论研读与分析（十二）
date: 2013-09-30 10:00:00
tags: 数据结构，图，遍历
author: maclaon
comments: false
---
# 图遍历
图的遍历可以根据数据结构的定义来处理，前篇[图的分类和存储](http://shieldme.cn/2013/09/23/introduction-to-algorithms-learning-chapter11/)中有邻接矩阵和邻接链表两种形式
## 深度优先算法

+ 访问指定的起始顶点；
+ 若当前访问的顶点的邻接顶点有未被访问的，则任选一个访问之；反之，退回到最近访问过的顶点；直到与起始顶点相通的全部顶点都访问完毕；
+ 若此时图中尚有顶点未被访问，则再选其中一个顶点作为起始顶点并访问之，转 2； 反之，遍历结束

### 无向图
![](https://raw.githubusercontent.com/wangkuiwu/datastructs_and_algorithm/master/pictures/graph/iterator/01.jpg)

<!--more-->

![](https://raw.githubusercontent.com/wangkuiwu/datastructs_and_algorithm/master/pictures/graph/iterator/02.jpg)

### 有向图
![](https://raw.githubusercontent.com/wangkuiwu/datastructs_and_algorithm/master/pictures/graph/iterator/03.jpg)

![](https://raw.githubusercontent.com/wangkuiwu/datastructs_and_algorithm/master/pictures/graph/iterator/04.jpg)

## 广度优先算法
从图中某顶点$$v$$出发，在访问了$$v$$之后依次访问$$v$$的各个未曾访问过的邻接点，然后分别从这些邻接点出发依次访问它们的邻接点，并使得“先被访问的顶点的邻接点先于后被访问的顶点的邻接点被访问，直至图中所有已被访问的顶点的邻接点都被访问到。如果此时图中尚有顶点未被访问，则需要另选一个未曾被访问过的顶点作为新的起始点，重复上述过程，直至图中所有顶点都被访问到为止。

### 无向图
![](https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/pictures/graph/iterator/05.jpg?raw=true&_=3711483)

![](https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/pictures/graph/iterator/06.jpg?raw=true&_=3711483)

## 拓扑排序
在图论中，拓扑排序（Topological Sorting）是一个有向无环图（DAG, Directed Acyclic Graph）的所有顶点的线性序列。且该序列必须满足下面两个条件：
+ 每个顶点出现且只出现一次
+ 若存在一条从顶点$$A$$到顶点$$B$$的路径，那么在序列中顶点$$A$$出现在顶点$$B$$的前面

下面的图为：
![](http://img.blog.csdn.net/20150507001028284)

它是一个$$DAG$$图，那么如何写出它的拓扑排序呢？这里说一种比较常用的方法：
+ 从 DAG 图中选择一个 没有前驱（即入度为0）的顶点并输出
+ 从图中删除该顶点和所有以它为起点的有向边
+ 重复 1 和 2 直到当前的 DAG 图为空或当前图中不存在无前驱的顶点为止。后一种情况说明有向图中必然存在环

上图的结果如下:
![](http://img.blog.csdn.net/20150507001759702)

于是，得到拓扑排序后的结果是$${1,2,4,3,5}$$。

> 一个有向无环图可以有一个或多个拓扑排序序列.

### 应用
> 通常用来“排序”具有依赖关系的任务。

+ 如果用一个DAG图来表示一个工程，其中每个顶点表示工程中的一个任务，用有向边 表示在做任务$$B$$之前必须先完成任务$$A$$。故在这个工程中，任意两个任务要么具有确定的先后关系，要么是没有关系，绝对不存在互相矛盾的关系（即环路）
+ 一本书的作者将书本中的各章节学习作为顶点，各章节的先学后修关系作为边，构成一个有向图。按有向图的拓扑次序安排章节，才能保证读者在学习某章节时，其预备知识已在前面的章节里介绍过。

![](http://images.cnblogs.com/cnblogs_com/xiaosuo/DataStructure/79.jpg)

## 强连通分量
在有向图G中,如果两个顶点间至少存在一条路径,称两个顶点强连通(strongly connected),如果有向图G的每两个顶点都强连通，称G是一个强连通图.
>从图G内任意一个点出发,存在通向图G内任意一点的的一条路径。

如图所示,蓝色框圈起来的是一个强连通分量

![](https://i2.wp.com/comzyh.com/upload/wordpress//2010/11/SCC.png?fit=300%2C240&ssl=1)

### 目的
>由于强连通分量内部的节点性质相同,可以将一个强连通分量内的节点缩成一个点,即消除了环,这样,原图就变成了一个有向无环图(directed acyclic graph,DAG)

