---
title: Java并发工具类：CyclicBarrier和CountDownLatch
date: 2017-5-24 
categories: 并发编程
tags:
- java
---

当我们启动一个系统的时候需要初始化许多数据，这时候我们可能需要启动很多线程来进行数据的初始化，只有这些系统初始化结束之后才能够启动系统。其实在Java的类库中已经提供了`CountDownLatch`、`CyclicBarrier`这3个类来帮我们实现这样类似的功能了。`CountDownLatch`和`CyclicBarrier`是jdk concurrent包下非常有用的两个并发工具类，它们提供了一种控制并发流程的手段。本文基于JDK8。

## CyclicBarrier
`CyclicBarrier`可以理解为栅栏的意思。通过它可以实现让一组线程等待至某个状态之后再全部同时执行。`Cyclic`是当所有等待线程都被释放以后，`CyclicBarrier`可以被重用。我们暂且把这个状态就叫做barrier，当调用`await()`方法之后，线程就处于barrier了。下面看一下该工具类中提供的方法。

```java
public class CyclicBarrier {

    public CyclicBarrier(int parties, Runnable barrierAction) {
    	//parties为正整数
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }
}
```
`CyclicBarrier`提供2个构造函数，参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

```java
public int await() throws InterruptedException, BrokenBarrierException { };
public int await(long timeout, TimeUnit unit) throws InterruptedException,BrokenBarrierException,TimeoutException { };
```
`CyclicBarrier`中最重要的方法就是await方法，没有参数的版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；另一个接收两个参数：时间和单位。让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。有若干个线程都要进行写数据操作，并且只有所有线程都完成写数据操作之后，这些线程才能继续做后面的事情，此时就可以利用CyclicBarrier了。

```java
public class CyclicBarrierTest {
	public static void main(String[] args) {
		int N = 4;
		CyclicBarrier barrier = new CyclicBarrier(N);

		for (int i = 0; i < N; i++) {
			new Writer(barrier).start();
		}

		try {
			Thread.sleep(5000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.println("\n CyclicBarrier重用 \n");

		for (int i = 0; i < N; i++) {
			new Writer(barrier).start();
		}
	}

	static class Writer extends Thread {
		private CyclicBarrier cyclicBarrier;

		public Writer(CyclicBarrier cyclicBarrier) {
			this.cyclicBarrier = cyclicBarrier;
		}

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "正在写入数据...");
			try {
				Thread.sleep(1000);      //以睡眠来模拟写入数据操作
				System.out.println(Thread.currentThread().getName() + "写入数据完毕，等待其他线程写入完毕");

				cyclicBarrier.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} catch (BrokenBarrierException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + "所有线程写入完毕，继续处理其他任务...");
		}
	}
}
```
每个写入线程执行完写数据操作之后，就在等待其他线程写入操作完毕。当所有线程线程写入操作完毕之后，所有线程就继续进行后续的操作了。在初次的4个线程越过barrier状态后，又可以用来进行新一轮的使用。

运行结果如下：

```
Thread-0正在写入数据...
Thread-1正在写入数据...
Thread-2正在写入数据...
Thread-3正在写入数据...
Thread-0写入数据完毕，等待其他线程写入完毕
Thread-1写入数据完毕，等待其他线程写入完毕
Thread-2写入数据完毕，等待其他线程写入完毕
Thread-3写入数据完毕，等待其他线程写入完毕
Thread-3所有线程写入完毕，继续处理其他任务...
Thread-1所有线程写入完毕，继续处理其他任务...
Thread-2所有线程写入完毕，继续处理其他任务...
Thread-0所有线程写入完毕，继续处理其他任务...

 CyclicBarrier重用 

Thread-4正在写入数据...
Thread-5正在写入数据...
Thread-6正在写入数据...
Thread-7正在写入数据...
Thread-4写入数据完毕，等待其他线程写入完毕
Thread-5写入数据完毕，等待其他线程写入完毕
Thread-7写入数据完毕，等待其他线程写入完毕
Thread-6写入数据完毕，等待其他线程写入完毕
Thread-6所有线程写入完毕，继续处理其他任务...
Thread-4所有线程写入完毕，继续处理其他任务...
Thread-5所有线程写入完毕，继续处理其他任务...
Thread-7所有线程写入完毕，继续处理其他任务...
```

## CountDownLatch
接受一个整数型的参数，可以通过`countDownLatch.countDown()`减少一个计时，`countDownLatch.await()`进行线程等待，等到`countDownLatch`中的计数到0之后就会恢复执行。`CountDownLatch` 与 `Semaphore` 的作用完全不同，`CountDownLatch` 是类似于集合点的一个类，当调用者到达一个数目就会触发一些操作。而 `Semaphore` 是一个类似于锁队列的东西，锁用完了就是用完了，而不会触发操作。

