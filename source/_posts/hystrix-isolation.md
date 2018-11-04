---
title: 详解Hystrix资源隔离
date: 2018-3-19
categories: 微服务
tags:
  - Spring Cloud
abbrlink: 20190
---
> 本文作者cangwu，文章节选自其即将出版的《Spring Cloud组件源码解析与高级应用》 一书。

在货船中，为了防止漏水和火灾的扩散，一般会将货仓进行分割，避免了一个货仓出事导致整艘船沉没的悲剧。同样的，在Hystrix中，也采用了这样的舱壁模式，将系统中的服务提供者隔离起来，一个服务提供者延迟升高或者失败，并不会导致整个系统的失败，同时也能够控制调用这些服务的并发度。

## 线程与线程池

Hystrix中通过将调用服务线程与服务访问的执行线程分隔开来，调用线程能够空出来去做其他的工作而不至于被服务调用的执行的阻塞过长的时间。

![](https://cl.ly/431c1Z2a3m2c/soa-5-isolation-focused-640.png)


在Hystrix中使用独立的线程池对应每一个服务提供者，来隔离和限制这些服务，于是，某个服务提供者的高延迟或者饱和资源受限只会发生在该服务提供者对用的线程池中。

如上图中，`Dependency I`的调用失败或者高延迟仅会影响自身对应的线程池中的5个线程的阻塞并不会影响其他服务提供者的线程池状况。系统完全与服务提供者请求隔离开来，即使服务提供者对应的线程完全耗尽，并不会影响系统中的其他请求。

注意在对应服务提供者的线程池被占满时，Hystrix会进入了`fallback`逻辑，快速失败，保护服务调用者的资源稳定。

## 信号量

除了线程池外，Hystrix还可以通过信号量(计数器)来限制单个服务提供者的并发量。如果通过信号量来控制系统负载，将不再允许设置超时和异步化，这就表示在服务提供者出现高延迟，其调用线程将会被阻塞，直至服务提供者的网络请求超时，如果对服务提供者有足够的信息，可以通过信号量来控制系统的负载。


## Hystrix执行流程


![](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/hystrix-command-flow-chart.png)

简单的流程的序号介绍如下

1. 构建`HystrixCommand`或者`HystrixObservableCommand`对象
2. 执行命令
3. 是否有Response缓存
4. 是否断路器打开
5. 是否线程池或者队列或者信号量被消耗完
6. `HystrixObservableCommand.construct()` or `HystrixCommand.run()`
7. 计算链路的健康情况
8. 获取fallback逻辑
9. 返回成功的Response

## 资源隔离实现

Hystrix在判断完断路器关行后(执行流程的第4步)，将会尝试获取信号量(`AbstractCommand#applyHystrixSemantics()`)中，在Hystrix中，主要有两种方式进行资源隔离操作，一种是通过信号量的隔离策略(`ExecutionIsolationStrategy.SEMAPHORE`)，另一种是线程隔离的策略(`ExecutionIsolationStrategy.THREAD`)，我们下面来关注一下相关的实现。

### 信号量隔离策略

信号量隔离主要通过`TryableSemaphore`接口实现：

```java
interface TryableSemaphore {

 	// 尝试获取信号量
	public abstract boolean tryAcquire();
	// 释放信号量	
  	public abstract void release();
	// 
  	public abstract int getNumberOfPermitsUsed();

}
```

它的主要实现类主要有`TryableSemaphoreNoOp`，顾名思义，不进行信号量隔离，当采取线程隔离策略的时候将会注入该实现到`HystrixCommand`中，如果采用信号量的隔离策略时，将会注入`TryableSemaphoreActual`，但此时无法超时和异步化，因为信号量隔离资源的策略无法指定命令的在特定的线程执行，从而无法控制线程的执行结果。

`TryableSemaphoreActual`实现相当简单，通过`AtomicInteger`记录当前请求的信号量的线程数(原子操作保证数据的一致性)，与初始化设置的允许最大信号量数进行比较`numberOfPermits`(可以动态调整)，从而判断是否允许获取信号量，轻量级的实现，保证`TryableSemaphoreActual`无阻塞的操作方式。

```java
static class TryableSemaphoreActual implements TryableSemaphore {
	protected final HystrixProperty<Integer> numberOfPermits;
   	private final AtomicInteger count = new AtomicInteger(0);

   	public TryableSemaphoreActual(HystrixProperty<Integer> numberOfPermits) {
   		this.numberOfPermits = numberOfPermits;
  	}

  	@Override
   	public boolean tryAcquire() {
  		int currentCount = count.incrementAndGet();
      	if (currentCount > numberOfPermits.get()) {
  			count.decrementAndGet();
         	return false;
      	} else {
        	return true;
      	}
    }

   	@Override
   	public void release() {
      	count.decrementAndGet();
   	}

   	@Override
   	public int getNumberOfPermitsUsed() {
   		return count.get();
  	}
}
```

需要注意的是每一个`TryableSemaphore`通过`CommandKey`与`HystrixCommand`一一绑定，在`AbstractCommand#getExecutionSemaphore()`有体现：

```java

protected TryableSemaphore getExecutionSemaphore() {
	if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.SEMAPHORE) {
   		if (executionSemaphoreOverride == null) {
       	TryableSemaphore _s = executionSemaphorePerCircuit.get(commandKey.name());
          if (_s == null) {
          	executionSemaphorePerCircuit.putIfAbsent(commandKey.name(), new TryableSemaphoreActual(properties.executionIsolationSemaphoreMaxConcurrentRequests()));
         		return executionSemaphorePerCircuit.get(commandKey.name());
        	} else {
              return _s;
         	}
      	} else {
   			return executionSemaphoreOverride;
     	}
   	} else {
 		return TryableSemaphoreNoOp.DEFAULT;
   	}
}
```

如果是采用信号量隔离的策略，将尝试从缓存中获取该`CommandKey`对应的`TryableSemaphoreActual`(缓存中不存在创建一个新的，并与`CommandKey`绑定放置到缓存中)，否则返回`TryableSemaphoreNoOp`不进行信号量隔离。

### 线程隔离策略

在`AbstractCommand#executeCommandWithSpecifiedIsolation()`的方法中，线程隔离策略与信号隔离策略的操作主要区别是将`Observable`的执行线程通过`threadPool.getScheduler()`进行了指定，我们先查看一下`HystrixThreadPool`的相关接口。

`HystrixThreadPool`是用来将`HystrixCommand#run()`(被HystrixCommand包装的代码)指定到隔离的线程中执行的。

```java
public interface HystrixThreadPool {

   	// 获取线程池
   public ExecutorService getExecutor();
	// 获取线程调度器
   public Scheduler getScheduler();
	//
   public Scheduler getScheduler(Func0<Boolean> shouldInterruptThread);

   // 标记一个命令已经开始执行 
   public void markThreadExecution();

   // 标记一个命令已经结束执行 
   public void markThreadCompletion();

   // 标记一个命令无法从线程池获取到线程
   public void markThreadRejection();

   // 线程池队列是否有空闲 
   public boolean isQueueSpaceAvailable();
    
 }

```

`HystrixThreadPool`是由`HystrixThreadPool.Factory`生成和管理的，是通过`ThreadPoolKey`(`@HystrixCommand`中`threadPoolKey`指定)与`HystrixCommand`进行绑定，它的默认实现为`HystrixThreadPoolDefault`，其内的线程池`ThreadPoolExecutor`是通过`HystrixConcurrencyStrategy`策略生成，生成方法如下：

```java
// HystrixConcurrencyStrategy
public ThreadPoolExecutor getThreadPool(final HystrixThreadPoolKey threadPoolKey, HystrixThreadPoolProperties threadPoolProperties) {
	final ThreadFactory threadFactory = getThreadFactory(threadPoolKey);
   	final boolean allowMaximumSizeToDivergeFromCoreSize = threadPoolProperties.getAllowMaximumSizeToDivergeFromCoreSize().get();
   	final int dynamicCoreSize = threadPoolProperties.coreSize().get();
   	final int keepAliveTime = threadPoolProperties.keepAliveTimeMinutes().get();
  	final int maxQueueSize = threadPoolProperties.maxQueueSize().get();
   	final BlockingQueue<Runnable> workQueue = getBlockingQueue(maxQueueSize);

   	if (allowMaximumSizeToDivergeFromCoreSize) {
   		final int dynamicMaximumSize = threadPoolProperties.maximumSize().get();
     	if (dynamicCoreSize > dynamicMaximumSize) {
      		return new ThreadPoolExecutor(dynamicCoreSize, dynamicCoreSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
      	} else {
			return new ThreadPoolExecutor(dynamicCoreSize, dynamicMaximumSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
      	}
   	} else {
		return new ThreadPoolExecutor(dynamicCoreSize, dynamicCoreSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
   	}
}

```
如果允许配置的`maximumSize`生效的话(`allowMaximumSizeToDivergeFromCoreSize`为true)，在`coreSize`小于`maximumSize`时，会创建一个线程最大值为`maximumSize`的线程池，但会在相对不活动期间返回多余的线程到系统。否则就只应用`coreSize`来定义线程池中线程的数量。`dynamic**`前缀说明这些配置都可以在运行时动态修改，如通过配置中心的方式。

接着我们重点关注`HystrixThreadPoolDefault#getScheduler()`方法，这是给rx的`Observable`进行线程绑定的提供调度器的核心方法：

```java
@Override
public Scheduler getScheduler() {
	//默认在超时可中断线程
   	return getScheduler(new Func0<Boolean>() {
   		@Override
      	public Boolean call() {
       	return true;
      	}
  	});
}
@Override
public Scheduler getScheduler(Func0<Boolean> shouldInterruptThread) {
	touchConfig();
  	return new HystrixContextScheduler(HystrixPlugins.getInstance().getConcurrencyStrategy(), this, shouldInterruptThread);
}
```

`touchConfig()`的方法中可以动态调整线程池线程大小、线程存活时间等线程池的关键配置，在配置中心存在的情况下可以动态设置。


`HystrixContextScheduler`是Hystrix对rx中`Scheduler`调度器的重写，主要为了实现在`Observable`未被订阅时，不获取线程执行命令，以及支持在命令执行过程中能够打断运行。

 首先关注一下`Scheduler`中的相关类图：
 ![](https://cl.ly/0L2I3O2p2F0c/Scheduler.png)
 
 在rx中，`Scheduler`将生成对应的`Worker`给`Observable`用于执行命令，由`Worker`具体负责相关执行线程的调度，`ThreadPoolWorker`是Hystrix自行实现的`Worker`，持有调度的核心方法：
 
```java
@Override
public Subscription schedule(final Action0 action) {
	if (subscription.isUnsubscribed()) {
     	return Subscriptions.unsubscribed();
   	}
  	ScheduledAction sa = new ScheduledAction(action);
   	subscription.add(sa);
  	sa.addParent(subscription);
  	ThreadPoolExecutor executor = (ThreadPoolExecutor) threadPool.getExecutor();
   	FutureTask<?> f = (FutureTask<?>) executor.submit(sa);
  	sa.add(new FutureCompleterWithConfigurableInterrupt(f, shouldInterruptThread, executor));
  	return sa;
}
```

在上述代码中，如果`Observable`没有订阅，那么将取消执行，此时还没有分配线程；如果已经被订阅，将会分配线程提交任务，此时如果线程池中的线程已被占满，就可能抛出`RejectedExecutionException`的异常，拒绝任务，引发失败回滚逻辑。同时添加一个`FutureCompleterWithConfigurableInterrupt`用于在任务已经提交的情况下取消任务时释放线程。

```java
// FutureCompleterWithConfigurableInterrupt
@Override
public void unsubscribe() {
	executor.remove(f);
  	if (shouldInterruptThread.call()) {
   		f.cancel(true);
  	} else {
     	f.cancel(false);
  	}
}

```

取消任务的时候将从线程池中移除任务，释放线程，同时根据配置是否强制中断任务的执行。

通过线程隔离的方式，可以将调用线程与执行命令的线程分隔开来，避免了调用线程被阻塞，同时通过线程池的方式对每种Command并发线程数量的控制也避免了一种Command的阻塞影响到了系统的其他请求的情况，很好的保护了调用方的线程资源。

