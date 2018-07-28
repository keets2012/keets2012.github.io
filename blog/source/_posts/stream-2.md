---
title: Spring Cloud Stream应用与自定义RocketMQ Binder：实现RocketMQ绑定器
date: 2018-6-25
categories: 微服务
tags:
- 微服务
- 消息中间件
---
前言： 本文作者张天，节选自笔者与其合著的《Spring Cloud微服务架构进阶》，即将在八月出版问世。本文将其中Spring Cloud Stream应用与自定义Rocketmq Binder的内容抽取出来，主要介绍实现Spring Cloud Stream 的RocketMQ绑定器。

## Stream的Binder机制
在上一篇中，介绍了Spring Cloud Stream基本的概念及其编程模型。除此之外，Spring Cloud Stream提供了Binder接口来用于和外部消息队列进行绑定。本文将讲述Binder SPI的基本概念，主要组件和实现细节。
Binder SPI通过一系列的接口，工具类和检测机制提供了与外部消息队列绑定的绑定器机制。SPI的关键点是Binder接口，这个接口负责提供和外部消息队列进行绑定的具体实现。

```java
public interface Binder<T, C extends ConsumerProperties, P extends ProducerProperties> {
    Binding<T> bindConsumer(String name, String group, T inboundBindTarget, C consumerProperties);
    Binding<T> bindProducer(String name, T outboundBindTarget, P producerProperties);
}
```

一个典型的自定义Binder组件实现应该包括以下几点：

- 一个实现Binder接口的类。
- 一个Spring的@Configuration类来创建上述类型的实例。
- 在classpath上一个包含自定义Binder相关配置类的META-INF/spring.binders文件，比如说：

```
kafka:\
org.springframework.cloud.stream.binder.kafka.config.KafkaBinderConfiguration
```

Spring Cloud Stream基于Binder SPI的实现来进行channel和消息队列的绑定任务。不同类型的消息队列中间件实现了不同的绑定器Binder。比如说：Spring-Cloud-Stream-Binder-Kafka是针对Kafka的Binder实现，而Spring-Cloud-Stream-Binder-Rabbit则是针对RabbitMQ的Binder实现。

Spring Cloud Stream依赖于Spring Boot的自动配置机制来配置Binder。如果一个Binder实现在项目的classpath中被发现，Spring Cloud Stream将会自动使用它。比如说，一个Spring Cloud Stream项目需要绑定RabbitMQ中间件的Binder，在pom文件中加入下面的依赖来轻松实现。

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

## Binder For RocketMQ
Spring Cloud Stream为接入不同的消息队列提供了一整套的自定义机制，通过为每个消息队里开发一个Binder来接入该消息队列。目前官方认定的Binder为rabbitmq binder和kafka binder。但是开发人员可以基于Stream Binder的机制来制定自己的Binder。下面我们就构建一个简单的RocketMQ的Binder。

### 配置类
需要在resources/META-INF/spring.binders文件中配置有关RocketMQ的Configuration类，该配置类会使用@Import来导入为RocketMQ制定的`RocketMessageChannelBinderConfiguration`。

```xml
rocket:\
org.springframework.cloud.stream.binder.rocket.config.RocketServiceAutoConfiguration
```

`RocketMessageChannelBinderConfiguration`将会提供两个极其重要的bean实例，分别为`RocketMessageChannelBinder`和`RocketExchangeQueueProvisioner`。`RocketMessageChannelBinder`主要是用于channel和消息队列的绑定，而`RocketExchangeQueueProvisioner`则封装了RocketMQ的相关API，可以用于创建消息队列的基础组件，比如说队列，交换器等。

```java
@Configuration
public class RocketMessageChannelBinderConfiguration {
    @Autowired
    private ConnectionFactory rocketConnectionFactory;
    @Autowired
    private RocketProperties  rocketProperties;
    @Bean
    RocketMessageChannelBinder rocketMessageChannelBinder() throws Exception {
        RocketMessageChannelBinder binder = new RocketMessageChannelBinder(this.rocketConnectionFactory,
                this.rocketProperties, provisioningProvider());
        return binder;
    }
    @Bean
    RocketExchangeQueueProvisioner provisioningProvider() {
        return new RocketExchangeQueueProvisioner(this.rocketConnectionFactory);
    }
}
```
`RocketMessageChannelBinder`继承了抽象类`AbstractMessageChannelBinder`，并实现了#producerMessageHandler和#createConsumerEndpoint函数。

