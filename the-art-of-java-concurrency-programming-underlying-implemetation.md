---
title: Java并发机制的底层实现原理
date: 2015-07-08 10:00:00
tags: Java并发编程艺术，底层实现
author: maclaon
comments: false
---
# Java并发机制底层实现原理
锁是存在于Java的对象头中，通过CAS算法来判断锁的线程持有，其中对这个流程的执行流程的不同导向，或者不同策略导致各种锁的出现，侧面的通过锁的粒度化来优化执行的性能问题。

## 题记
### 乐观锁和悲观锁
独占锁是一种悲观锁，synchronized就是一种独占锁，它假设最坏的情况，并且只有在确保其它线程不会造成干扰的情况下执行，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。

## CAS
要实现无锁（lock-free）的非阻塞算法有多种实现方法，其中CAS（比较与交换，Compare and swap）是一种有名的无锁算法。借助C来调用CPU的底层指令来保证在硬件层面上实现原子操作的。
> CAS操作很容易理解，一般来说有三个值：**内存值V，期望值A，更新值B，如果内存值V和期望值A相等，那么就用更新值B替换内存值，否则什么都不做**。

<!--more-->
想象一下，假设AtomicInteger当前的value值为1，某个线程A正在执行上述的getAndSet方法，当执行到compareAndSet方法的时候，被另一个线程B抢占了，线程B成功将内存值更新为2，然后轮到线程A来继续执行上述还没有执行完的比较并更新操作，由于线程A上次获得到的current值是1，然后开始执行compareAndSet方法（最后交由CPU的原子执行来执行的），comareAndSet方法发现当前内存值V=2，而期望值A=1（current变量值），所以就不会产生值交换，然后继续下一次重试，在没有别的线程抢占的情况下，下一个循环（在并发很高的情况下可能经过更多次的循环）线程A就能够设置成功，如果线程A是在还没有运行int current = get()这一行操作时被抢占了，那么线程B运行完毕后，线程A获得的将是线程B修改后的值然后进行CAS操作可能就一次成功（在没有其他线程抢占的情况下）。因此，CAS Try-Loop操作能够很好的提供线程同步机制，我们又将此实现过程称之为线程同步的无阻塞算法，又叫”CAS循环”，”lock-free“或”wait-free“算法。

CAS是项**乐观锁技术**，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。