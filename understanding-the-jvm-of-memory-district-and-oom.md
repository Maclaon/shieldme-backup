---
title: 深入理解JVM虚拟机之Java内存区域与内存溢出异常
date: 2015-07-01 10:00:00
tags: 深入理解JVM虚拟机，内存区域，内存异常
author: maclaon
comments: false
---
# Java内存区域与内存溢出异常
## 运行时的数据区域
![](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA.jpg)

## 程序计数器
> + 程序计数器（Program Counter Register）是一块比较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器；
> + **PCR为线程私有内存**；
> + 是唯一一个在Java虚拟机规范中没有规定任何OOM情况的区域；

<!--more-->

## Java虚拟机栈
![](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/JavaStacks.jpg)

> + Java虚拟机栈（Java Virtual Machine Stacks）描述的是Java方法执行的内存模型：每个方法在在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法接口等信息。每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈出栈的过程。
> + Java虚拟机栈也是线程私有，它的生命周期与线程相同。
> + Java内存区常分为堆内存（Heap）和栈内存（Stack）；
> + OOM情况：（1）线程请求的栈深度>虚拟机所运行的最大深度；（2）虚拟机动态扩展时无法申请到足够的内存

## 本地方法栈
![](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/Java%E6%9C%AC%E5%9C%B0%E6%96%B9%E6%B3%95%E6%A0%88.png)

本地方法栈（Native Method Stack）与虚拟机所发挥的作用非常相似的，他们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机所使用的Native方法服务。

> + HotSpot虚拟机把本地方法栈和虚拟机栈合二为一；
> + 此区域会抛StackOverflowError 和 OutofMemoryError异常

## Java堆
![](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/JavaHeap.gif)

Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块，Java Heap是所有线程共享的一块内存区域，在VM启动时创建。

> + 所有的对象实例以及数组都要在堆上分配（不绝对：栈上分配、标量替换优化技术）；
> + Java堆是垃圾收集器管理的主要区域，也可称做GC堆（Garbage Collected Heap）
> + 从内存回收的角度，现代收集器基本都采用分代收集算法，Java Heap可细分为新生代和老年代，再细致可分为Eden空间、From Survivor空间、To Survivor空间等–>更好回收内存。
> + 从内存分配的角度，线程共享的Java堆中可能分出多个线程私有的分配缓存区（TLAB：Thread Local Allocation Buffer）–>更快分配内存。
> + Java堆出于逻辑连续的内存空间中，物理上可不连续，如磁盘空间一样；
> + Java堆在实现上可时，可以实现成固定大小的，也可以按照可扩展实现（-Xmx和-Xms控制）；
> + OOM情况：堆中没有内存完成实例分配，堆也无法再扩展时

## 方法区
方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

> + 也称为永久代（Permanent Generation）但随着Java8的到来，已放弃永久代改为采用Native Memory来实现方法区的规划。
> + 此区域回收目标主要是针对常量池的回收和对类型的卸载。

## 运行时常量池
![](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA%E6%8B%93%E6%89%91%E5%85%B3%E7%B3%BB.png)

运行时常量池（Runtime Constants Pool）是方法区的一部分.
+ Class文件中除了有类的版本、字段、方法、接口等描述的信息外，还有一项信息是常量池（Constant Pool Table）,用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。

## 直接内存
直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域。
> + 能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。
> + 直接内存的分配不会受到Java堆大小的限制，但会收到本机总内存（RAM以及SWAP/分页文件）大小以及处理器寻址空间的限制。
> + 设置Xmx等参数信息时注意不能忽略直接内存，不然会引起OOM。

## 整体性学习
从编写代码Java代码的角度来关联上述的内存区域：
> 方法区: 编译器编译的类的信息，常量，静态变量等相关信息。
> 堆: 对代码中new的一些对象进行归属
> 虚拟机栈: 方法区域中局部变量的存储地方
> 本地方法栈: 调用本地方法时局部变量进行数据存储的地方
> 程序计数器：程序从上到下进行执行的顺序，因为CPU会进行计数的切换等。