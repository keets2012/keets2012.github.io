---
title: 设计模式之备忘录模式
date: 2017-03-1
categories: 设计模式
tags:
  - 设计模式
abbrlink: 15653
---
# 设计模式之备忘录模式
属于行为型设计模式。

## 备忘录模式的定义

在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。
这样以后就可将该对象恢复到原先保存的状态。

## 备忘录模式的结构
备忘录模式中的角色对象：

- 发起者对象`Originator`：负责创建一个备忘录来记录当前对象的内部状态，并可使用备忘录恢复内部状态。 
- 备忘录对象`Memento`：负责存储发起者对象的内部状态，并防止其他对象访问备忘录。 
- 管理者对象`Caretaker`:负责备忘录权限管理，不能对备忘录对象的内容进行访问或者操作。 

![](http://ovcjgn2x0.bkt.clouddn.com/memo-design.png)

## 备忘录模式的实现
下面我们以机器使用的状态作为例子，运行一段时间，将会保存该机器运行的时长和状态等级。下一次启动时，将会恢复这些信息。
### 备忘录

```java
@lombok.Data
public class MemoTo {
    private int useTime;//使用时间
    private String deviceName;//设备名称
    private int stateLevel;//状态
}
```
用以记录三项信息：使用时间、设备名以及状态。
### 管理者

```java
@lombok.Data
public class MemoManager {
    MemoTo memento;
}
```
负责管理机器的状态，恢复时将会用到。
### 发起者

```java
@lombok.Data
public class Originator {
    private int useTime;// 使用时间
    private String deviceName;// 设备名称
    private int stateLevel;// 状态

    public Originator(String deviceName, int useTime, int stateLevel) {
        super();
        this.useTime = useTime;
        this.deviceName = deviceName;
        this.stateLevel = stateLevel;
    }

    public Originator() {
    }

    public MemoTo createMemoObject() {
        MemoTo memento = new MemoTo();
        memento.setDeviceName(deviceName);
        memento.setStateLevel(stateLevel);
        memento.setUseTime(useTime);
        return memento;
    }

    public void setMemento(MemoTo memento) {
        this.deviceName = memento.getDeviceName();
        this.stateLevel = memento.getStateLevel();
        this.useTime = memento.getUseTime();
    }

    //获取对象当前状态
    public void getCurrentState() {
        System.out.println("当前设备名称：" + this.deviceName + "当前使用时间：" + this.useTime + "当前工作状态：" + this.stateLevel);
    }
}
```
管理者和发起者中的属性相同，这里创建一个备忘录Memento，用以记录当前时刻自身的内部状态，并可使用备忘录恢复内部状态。Originator可以根据需要决定Memento存储自己的哪些内部状态。

### 测试类

```java
public class Test {

    public static void main(String[] args) {
        // 新建备忘录发起者对象
        Originator role = new Originator("发电机", 0, 1);
        // 新建备忘录管理者
        MemoManager manager = new MemoManager();
        // 角色初始状态
        System.out.println("机器开始发电:");
        role.getCurrentState();
        // 利用备忘录模式保存当前状态
        System.out.println("---保存当前的机器状态---");
        manager.setMemento(role.createMemoObject());
        role.setDeviceName("发电机");
        role.setStateLevel(5);
        role.setUseTime(1000);
        System.out.println("已经持续发电1000小时");
        role.getCurrentState();
        // 恢复保存的角色状态
        role.setMemento(manager.getMemento());
        System.out.println("恢复后发电机当前状态：");
        role.getCurrentState();
    }

}
```
测试时，先创建发起者对象和备忘录管理者，然后`启动机器发电`，运行一段时间后，保存机器的状态到备忘录中，后面还可以继续恢复保存的状态。

## 总结
本文介绍了行为型的设计模式：备忘录模式。有点系统中的快照的意思，在不破坏封装性的前提下，获取到一个对象的内部状态，并在对象之外记录或保存这个状态。
在有需要的时候可将该对象恢复到原先保存的状态。我们相当于把对象原始状备份保留，所以叫备忘录模式。 

优点： 
1、备忘录模式可以把发起人内部信息对象屏蔽起来，从而可以保持封装的边界。 
2、简化了发起人。当发起人角色的状态改变的时候，有可能这个状态无效（发布应用的时候，出现问题），这时候就可以使用存储起来的备忘录将状态复原。

缺点： 
1、如果状态需要完整地存储到备忘录对象中，那么在资源消耗上面备忘录对象比较昂贵。 
2、当发起者对象的状态改变的时候，有可能这个备忘录用不上。

### 参考
1. [JAVA设计模式之：备忘录模式](http://blog.csdn.net/true100/article/details/50561081)
