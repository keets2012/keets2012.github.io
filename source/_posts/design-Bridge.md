---
title: 设计模式之桥接模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 60984
date: 2017-01-16 00:00:00
---
属于结构型模式。
## 桥接模式的定义
桥接模式，基于类的最小设计原则，通过使用封装、聚合及继承等行为让不同的类承担不同的职责。它的主要特点是把抽象(Abstraction)与行为实现(Implementation)分离开来，
从而可以保持各部分的独立性以及应对他们的功能扩展。

## 桥接模式的结构
桥接模式的角色和职责：

- Client 调用端：这是Bridge模式的调用者。
- 抽象类（Abstraction）：抽象类接口（接口这货抽象类）维护队行为实现（implementation）的引用。它的角色就是桥接类。
- Refined Abstraction：这是Abstraction的子类。
- Implementor：行为实现类接口（Abstraction接口定义了基于Implementor接口的更高层次的操作）
- ConcreteImplementor：Implementor的子类

桥接模式的UML图如下：

![bridge-pattern](../../../../pic/bridge-pattern.png ""桥接模式UML图")


## 桥接模式的实现
下面将会给出一个桥接模式的实现，重点包括两部分：

1、定义一个桥接口，使其与一方绑定，这一方的扩展全部使用实现桥接口的方式。
2、定义一个抽象类，来表示另一方，在这个抽象类内部要引入桥接口，而这一方的扩展全部使用继承该抽象类的方式。

### Implementor

```java
public interface Implementor {
    public void operation();
}
```

### ConcreteImplementor

```java
public class ConcreateImplementorA implements Implementor {
    
    @Override
    public void operation() {
        System.out.println("this is concreteImplementorA's operation...");
    }
}
```

```java
public class ConcreateImplementorB implements Implementor {
    
    @Override
    public void operation() {
        System.out.println("this is concreteImplementorB's operation...");
    }
}
```

### 桥接Abstraction

```java
public abstract class Abstraction {
    private Implementor implementor;

    public Implementor getImplementor() {
        return implementor;
    }

    public void setImplementor(Implementor implementor) {
        this.implementor = implementor;
    }

    protected void operation(){
        implementor.operation();
    }
}
```

### RefinedAbstraction

```java
public class RefinedAbstraction extends Abstraction {
    @Override
    protected void operation() {
        super.getImplementor().operation();
    }
}
```

### 客户端测试

```java
public class BridgeTest {
    public static void main(String[] args) {
        Abstraction abstraction = new RefinedAbstraction();

        //调用第一个实现类
        abstraction.setImplementor(new ConcreateImplementorA());
        abstraction.operation();

        //调用第二个实现类
        abstraction.setImplementor(new ConcreateImplementorB());
        abstraction.operation();

    }
}
```

## 总结
 本文主要讲解了桥接模式。通过对Abstraction桥接类的调用，实现了对接口Implementor的实现类ConcreteImplementorA和ConcreteImplementorB的调用。实现了抽象与行为实现的分离。
 
1.桥接模式的优点
 
- 实现了抽象和实现部分的分离   
桥接模式分离了抽象部分和实现部分，从而极大的提供了系统的灵活性，让抽象部分和实现部分独立开来，分别定义接口，这有助于系统进行分层设计，从而产生更好的结构化系统。对于系统的高层部分，只需要知道抽象部分和实现部分的接口就可以了。
 
- 更好的可扩展性   
由于桥接模式把抽象部分和实现部分分离了，从而分别定义接口，这就使得抽象部分和实现部分可以分别独立扩展，而不会相互影响，大大的提供了系统的可扩展性。
 
- 可动态的切换实现   
由于桥接模式实现了抽象和实现的分离，所以在实现桥接模式时，就可以实现动态的选择和使用具体的实现。
 
- 实现细节对客户端透明，可以对用户隐藏实现细节
 
2.桥接模式的缺点
 
- 桥接模式的引入增加了系统的理解和设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计和编程。
- 桥接模式要求正确识别出系统中两个独立变化的维度，因此其使用范围有一定的局限性。

 
### vs 适配器模式
桥接模式和适配器模式的共同点是让两方配合工作。但是其出发点是不一样的：
         
- 适配器：改变已有的两个接口，让他们相容。  
- 桥接模式：分离抽象化和实现，使两者的接口可以不同，目的是分离。  

### 参考
1. [设计模式之桥接模式](https://www.cnblogs.com/lixiuyu/p/5923160.html)
2. [Java设计模式之《桥接模式》及应用场景](https://www.cnblogs.com/V1haoge/p/6497919.html)
