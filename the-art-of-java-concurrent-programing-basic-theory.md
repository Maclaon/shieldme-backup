---
title: Java并发机制的底层实现原理
date: 2015-07-11 10:00:00
tags: 并发机制，并发基础
author: maclaon
comments: false
---
# Java并发编程基础
线程作为操作系统调度的最小单元，多个线程能够同时执行，显著的提升程序性能，在多核环境中表现得更加明显。但是过多的创建线程和对线程的不当管理也容易造成问题，所以良好的并发编程基础成为至关重要的决定。

## 线程
操作系统在运行一个程序时，会为其创建一个进程，操作系统调度的最小单元是线程，在一个进程里可以创建多个线程，这些线程拥有各自的计数器，堆栈和局部变量等属性，并且能够访问共享的内存变量，处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。如: main()方法就是一个天然的线程。

### 线程状态
> Java线程状态在运行的生命周期中处于下列不同的状态，**在给定的一个时刻，线程只能处于其中一个状态**。

| 状态名称       | 说明          |
|:------------- |:-------------|
| **NEW**      |初始状态，线程被构建，但是还没有调用`start()`方法 |
| **RUNNABLE** |运行状态，Java线程将操作系统中的就绪和运行都称作"运行中"      |
| **BLOCKED**|阻塞状态，表示线程阻塞于锁|
|**WAITING**|等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作(通知或中断)|
|**TIME_WAITING**|超时等待状态，该状态不同于WAITING，它是可以在指定的时间自行返回|
|**TERMINATED**|终止状态，表示当前线程已经执行完毕|

<!--more-->

### 图示

![](http://img.blog.csdn.net/20150401114622564)

### 线程解决的问题
> + 线程之间的内存共享
> + 线程之间的通信

### Daemon线程
是一种支持性线程，用于程序中后台调度以及支持性工作，当一个Java虚拟机中不存在Daemon线程的时候，Java虚拟机将会推出。

## 线程间通信
线程开始运行，拥有自己的栈空间，如果仅仅是孤立的运行，那么所产生的价值很少。因为目前的操作系统的多核处理器设计上，都在每个核中拥有对应的本地拷贝，用于加速程序的执行，对于JVM从主内存中申请的内存，线程之间都是共享的，所以程序在执行的过程中，一个线程看到的变量并不一定是最新的。

### volatile和synchronized关键字
+ `volatile`: 可以用来修饰字段(成员变量)，就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。**但是过多的使用volatile的会降低程序的执行效率，适合一写多读的场景**。

+ `synchronized`: 可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。

#### synchronized字段剖析

``` Java
public class Synchronized {
	public static void main(String[] args) {
		/** 对Synchronized Class对象进行加锁，锁信息写在对象头中 */
		synchronized(Synchronized.class) {
		}
		
		/** 静态同步方法，对Synchronized Class对象进行加锁*/
		m();
	}
}

	public static synchronized void m() {
	}
```

在Synchronized.class的目录下执行`javap -v Synchronized.class`，部分相关输出如下所示:
``` python
public static void main(java.lang.String[]);
	/** 方法修饰符，表示: public staticflags: ACC_PUBLIC, ACC_STATIC */
	Code:
		stack=2, locals=1, args_size=1
		0: ldc		#1		// class com/murdock/books/multithread/book/Synchronized
		2: dup
		3: monitorenter		// monitorenter: 监视器进入，获取锁
		4: monitorexit		// monitorexit: 监视器退出，释放锁
		5: invokestatic		#16 // Method m:()V
		8: return
		
	public static synchronized void m();
	// 方法修饰符，表示: public static synchronized
	flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
		Code:
			stack = 0, locals = 0, args_size = 0
			0: return
```

上述的信息汇总，对于同步代码块的实现使用了`monitorenter`和`monitoexit`指令，而同步方法则是依靠方法修饰符上的`ACC_SYNCHRONIZED`来完成。无论采用哪种方式都是对一个对象的监视器(`monitor`)获取。

> 任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步代码块或者同步方法，而没有获取到监视器(执行该方法)的线程将会被阻塞在同步块和同步方法的入口处，进入`BLOCKED`状态。

![](http://oh8mi0yav.bkt.clouddn.com/relationships-between-monitor-blocked-quene-thread.png)

上图中任意线程对`Object`的访问获取`Object`的监视器，如果获取失败，线程进入同步队列，线程状态变为`BLOCKED`，当访问`Object`的前驱(获得了锁的线程)释放了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

### 等待/通知机制
等待/通知的相关方法是任意的Java对象都具备的，因为这些方法被定义在所有对象的超累`java.lang.Object`上，方法和描述如下:

|方法名称|描述|
|:--|:--|
|notify()|通知一个在对象上等待的线程，使其从`wait()`方法返回，而返回的前提是该线程获取了对象的锁|
|notifyAll()|通知所有等待在该对象上的线程|
|wait()|调用该方法的线程进入`WAITING`的状态，只有等待另外线程的通知或被中断才会返回，需要注意，调用`wait`方法后，会释放对象的锁|
|wait(long)|超时等待一段时间，这里的参数是毫秒，也就是等待长达$$n$$毫秒，如果没有通知就超时返回|
|wait(long,int)|对于超时时间更新粒度的控制，可以达到纳秒|

### 等待/通知模式的经典范式(方法论)
提炼出上述的通知方式，针对等待方（消费者）和通知方（生产者），原则如下:
> + 获取对象的锁
> + 如果条件不足，调用对象`wait()`方法，被通知后仍要检查条件
> + 条件满足时执行对应的条件。 

对应的伪代码如下:
``` Java
	synchronized(对象) {
		while(condition is not true) {
			对象.wait();
		}
		对应的处理逻辑
	}
```

通知方遵循如下原则:
> + 获取对象的锁
> + 改变条件
> + 通知所有等待在对象上的线程

对应的伪代码如下：
``` Java
	synchronized(对象) {
		改变条件
		对象.notifyAll();
	}
```
### 管道输入/输出
管道输入/输出流和普通的文件输入/输出流或者网络输入/输出流不同之处在于，它主要用于线程之间的数据传输，而传输的媒介为内存。

### `Thread.join()`和`ThreadLocal`的使用



