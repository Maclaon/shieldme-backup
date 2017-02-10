---
title: 深入理解JVM虚拟机之垃圾回收器的分类
date: 2015-07-05 10:00:00
tags: 深入理解JVM虚拟机，垃圾回收器，内存分配策略
author: maclaon
comments: false
---
# 垃圾回收器
垃圾回收器是基于前一章节[垃圾回收器与内存分配策略](http://shieldme.cn/2015/07/05/understanding-the-jvm-of-gc-and-memory-allocation-strategy/)，垃圾回收器只给出了相关算法，至于垃圾回收器的就是具体的实现，每个实现方都不一样。Hotspot具体的有如下几种:

![](http://img.my.csdn.net/uploads/201210/03/1349278110_8410.jpg)

<!--more-->

![](http://oh8mi0yav.bkt.clouddn.com/gc-implementation.png)