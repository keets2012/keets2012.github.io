---
title: Spring中的事件驱动模型（二）
date: 2018-3-15
categories: Spring
tags:
  - Spring
  - Java
abbrlink: 16090
---
## 前文回顾
[前一篇文章](http://blueskykong.com/2018/02/22/spring-event-sourcing/)讲了Spring中的事件驱动模型相关概念。重点篇幅介绍了Spring的事件机制，Spring的事件驱动模型由事件、发布者和订阅者三部分组成，结合Spring的源码分析了这三部分的定义与实现。本文主要结合具体例子讲解Spring中的事件驱动。笔者在写[Spring Cloud Bus中的事件的订阅与发布](http://blueskykong.com/2018/02/13/spring-cloud-bus-event/)两篇文章的时候，想到要把Spring中的事件驱动模型的讲解给补充一下，这块也是属于更加基础的知识点。

## 应用Spring中的事件驱动模式
我们示例配置信息的刷新，当配置服务器收到提交的配置事件之后，将会触发各个服务响应的更新自己的配置。具体代码如下：

### 事件

```java
public class ConfigRefreshEvent extends ApplicationEvent {
    public ConfigRefreshEvent(final String content) {
        super(content);
    }
}
```
定义一个配置刷新的事件。继承`ApplicationEvent`即可，content即为需要传递的object。
### 定义监听器
定义两个服务，都实现了`SmartApplicationListener`，该接口继承自`ApplicationListener`和`Ordered`接口，属于标准监听器的扩展接口，公开更多的元数据，例如支持的事件类型和可排序的监听器。

```java
public interface SmartApplicationListener extends ApplicationListener<ApplicationEvent>, Ordered {

	/**
	 * 决定该监听器是否支持给定的事件
	 */
	boolean supportsEventType(Class<? extends ApplicationEvent> eventType);

	/**
	 * 决定该监听器是否支持给定的目标类型，支持才会调用
	 */
	boolean supportsSourceType(Class<?> sourceType);

}
```
下面贴出两个Service的代码。
#### ServiceAListener

```java
@Component
public class ServiceAListener implements SmartApplicationListener {

    @Override
    public boolean supportsEventType(final Class<? extends ApplicationEvent> eventType) {
        return eventType == ConfigRefreshEvent.class;
    }

    @Override
    public boolean supportsSourceType(final Class<?> sourceType) {
        return sourceType == String.class;
    }

    @Override
    public void onApplicationEvent(final ApplicationEvent event) {
        System.out.println("ServiceA收到新的配置：" + event.getSource());
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
} 
```
#### ServiceBListener

```java
@Component
public class ServiceBListener implements SmartApplicationListener {

    @Override
    public boolean supportsEventType(final Class<? extends ApplicationEvent> eventType) {
        return eventType == ConfigRefreshEvent.class;
    }

    @Override
    public boolean supportsSourceType(final Class<?> sourceType) {
        return sourceType == String.class;
    }

    @Override
    public void onApplicationEvent(final ApplicationEvent event) {
        System.out.println("ServiceB收到新的配置：" + event.getSource());
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```
### 测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTest {
    @Autowired
    private ApplicationContext applicationContext;

    @Test
    public void testPublishEvent() {
        System.out.println("发布配置更新：");
        ConfigRefreshEvent event = new ConfigRefreshEvent("配置信息更新了")
        applicationContext.publishEvent(event);
    }
}
```
上面是我们的测试类，相当于事件的发布者，首先定义一个配置刷新的事件，然后通过注入的`ApplicationContext`发布该事件。

由于ServiceA的优先级高于ServiceB，所以我们看到如下的结果：

```
发布配置更新：
ServiceA收到新的配置：配置信息更新了
ServiceB收到新的配置：配置信息更新了
```

## 总结
本文比较简单，在上一篇介绍Spring中的事件驱动模型基础上，具体应用到配置刷新的场景中。Spring的事件驱动模型使用的是观察者模式。通过`ApplicationEvent`抽象类和`ApplicationListener`接口，可以实现事件的定义与监听，`ApplicationContext`则实现了事件的发布。例子中使用的`SmartApplicationListener`扩展了标准的事件监听接口，监听器在处理Event时，可以对传入的Event进行判断，并且可以设定监听器的优先级。后面抽时间会写一下 Spring Cloud 的热更新机制，也是基于Spring中的事件驱动模型。

