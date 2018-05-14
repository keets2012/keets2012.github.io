---
title: 设计模式之状态模式
date: 2017-01-28
categories: 设计模式
tags:
- 设计模式
---
状态模式属于行为型模式。

## 状态模式的定义
定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。允许一个对象在其内部状态改变时改变它的行为，对象看起来似乎修改了它的类。

### 状态模式的组成
状态模式的类图如下所示：

![](https://images2015.cnblogs.com/blog/653266/201604/653266-20160418152108273-1472856947.png)

包含如下的角色：

- Context：定义客户感兴趣的接口。维护一个ConcreteState子类的实例，这个实例定义当前状态。
- State：定义一个接口以封装与Context的一个特定状态相关的行为。
- ConcreteStatesubclasses：每一子类实现一个与Context的一个状态相关的行为。

## 状态模式的实现
下面的示例，展示获取机器运行的状态。
### 定义State

```java
public interface State {
     //获取机器的状态
     String getState();
}
```

### 定义Context

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

### 定义ConcreteStatesubclasses

```java
class Running implements State {

    @Override
    public String getState() {   
        return "运行";
    }
}
class Stopping implements State {

    @Override
    public String getState() {
        return "停止";
    }
}
```
### 测试类

```java
public class StateTest {

    public static void main(String args[]){
        Context context=new Context();
        context.setState(new Running());
        System.out.println(context.stateMessage());
        context.setState(new Stopping());
        System.out.println(context.stateMessage());
    }
}
```
## 总结
本文主要讲解了状态模式的概念及应用。

- 状态模式优点：

封装了状态的转换规则，在状态模式中可以将状态的转换代码封装在环境类或者具体状态类中，可以对状态转换代码进行集中管理，而不是分散在一个个业务方法中。

将所有与某个状态有关的行为放到一个类中，只需要注入一个不同的状态对象即可使环境对象拥有不同的行为。

允许状态转换逻辑与状态对象合成一体，而不是提供一个巨大的条件语句块，状态模式可以让我们避免使用庞大的条件语句来将业务方法和状态转换代码交织在一起。

可以让多个环境对象共享一个状态对象，从而减少系统中对象的个数。

- 状态模式缺点：

状态模式的使用必然会增加系统中类和对象的个数，导致系统运行开销增大。

状态模式的结构与实现都较为复杂，如果使用不当将导致程序结构和代码的混乱，增加系统设计的难度。

状态模式对“开闭原则”的支持并不太好，增加新的状态类需要修改那些负责状态转换的源代码，否则无法转换到新增状态；而且修改某个状态类的行为也需修改对应类的源代码。

### 状态模式 vs 策略模式
- 区别：
状态模式将各个状态所对应的操作分离开来，即对于不同的状态，由不同的子类实现具体操作，不同状态的切换由子类实现，当发现传入参数不是自己这个状态所对应的参数，则自己给Context类切换状态；而策略模式是直接依赖注入到Context类的参数进行选择策略，不存在切换状态的操作。
- 联系：
状态模式和策略模式都是为具有多种可能情形设计的模式，把不同的处理情形抽象为一个相同的接口，符合对扩展开放，对修改封闭的原则。还有就是，策略模式更具有一般性一些，在实践中，可以用策略模式来封装几乎任何类型的规则，只要在分析过程中听到需要在不同实践应用不同的业务规则，就可以考虑使用策略模式处理，在这点上策略模式是包含状态模式的功能的，策略模式是一个重要的设计模式。


### 参考
1. [设计模式(行为型)之状态模式(State Pattern)](https://blog.csdn.net/yanbober/article/details/45502665)
2. [Java设计模式系列之状态模式](https://www.cnblogs.com/ysw-go/p/5404918.html)
