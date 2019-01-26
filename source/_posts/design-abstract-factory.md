---
title: 设计模式之抽象工厂模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 23356
date: 2017-01-31 00:00:00
---
工厂模式属于创建型模式。

## 工厂模式的定义
工厂模式包括：简单工厂，工厂方法，抽象工厂三种。本文介绍前两种。

- 目的：松耦合，不针对具体类。
- 作用：减少应用程序与具体类的依赖，实现松耦合。针对抽象编程，不针对具体类编程。

在上一篇[设计模式之工厂模式](http://blueskykong.com/2017/01/30/design-factory/)中讲了工厂方法的两种：简单工厂和工厂方法模式。 本文将会讲解抽象工厂模式。
## 抽象工厂模式
为创建一组相关或相互依赖的对象提供一个接口，而且无需指定他们的具体类。

提供一个接口，用于创建相关或者依赖对象的家族，而不需要明确指定具体类。
允许客户使用抽象的接口来创建一组相关产品，不需要知道生产出来的是什么，客户从具体的产品中被解耦。

### 组成
抽象工厂模式的类图如下所示：

![](../../../../pic/abstract-factory.gif )

包含如下的角色：

- 抽象产品：抽象产品是(许多)不同产品抽象出来的。Product可以是接口或者抽象类。
- 具体产品：工厂中返回的产品对象，实际上是通过ConcreteProduct来创建的。
- 工厂实体：提供了对外接口。客户端或其它程序要获取Product对象，都是通过Factory的接口来获取的。
- 抽象工厂：用于创建相关或者依赖对象的家族，而不需要明确指定具体类。

### 抽象工厂模式的实现

#### 产品接口

```java
public interface ProductBI { 
  public void productName(); 
} 

public interface ProductAI { 
  public void productName(); 
}
```
#### 产品实体类

```java
public class ProductA1 implements ProductAI { 
  @Override 
  public void productName() { 
      System.out.println("product A1"); 
  } 
}
...
```
#### 抽象工厂

```java
public interface AbstractFactory { 
  public ProductAI createProductA(); 
  public ProductBI createProductB(); 
} 
```

#### 工厂实体类

```java
public class Factory1 implements AbstractFactory { 
  @Override 
  public ProductAI createProductA() { 
      return new ProductA1(); 
  } 

  @Override 
  public ProductBI createProductB() { 
      return new ProductB1(); 
  } 
}

public class Factory2 implements AbstractFactory { 
  @Override 
  public ProductAI createProductA() { 
      return new ProductA2(); 
  } 

  @Override 
  public ProductBI createProductB() { 
      return new ProductB2(); 
  } 
} 
```



## 总结
抽象工厂每个方法经常以工厂方法来实现，抽象工厂的任务是定义一个负责创建一组产品的接口。这个接口内的每个方法负责创建一个具体产品，同时利用实现抽象工厂子类来提供具体的做法。优点是可以把相关的产品组合起来，缺点是扩展需要修改接口。

### 工厂方法模式 vs 抽象工厂模式

- 抽象工厂关键在于产品之间的抽象关系，所以至少要两个产品；工厂方法在于生成产品，不关注产品间的关系，所以可以只生成一个产品。
- 抽象工厂中客户端把产品的抽象关系理清楚，在最终使用的时候，一般使用客户端（和其接口），产品之间的关系是被封装固定的；而工厂方法是在最终使用的时候，使用产品本身（和其接口）。
- 抽象工厂的工厂是类；工厂方法的工厂是方法。

抽象工厂更像一个复杂版本的策略模式，策略模式通过更换策略来改变处理方式或者结果；而抽象工厂的客户端，通过更改工厂还改变结果。所以在使用的时候，就使用客户端和更换工厂，而看不到产品本身。工厂方法目的是生产产品，所以能看到产品，而且还要使用产品。当然，如果产品在创建者内部使用，那么工厂方法就是为了完善创建者，从而可以使用创建者。另外创建者本身是不能更换所生产产品的。

抽象工厂的工厂类就做一件事情生产产品。生产的产品给客户端使用，绝不给自己用。
工厂方法生产产品，可以给系统用，可以给客户端用，也可以自己这个类使用。自己这个类除了这个工厂方法外，还能有其他功能性的方法

### 参考
1. [设计模式之三种工厂模式](http://www.iteye.com/topic/1145602)
2. [抽象工厂模式和工厂模式的区别？](https://www.zhihu.com/question/20367734)
