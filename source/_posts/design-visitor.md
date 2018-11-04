---
title: 设计模式之访问者模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 1606
date: 2017-03-13 00:00:00
---
属于行为型模式。
## 访问者模式的定义

> 封装一些作用于某种数据结构中的各元素的操作，它可以在不改变这个数据结构的前提下定义作用于这些元素的新的操作。

在软件开发中，有时候也需要处理像处方单这样的集合对象结构，在该对象结构中存储了多个不同类型的对象信息，而且对同一对象结构中的元素的操作方式并不唯一，
可能需要提供多种不同的处理方式。

## 访问者模式的结构
访问者模式结构中包含以下5个角色：

- Visitor（抽象访问者）：抽象访问者为对象结构中每一个具体元素类ConcreteElement声明一个访问操作，从这个操作的名称或参数类型可以清楚知道需要访问的具体元素的类型，具体访问者则需要实现这些操作方法，定义对这些元素的访问操作。
- ConcreteVisitor（具体访问者）：具体访问者实现了抽象访问者声明的方法，每一个操作作用于访问对象结构中一种类型的元素。
- Element（抽象元素）：一般是一个抽象类或接口，定义一个Accept方法，该方法通常以一个抽象访问者作为参数。
- ConcreteElement（具体元素）：具体元素实现了Accept方法，在Accept方法中调用访问者的访问方法以便完成一个元素的操作。
- ObjectStructure（对象结构）：对象结构是一个元素的集合，用于存放元素对象，且提供便利其内部元素的方法。


![design-visitor](http://ovcjgn2x0.bkt.clouddn.com/design-visitor.png "访问者模式")
## 访问者模式的实现

汽车作为对象结构的角色，里面包含引擎，车身等部分对象，访问者角色对象为PrintVisitor，车接受该访问者让其访问车的各个组成对象并打印信息；
### visitor

```java
public interface Visitor {
    void visit(Engine engine);

    void visit(Body body);

    void visit(Car car);
}
```
访问者接口，包含三个方法

### 具体访问者
汽车打印访问者：

```java
public class PrintCar implements Visitor {
    public void visit(Engine engine) {
        System.out.println("Visiting engine");
    }

    public void visit(Body body) {
        System.out.println("Visiting body");
    }

    public void visit(Car car) {
        System.out.println("Visiting car");
    }
}
```
汽车检修的访问者：

```java
public class CheckCar implements Visitor {
    public void visit(Engine engine) {
        System.out.println("Check engine");
    }

    public void visit(Body body) {
        System.out.println("Check body");
    }

    public void visit(Car car) {
        System.out.println("Check car");
    }
}
```
### 抽象元素

```java
public interface Visitable {
    void accept(Visitor visitor);
}
```

### 具体元素

```java
public class Body implements Visitable {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

```java
public class Engine implements Visitable {
    @Override    
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

### 对象结构

```java
public class Car {
    private List<Visitable> visit = new ArrayList<>();
    public void addVisit(Visitable visitable) {
        visit.add(visitable);
    }

    public void show(Visitor visitor) {
        for (Visitable visitable: visit) {
            visitable.accept(visitor);
        }
    }
}
```
当前访问者模式例子中的对象结构，这个列表是元素（Visitable）的集合，这便是对象结构的通常表示，它一般会是一堆元素的集合，不过这个集合不一定是列表，也可能是树，
链表等等任何数据结构，甚至是若干个数据结构。其中show方法，就是汽车类的精髓，它会枚举每一个元素，让访问者访问。
### 客户端

```java
public class Client {
    static public void main(String[] args) {
        Car car = new Car();
        car.addVisit(new Body());
        car.addVisit(new Engine());
        
        Visitor print = new PrintCar();
        car.show(print);

    }
}
```
汽车中的元素很稳定，这些几乎不可能改变，而最容易改变的就是访问者这部分。
访问者模式最大的优点就是增加访问者非常容易，我们从代码上来看，如果要增加一个访问者，你只需要做一件事即可，新增访问者实现Visitor接口，然后就可以直接调用对象结构的show方法去访问汽车了。
## 总结
本文主要介绍了访问者模式，参考维基百科的例子，笔者给出了一个实现。从上面讲解得出，访问者模式适用于对象结构比较稳定，但经常需要在此对象结构上定义新的操作。
需要对一个对象结构中的对象进行很多不同的且不相关的操作，而需要避免这些操作“污染”这些对象的类，也不希望在增加新操作时修改这些类。

优点：

- 增加新的访问操作十分方便，符合开闭原则；
- 将有关元素对象的访问行为集中到一个访问者对象中，而不是分散在一个个的元素类中，类的职责更加清晰，符合单一职责原则

主要缺点：

- 增加新的元素类很困难，需要在每一个访问者类中增加相应访问操作代码，这违背了开闭原则；
- 元素对象有时候必须暴露一些自己的内部操作和状态，否则无法供访问者访问，这破坏了元素的封装性。

### 参考
1. [wikipedia](https://zh.wikipedia.org/wiki/%E8%AE%BF%E9%97%AE%E8%80%85%E6%A8%A1%E5%BC%8F)
2. [设计模式的征途—16.访问者（Visitor）模式](https://www.cnblogs.com/edisonchou/p/7247990.html)
