---
title: Java并发编程艺术之Java并发工具类
date: 2015-07-15 10:00:00
tags: 并发编程，Java中的锁
author: maclaon
comments: false
---
# Java中的并发工具类
JDK的并发包提供了几个非常有用的并发工具类，`CountDownLatch`,`CyclicBarrier`和`Semaphore`工具类提供了并发流程控制手段，`Exchanger`工具类则提供了在线程间交换数据的一种手段。

## 等待多线程完成的`CountDownLatch`
> `CountDownLatch`允许一个或多个线程等待其他线程完成操作。

等待多个线程执行完毕，最简单的做法是使用`join()`方法，代码如下:

``` Java
public class JoinCountDownLatchTest {
	public static void main(String[] args) throws InterruptedException {
		Thread sheetParser1 = new Thread(new Runnable() {
			@Override
			public void run(){
			}
		});
		
		Thread sheetParser2 = new Thread(new Runnable() {
			@Override
			public void run(){
				System.out.println("parser2 finish");
			}
		});
		
		sheetParser1.start();
		sheetParser2.start();
		sheetParser1.join();
		sheetParser2.join();
		System.out.println("all parser finish");
	}
}
```

<!--more-->

`join`用于让当前执行线程等待`join`线程执行结束。原理如下:
> 是不停检查`join`线程是否存活，如果`join`线程存活则让当前线程永远等待。其中，`wait(0)`表示永远等待下去.

上述的代码片段如下:

```Java
while(isAlive()){
	wait(0);
}
```

直到`join`线程中止后，线程的`this.notifyAll()`方法会被调用，调用`notifyAll()`方法是在JVM里实现的，所以JDK里看不到，可以看到JVM的源码




## 同步屏障`CyclicBarrier`
## 控制并发线程数的`Semaphore`
## 线程间交换数据的`Exchanger`