MessageHandler有向消息队列发送消息的能力，#createProducerMessageHandler函数就是为了创建MessageHandler对象，来将输出型Channel的消息发送到消息队列上。

```java
protected MessageHandler createProducerMessageHandler(
        ProducerDestination destination,             
        ExtendedProducerProperties<RocketProducerProperties> producerProperties,                    
        MessageChannel errorChannel) 
        throws Exception {
    final AmqpOutboundEndpoint endpoint = new AmqpOutboundEndpoint(
            buildRocketTemplate(producerProperties.getExtension(), errorChannel != null));
    return endpoint;
}
```

MessageProducer能够从消息队列接收消息，并将该消息发送输入型Channel。

```java
@Override
protected MessageProducer createConsumerEndpoint(ConsumerDestination consumerDestination, String group,
                                                    ExtendedConsumerProperties<RocketConsumerProperties> properties) throws Exception {
    SimpleRocketMessageListenerContainer listenerContainer = new SimpleRocketMessageListenerContainer();
    RocketInboundChannelAdapter rocketInboundChannelAdapter = new RocketInboundChannelAdapter(listenerContainer);
    return rocketInboundChannelAdapter;
}
```
### 消息接收功能的实现流程
类似于RabbitMQ的Binder，需要实现下面一系列的类来实现从RocketMQ到对应MessageChannel的消息传递。
1. RocketBlockingQueueConsumer.InnerConsumer实现了MessageListenerConcurrently来接收RocketMQ传递的消息。
2. RocketBlockingQueueConsumer将InnerConsumer注册给RocketMQ的DefaultMQPushConsumer来接收RocketMQ传递过来的消息，并存储在自身的阻塞队列中。供SimpleRocketMessageListenerContainer获取。
3. SimpleRocketMessageListenerContainer,启动一个线程来不停从RocketBlockingQueueConsumer获取消息，然后调用RocketInboundChannelAdapter.Listener的回调函数，将消息传递给RocketInboundChannelAdapter。
4.	RocketInboundChannelAdapter.Listener供SimpleRocketMessageListenerContainer回调，将消息发送给RocketInboundChannelAdapter。
5.	RocketInboundChannelAdapter，接受SimpleRocketMessageListenerContainer传递过来的消息，然后通过MessageTemplate发送给相应的MessageChannel。从而传递给由@StreamListener的修饰的函数。

### InnerConsumer接收RocketMQ消息

InnerConsumer实现的MessageListenerConcurrently接口是RocketMQ中用于并发接受异步消息的接口，该接口可以接收到RocketMQ发送过来的异步消息。而InnerConsumer在接受到消息之后，会将消息封装成RocketDelivery加入到阻塞队列中。


RocketBlockingQueueConsumer有一个阻塞队列来存储RocketMQ传递给RocketBlockingQueueConsumer.InnerConsumer的消息，而nextMessage函数可以从阻塞队列中拉取一个消息并返回。

### AsyncMessageProcessingConsumer获取消息

SimpleRocketMessageListenerContainer.AsyncMessageProcessingConsumer是实现了Runnable接口，在run()接口中会无限循环地调用SimpleRocketMessageListenerContainer本身的receiveAndExecute。

```java
@Override
public void run() {
    if (!isActive()) {
        return;
    }
    try {
        //只要consumer的状态正常，就会一直循环
        while (isActive(this.consumer) || this.consumer.hasDelivery() || !this.consumer.cancelled()) {
            try {
                boolean receivedOk = receiveAndExecute(this.consumer);
            }
            catch (ListenerExecutionFailedException ex) {
                if (ex.getCause() instanceof NoSuchMethodException) {
                    throw new FatalListenerExecutionException("Invalid listener", ex);
                }
            }
            catch (AmqpRejectAndDontRequeueException rejectEx) {
            } catch (Throwable e) {
            }
        }
    } catch (Exception e) {
    }
    finally {
        if (getTransactionManager() != null) {
            ConsumerChannelRegistry.unRegisterConsumerChannel();
        }
    }
    this.start.countDown();
    if (!isActive(this.consumer) || aborted) {
        this.consumer.stop();
    }
    else {
        restart(this.consumer);
    }
}
```
函数#receiveAndExecute最终的作用就是调用RocketBlockingQueueConsumer的nextMessage，然后再将消息调用messageListener.onMessage函数将消息传递出去。

