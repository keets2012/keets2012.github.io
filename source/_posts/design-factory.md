---
title: 设计模式之工厂模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 20533
date: 2017-01-30 00:00:00
---
工厂模式属于创建型模式。

## 工厂模式的定义
工厂模式包括：简单工厂，工厂方法，抽象工厂三种。本文介绍前两种。

- 目的：松耦合，不针对具体类。
- 作用：减少应用程序与具体类的依赖，实现松耦合。针对抽象编程，不针对具体类编程。
## 简单工厂模式

简单工厂模式的工厂类一般是使用静态方法，通过接收的参数的不同来返回不同的对象实例。不修改代码的话，是无法扩展的。    
通常，我们利用简单工厂模式来进行类的创建。例如，获取线程池对象，就是通过简单工厂模式来实现的。

### 组成
工厂模式的类图如下所示：

![](http://files.jb51.net/file_images/article/201605/201651294056383.jpg?201641294113 )

包含如下的角色：

- 工厂：工厂是简单工厂模式的核心，提供了对外接口。客户端或其它程序要获取Product对象，都是通过Factory的接口来获取的。
- 抽象产品：抽象产品是(许多)不同产品抽象出来的。Product可以是接口或者抽象类。
- 具体产品：工厂中返回的产品对象，实际上是通过ConcreteProduct来创建的。

### 简单工厂模式的实现

#### 产品接口

```java
public interface ProductI { 
  public void productName(); 
} 
```

#### 产品实体类

```java
public class ProductA implements ProductI { 
  @Override 
  public void productName() { 
      System.out.println("product A"); 
  } 
} 
public class ProductB implements ProductI { 
  @Override 
  public void productName() { 
      System.out.println("product B"); 
  } 
} 
```
####  简单工厂类

```java
public class Factory { 
  public ProductI create(String productName) { 
      switch (productName) { 
          case "A": 
              return new ProductA(); 
          case "B": 
              return new ProductB(); 
          default: 
              return null; 
      } 
  } 
} 
```
#### 测试

```java
public class Client { 
  public static void main(String[] args) { 
      Factory factory = new Factory(); 
      ProductI productA = factory.create("A"); 
      productA.productName(); 
      // 
      ProductI productB = factory.create("B"); 
      productB.productName(); 
  } 
} 
```

### 工厂方法模式
工厂方法是针 对每一种产品提供一个工厂类。通过不同的工厂实例来创建不同的产品实例。 
在同一等级结构中，支持增加任意产品。 

#### 组成
工厂方法模式的类图如下：

![](http://ovcjgn2x0.bkt.clouddn.com/factory-method.png)
包含如下的角色：

- 抽象工厂角色： 这是工厂方法模式的核心，它与应用程序无关。是具体工厂角色必须实现的接口或者必须继承的父类。在java中它由抽象类或者接口来实现。 
- 具体工厂角色：它含有和具体业务逻辑有关的代码。由应用程序调用以创建对应的具体产品的对象。 
- 抽象产品角色：它是具体产品继承的父类或者是实现的接口。在java中一般有抽象类或者接口来实现。 
- 具体产品角色：具体工厂角色所创建的对象就是此角色的实例。在java中由具体的类来实现。 

#### 产品接口和产品实体类
与上面相同。

#### 工厂接口 

```java
public interface FactoryI { 
  // 工厂的目的是为了生产产品 
  public ProductI create(); 
} 
```


```java
public class FactoryA implements FactoryI { 
  @Override 
  public ProductI create() { 
      return new ProductA(); 
  } 
} 
public class FactoryB implements FactoryI { 
  @Override 
  public ProductI create() { 
      return new ProductB(); 
  } 
} 
```

#### 工厂实体类

```java
//定义当前的状态
public class Context {

    private State state;

    public State getState() {
        return state;
    }

    public void setState(State state) {
        this.state = state;
    }
    public String stateMessage(){
        return state.getState();
    }
}
```

#### 测试类

```java
public class Client { 
  public static void main(String[] args) { 
      FactoryI factoryA = new FactoryA(); 
      ProductI productA = factoryA.create(); 
      productA.productName(); 

      FactoryI factoryB = new FactoryB(); 
      ProductI productB = factoryB.create(); 
      productB.productName(); 
  } 
} 
```

## 总结
简单工厂模式又称静态工厂方法模式。重命名上就可以看出这个模式一定很简单。它存在的目的很简单：定义一个用于创建对象的接口。而工厂方法模式使用继承自抽象工厂角色的多个子类来代替简单工厂模式中的“上帝类”。正如上面所说，这样便分担了对象承受的压力；而且这样使得结构变得灵活 起来——当有新的产品产生时，只要按照抽象产品角色、抽象工厂角色提供的合同来生成，那么就可以被客户使用，而不必去修改任何已有 的代码。可以看出工厂角色的结构也是符合开闭原则的！ 

### 工厂方法模式 vs 简单工厂模式

- 工厂方法模式是针对每种商品提供一个工厂类，在客户端中判断使用哪个工厂类来创建对象；
- 对于简单工厂而言，创建对象的逻辑判断放在了工厂类中，客户不感知具体的类，但是工厂类违反了开闭原则，如果要新增新的具体类，就必须修改工厂类；
- 对于工厂模式而言，是通过扩展来新增具体类的，但是在客户端就必须感知到具体的类了，要通过具体类的创建工厂来创建具体的实例，也就是判断逻辑由简单工厂的工厂类挪到了客户端中；
- 工厂模式横向扩展很方便；

### 参考
1. [设计模式之三种工厂模式](https://www.cnblogs.com/LUO77/p/5785906.html)
2. [JAVA设计模式之工厂模式(简单工厂模式+工厂方法模式)](https://www.cnblogs.com/yumo1627129/p/7197524.html)
