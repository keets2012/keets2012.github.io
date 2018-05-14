---
title: 设计模式之适配器模式
date: 2017-01-29
categories: 设计模式
tags:
- 设计模式
---
状态模式属于结构型模式。

## 适配器模式的定义
适配器模式将某个类的接口转换成客户端期望的另一个接口表示，目的是兼容性，让原本因接口不匹配的两个类可以协同工作，其别名为包装器(Wrapper)。

最典型的例子就是很多功能手机，每一种机型都自带有从电器，有一天自带充电器坏了，而且市场没有这类型充电器可买了。怎么办？万能充电器就可以解决。这个万能充电器就是适配器。

分为三类：类适配器、对象的适配器、接口的适配器。

## 结构
适配器模式的组成

- 目标角色（Target）：— 定义Client使用的与特定领域相关的接口。
- 客户角色（Client）：与符合Target接口的对象协同。
- 被适配角色（Adaptee)：定义一个已经存在并已经使用的接口，这个接口需要适配。
- 适配器角色（Adapte) ：适配器模式的核心。它将对被适配Adaptee角色已有的接口转换为目标角色Target匹配的接口。对Adaptee的接口与Target接口进行适配.
### 类适配器
通过继承来实现适配器功能。

![](http://ovci1ihdy.bkt.clouddn.com/class-adapter.jpg)
Adapter与Adaptee是继承关系：

- 用一个具体的Adapter类和Target进行匹配。结果是当我们想要一个匹配一个类以及所有它的子类时，类Adapter将不能胜任工作
- 使得Adapter可以重定义Adaptee的部分行为，因为Adapter是Adaptee的一个子集
- 仅仅引入一个对象，并不需要额外的指针以间接取得adaptee

### 对象适配器
适配器容纳一个它包裹的类的实例。在这种情况下，适配器调用被包裹对象的物理实体。

![](http://ovci1ihdy.bkt.clouddn.com/object-adapter.jpg)
Adapter与Adaptee是委托关系：

- 允许一个Adapter与多个Adaptee同时工作。Adapter也可以一次给所有的Adaptee添加功能
- 使用重定义Adaptee的行为比较困难

### 接口的适配器
接口的适配器也会有很多地方射击，如Spring 中使用适配器模式的典型应用：

在 Spring 的 AOP 里通过使用的 Advice（通知）来增强被代理类的功能。Spring 实现这一 AOP 功能的原理就使用代理模式（JDK 动态代理和CGLib 字节码生成技术代理。）对类进行方法级别的切面增强，即，生成被代理类的代理类，并在代理类的方法前，设置拦截器，通过执行拦截器中的内容增强了代理方法的功能，实现的面向切面编程。

Advice的类型有：BeforeAdvice、AfterReturningAdvice、ThrowSadvice 等。每个类型 Advice都有对应的拦截器，MethodBeforeAdviceInterceptor、AfterReturningAdviceInterceptor、ThrowsAdviceInterceptor。Spring 需要将每个 Advice（通知）都封装成对应的拦截器类型，返回给容器，所以需要使用适配器模式对 Advice 进行转换。

```java
public interface MethodBeforeAdvice extends BeforeAdvice {
    void before(Method method, Object[] args, Object target) throws Throwable;
}

public interface AdvisorAdapter {

    boolean supportsAdvice(Advice advice);

    MethodInterceptor getInterceptor(Advisor advisor);
}
```
## 总结
无论哪种适配器，都是为了保留现有类所提供的服务，向客户提供接口，以满足客户的期望。即在不改变原有系统的基础上，提供新的接口服务。

优点是：

- 可以让任何两个没有关联的类一起运行；
- 提高了类的复用，想使用现有的类，而此类的接口标准又不符合现有系统的需要。通过适配器模式就可以让这些功能得到更好的复用；
- 增加了类的透明度，客户端只关注结果；
- 使用适配器的时候，可以调用自己开发的功能，从而自然地扩展系统的功能。

缺点：

- 过多使用会导致系统凌乱，追溯困难。

适用场景：

- 系统需要使用一些现有的类，而这些类的接口不符合系统的需要，甚至没有这些类的源代码；
- 创建一个可以重复使用的类，用于和一些彼此之间没有太大关联的类，包括一些可能在将来引进的类一起工作。

### vs 桥接模式
桥接模式与对象适配器类似，但是桥接模式的出发点不同：桥接模式目的是将接口部分和实现部分分离，从而对它们可以较为容易也相对独立的加以改变。而对象适配器模式则意味着改变一个已有对象的接口。

### vs 装饰器模式
装饰模式增强了其他对象的功能而同时又不改变它的接口。因此装饰模式对应用的透明性比适配器更好。结果是decorator模式支持递归组合，而纯粹使用适配器是不可能实现这一点的。

### vs 外观模式
适配器模式的重点是改变一个单独类的API。Facade的目的是给由许多对象构成的整个子系统，提供更为简洁的接口。而适配器模式就是封装一个单独类，适配器模式经常用在需要第三方API协同工作的场合，设法把你的代码与第三方库隔离开来。

适配器模式与外观模式都是对现存系统的封装。但这两种模式的意图完全不同，前者使现存系统与正在设计的系统协同工作而后者则为现存系统提供一个更为方便的访问接口。简单地说，适配器模式为事后设计，而外观模式则必须事前设计，因为系统依靠于外观。总之，适配器模式没有引入新的接口，而外观模式则定义了一个全新的接口。

### 参考
1. [设计模式：适配器模式](https://www.cnblogs.com/songyaqi/p/4805820.html)
2. [设计模式（五）适配器模式Adapter（结构型）](https://blog.csdn.net/hguisu/article/details/7527842)
