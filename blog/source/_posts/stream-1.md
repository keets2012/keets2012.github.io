---
title: Spring Cloud Stream应用与自定义RocketMQ Binder：编程模型
date: 2018-6-22
categories: 微服务
tags:
- 微服务
- 消息中间件
---
前言： 本文作者张天，节选自笔者与其合著的《Spring Cloud微服务架构进阶》，即将在八月出版问世。本文将其中Spring Cloud Stream应用与自定义Rocketmq Binder的内容抽取出来，主要介绍Spring Cloud Stream的相关概念，并概述相关的编程模型。

## 概述
### Spring Cloud Stream 简介
Spring Cloud Stream 是一个用来为微服务应用构建消息驱动能力的框架。它可以基于Spring Boot 来创建独立的，可用于生产的Spring 应用程序。他通过使用Spring Integration来连接消息代理中间件以实现消息事件驱动。Spring Cloud Stream 为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅、消费组、分区的三个核心概念。Spring Cloud Stream目前仅支持RabbitMQ、Kafka。
### 消息队列简介
消息队列中间件是分布式系统中最为重要的组件之一，主要解决应用耦合，异步消息，流量削锋等问题，是大型分布式系统不可缺少的中间件。消息队列技术是分布式应用间交换信息的一种技术，消息可驻留在内存或磁盘上，队列存储消息直到它们被应用程序读走。通过消息队列，应用程序可以相对独立地执行，它们不需要知道彼此的位置，只需要处理从消息队列发送来的消息和向消息队列发送消息。

消息队列的主要特点是异步处理和解耦。其主要的使用场景就是将比较耗时而且不需要同步返回结果的操作作为消息放入消息队列。同时由于使用了消息队列，只要保证消息格式不变，消息的发送方和接受者并不需要彼此联系，也不需要受对方的影响，即解耦。

消息队列的使用场景有:

- 跨系统的异步通信，需要异步交互的场景都可以使用消息队列。
- 消息驱动的架构(EDA)，系统分解为消息队列，消息队列制造者和消息队列消费者，一个是处理流程可以根据需求拆分成多个阶段，每个阶段之间通过队列连接起来。
- 流量削锋，它是消息队列中的常用场景之一，一般在秒杀或团抢活动中使用广泛。秒杀活动，一般会因为流量过大，导致流量暴增，应用挂掉，为解决这个问题，一般需要在应用前端加入消息队列，来缓和流量的暴增。

在软件的正常功能开发过程中，开发人员并不需要去刻意的寻找消息队列的使用场景，而是当出现性能瓶颈时，去查看业务逻辑是否存在可以异步处理的耗时操作，如果存在的话便可以引入消息队列来解决。否则盲目的使用消息队列可能会增加维护和开发的成本却无法得到可观的性能提升，那就得不偿失了。

### 常见的消息队列
目前业界有四款常用的消息队列，它们分别是RabbitMQ、RocketMQ、ActiveMQ和Kafka。我们这里主要介绍前两种。
#### RabbitMQ
RabbitMQ在2007年发布，是一个在AMQP(高级消息队列协议)基础上完成的，可复用的企业消息系统，是当前最流行的消息中间件之一。
RabbitMQ的主要特性有：

- 可靠性: RabbitMQ提供了多种技术可以让你在性能和可靠性之间进行权衡。这些技术包括持久性机制、投递确认、发布者证实和高可用性机制；
- 灵活的路由：消息在到达队列前是通过交换机进行路由的。RabbitMQ为典型的路由逻辑提供了多种内置交换机类型。如果你有更复杂的路由需求，可以将这些交换机组合起来使用，你甚至可以实现自己的交换机类型，并且当做RabbitMQ的插件来使用；
- 消息集群：在相同局域网中的多个RabbitMQ服务器可以聚合在一起，作为一个独立的逻辑代理来使用；
- 队列高可用：队列可以在集群中的机器上进行镜像，以确保在硬件问题下还保证消息安全；
- 多种协议的支持：RabbitMQ支持多种消息队列协议；
- 多语言支持：RabbitMQ的服务器端用Erlang语言编写，其客户端支持基本所有编程语言；
- 管理界面: RabbitMQ有一个易用的用户界面，使得用户可以监控和管理消息Broker的许多方面；
- 跟踪机制：如果消息异常，RabbitMQ提供消息跟踪机制，使用者可以跟踪发现异常；
- 插件机制：提供了许多插件，来从多方面进行扩展，也可以编写自己的插件；

