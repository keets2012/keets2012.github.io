---
title: Java 8新特性之CompletableFuture（二）
date: 2017-5-24 
categories: 并发编程
tags:
- java
---
我们在上一篇[Java 8特性之CompletableFuture（一）](http://blueskykong.com/2017/05/21/completeblefuture/)介绍了Java8 中新增的处理类CompletableFuture部分使用特性，包括：创建、计算结果处理、进行转换和结果消费。本文将会继续讲解CompletableFuture剩余的用法。

### 组合
通常，我们会有多个需要独立运行但又有所依赖的的任务。比如先等用于的订单处理完毕然后才发送邮件通知客户。

thenCompose 方法允许你对两个异步操作进行流水线，第一个操作完成时，将其结果作为参数传递给第二个操作。你可以创建两个CompletableFutures 对象，对第一个 CompletableFuture 对象调用thenCompose ，并向其传递一个函数。当第一个CompletableFuture 执行完毕后，它的结果将作为该函数的参数，这个函数的返回值是以第一个 CompletableFuture 的返回做输入计算出的第二个 CompletableFuture 对象。

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T,? extends CompletionStage<U>> fn);

public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn);

public <U> CompletableFuture<U> thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn, Executor executor);
```
这一组方法接受一个Function作为参数，这个Function的输入是当前的CompletableFuture的计算值，返回结果将是一个新的CompletableFuture，这个新的CompletableFuture会组合原来的CompletableFuture和函数返回的CompletableFuture。

`thenCompose`返回的对象并不一是函数fn返回的对象，如果原来的CompletableFuture还没有计算出来，它就会生成一个新的组合后的CompletableFuture。
```java
    @Test
    public void testCompose() {
        int f = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "first";
        }).thenCompose(str -> CompletableFuture.supplyAsync(() -> {
            String str2 = "second";
            return str.length() + str2.length();
        })).join();
        System.out.println("字符串长度为：" + f);

    }
```
如上代码组合了两个字符串长度的计算，最后返回字符串的长度。
另一种比较常见的情况是，你需要将两个完全不相干的 CompletableFuture 对象的结果整合起来，而且你也不希望等到第一个任务完全结
束才开始第二项任务。thenCombine用来复合另外一个CompletionStage的结果。两个CompletionStage是并行执行的，它们之间并没有先后依赖顺序.

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);

public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);

public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor);
```
其中的other，并不会等待先前的CompletableFuture执行完毕后再执行。

```java
    @Test
    public void testCombine() {
        String f = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "first";
        }).thenCombineAsync(CompletableFuture.supplyAsync(() -> "second"), (x, y) -> y + ":" + x).join();
        System.out.println(f); 
    }
```
如上面的例子所示，输出结果为： second:first。从功能上来讲，类似thenAcceptBoth，只不过thenAcceptBoth是纯消费，它的函数参数没有返回值，而thenCombine的函数参数fn有返回值。

### 任意一个
在第一篇中讲到thenAcceptBoth和runAfterBoth是当两个CompletableFuture都计算完成，CompletableFuture中还提供了当任意一个完成时执行计算的方法。

```java
public CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action);

public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action);

public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor);
```
`acceptEither`方法是当任意一个CompletionStage完成的时候，action这个消费者就会被执行。这个方法返回`CompletableFuture<Void>`。

```java
public <U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn);

public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn);

public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor);
```
`applyToEither`方法是当任意一个CompletionStage完成的时候，fn会被执行，它的返回值会当作新的`CompletableFuture<U>`的计算结果。

```java
    @Test
    public void testApplyEither() {
        Random rand = new Random();
        String f = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100 + rand.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "first";
        }).applyToEither(CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100 + rand.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "second";
        }), i -> "result: " + i).join();
        System.out.println(f);
    }
```
上面的例子有时会输出first，有时候会输出second，哪个Future先完成就会根据它的结果计算。

### allOf 和 anyOf
除了上述的几个静态方法：`completedFuture`、`runAsync` 和` supplyAsync`，下面介绍的这两个方法用来组合多个CompletableFuture。


```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs);

public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs);
```
`allOf`方法是当所有的CompletableFuture都执行完后执行计算。`anyOf`方法是当任意一个CompletableFuture执行完后就会执行计算，计算的结果相同。

```java
    @Test
    public void testAnyAll() {
        Random rand = new Random();
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100 + rand.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "first";
        });
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100 + rand.nextInt(100));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "second";
        });
//        CompletableFuture.allOf(future1, future2).join();
        Object cf = CompletableFuture.anyOf(future1, future2).join();
        System.out.println(cf);
    }
```
上面的例子应用了 `allOf` 和 `anyOf` 方法，较为简单。最后的结果可能为"first"或者"second"。

## 总结
CompletableFuture为构建异步应用提供了很多实现的方法，很好的弥补了`Future`接口的局限性。在Java8 之前，很难直接表述多个Future 结果之间的依赖性。新的CompletableFuture将使得这些成为可能。

当然CompletableFuture提供了方法很多，参数各有不同，我们可以根据方法的参数的类型来加速记忆。Runnable类型的参数会忽略计算的结果，Consumer是纯消费计算结果，BiConsumer会组合另外一个CompletionStage纯消费，Function会对计算结果做转换，BiFunction会组合另外一个CompletionStage的计算结果做转换。

#### 参考文档
1. Java8 Doc
2. [Java CompletableFuture 详解](http://www.cnblogs.com/tian666/p/7840232.html)
3. [CompletableFuture 详解](https://www.jianshu.com/p/6f3ee90ab7d3)
4. [Java8新特性8--使用CompletableFuture构建异步应用](https://www.jianshu.com/p/4897ccdcb278)


