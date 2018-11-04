---
title: Java 8新特性之CompletableFuture（一）
date: 2017-5-21
categories: 并发编程
tags:
  - java
abbrlink: 16234
---
## Future

自Java 5开始添加了`Future`，用来描述一个异步计算的结果。获取一个结果时方法较少，要么通过轮询`isDone`，确认完成后调用`get()`获取值，要么调用`get()`设置一个超时时间。但是`get()`方法会阻塞调用线程，这种阻塞的方式显然和我们的异步编程的初衷相违背。如：

```java
    @Test
    public void testFuture() throws InterruptedException {
        ExecutorService es = Executors.newSingleThreadExecutor();
        Future<Integer> f = es.submit(() -> {
            // 长时间的异步计算
            Thread.sleep(2000L);
            System.out.println("长时间的异步计算");
            return 100;
        });
        while (true) {
            System.out.println("阻断");
            if (f.isDone()) {

                try {
                    System.out.println(f.get());
                    es.shutdown();
                    break;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }
            Thread.sleep(100L);
        }
    }
```
虽然`Future`以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们的异步编程的初衷相违背，轮询的方式又会耗费无谓的CPU资源，而且也不能及时地得到计算结果，为什么不能用观察者设计模式当计算结果完成及时通知监听者呢？如Netty扩展Future的ChannelFuture接口，Node.js采用回调的方式实现异步编程。

为了解决这个问题，自Java 8开始，吸收了guava的设计思想，加入了`Future`的诸多扩展功能形成了`CompletableFuture`。
当一个Future可能需要显示地完成时，使用CompletionStage接口去支持完成时触发的函数和操作。
当两个及以上线程同时尝试完成、异常完成、取消一个CompletableFuture时，只有一个能成功。
CompletableFuture实现了CompletionStage接口的如下策略：

1. 为了完成当前的CompletableFuture接口或者其他完成方法的回调函数的线程，提供了非异步的完成操作。
2. 没有显式入参Executor的所有async方法都使用ForkJoinPool.commonPool()为了简化监视、调试和跟踪，所有生成的异步任务都是标记接口AsynchronousCompletionTask的实例。
3. 所有的CompletionStage方法都是独立于其他共有方法实现的，因此一个方法的行为不会受到子类中其他方法的覆盖。

CompletableFuture实现了Future接口的如下策略：

  * CompletableFuture无法直接控制完成，所以cancel操作被视为是另一种异常完成形式。方法`isCompletedExceptionally`可以用来确定一个CompletableFuture是否以任何异常的方式完成。
  * 方法get()和get(long,TimeUnit)抛出一个ExecutionException，对应CompletionException。为了在大多数上下文中简化用法，这个类还定义了方法`join()`和`getNow`(如果结果已经计算完则返回结果或者抛出异常，否则返回给定的valueIfAbsent值)，而不是直接在这些情况中直接抛出CompletionException。

## CompletableFuture
CompletableFuture类实现了CompletionStage和Future接口，所以你还是可以像以前一样通过阻塞或者轮询的方式获得结果，尽管这种方式不推荐使用。

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    //...
}
```

### 创建CompletableFuture对象
在该类中提供了四个静态方法创建CompletableFuture对象：

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }

public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(asyncPool, runnable);
}

public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```
以Async结尾并且没有指定Executor的方法会使用`ForkJoinPool.commonPool()`作为线程池执行异步代码。

- `runAsync`方法用于没有返回值的任务，它以Runnable函数式接口类型为参数，所以CompletableFuture的计算结果为空。

- `supplyAsync`方法用于有返回值的任务，以`Supplier<U>`函数式接口类型为参数，CompletableFuture的计算结果类型为U。

这些线程都是Daemon线程，主线程结束Daemon线程不结束，只有JVM关闭时，生命周期终止。

```java
    @Test
    public void testForCreate() {
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");//设置日期格式

        String result = CompletableFuture.supplyAsync(() -> {
            return df.format(new Date());
        }).thenApply(s -> "当前时间为： " + s).join();
        System.out.println(result);

        CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("sleep for 1s :" + df.format(new Date()));// new Date()为获取当前系统时间
        }).join(); 
    }
```

### 计算结果完成时的处理
当CompletableFuture的计算结果完成，或者抛出异常的时候，有如下四个方法：

```java
    public CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(null, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(asyncPool, action);
    }

    public CompletableFuture<T> whenCompleteAsync(
        BiConsumer<? super T, ? super Throwable> action, Executor executor) {
        return uniWhenCompleteStage(screenExecutor(executor), action);
    }
    public CompletableFuture<T> exceptionally(
        Function<Throwable, ? extends T> fn) {
        return uniExceptionallyStage(fn);
    }
```
可以看到Action的类型是`BiConsumer<? super T,? super Throwable>`它可以处理正常的计算结果，或者异常情况。

方法不以Async结尾，意味着Action使用相同的线程执行，而Async可能会使用其他线程执行（如果是使用相同的线程池，也可能会被同一个线程选中执行）

