---
title: 设计模式之单例模式
date: 2017-01-24
categories: 设计模式
tags:
- 设计模式
---
属于创建型模式。
## 单例模式的定义
单例模式是一种常用的软件设计模式。在它的核心结构中只包含一个被称为单例类的特殊类。通过单例模式可以保证系统中一类只有一个实例而且该实例易于外界访问，从而达到使用目的，同时还能方便对实例个数的控制并节约系统资源（主要目的之外的好处）。如果希望在系统中某个类的对象只能存在一个，单例模式是最好的解决方案。

在Java的Spring组件中，Bean的默认scope是singleton。

## 单例模式的结构
![Component](http://ovcjgn2x0.bkt.clouddn.com/singleton-uml.png "单例模式的结构图")
从上图中可以看出，单例模式结构图中只包含了一个单例的角色。

在单例类的内部实现只生成一个实例，同时它提供一个静态的GetInstance()方法，让客户可以访问它的唯一实例；
为了防止在外部对单例类实例化，它的构造函数被设为private；
在单例类的内部定义了一个Singleton类型的静态对象，作为提供外部共享的唯一实例。
## 单例模式的实现
单体模式有好几种实现，下面分别介绍这几种方法。
### 懒汉式
“懒汉式”是在你真正用到的时候才去建这个单例对象：

```java
public class SingletonClass{
    private static SingletonClass instance=null;
    public static SingletonClass getInstance()
    {
        if(instance==null)
        {
            synchronized(SingletonClass.class)
            {
                if(instance==null)
                    instance=new SingletonClass();
            }
        }
        return instance;
    }
    private SingletonClass(){
    }
}
```

### 饿汉式
在不管是不是马上用上，一开始就建立这个单例对象。

```java
public static class Singleton{
    //在自己内部定义自己的一个
    //实例，只供内部调用
    private static final Singleton instance = new Singleton();
    private Singleton(){
        //do something
    }
    //这里提供了一个供外部访问本class的静态方法，可以直接访问
    public static Singleton getInstance(){
        return instance;
    }
}
```

### 双重锁的形式

```java
public static class Singleton{
    private static Singleton instance=null;
    private Singleton(){
    }
    public static Singleton getInstance(){
        if(instance==null){
            synchronized(Singleton.class){
                if(null==instance){
                    instance=new Singleton();
                }
            }
        }
        return instance;
    }
}
```
可以看到，这种方式也是将实例化延迟到了getInstance方法被调用时，区别于经典单例模式，该方式引入了“双重检查”，在多线程并行执行到同步代码块时，会再次判断uniqueInsance是否为null，有效防止了多次实例化的发生。并且这种方式并没有对整个getInstance方法加锁，只是在第一次生成Singleton的唯一实例时进行了一次同步，并没有降低程序的并发性。

### 静态内部类

```java
public class Singleton {

    // 1. 创建静态内部类
    private static class Singleton2 {
       // 在静态内部类里创建单例
      private static  Singleton ourInstance  = new Singleton()；
    }

    // 私有构造函数
    private Singleton() {
    }

    // 延迟加载、按需创建
    public static  Singleton newInstance() {
        return Singleton2.ourInstance;
    }

}
```
在静态内部类里创建单例，在装载该内部类时才会去创建单例；类是由 JVM加载，而JVM只会加载1遍，保证只有1个单例。
## 总结
本文主要介绍了单例模式的概念和四种实现方式，单例模式目标明确，结构简单，在软件开发中使用频率相当高。
### 主要优点
（1）提供了对唯一实例的受控访问。单例类封装了它的唯一实例，所以它可以严格控制客户怎样以及何时访问它。
（2）由于在系统内存中只存在一个对象，因此可以节约系统资源，对于一些需要频繁创建和销毁的对象，单例模式无疑可以提高系统的性能。
（3）允许可变数目的示例。基于单例模式，开发人员可以进行扩展，使用与控制单例对象相似的方法来获得指定个数的实例对象，既节省系统资源，又解决了单例对象共享过多有损性能的问题。（Note：自行提供指定书目的实例对象的类可称之为多例类）例如，数据库连接池，线程池，各种池。

### 主要缺点
（1）单例模式中没有抽象层，因此单例类的扩展有很大的困难。
（2）单例类的职责过重，在一定程度上违背了单一职责的原则。因为单例类既提供了业务方法，又提供了创建对象的方法（工厂方法），将对象的创建和对象本身的功能耦合在一起。不够，很多时候我们都需要取得平衡。
（3）很多高级面向对象编程语言如C#和Java等都提供了垃圾回收机制，如果实例化的共享对象长时间不被利用，系统则会认为它是垃圾，于是会自动销毁并回收资源，下次利用时又得重新实例化，这将导致共享的单例对象状态的丢失。

### 参考
1. [设计模式的征途—1.单例（Singleton）模式](https://www.cnblogs.com/snaildev/p/7647190.html)
2. [设计模式之单例模式](https://www.cnblogs.com/tjudzj/p/4450975.html)
