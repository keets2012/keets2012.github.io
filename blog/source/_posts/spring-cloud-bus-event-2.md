---
title: Spring Cloud Bus中的事件的订阅与发布（二）
date: 2018-3-8
categories: 微服务
tags:
- Spring Cloud
---
在之前的文章[Spring Cloud Bus中的事件的订阅与发布（一）](http://blueskykong.com/2018/02/13/spring-cloud-bus-event/)介绍了消息总线的相关事件。
本文主要介绍消息总线的事件监听器以及消息的订阅与发布。

## 事件监听器

`Spring Cloud Bus`中，事件监听器的定义可以是实现`ApplicationListener`接口，或者是使用`@EventListener`注解的形式。我们看一下事件监听器的类图。

![listener](http://ovcjgn2x0.bkt.clouddn.com/bus-listener.png "监听器")
`ApplicationListener`接口实现有两个：刷新监听器`RefreshListener`和环境变更监听器`EnvironmentChangeListener`。

### RefreshListener   
`RefreshListener`对应的事件是`RefreshRemoteApplicationEvent`，

```java
public class RefreshListener
		implements ApplicationListener<RefreshRemoteApplicationEvent> {
	private ContextRefresher contextRefresher;

	public RefreshListener(ContextRefresher contextRefresher) {
		this.contextRefresher = contextRefresher;
	}

	@Override
	public void onApplicationEvent(RefreshRemoteApplicationEvent event) {
		Set<String> keys = contextRefresher.refresh();
		log.info("Received remote refresh request. Keys refreshed " + keys);
	}
}
```
对于刷新时间的处理，调用`ContextRefresher`的`refresh()`方法，而定义在Spring Cloud Context中的`ContextRefresher`用于提供上下文刷新的功能。我们具体看一下`refresh()`方法。

```java
	public synchronized Set<String> refresh() {
		Map<String, Object> before = extract(
				this.context.getEnvironment().getPropertySources());
		addConfigFilesToEnvironment();
		Set<String> keys = changes(before,
				extract(this.context.getEnvironment().getPropertySources())).keySet();
		this.context.publishEvent(new EnvironmentChangeEvent(keys));
		this.scope.refreshAll();
		return keys;
	}
```
实现很简单，先获取之前环境变量的key-value，然后重新加载新的配置环境文件，通过比对新旧环境变量的map集合，然后发布新的环境变更`EnvironmentChangeEvent`的事件。`this.scope.refreshAll()`销毁了在这个范围内，当前实例的所有bean并在下次方法的执行时强制刷新。

### EnvironmentChangeListener   
`EnvironmentChangeListener`对应的事件类是`EnvironmentChangeRemoteApplicationEvent`。

```java
public class EnvironmentChangeListener
		implements ApplicationListener<EnvironmentChangeRemoteApplicationEvent> {
	@Autowired
	private EnvironmentManager env;

	@Override
	public void onApplicationEvent(EnvironmentChangeRemoteApplicationEvent event) {
		Map<String, String> values = event.getValues();
		for (Map.Entry<String, String> entry : values.entrySet()) {
			env.setProperty(entry.getKey(), entry.getValue());
		}
	}
}
```
在`RefreshListener`的实现中，可以知道该事件的实现最终又发布了一个新的事件`EnvironmentChangeListener`。在刷新监听器中，构造了变更了的环境变量的map，交给环境变更监听器。上面对环境变更事件的处理，遍历变更了的配置环境属性，并在本地应用程序的环境中将新的属性值设置到对应的键。

### TraceListener   
`TraceListener`的实现是通过注解`@EventListener`的形式，监听的事件为：确认事件`AckRemoteApplicationEvent`和发送事件`SentApplicationEvent`。

```java
@EventListener
	public void onAck(AckRemoteApplicationEvent event) {
		this.repository.add(getReceivedTrace(event));
	}

	@EventListener
	public void onSend(SentApplicationEvent event) {
		this.repository.add(getSentTrace(event));
	}

	protected Map<String, Object> getSentTrace(SentApplicationEvent event) {
		Map<String, Object> map = new LinkedHashMap<String, Object>();
		map.put("signal", "spring.cloud.bus.sent");
		map.put("type", event.getType().getSimpleName());
		map.put("id", event.getId());
		map.put("origin", event.getOriginService());
		map.put("destination", event.getDestinationService());
		if (log.isDebugEnabled()) {
			log.debug(map);
		}
		return map;
	}

	protected Map<String, Object> getReceivedTrace(AckRemoteApplicationEvent event) {
		Map<String, Object> map = new LinkedHashMap<String, Object>();
		map.put("signal", "spring.cloud.bus.ack");
		map.put("event", event.getEvent().getSimpleName());
		map.put("id", event.getAckId());
		map.put("origin", event.getOriginService());
		map.put("destination", event.getAckDestinationService());
		if (log.isDebugEnabled()) {
			log.debug(map);
		}
		return map;
	}
```
在SentTrace中，主要记录了signal、事件类型type、id、源服务origin和目的服务destination的属性值。而在ReceivedTrace中，表示对事件的确认，主要记录了signal、事件类型event、id、源服务origin和目的服务destination的属性值。这些信息默认存储于内存中，可以通过`/trace`端点获取最近的事件信息，如下图所示：

```
{
    "timestamp": 1517229555629,
    "info": {
        "signal": "spring.cloud.bus.sent",
        "type": "RefreshRemoteApplicationEvent",
        "id": "c73a9792-9409-47af-993c-65526edf0070",
        "origin": "config-server:8888",
        "destination": "config-client:8000:**"
    }
},
{
	"timestamp": 1517227659384,
	"info": {
	    "signal": "spring.cloud.bus.ack",
	    "event": "RefreshRemoteApplicationEvent",
	    "id": "846f3a17-c344-4d29-93f3-01b73c5bf58f",
	    "origin": "config-client:8000",
	    "destination": "config-client:8000:**"
	}
}
```

至于事件的发起，我们将在下一节结合消息的订阅与发布一起讲解。

## 消息的订阅与发布
`Spring Cloud Bus`基于`Spring Cloud Stream`，对特定主题的消息进行订阅与发布，事件以消息的形式传递到其他服务实例。
### 通道定义
既然是基于stream，我们首先看一下input和output的通道定义。

```java
public interface SpringCloudBusClient {

String INPUT = "springCloudBusInput";

String OUTPUT = "springCloudBusOutput";

@Output(SpringCloudBusClient.OUTPUT)
MessageChannel springCloudBusOutput();

@Input(SpringCloudBusClient.INPUT)
SubscribableChannel springCloudBusInput();
}
```

可以看到，bus中定义了`springCloudBusInput`和`springCloudBusOutput`两个通道，分别用于定于订阅与发布`springCloudBus`的消息。


### bus属性定义
其次，我们看一下bus中关于stream的属性定义。在基础应用中我们就知道bus订阅的话题是`springCloudBus`，下面看一下在bus中的其他属性的定义。

```java
@ConfigurationProperties("spring.cloud.bus")
public class BusProperties {

//环境变更相关的属性
private Env env = new Env();
// 刷新事件相关的属性
private Refresh refresh = new Refresh();
//与ack相关的属性
private Ack ack = new Ack();
//与追踪ack相关的属性
private Trace trace = new Trace();
//Spring Cloud Stream消息的话题
private String destination = "springCloudBus";

//标志位，bus是否可用
private boolean enabled = true;

...
}
```
上面的bus属性，设置了一些默认值，正好与事实也是相符的，我们没有进行任何`spring.cloud.bus`配置也能够进行正常运行。通过在配置文件中修改相应的属性，实现bus的更多功能扩展。env、refresh、ack和trace分别对应不同的事件，在配置文件中有一个开关属性，默认都是开启的，我们可以根据需要进行关闭。

### 消息的监听与发送
上面两部分讲了stream通道和基本属性的定义，最后我们看下bus中对指定主题的消息如何发送与监听处理。在META-INF/spring.factories配置了`EnableAutoConfiguration`配置项为`BusAutoConfiguration`，在服务启动时会自动加载到Spring容器中，其中对于指定主题的消息如何发送与监听处理如下：

```java
@Configuration
@ConditionalOnBusEnabled //bus启用的开关
@EnableBinding(SpringCloudBusClient.class) //绑定通道
@EnableConfigurationProperties(BusProperties.class)
public class BusAutoConfiguration implements ApplicationEventPublisherAware {

//注入source接口，用于发送消息
@Autowired
@Output(SpringCloudBusClient.OUTPUT)
private MessageChannel cloudBusOutboundChannel;

// 监听RemoteApplicationEvent事件
@EventListener(classes = RemoteApplicationEvent.class)
public void acceptLocal(RemoteApplicationEvent event) {
    if (this.serviceMatcher.isFromSelf(event)
            && !(event instanceof AckRemoteApplicationEvent)) {
        //当事件是来自自己的并且不是ack事件，则发送消息
    this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(event).build());
    }
}
//消息的消费，也是事件的发起
@StreamListener(SpringCloudBusClient.INPUT)
public void acceptRemote(RemoteApplicationEvent event) {
    if (event instanceof AckRemoteApplicationEvent) {
        //ack事件
        if (this.bus.getTrace().isEnabled() && !this.serviceMatcher.isFromSelf(event)
                && this.applicationEventPublisher != null) {
            //当开启bus追踪且不是自己的ack事件，则通知所有的注册该事件的监听者，否则直接返回
            this.applicationEventPublisher.publishEvent(event);
        }
        return;
    }
    //消费消息，该消息属于自己
    if (this.serviceMatcher.isForSelf(event)
            && this.applicationEventPublisher != null) {
        //不是自己发布的事件，正常处理
        if (!this.serviceMatcher.isFromSelf(event)) {
            this.applicationEventPublisher.publishEvent(event);
        }
        //消费之后，需要发送ack确认事件
        if (this.bus.getAck().isEnabled()) {
            AckRemoteApplicationEvent ack = new AckRemoteApplicationEvent(this,
                    this.serviceMatcher.getServiceId(),
                    this.bus.getAck().getDestinationService(),
                    event.getDestinationService(), event.getId(), event.getClass());
            this.cloudBusOutboundChannel
                    .send(MessageBuilder.withPayload(ack).build());
            this.applicationEventPublisher.publishEvent(ack);
        }
    }
    //事件追踪相关，若是开启追踪事件则执行
    if (this.bus.getTrace().isEnabled() && this.applicationEventPublisher != null) {
        // 不论其来源，准备发送事件，发布了之后供本地消费
        this.applicationEventPublisher.publishEvent(new SentApplicationEvent(this,
                event.getOriginService(), event.getDestinationService(),
                event.getId(), event.getClass()));
    }
}

//...
}
```

`@ConditionalOnBusEnabled`注解是bus的开关，默认开启。`@EnableBinding`绑定了`SpringCloudBusClient`中定义的通道。在应用服务启动时，自动化配置类加载了bus的API端点、刷新、ACK追踪以及bus环境变量的配置等beans。`@Output`表示输出output绑定目标将由框架创建，由该通道发送消息。
还涉及到上面列出来的两个主要方法：`acceptLocal`和`acceptRemote`。

`acceptLocal`是一个基于注解实现的事件监听器，监听的事件类型是`RemoteApplicationEvent`，对于该事件的处理方法是，当事件是来自自己的并且不是ack事件，则发送消息。

`@StreamListener`注解是`Spring Cloud Stream`中提供的，用来标识一个方法作为`@EnableBinding`绑定的input通道的监听器。`acceptRemote`方法，传递的参数`RemoteApplicationEvent`就是stream中的消息。如果是确认类事件，当开启了事件追踪且事件不是来自于自身，则发布该事件，对于确认类事件，处理已经完成；
如果自身需要处理该事件且该事件不是来自自身，则发布该事件。需要注意的是，当开启事件追踪时，构造一个确认事件并将该事件发布；最后，当开启了事件追踪，这边的处理是注册已发送的事件，以便发布供本地消费，而不论其来源。

## 总结
本文在[上一篇](http://blueskykong.com/2018/02/13/spring-cloud-bus-event/)介绍Spring Cloud Bus中的事件基础上，结合源码继续介绍事件的监听器以及事件的订阅与发布是如何在消息总线中实现的。
消息总线常用于传播状态的变更和管理指令的发布。而消息总线最常用的场景就是更新应用服务的配置信息，需要结合Config Server使用，当然消息总线的实现其实是基于Spring Cloud Stream，Stream封装了各种不同的MQ中间件，产生的消息实则是推送配置信息的变更。

### 参考
[Spring Cloud Bus-v1.3.3](http://cloud.spring.io/spring-cloud-static/spring-cloud-bus/1.3.3.RELEASE/single/spring-cloud-bus.html)

