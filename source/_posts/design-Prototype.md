---
title: 设计模式之原型模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 8028
date: 2017-01-26 00:00:00
---
属于创建型模式。
## 原型模式的定义
原型模式（Prototype），使用原型实例指定待创建对象的类型，并且通过复制这个原型来创建新的对象。

Java提供了Cloneable接口来标识运行克隆的对象。这个接口是一个标记接口，因此不包含任何的方法声明。当在类中实现时，Cloneable标识类的对象能够被克隆。为了执行克隆，你需要调用Object类的这个protected clone()方法，通过一个调用super.clone()。由此引出了深拷贝和浅拷贝：

- 浅拷贝：当原型对象被复制时，只复制它本身和其中包含的值类型的成员变量，而引用类型的成员变量并没有复制。
- 深拷贝：除了对象本身被复制外，对象所包含的所有成员变量也将被复制。


## 原型模式的结构
原型模式中的主要角色：

- Prototype(抽象原型类)：声明克隆方法的接口，是所有具体原型类的公共父类，它可是抽象类也可以是接口，甚至可以是具体实现类。
- ConcretePrototype(具体原型类)：它实现抽象原型类中声明的克隆方法，在克隆方法中返回自己的一个克隆对象。
- Client(客户端)：在客户类中，让一个原型对象克隆自身从而创建一个新的对象。

原型模式UML图如下：
![原型模式UML图](http://ovcjgn2x0.bkt.clouddn.com/proto-design.png "模式UML图")
## 组合模式的实现
本文主要介绍浅拷贝和深拷贝的用法案例。 实现深拷贝有两种方法，一种是实现Cloneable接口，重写clone()方法。另一种是通过序列化反序列化来获取对象的拷贝。

### 原型类

```java
public class Prototype implements Cloneable , Serializable{
    String name;
    Date date;
     
    public Prototype(String name, Date date) {
        this.name = name;
        this.date = date;
    }
     
    public Prototype() {
    }
     
    public void setDate(Date date) {
        this.date = date;
    }public void setName(String name) {
        this.name = name;
    }
    @Override
    protected Object clone() throws CloneNotSupportedException {
         
        return super.clone();
    }
     
    protected Object deepClone() throws CloneNotSupportedException {
        Object obj = this.clone();
        Prototype p = (Prototype) obj;
        p.date = (Date) this.date.clone();
        return obj;
    }
     
}
```
这里混合一起写了抽象原型类和具体原型类。

### 客户端

```java
public class ClientOne {
     //浅拷贝
    public static void shallowClone() throws Exception {
        Date date = new Date(12356565656L);
        Prototype p1 = new Prototype("原型对象", date);
        Prototype p2 = (Prototype) p1.clone();
        System.out.println(p1);
        System.out.println(p1.date);
         

        date.setTime(36565656562626L);
        System.out.println(p2);
        System.out.println(p2.date);
        System.out.println(p1.date);
    }
    //clone方式实现深拷贝
    public static void deepClone() throws Exception {
        Date date = new Date(12356565656L);
        Prototype p1 = new Prototype("原型对象",date);
        Prototype p2 = (Prototype) p1.deepClone();
        System.out.println(p1);
        System.out.println(p1.date);
        date.setTime(36565656562626L);
        System.out.println(p2);
        System.out.println(p2.date);
        System.out.println(p1.date);

    }
    //序列化方式实现深拷贝
    public static void deepCloneSerialize() throws Exception {
        Date date = new Date(12356565656L);
        Prototype p1 = new Prototype("原型对象",date);
        //通过序列化反序列化来新建一个对象
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream    oos = new ObjectOutputStream(bos);
        oos.writeObject(p1);
        byte[] bytes = bos.toByteArray();
         
        ByteArrayInputStream  bis = new ByteArrayInputStream(bytes);
        ObjectInputStream     ois = new ObjectInputStream(bis);
        Prototype p2 = (Prototype) ois.readObject();
        System.out.println(p1);
        System.out.println(p1.date);
        date.setTime(36565656562626L);
        System.out.println(p2);
        System.out.println(p2.date);
        System.out.println(p1.date);

    }
     
    public static void main(String[] args) throws Exception {
        System.out.println("Shallow clone:");
        shallowClone();
        System.out.println("Deep clone:");
        deepClone();
        System.out.println("Deep clone serialize:");
        deepCloneSerialize();
        System.exit(0);
    }
 
}
```

测试结果如下：

```
Shallow clone:
com.blueskykong.test.SimpleObject@45fe3ee3
1970-05-24
com.blueskykong.test.SimpleObject@4cdf35a9
3128-09-20
3128-09-20
Deep clone:
com.blueskykong.test.SimpleObject@4c98385c
1970-05-24
com.blueskykong.test.SimpleObject@5fcfe4b2
1970-05-24
3128-09-20
Deep clone serialize:
com.blueskykong.test.SimpleObject@aec6354
1970-05-24
com.blueskykong.test.SimpleObject@6d86b085
1970-05-24
3128-09-20
```
可以看到，浅拷贝的结果，在p2的日期修改之后，p1的日期也跟着改变了，而深拷贝不会出现这种情况。
## 总结
当使用原型模式时，我们主张使用浅拷贝还是深拷贝呢？这里没有严格的规则，主要还是依据需求。如果一个对象只有简单基础数据类型字段或者不可变对象，使用浅拷贝。当对象引用了其他可变的对象，可使用浅拷贝也可以使用深拷贝。例如，如果引用不会被修改就避免使用深拷贝而使用浅拷贝。但是如果你知道这个引用将会被更改，并且这个更改可能会影响应用程序预期的行为，那么就应该使用深拷贝。

优点：

- 当创建对象的实例较为复杂的时候，使用原型模式可以简化对象的创建过程，通过复制一个已有的实例可以提高实例的创建效率。
- 扩展性好，由于原型模式提供了抽象原型类，在客户端针对抽象原型类进行编程，而将具体原型类写到配置文件中，增减或减少产品对原有系统都没有影响。
- 原型模式提供了简化的创建结构，工厂方法模式常常需要有一个与产品类等级结构相同的工厂等级结构，而原型模式不需要这样，圆形模式中产品的复制是通过封装在类中的克隆方法实现的，无需专门的工厂类来创建产品。
- 可以使用深克隆方式保存对象的状态，使用原型模式将对象复制一份并将其状态保存起来，以便在需要的时候使用(例如恢复到历史某一状态)，可辅助实现撤销操作。

缺陷：

- 需要为每一个类配置一个克隆方法，而且该克隆方法位于类的内部，当对已有类进行改造的时候，需要修改代码，违反了开闭原则。
- 在实现深克隆时需要编写较为复杂的代码，而且当对象之间存在多重签到引用时，为了实现深克隆，每一层对象对应的类都必须支持深克隆，实现起来会比较麻烦。

### 参考
1. [设计模式：原型模式](https://www.cnblogs.com/songyaqi/p/PrototypePattern.html)
2. [设计模式——原型模式](https://www.cnblogs.com/wxisme/p/4540634.html)