```
    @Test
    public void testComplete() {
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");//设置日期格式

        CompletableFuture.runAsync(() -> {
            System.out.println("当前时间为：" + df.format(new Date()));

            throw new ArithmeticException("illegal exception!");
        }).exceptionally(e -> {
            System.out.println("异常为： "+e.getMessage());
            return null;
        }).whenComplete((v, e) -> System.out.println("complete")).join();
    }
```

`exceptionally`方法返回一个新的CompletableFuture，当原始的CompletableFuture抛出异常的时候，就会触发这个CompletableFuture的计算，调用function计算值，否则如果原始的CompletableFuture正常计算完后，这个新的CompletableFuture也计算完成，它的值和原始的CompletableFuture的计算的值相同。也就是这个`exceptionally`方法用来处理异常的情况。

除了上述四个方法之外，一组`handle`方法也可用于处理计算结果。当原先的CompletableFuture的值计算完成或者抛出异常的时候，会触发这个CompletableFuture对象的计算，结果由BiFunction参数计算而得。因此这组方法兼有`whenComplete`和转换的两个功能。

```java
public <U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn);

public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn);

public <U> CompletableFuture<U> handleAsync(BiFunction<? super T,Throwable,? extends U> fn, Executor executor);
```
我们将上面的例子进行修改为使用`handle`实现的例子如下：

```java
    @Test
    public void testHandle() {
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");//设置日期格式

        String f = CompletableFuture.supplyAsync(() -> {
            System.out.println("当前时间为：" + df.format(new Date()));
            return "normal";
//          throw new ArithmeticException("illegal exception!");
        }).handleAsync((v, e) -> "value is: " + v + " && exception is: " + e).join();
        System.out.println(f);
    }
```
从结果可以看出，handle实现了`whenComplete`和转换的两个功能。

### 进行转换
我们可以将操作串联起来，或者将CompletableFuture组合起来。关键的入参只有一个Function，它是函数式接口，所以使用Lambda表示起来会更加优雅。它的入参是上一个阶段计算后的结果，返回值是经过转化后结果。

```java
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);

public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);

public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn,Executor executor);
```
函数的功能是当原来的CompletableFuture计算完后，将结果传递给函数fn，将fn的结果作为新的CompletableFuture计算结果。因此它的功能相当于将`CompletableFuture<T>`转换成`CompletableFuture<U>`。

```java
    @Test
    public void testFConvert() {
        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> 100);
        String f = future.thenApplyAsync(i -> i * 10).thenApply(i -> i.toString()).join();
        System.out.println(f);
    }
```
需要注意的是，这些转换并不是马上执行的，也不会阻塞，而是在前一个stage完成后继续执行。

它们与handle方法的区别在于handle方法会处理正常计算值和异常，因此它可以屏蔽异常，避免异常继续抛出。而thenApply方法只是用来处理正常值，因此一旦有异常就会抛出。

### 消费
上面的方法是当计算完成的时候，会生成新的计算结果(`thenApply`, `handle`)，或者返回同样的计算结果`whenComplete`。我们可以在每个CompletableFuture 上注册一个操作，该操作会在 CompletableFuture 完成执行后调用它。

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action);

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action);

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor);
```
CompletableFuture 通过 `thenAccept` 方法提供了这一功能，它接收CompletableFuture 执行完毕后的返回值做参数，只对结果执行Action，而不返回新的计算值，因此计算值为空：

```java
    @Test
    public void testAccept() {
        CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello world";
        }).thenAccept(System.out::println);
    }
```
`thenAcceptBoth`以及相关方法提供了类似的功能，当两个CompletionStage都正常完成计算的时候，就会执行提供的action，它用来组合另外一个异步的结果。
`runAfterBoth`是当两个CompletionStage都正常完成计算的时候，执行一个Runnable，这个Runnable并不使用计算的结果。

```java
public <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action);

public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action);

public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor);

public CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action);
```
如下的实现中，将会在两个CompletionStage都正常完成后，输出这两个计算的结果：

```java
    @Test
    public void testAcceptBoth() {
        CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "first";
        }).thenAcceptBoth(CompletableFuture.completedFuture("second"), (first, second) -> System.out.println(first + " : " + second)).join();
    }
```
下面一组方法当计算完成的时候会执行一个Runnable，与thenAccept不同，Runnable并不使用CompletableFuture计算的结果。

```java
public CompletableFuture<Void> thenRun(Runnable action);

public CompletableFuture<Void> thenRunAsync(Runnable action);

public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor);
```
我们进行如下的应用：

```java
    @Test
    public void testRun() {
        CompletableFuture.supplyAsync(() -> {
            System.out.println("执行CompletableFuture");
            return "first";
        }).thenRun(() -> System.out.println("finished")).join();
    }
```
先前的CompletableFuture计算的结果被忽略了,这个方法返回`CompletableFuture<Void>`类型的对象。

#### 参考文档
1. Java8 Doc
2. [Java CompletableFuture 详解](http://www.cnblogs.com/tian666/p/7840232.html)
3. [CompletableFuture 详解](https://www.jianshu.com/p/6f3ee90ab7d3)




