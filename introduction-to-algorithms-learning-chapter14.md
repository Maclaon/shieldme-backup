---
title: 算法导论研读与分析（十四）
date: 2013-10-18 10:00:00
tags: 图，最大流
author: maclaon
comments: false
---
#最大流
可以将道路交通图模型化为有向图来找到从一个城市到另一个城市之间的最短路径，我们也可以将一个有向图看做是一个“流网络”并使用它来回答关于物料流动方面的问题。设想一种物料从产生它的源节点经过一个系统，流向消耗该物料的汇点这样一个过程。

> 从源点到经过的所有路径的最终到达汇点的所有流量和。可能是最小费用，最大流量这样的问题，比如说物流的信息。

## 问题
> + 有一个自来水管道运输系统，起点是$$s$$，终点是$$t$$，途中经过的管道都有一个最大的容量。求从$$s$$到$$t$$的最大水流量是多少？
> + $$n$$辆卡车要运送物品，从$$A$$地到$$B$$地，由于每条路段都有不同的路费要缴纳，每条路能容纳的车的数量有限制，最小费用最大流问题指如何分配卡车的出发路径可以达到费用最低，物品又能全部送到。

<!--more-->

## 定义
流网络$$G=(V,E)$$是一个有向图，其中每条边$$(u,v)\in E$$均有一个非负容量$$c(u,v)>=0$$。如果$$(u,v)$$不属于$$E$$，则假定$$c(u,v)=0$$。流网络中有两个特别的顶点: 源点$$s$$和汇点$$t$$。如下图:

![](http://img.blog.csdn.net/20140410224125500)

## 特性
对一个流网络$$G=(V,E)$$，其容量函数为$$c$$，源点和汇点分别为$$s$$和$$t$$。$$G$$的流$$f$$满足下列三个性质：
> + 容量限制：对所有的$$u,v\in V$$，要求$$f(u,v)\le c(u,v)$$。
> + 反对称性：对所有的$$u,v\in V$$，要求$$f(u,v)=-f(v,u)$$。
> + 流守恒性：对所有$$u\in V-\lbrace s,t\rbrace，要求\sum f(u,v)=0, (v\in V)$$。 

容量限制说明了从一个顶点到另一个顶点的网络流不能超过设定的容量，就好像是一个管道只能传输一定容量的水，而不可能超过管道体积的限制；反对称性说明了从顶点u到顶点v的流是其反向流求负所得，就好像是当参考方向固定后，站在不同的方向看，速度一正一负；而流守恒性说明了从非源点或非汇点的顶点出发的点网络流之和为0，这有点类似于基尔霍夫电流定律，且不管什么是基尔霍夫电流定律，通俗的讲就是进入一个顶点的流量等于从该顶点出去的流量，如果这个等式不成立，必定会在该顶点出现聚集或是枯竭的情况，而这种情况是不应该出现流网络中的，所以一般的最大流问题就是在不违背上述原则的基础上求出从源点$$s$$到汇点$$t$$的最大的流量值，显然这个流量值应该定义为从源点出发的总流量或是最后聚集到t的总流量，即流$$f$$的值定义为

$$|f|=\sum_{v\in V} f(s,v) - \sum_{v\in V} f(v,s)$$

如果没有流量进入到$$s$$中的时候，上述的:

$$\sum_{v\in V} f(v,s) = 0$$

现实生活中可能是多$$s,t$$的节点，这样的网络就比较复杂了。

## 概念
> + 流量和容量：节点存储的量是容量，网络中流向的是流量
> + 残存网络：给定流网络$$G_f$$和流量$$f$$，残存网络$$G_f$$由那些仍有**空间对流量进行调整的边构成**。流网络的一条边可以允许的额外流量等于该边的容量减去该边上的流量。如果该差值为正，则将该条边置于图$$G_f$$中，并将其残存容量设置为$$c_f(u,v)=c(u,v)-f(u,v)$$。对于图$$G$$中的边来说，只有能够允许额外的流量的边才能加入到图$$G_f$$中。如果边$$(u,v)$$的流量等于其容量，则其$$c_f(u,v)=0$$，该条边不属于图$$G_f$$。残存网络$$G_f$$还可能包含图$$G$$中不存在的边。算法对流量进行操作的目标是增加总流量，算法可能对某些特定边上的流量进行缩减，那么可以标识一个正流量$$f(u,v)$$的缩减，可以将边$$(v,u)$$加入到图$$G_f$$中，并将残存容量设置为$$c_f(v,u)=f(u,v)$$，一条边的反向流量最多将其正向流量抵消。如下图:
> ![](http://img.blog.csdn.net/20130725155231640)![](http://img.blog.csdn.net/20130725155303421)
> + 残存容量：根据上述的流网络中残存网络的定义，可以得出残存容量$$c_f(u,v)=\{_f(v,u)^c(u,v)-f(u,v)$$
> + 增广路径：增广路径$$p$$是**残存网络$$G_f$$中一条从源节点$$S$$到汇点$$T$$的简单路径**。我们可以增加其流量的幅度最大为$$c_f(u,v)$$中，而不会违反原始流网络$$G$$中对边$$(u,v)$$或$$(v,u)$$的容量限制，下图是对残存网络$$G_f$$中一条增广路径，因为上述的增广路径可以在每条边上增加$$4$$个单位(最小值)，而不会违反容量限制。
> ![](http://img.blog.csdn.net/20130725155342015)
> + 流网络切割:

## 算法
> 最大流的最大, 说的是流的最大; 最小割的最小, 说的是容量的最小。残留网络$$G_f$$中不包含增广路径时，$$f$$就是$$G$$的最大流（或者说，最大流的流量等于最小割的容量）。

### 定理
> 最大割最小切切割定理：设$$f$$为流网络$$G=(V,E)$$中的一个流，该流网络的源节点为$$s$$，汇点为$$t$$，则下面的条件是等价的：
> + $$f$$是$$G$$的一个最大流
> + 残存网络$$G_f$$不包括任何增广路径。
> + $$|f|=c(S,T)$$，其中(S，T)是流网络$$G$$的某个切割。

### 描述
Ford-Fulkerson是一种求解最大流的方法，依赖于上面积淀的基础知识（主要是残留网络、增广路径、割的功劳），也称作“扩充路径方法”。之所以称之为方法而不是算法，是因为这个只是一种指导思想，在此指导之下，有很多种实现方式.

> Ford-Fulkerson是一种迭代法，过程如下：
> + 流网络中所有顶点对的流大小清零（此时，网络流为零）
> + 每次迭代，通过寻找一条增广路径来增加流的值
> + 无法找到增广路径时，迭代结束

可以看到，最关键问题是如何寻找增广路径，而$$Ford-Fulkerson$$方法的效率正取决于此。如果选择方法不好，就有可能每次增加的流非常少，而算法运行时间非常长，甚至无法终止。