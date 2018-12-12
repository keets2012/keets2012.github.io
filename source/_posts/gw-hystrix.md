---
title: Hystrix断路器在微服务网关中的应用（Spring Cloud Gateway）
categories: 微服务
tags:
  - zuul
  - gateway
date: 2018-12-11 00:00:00
---

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
### 依赖版本
spring-boot-starter-parent的版本为`2.0.3.RELEASE`。   
Spring Cloud的版本为`Finchley.RELEASE`，对应的spring-cloud-gateway版本为`2.0.0.RELEASE`。

### 报错分析
使用POSTMAN发送GET请求，不会出现第一小节的异常。当改为POST请求之后，`HystrixGatewayFilterFactory`抛出异常。使得刚开始的猜想往为什么不支持POST请求上考虑。打开debug日志，我们得到如下更为详细的输出：

```
AbstractCommand$22.call[821] : HystrixCommand execution COMMAND_EXCEPTION and fallback failed.

java.lang.IllegalArgumentException: Actual request host must not be null
at org.springframework.util.Assert.notNull(Assert.java:193)
at org.springframework.web.cors.reactive.CorsUtils.isSameOrigin(CorsUtils.java:74)
at org.springframework.web.cors.reactive.DefaultCorsProcessor.process(DefaultCorsProcessor.java:70)
at org.springframework.web.reactive.handler.AbstractHandlerMapping.lambda$getHandler$1(AbstractHandlerMapping.java:152)
at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:107)
at reactor.core.publisher.Operators$ScalarSubscription.request(Operators.java:1640)
at 
```
从上面的日志可以看出，最后的错误定位到了默认的CORS处理器`DefaultCorsProcessor`的实现。

```java
public boolean processRequest(@Nullable CorsConfiguration config, HttpServletRequest request,
			HttpServletResponse response) throws IOException {
		//是否为CORS请求（包含Origin头部）
		if (!CorsUtils.isCorsRequest(request)) {
			return true; //不是则直接返回
		}

		ServletServerHttpResponse serverResponse = new ServletServerHttpResponse(response);
		//根据响应serverResponse判断
		if (responseHasCors(serverResponse)) {
			logger.debug("Skip CORS processing: response already contains \"Access-Control-Allow-Origin\" header");
			return true;
		}

		ServletServerHttpRequest serverRequest = new ServletServerHttpRequest(request);
		if (WebUtils.isSameOrigin(serverRequest)) {
			logger.debug("Skip CORS processing: request is from same origin");
			return true;
		}

		boolean preFlightRequest = CorsUtils.isPreFlightRequest(request);
		if (config == null) {
			if (preFlightRequest) {
				rejectRequest(serverResponse);
				return false;
			}
			else {
				return true;
			}
		}

		return handleInternal(serverRequest, serverResponse, config, preFlightRequest);
	}
```
Access-Control-Allow-Origin是HTML5中定义的一种解决资源跨域的策略。

他是通过服务器端返回带有Access-Control-Allow-Origin标识的Response header，用来解决资源的跨域权限问题。

基于Origin、Host、Forwarded、X-Forwarded-Proto、X-Forwarded-Host、X-Forwarded-Port等头部，校验请求是否同源。
## 解决思路

### 移除

### CORS配置

### Spring Cloud Gateway版本升级

## 小结

#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](http://image.blueskykong.com/wechat-public-code.jpg)
