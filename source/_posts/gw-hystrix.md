---
title: Hystrix断路器在微服务网关中的应用（Spring Cloud Gateway）
categories: 微服务
tags:
  - zuul
  - gateway
date: 2018-12-11 00:00:00
---
# Hystrix断路器在微服务网关中的应用（Spring Cloud Gateway）

## 前文回顾

在之前的一篇文章：[微服务网关Zuul迁移到Spring Cloud Gateway](http://blueskykong.com/2018/09/20/integration-enhanced/)，我们讲解了如何从Zuul迁移到新的组件：Spring Cloud Gateway，以及扩展了微服务网关的功能，包括限流过滤器、断路器过滤器等。然而很多读者在使用的时候反馈，使用POSTMAN发送GET请求测试断路器是正常的，然而POST请求会出现：

```
{
    "timestamp": "2018-10-11T13:07:07.790+0000",
    "path": "/user/body",
    "status": 500,
    "error": "Internal Server Error",
    "message": "fallbackcmd failed and fallback failed."
}
```
看一下网关服务的控制台，`HystrixGatewayFilterFactory`也报错。

```
com.netflix.hystrix.exception.HystrixRuntimeException: fallbackcmd failed and fallback failed.
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:825) ~[hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:804) ~[hystrix-core-1.5.12.jar:1.5.12]
	at rx.internal.operators.OperatorOnErrorResumeNextViaFunction$4.onError(OperatorOnErrorResumeNextViaFunction.java:140) ~[rxjava-1.3.8.jar:1.3.8]
	at rx.internal.operators.OnSubscribeDoOnEach$DoOnEachSubscriber.onError(OnSubscribeDoOnEach.java:87) ~[rxjava-1.3.8.jar:1.3.8]
	at rx.internal.operators.OnSubscribeDoOnEach$DoOnEachSubscriber.onError(OnSubscribeDoOnEach.java:87) ~[rxjava-1.3.8.jar:1.3.8]
	at com.netflix.hystrix.AbstractCommand$DeprecatedOnFallbackHookApplication$1.onError(AbstractCommand.java:1472) ~[hystrix-core-1.5.12.jar:1.5.12]
	at com.netflix.hystrix.AbstractCommand$FallbackHookApplication$1.onError(AbstractCommand.java:1397) ~[hystrix-core-1.5.12.jar:1.5.12]
	at rx.internal.operators.OnSubscribeDoOnEach$DoOnEachSubscriber.onError(OnSubscribeDoOnEach.java:87) ~[rxjava-1.3.8.jar:1.3.8]
	at rx.internal.reactivestreams.SubscriberAdapter.onError(SubscriberAdapter.java:59) ~[rxjava-reactive-streams-1.2.1.jar:1.2.1]

```
本文主要是解决Hystrix过滤器应用过程中的报错问题，并提供正确的使用方式。

## 问题分析


## 解决思路

### 移除

### CORS配置

### Spring Cloud Gateway版本升级

## 小结

#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](http://image.blueskykong.com/wechat-public-code.jpg)

