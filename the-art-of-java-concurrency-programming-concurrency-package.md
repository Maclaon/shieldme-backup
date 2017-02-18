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

``` Java
public class CountDownLatchTest {
	staticCountDownLatch = new CountDownLatch(2);
	
	public static void main(String[] args) throws InterruptedException {
		new Thread(new Runnable() {
			@Override
			public void Run() {
				System.out.println(1);
				c.countDown();
				System.out.println(2);
				c.countDown();
			}
		}).start();
		c.await();
		System.out.println("3");
	}
}
```

CountDownLatch的构造函数接收`int`类型的参数作为计数器，如果你想等待$$N$$个点完成，这里就传入$$N$$。当调用CountDownLatch的`countDown`方法时，$$N$$就会减$$1$$，`CountDownLatch`的`await`方法会阻塞当前线程，直到$$N$$变成零。由于`countDown`方法可以用在任何地方，所以这里说的$$N$$个点，可以是$$N$$个线程，也可以是$$1$$个线程里的$$N$$个执行步骤。用在多个线程时，只需要把这个`CountDownLatch`引用传递到线程里即可。同样，可以有等待超时的方法，在线程没有在规定时间内执行完，就不会阻塞当前线程。

## 同步屏障`CyclicBarrier`
`CyclicBarrier`的字面意思是可循环使用的屏障。让一组线程到达一个屏障(也可以是同步点)时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

### 说明
每个线程调用`await`方法告诉`CyclicBarrier`我已经到达屏障，然后当前线程被阻塞。示例代码如下:

```Java
public class CyclicBarrierTest {
	staticCyclicBarrier c = new CyclicBarrier(2);
	
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					c.await();
				} catch(Exception e){
				
				}
				System.out.println(1);
			}
		}).start();
		
		try {
			c.await();
		} catch(Exception e) {
		}
		
		System.out.println(2);
	}
}
```
因为主线程和子线程的调度由CPU决定，两个线程都有可能先执行，所以会产生两种输出。

``` Python
1
2
```
或者

``` Python
2
1
```

如果把`new CyclicBarrier(2)`改为`new CyclicBarrier(3)`，则主线程和子线程会永远等待，因为没有第三个线程执行`await`方法，即没有第三个线程到达屏障，所以之前到达屏障的两个线程都不会继续执行。`CyclicBarrier`还提供一个更高级的构造函数`CyclicBarrier(int parties, Runnable barrierAction)`，用于在线程到达屏障时，优先执行`barrierAction`，方便处理更复杂的业务场景。

### `CyclicBarrier`和`CountDownLatch`的区别
> + `CountDownLatch`的计数器只能使用一次，而`CyclicBarrier`的计数器可以使用`reset()`方法重置。所以`CyclicBarrier`能处理更为复杂的业务场景。除此之外`CyclicBarrier`还提供其他有用的方法。

### 应用场景
目前在设计前后端数据透出的时候，前端在调用各种数据的服务接口的时候，将数据进行组装，有的时候需要等到某个数据的来临后，才能进一步的做后续的操作。

## 控制并发线程数的`Semaphore`
> `Semaphore`是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。从字面意义上很难理解`Semaphore`，只能把它比作是控制流量的红绿灯。

### 应用场景
`Semaphore`可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库链接。假如有一个需求，需要读取几万个文件的数据，因为都是IO密集型的，可以启动几十个线程并发的读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有$$10$$个，这个时候必须控制只有$$10$$个线程同时获取数据库，否则会报错无法获取数据库链接。这个时候就需要使用`Semaphore`来做流量控制。

``` Java
public class SemaphoreTest {
	private static final int THREAD_COUNT = 30;
	private static ExecutorServierThreadPool = Executors.newFixedThreadPool(THREAD_COUNT);
	private static Semaphore s = new Semaphore(10);
	
	private static void main(String[] args) {
		for( int i = 0; i < THREAD_COUNT; i++ ) {
			threadPool.execute(new Runnable() {
				@Override
				public void run(){
					try {
						s.acquire();
						System.out.println("save data");
						s.release();
					} catch (InterruptedExecption e) {
					
					}
				}
			});
		}
		
		threadPool.shutdown();
	}
}
```

代码中，虽然有$$30$$个线程在执行，但是只允许$$10$$个并发执行。


## 线程间交换数据的`Exchanger`