### 初始化RocketBlockingQueueConsumer和AsyncMessageProcessingConsumer
SimpleRocketMessageListenerContainer的doStart函数会初始化RocketBlockingQueueConsumer并且启动SimpleRocketMessageListenerContainer的AsyncMessageProcessingConsumer会无限循环地从RocketBlockingQueueConsumer中获取RocketMQ传递过来的消息。

```java
private void doStart() {
    synchronized (this.lifecycleMonitor) {
        this.active = true;
        this.running = true;
        this.lifecycleMonitor.notifyAll();
    }
    synchronized (this.consumersMonitor) {
        if (this.consumers != null) {
            throw new IllegalStateException("A stopped container should not have consumers");
        }
        //初始化Consumer
        int newConsumers = initializeConsumers();
        if (this.consumers == null) {
            return;
        }
        if (newConsumers <= 0) {
            return;
        }
        Set<SimpleRocketMessageListenerContainer.AsyncMessageProcessingConsumer> processors =
                new HashSet<>();
        //对于每个RocketBlockingQueueConsumer启动一个
        //AsyncMessageProcessingConsumer来执行任务
        for (RocketBlockingQueueConsumer consumer : this.consumers) {
            SimpleRocketMessageListenerContainer.AsyncMessageProcessingConsumer
                    processor = new SimpleRocketMessageListenerContainer.AsyncMessageProcessingConsumer(consumer);
            processors.add(processor);
            getTaskExecutor().execute(processor);
        }
    }
}
```

### 发送消息给MessageChannel

RocketInboundChannelAdapter实现了MessageProducer接口。它主要将SimpleRocketMessageListenerContainer传递过来的消息经过MessageTemplate传递给MessageChannel。

接下来则是RocketInboundChannelAdapter.Listener的实现，它就是RocketBlockingQueueConsumer.nextMessage函数中的messageListener。

```java
public class Listener implements ChannelAwareMessageListener, RetryListener {
    public void onMessage(Message message, Channel channel) throws Exception {
        try {
            this.createAndSend(message, channel);
        } catch (RuntimeException var7) {
            if (RocketInboundChannelAdapter.this.getErrorChannel() == null) {
                throw var7;
            }
       RocketInboundChannelAdapter.this.getMessagingTemplate().send(RocketInboundChannelAdapter.this.getErrorChannel(), RocketInboundChannelAdapter.this.buildErrorMessage((org.springframework.messaging.Message)null, new ListenerExecutionFailedException("Message conversion failed", var7, message)));
        }
    }
    private void createAndSend(Message message, Channel channel) {
        org.springframework.messaging.Message<Object> messagingMessage = this.createMessage(message, channel);
        RocketInboundChannelAdapter.this.sendMessage(messagingMessage);
    }
    private org.springframework.messaging.Message<Object> createMessage(Message message, Channel channel) {
        Object payload = RocketInboundChannelAdapter.this.messageConverter.fromMessage(message);
        org.springframework.messaging.Message<Object> messagingMessage = RocketInboundChannelAdapter.this.getMessageBuilderFactory().withPayload(payload).build();
        return messagingMessage;
    }
}
```

### RocketMQ的管理器

RocketProvisioningProvider实现了ProvisioningProvider接口，它有两个函数：provisionProducerDestination和provisionConsumerDestination，分别用于创建ProducerDestination和ConsumerDestination。RocketProvisioningProvider的实现类似于RabbitProvisioningProvider。只不过在声明队列，交换器和绑定时使用了RocketAdmin所实现的RocketMQ的相关API。

## 总结
本文概要介绍了Spring Cloud Stream的Rocketmq绑定器的实现，限于篇幅不展开具体的代码讲解。读者感兴趣，可以关注GitHub上的代码。根据Spring Cloud Stream抽象的接口，我们可以自由地实现各种消息队列的绑定器。

项目GitHub地址：https://github.com/ztelur/spring-cloud-stream-binder-rocket

