---
title: 设计模式之享元模式
date: 2017-03-14
categories: 设计模式
tags:
- 设计模式
---
享元模式属于结构性模式。
## 享元模式的定义
采用一个共享来避免大量拥有相同内容对象的开销。这种开销中最常见、直观的就是内存的损耗。享元模式以共享的方式高效的支持大量的细粒度对象。

享元模式是为数不多的、只为提升系统性能而生的设计模式。它的主要作用就是复用大对象（重量级对象），以节省内存空间和对象创建时间。
## 享元模式的结构

![flyweight](http://ovcjgn2x0.bkt.clouddn.com/flyweight.jpg "享元模式UML")

在享元模式结构图中包含如下几个角色：

- Flyweight（抽象享元类）：通常是一个接口或抽象类，在抽象享元类中声明了具体享元类公共的方法，这些方法可以向外界提供享元对象的内部数据（内部状态），同时也可以通过这些方法来设置外部数据（外部状态）。
- ConcreteFlyweight（具体享元类）：它实现了抽象享元类，其实例称为享元对象；在具体享元类中为内部状态提供了存储空间。通常我们可以结合单例模式来设计具体享元类，为每一个具体享元类提供唯一的享元对象。
- UnsharedConcreteFlyweight（非共享具体享元类）：并不是所有的抽象享元类的子类都需要被共享，不能被共享的子类可设计为非共享具体享元类；当需要一个非共享具体享元类的对象时可以直接通过实例化创建。
- FlyweightFactory（享元工厂类）：享元工厂类用于创建并管理享元对象，它针对抽象享元类编程，将各种类型的具体享元对象存储在一个享元池中，享元池一般设计为一个存储“键值对”的集合（也可以是其他类型的集合），
可以结合工厂模式进行设计；当用户请求一个具体享元对象时，享元工厂提供一个存储在享元池中已创建的实例或者创建一个新的实例（如果不存在的话），返回新创建的实例并将其存储在享元池中。


## 享元模式的实现

### 抽象享元类

```java
public abstract class Flyweight {

    public abstract void operation(int extrinsicstate);

}
```

### 具体享元类

```java
public class ConcreteFlyweight extends Flyweight {

    @Override
    public void operation(int extrinsicstate) {
        System.out.println("具体Flyweight：" + extrinsicstate);        
    }

}
```

### 非共享具体享元类

```java
public class UnsharedConcreteFlyweight extends Flyweight {

    @Override
    public void operation(int extrinsicstate) {
        System.out.println("不共享的的具体Flyweight：" + extrinsicstate);       
    }

}
```

### 享元工厂类

```java
public class FlyweightFactory {

    private Hashtable flyweights = new Hashtable();

    //获取Flyweight实例对象,如果实例对象不存在就构建，存在直接返回
    public Flyweight getFlyweight(String key){

        if ((Flyweight) flyweights.get(key)== null) {
            flyweights.put(key, new ConcreteFlyweight());
        }
        return (Flyweight) flyweights.get(key);
    }

}
```

### 客户端测试


```java
public class Client {

    public static void main(String[] args) {
        int extrinsicstate = 2;//外部状态

        FlyweightFactory flyweightFactory = new FlyweightFactory();

        Flyweight flyweightA = flyweightFactory.getFlyweight("instanceA");
        flyweightA.operation(--extrinsicstate);

        Flyweight flyweightB = flyweightFactory.getFlyweight("instanceB");
        flyweightB.operation(--extrinsicstate);
    }

}
```
## 总结

本文主要介绍了享元模式，享元模式优点就在于它能够大幅度的降低内存中对象的数量；而为了做到这一步也带来了它的缺点：它使得系统逻辑复杂化，而且在一定程度上外蕴状态影响了系统的速度。
            
适用的场景： 
- 如果一个应用程序使用了大量的对象，而大量的这些对象造成了很大的存储开销时就应该考虑使用； 
- 对象的大多数状态可以是外部状态，如果删除对象的外部状态，那么可以用相对较少的共享对象取代很多组对象，此时可以考虑使用享元模式。

### 参考
1. [深入浅出享元模式](https://blog.csdn.net/ai92/article/details/224598)
2. [ 大话设计模式—享元模](https://blog.csdn.net/lmb55/article/details/51057570)