![](http://image.blueskykong.com/amqp-theory.jpg)

RabbitMQ的优点有：

- 由于erlang语言的特性，mq 性能较好，高并发；
- 健壮、稳定、易用、跨平台、支持多种语言、文档齐全；
- 有消息确认机制和持久化机制，可靠性高；
- 高度可定制的路由；
- 管理界面较丰富，在互联网公司也有较大规模的应用；
- 社区活跃度高；

RabbitMQ的缺点有：

- 尽管结合erlang语言本身的并发优势，性能较好，但是不利于做二次开发和维护；
- 实现了代理架构，意味着消息在发送到客户端之前可以在中央节点上排队。此特性使得RabbitMQ易于使用和部署，但是使得其运行速度较慢，因为中央节点增加了延迟，消息封装后也比较大；
- 需要学习比较复杂的接口和协议，学习和维护成本较高；

##### RocketMQ
RocketMQ出自阿里公司的开源产品，用 Java 语言实现，在设计时参考了 Kafka，并做出了自己的一些改进，消息可靠性上比 Kafka 更好。RocketMQ在阿里集团被广泛应用在订单，交易，充值，流计算，消息推送，日志流式处理，binglog分发等场景。

RocketMQ的主要特性有：

- 是一个队列模型的消息中间件，具有高性能、高可靠、高实时、分布式特点；
- Producer、Consumer、队列都可以分布式；
- Producer向一些队列轮流发送消息，队列集合称为Topic，Consumer如果做广播消费，则一个consumer实例消费这个Topic对应的所有队列，如果做集群消费，则多个Consumer实例平均消费这个topic对应的队列集合；
- 能够保证严格的消息顺序；
- 提供丰富的消息拉取模式；
- 高效的订阅者水平扩展能力；
- 实时的消息订阅机制；
- 亿级消息堆积能力；
- 较少的依赖；

![](http://image.blueskykong.com/rocketmq.png)
RocketMQ的优点有：

- 单机支持 1 万以上持久化队列；
- RocketMQ 的所有消息都是持久化的，先写入系统 PAGECACHE，然后刷盘，可以保证内存与磁盘都有一份数据；
- 模型简单，接口易用（JMS 的接口很多场合并不太实用）；
- 性能非常好，可以大量堆积消息在broker中；
- 支持多种消费，包括集群消费、广播消费等。
- 各个环节分布式扩展设计，主从HA；

RocketMQ的缺点有：

- 支持的客户端语言不多，目前是java及c++，其中c++不成熟；
- RocketMQ社区关注度及成熟度也不及前两者；
- 没有web管理界面，提供了一个CLI(命令行界面)管理工具带来查询、管理和诊断各种问题；
- 没有在消息队列的核心部分实现JMS等接口；


## 原理简介

![](http://image.blueskykong.com/stream-source.jpg)

如图是Stream源码的流程图。Stream首先会动态注册相关BeanDefinition，并且处理@StreamListener注解；然后在Bean实例初始化之后，会调用BindingService进行服务绑定；BindingService在绑定服务时会首先获取特定的Binder绑定器，然后绑定Producer和Consumer；最后Stream的相关实例就会进行发送和接受消息的处理。

## 编程模型
Spring Cloud Stream提供了一系列的预先定义的注解来声明输入型和输出型channel，业务系统基于这些channel与消息中间件进行通信，而不是直接与消息中间件进行通信。

### 声明和绑定Channels
通过给业务应用的配置类添加`@EnableBinding`注解来将一个Spring应用转变成Spring Cloud Stream应用。`@EnableBinding`注解本身拥有`@Configuration`元注解来进行相关配置并且会触发Spring Cloud Stream框架的初始化机制。

```java
@Configuration
@EnableIntegration
public @interface EnableBinding {
    ...
    Class<?>[] value() default {};
}
```

`@EnableBinding`注解可以使用声明输入型和输出行channel的接口类作为其value属性值。`@EnableBinding`注解只能使用在业务系统的Configuration类上，可以提供尽可能多的接口类作为该注解的value属性值，比如说`@EnableBinding(value={Order.class, Payment.class})`，Order和Payment都是声明了channel的接口类。
在Spring Cloud Stream应用中，接口类可以通过被`@Input`和`@Output`注解修饰的函数来声明的输入型和输出型channels。

```java
public interface OnlineStore{
    @Input
    SubscribableChannel orders();  #声明输入型channel,表示接收订单
    @Output
    MessageChannel stock();         #声明输出型channel，表示向供应商进货
}
```

使用这个接口类当作`@EnableBinding`的value属性值可以触发Stream框架的初始化机制，创建两个channel，名字分别为orders和stock，orders是输入型channel，而stock是输出型channel。

```java
@EnableBinding(OnlineStore.class)
public class ShopConfiguration {
   ...
}
```


### 自定义信道
使用`@Input`和`@Output`注解，编程人员可以给每个信道一个自定义的名称，使用这个自定义信道，可以与消息对立中相应的Channel进行交互。

```java
public interface OnlineStore{
    @Input("inboundOrders")
    SubscribableChannel orders();
}
```

在上边代码示例中，自定义信道的名称为inboundOrders，Stream框架会创建出名为inboundOrders的信道。

Spring Cloud Stream提供了预先设置的三种接口来定义输入型channel和输出型channel，它们是Source、Sink和Processor。Source用来声明输出型channel，它的信道名称为output。Sink用来声明输入型channel，它的信道名称为input。Processor则用来声明输出输入型的channel。

```java
# Source
public interface Source {
  String OUTPUT = "output";
  @Output(Source.OUTPUT)
  MessageChannel output();
}
# Sink
public interface Sink {
  String INPUT = "input";
  @Input(Sink.INPUT)
  SubscribableChannel input();
}
# Processor
public interface Processor extends Source, Sink {
}
```
### 产生和消费消息
使用Spring Integration注解或者Spring Cloud Stream的@StreamListener注解可以进行消息的发送和消费。`@StreamListener`注解基于Spring Messaging注解(比如说@MessageMapping，@JmsListener，@RabbitListener)，除此之外，该注解添加了内容（content）类型管理和类型强制等特性。

作为Spring Integration的补充，Spring Cloud Stream提供了它自己的@StreamListener注解，该注解构建在Spring Messaging注解的基础上，比如说@MessageMapping、@JmsListener和`@RabbitListener`。`@StreamListener`注解提供了更加简便处理输入消息的模型。

Spring Cloud Stream提供了可扩展的消息转换（MessageConverter）机制来处理数据转换，并将转换后的数据分配给对应的被`@StreamListener`修饰的方法。下面这个例子展示了一个处理外部订单消息的应用。

```java
@EnableBinding(Sink.class)
public class OrderHandler {
  @Autowired
  OrderService orderService;
  @StreamListener(Sink.INPUT)
  public void handle(Order order) {
    orderService.handle(order);
  }
}
```
假设，输入的Message对象有一个string类型的Payload和一个值为application/json的contentType。在使用`@StreamListener`时，`MessageConverter`会使用消息的contentType来解析String类型的Payload并赋值给Order对象。
就像其他的Spring Messaging方法一样，被`@StreamListener`注解的方法的参数可以使用`@Payload`和`@Headers`进行注解。对于返回数据的方法，必须使用`@SendTo`注解来指定该返回数据发送到哪个输出型channel。

```java
@EnableBinding(Processor.class)
public class TransformProcessor {
  @Autowired
  VotingService votingService;
  @StreamListener(Processor.INPUT)
  @SendTo(Processor.OUTPUT)
  public VoteResult handle(Vote vote) {
    return votingService.record(vote);
  }
}
```

Spring Cloud Stream支持将消息分配到多个`@StreamListener`修饰的方法。为了能使用该分配机制，一个方法必须首先满足下列条件：

- 方法不能有返回值。
- 方法必须是单独一类消息的处理函数。

使用注解的condition属性中的SpEL表达式可以设置`@StreamListener`接收消息的条件判断。所有匹配了该condition的方法都会在同一个线程中被调用，但是方法调用相对顺序不能保证。

下面就是一个`@StreamListener`分配消息的例子。在这个例子中，所有头部属性type对应的值为food的消息都会被分配给receiveFoodOrder方法，所有头部属性type对应的值为compute的消息都会被分配给receiveComputeOrder方法。

```java
@EnableBinding(Sink.class)
@EnableAutoConfiguration
public static class TestPojoWithAnnotatedArguments {
    @StreamListener(target = Sink.INPUT, condition = "headers['type']=='food'")
    public void receiveFoodOrder(@Payload FoodOrder foodOrder) {
       // handle the message
    }
    @StreamListener(target = Sink.INPUT, condition = "headers['type']=='compute'")
    public void receiveComputeOrder(@Payload ComputeOrder computeOrder) {
       // handle the message
    }
}
```

## 小结
本文主要介绍了Spring Cloud Stream中涉及到的相关概念，重点介绍了Spring Cloud Stream的编程模型，为后面文章实战应用和自定义奠定一些基础。Spring Cloud Stream封装了多种消息中间件的操作接口，目前只有kafka和rabbitmq，下一篇将会介绍如何自已实现一个Rocketmq的绑定器。

