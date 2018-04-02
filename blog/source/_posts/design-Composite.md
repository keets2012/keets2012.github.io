---
title: 设计模式之组合模式
date: 2017-01-20
categories: 设计模式
tags:
- 设计模式
---
属于结构型模式。
## 组合模式的定义
组合模式（Composite Pattern），又叫部分整体模式，是用于把一组相似的对象当作一个单一的对象。组合模式依据树形结构来组合对象，用来表示部分以及整体层次。这种类型的设计模式属于结构型模式，它创建了对象组的树形结构。
如公司、子公司以及部门的例子，以及文件系统和书的目录等等，生活中的例子很多。

## 组合模式的结构
组合模式中的主要角色：

- 组合部件（Component）：它是一个抽象角色，为要组合的对象提供统一的接口。
- 叶子（Leaf）：在组合中表示子节点对象，叶子节点不能有子节点。
- 合成部件（Composite）：定义有枝节点的行为，用来存储部件，实现在Component接口中的有关操作，如增加（Add）和删除（Remove）。

组合模式UML图如下：
![composite-pattern](http://ovcjgn2x0.bkt.clouddn.com/composite-pattern "组合模式UML图")
## 组合模式的实现
下面我们使用组合模式构造一棵树，有根节点、叶子节点，并遍历这颗组合的树。
### Component

```java
public abstract class Component {
     
    protected String name;
     
    public Component(String name){
        this.name = name;
    }
     
    public abstract void add(Component component);
     
    public abstract void delete(Component component);
     
    public abstract void show(int index);
 
}
```

### Composite

```java
public class Composite extends Component {
     
    List<Component> components = new ArrayList<>();
 
    public Composite(String name) {
        super(name);
    }
 
    @Override
    public void add(Component component) {
        components.add(component);
    }
 
    @Override
    public void delete(Component component) {
        components.remove(component);
    }
 
    @Override
    public void show(int index) {
        StringBuilder stringBuilder = new StringBuilder();
        for(int i = 0; i < index; i++){
            stringBuilder.append("-");
        }
        System.out.println(stringBuilder.toString() + this.name);
        for(Component component : components){
            component.show(index + 2);
        }
    }
 
}
```

### Leaf

```java
public class Leaf extends Component {
 
    public Leaf(String name) {
        super(name);
    }
 
    @Override
    public void add(Component component) {
        System.out.println("A leaf could not add component");
    }
 
    @Override
    public void delete(Component component) {
        System.out.println("A leaf could not delete component");
    }
 
    @Override
    public void show(int index) {
        StringBuilder stringBuilder = new StringBuilder();
        for(int i = 0; i < index; i++){
            stringBuilder.append("-");
        }
        System.out.println(stringBuilder.toString() + this.name);
    }
 
}
```
### 客户端测试

```java
public class CompositeDemo {
     
    public static void main(String[] args) {
        Composite root = new Composite("root");
        root.add(new Leaf("leaf1"));
        root.add(new Leaf("leaf2"));
         
        Composite composite1 = new Composite("composite1");
        composite1.add(new Leaf("leaf3"));
        composite1.add(new Leaf("leaf4"));
        root.add(composite1);
         
        Leaf leaf5 = new Leaf("leaf5");
        root.add(leaf5);
        root.delete(leaf5);
        root.show(1);
    }
}
```
最后输出结果为：

```
-root
---leaf1
---leaf2
---composite1
-----leaf3
-----leaf4
```
## 总结
本文主要介绍了组合模式。组合模式用来表示部分与整体的层次结构（类似于树结构），而且也可以使用户对单个对象（叶子节点）以及组合对象（非叶子节点）的使用具有一致性，一致性的意思就是说，这些对象都拥有相同的接口。

优点：

- 组合模式使得客户端代码可以一致地处理对象和对象容器，无需关心处理的是单个对象，还是组合的对象容器。
- 将”客户代码与复杂的对象容器结构“解耦。
- 可以更容易地往组合对象中加入新的构件。

缺点： 使得设计更加复杂。客户端需要花更多时间理清类之间的层次关系。（这个是几乎所有设计模式所面临的问题）。
有时候系统需要遍历一个树枝结构的子构件很多次，这时候可以考虑把遍历子构件的结构存储在父构件里面作为缓存。
客户端尽量不要直接调用树叶类中的方法（在我上面实现就是这样的，创建的是一个树枝的具体对象;），而是借用其父类（Graphics）的多态性完成调用，这样可以增加代码的复用性。

### 参考
1. [设计模式之组合模式](https://www.cnblogs.com/snaildev/p/7647190.html)
2. [设计模式15：组合模式](https://www.cnblogs.com/zcy-backend/p/6719100.html)
