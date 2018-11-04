---
title: 设计模式之装饰器模式
date: 2017-02-2
categories: 设计模式
tags:
  - 设计模式
abbrlink: 7651
---
装饰器模式属于结构型模式。

## 装饰器模式的定义
装饰模式可以动态的给一个对象增加一些额外的功能(增强功能) 相比于继承，装饰模式能对不支持继承的类进行增强；并且比继承更灵活，不需要生成大量的子类。

## 装饰器模式的组成
装饰器模式的类图如下：

![](http://ovcjgn2x0.bkt.clouddn.com/decorator-design.jpg)

包含如下的角色：

- Component抽象构件角色：真实对象和装饰对象有相同的接口。这样，客户端对象就能够以与真实对象相同的方式同装饰对象交互。
- ConcreteComponent具体构件角色：定义一个具体的对象，也可以给这个对象添加一些职责。
- Decorator装饰角色：持有一个抽象构件的引用。装饰对象接受所有客户端的请求，并把这些请求转发给真实的对象。这样，就能在真实对象调用前后增加新的功能。
- ConcreteDecorator具体装饰角色：负责给构件对象增加新的责任。Target匹配的接口。对Adaptee的接口与Target接口进行适配。

## 装饰器模式的实现

### 抽象构件  

```java
public abstract class House {  
    String description = "";  
    public String getDescription() {  
        return description;  
    }  
    public abstract void sleep();  
}  
```

### 具体构件

```java
public class MyHouse extends House {  
    public MyHouse() {  
        description = "自己的房子";  
    }  
  
    @Override  
    public void sleep() {  
        return "在自己房子睡觉";  
    }  
}
```

### 抽象装饰器

```java
public abstract class BigHouseDecorator extends House {  
    @Override  
    public abstract String getDescription();  
}  
```
### 具体装饰器

```java
public class European extends BigHouseDecorator {  
    House house ;  
    public European(House house) {  
        this.house = house;  
    }  
  
    @Override  
    public String getDescription() {  
        return house.getDescription()+"欧式装修";  
    }  
  
    @Override  
    public void sleep() {  
        return "欧式装修，睡着更舒服";  
    }  
}  
```

### 测试类

```java
@Test
public void testDecorator() {
    House house = new MyHouse();  
    System.out.println(house.getDescription() + " : " + house.sleep());  
    house = new European(house);  
    System.out.println(house.getDescription()+" : " + house.sleep());  
} 
```

## 总结
本文主要介绍了装饰器模式的概念及其实现。一般情况下，为了扩展一个类经常使用继承方式实现，由于继承为类引入静态特征，并且随着扩展功能的增多，子类会很膨胀。在不想增加很多子类的情况下扩展类，可以使用装饰器模式。

装饰器模式能够动态地为对象添加功能，是从一个对象外部来给对象添加功能，相当于改变了对象的外观。从外部使用系统的角度看，就不再是使用原始的那个对象了，而是使用被一系列装饰器装饰过后的对象。这样就能够灵活的改变一个对象的功能，只要动态组合的装饰器发生了改变，那么最终所得到的对象的功能就发生了改变。另一个好处是装饰器功能的复用，可以给一个对象多次增加同一个装饰器，也可以用同一个装饰器来装饰不同的对象。而且符合面向对象设计中的一条基本规则："尽量使用对象组合，而不是对象继承"。

优点是：

- 比继承更灵活。继承是静态的，而且一旦继承所有子类都有一样的功能。而装饰模式采用把功能分离到每个装饰器中，然后通过对象组合的方式，在运行时动态的组合功能，每个被装饰的对象最终有哪些功能，是由运行期动态组合的功能决定的
- 更容易复用功能。有利于装饰器功能的复用，可以给一个对象多次增加同一个装饰器，也可以用同一个装饰器来装饰不同的对象
- 简化高层定义。装饰模式可以通过组合装饰器方式给对象增添任意多的功能，因此在进行高层定义的时候，只需要定义最基本的功能就可以了，需要的时候结合相应装饰器完成需要的功能

缺点：

- 多层装饰比较复杂

### 参考
1. [装饰器模式](http://www.runoob.com/design-pattern/decorator-pattern.html)
2. [[设计模式]-装饰器模式(Decorator)](https://blog.csdn.net/qust_2011/article/details/38082729)
