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

## 分析
因为新生代对象的存活时间比较短，所以一般采用**复制算法**来进行垃圾回收器效率比较高，一般情况下新生代都采用这种算法，老年代的对象生存时间比较长，一般收集的时候采用标记-清除，标记-整理。如果发生`Full GC`，说明这次发生了`Stop-The-World`，那么可能jvm就需要停顿下来进行垃圾回收器的工作，间接导致服务的请求RT超时，影响服务的性能，那么就可以从痛点出发来进行分析，定位到触发`Full GC`的几个点。

## 虚拟机区域
年轻代（Young Generation）、年老代（Old Generation）和持久代（Permanent 
Generation）。其中持久代主要存放的是Java类的类信息，与垃圾收集要收集的Java对象关系不大。年轻代和年老代的划分是对垃圾收集影响比较大的。
> + **年轻代**: 所有新生成的对象首先都是放在年轻代的。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。年轻代分三个区。一个Eden区，两个 Survivor区(一般而言)。大部分对象在Eden区中生成。当Eden区满时，还存活的对象将被复制到Survivor区（两个中的一个），当这个 Survivor区满时，此区的存活对象将被复制到另外一个Survivor区，当这个Survivor去也满了的时候，从第一个Survivor区复制过来的并且此时还存活的对象，将被复制“年老区(Tenured)”。需要注意，Survivor的两个区是对称的，没先后关系，所以同一个区中可能同时存在从Eden复制过来对象，和从前一个Survivor复制过来的对象，而复制到年老区的只有从第一个Survivor去过来的对象。而且，Survivor区总有一个是空的。同时，根据程序需要，Survivor区是可以配置为多个的（多于两个），这样可以增加对象在年轻代中的存在时间，减少被放到年老代的可能。
> **老年代**：在年轻代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代中。因此，可以认为年老代中存放的都是一些生命周期较长的对象。
> **持久代**：用于存放静态文件，如今Java类、方法等。持久代对垃圾回收没有显著影响，但是有些应用可能动态生成或者调用一些class，例如Hibernate等，在这种时候需要设置一个比较大的持久代空间来存放这些运行过程中新增的类。持久代大小通过-XX:MaxPermSize=<N>进行设置。
> **Eden Space (heap)**：内存最初从这个线程池分配给大部分对象。
> **Survivor Space (heap)**：用于保存在eden space内存池中经过垃圾回收后没有被回收的对象。
> **Tenured Generation (heap)**：用于保持已经在survivor space内存池中存在了一段时间的对象。
> **Permanent Generation (non-heap)**：保存虚拟机自己的静态(reflective)数据，例如类（class）和方法（method）对象。Java虚拟机共享这些类数据。这个区域被分割为只读的和只写的。
> **Code Cache (non-heap)**：HotSpot Java虚拟机包括一个用于编译和保存本地代码（native code）的内存，叫做“代码缓存区”（code cache）。

## 整体性学习
> jvm区域总体分两类，heap区和非heap区。heap区又分：Eden Space（伊甸园）、Survivor Space(幸存者区)、Tenured Gen（老年代-养老区）。 非heap区又分：Code Cache(代码缓存区)、Perm Gen（永久代）、Jvm Stack(java虚拟机栈)、Local Method Statck(本地方法栈)。
> HotSpot虚拟机GC算法采用分代收集算法：**一个人（对象）出来（new 出来）后会在Eden Space（伊甸园）无忧无虑的生活，直到GC到来打破了他们平静的生活。GC会逐一问清楚每个对象的情况，有没有钱（此对象的引用）啊，因为GC想赚钱呀，有钱的才可以敲诈嘛。然后富人就会进入Survivor Space（幸存者区），穷人的就直接kill掉。并不是进入Survivor Space（幸存者区）后就保证人身是安全的，但至少可以活段时间。GC会定期（可以自定义）会对这些人进行敲诈，亿万富翁每次都给钱，GC很满意，就让其进入了Genured Gen(养老区)。万元户经不住几次敲诈就没钱了，GC看没有啥价值啦，就直接kill掉了。进入到养老区的人基本就可以保证人身安全啦，但是亿万富豪有的也会挥霍成穷光蛋，只要钱没了，GC还是kill掉**，其中判断富有与否的是根据可达性分析来进行的，可以做为可达性对象的在这里就可以认为是GC的亲戚或者亲信。
> 分区的目的：新生区由于对象产生的比较多并且大都是朝生夕灭的，所以直接采用标记-清理算法。而养老区生命力很强，则采用复制算法，针对不同情况使用不同算法。

**简单来说:** 对象在Eden Space创建，当Eden Space满了的时候，gc就把所有在Eden Space中的对象扫描一次，把所有有效的对象复制到第一个Survivor Space，同时把无效的对象所占用的空间释放。当Eden Space再次变满了的时候，就启动移动程序把Eden Space中有效的对象复制到第二个Survivor Space，同时，也将第一个Survivor Space中的有效对象复制到第二个Survivor Space。如果填充到第二个Survivor Space中的有效对象被第一个Survivor Space或Eden Space中的对象引用，那么这些对象就是长期存在的，此时这些对象将被复制到Old Generation。

## `Full Gc`发生点
> + Old Gen区空间用完
> + Perm Gen区空间用完
> + System.gc()被主动调用

## 总结
> + 对象有限在Eden区分配
> + 大对象直接进入老年代，写程序的时候避免出现朝生夕死的短命的大对象。
> + 长期存活的对象将进入老年代，对象在Survivor区每熬过一次Minor Gc，年龄就增加1岁，当它的年龄增加到一定程度（默认是15），就会被晋升为老年区。

![](http://oh8mi0yav.bkt.clouddn.com/gc-implementation.png)


## 引用
[Understanding Java Garbage Collection](http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/)