```java
public CountDownLatch(int count) {  };
```
其中参数参数count为计数值。

```java
public void await() throws InterruptedException { };
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };
public void countDown() { };  //将count值减1
```
第一个方法，调用`await()`方法的线程会被挂起，它会等待直到count值为0才继续执行；第二个方法，和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行；第三个是将count的值减1。下面看一下具体的使用方法。

```java
public class CountDownLatchTest {
    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(2);

        new Thread(() -> {
            try {
                System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                Thread.sleep(1000);
                System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                latch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            try {
                System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                Thread.sleep(100);
                System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                latch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        try {
            System.out.println("等待2个子线程执行完毕...");
            latch.await();
            System.out.println("2个子线程已经执行完毕");
            System.out.println("继续执行主线程");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
主线程同时启动两个子线程，主线程等待子线程执行完，即调用了`await`方法，子线程执行完则调用`countDown()`方法；当子线程都执行完，继续执行主线程。
运行结果，读者可以自行尝试一下，比较简单。
## Semaphore
`Semaphore`很多时候会拿来和`CountDownLatch`进行比较，不同的地方在于`Semaphore`的值被获取到后是可以释放的，并不像`CountDownLatch`那样一直减到底。`Semaphore`是信号量的意思，可以控制同时访问的线程个数，通过 `acquire()` 获取一个许可，如果没有就等待，而 `release()` 释放一个许可。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
}
```
参数permits表示许可数目，即同时可以允许多少线程进行访问；参数fair表示是否是公平的，即等待时间越久的越先获取许可。`Semaphore`类中比较重要的几个方法，首先是`acquire()`、`release()`方法：


```java
public void acquire() throws InterruptedException {  }     //获取一个许可
public void acquire(int permits) throws InterruptedException { }    //获取permits个许可
public void release() { }          //释放一个许可
public void release(int permits) { }    //释放permits个许可
```
`acquire()`用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。`release()`用来释放许可。注意，在释放许可之前，必须先获获得许可。不过都是阻塞的方法，该工具类还提供了非阻塞的方法：

```java
public boolean tryAcquire() { };    //尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  //尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits) { }; //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; //尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public int availablePermits() //返回该信号量中可用的许可
```
下面看一下具体的使用。

```java
public class SemaphoreTest {
    public static void main(String[] args) {
        final Semaphore sp = new Semaphore(1);  //只声明一盏信号灯
        //业务线程1
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "准备获取信号灯-A");
                sp.acquire();  //获取信号灯
                System.out.println(Thread.currentThread().getName() + "已获取信号灯-A");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        //业务线程2
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "准备获取信号灯-B");
                sp.acquire();  //获取信号灯
                System.out.println(Thread.currentThread().getName() + "已获取信号灯-B");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        //业务线程3
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "准备获取信号灯-C");
                sp.acquire();  //获取信号灯
                System.out.println(Thread.currentThread().getName() + "已获取信号灯-C");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
        //检查线程
        new Timer().schedule(new TimerTask() {
            public void run() {
                System.out.println("每10s释放一次信号灯");
                sp.release();
                System.out.println("信号灯已释放");
            }
        }, 2000, 2000);    //每2秒释放一次信号灯
    }
}
```
我们声明了只有 1 个灯的信号灯，然后启动 3 个线程同时去获取信号灯，另外还启动了 1 个线程每 2 秒就释放一次信号灯。运行结果简单，可以自行尝试。

## 总结
对于 `CountDownLatch` 和 `CyclicBarrier` 两个类，我们可以看到`CountDownLatch` 类都是一个类似于集结点的概念，很多个线程做完事情之后等待其他线程完成，
全部线程完成之后再恢复运行。不同的是`CountDownLatch` 类需要你自己调用 `countDown()` 方法减少一个计数，然后调用 `await()` 方法即可。而 `CyclicBarrier` 则直接调用 `await()` 方法即可。
`CountDownLatch` 更倾向于多个线程合作的情况，等你所有东西都准备好了，我这边就自动执行了。而 `CyclicBarrier` 则是我们都在一个地方等你，大家到齐了，大家再一起执行。
`Semaphore`其实和锁有点类似，它一般用于控制对某组资源的访问权限。

### 参考
1. [Java并发编程：Semaphore、CountDownLatch、CyclicBarrier](http://www.cnblogs.com/chanshuyi/p/4457499.html)
2. [Java并发编程：CountDownLatch、CyclicBarrier和Semaphore](https://www.cnblogs.com/dolphin0520/p/3920397.html)
