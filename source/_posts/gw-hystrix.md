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
熔断机制和日常生活中见到电路保险丝是非常相似的，当出现了问题之后，保险丝会自动烧断，以保护我们的电器。在我们的对外提供服务时，当现在服务的提供方出现了问题之后整个的程序将出现错误的信息显示，而这个时候如果不想出现这样的错误信息，而希望替换为一个错误时的内容。

一个服务挂了后续的服务跟着不能用了，这就是雪崩效应。




我们在网关配置了Hystrix断路器的过滤器：

```yaml
      routes:
      - id: hytstrix_route
        uri: lb://user
        order: 6000
        predicates:
        - Path=/user/**
        filters:
        - StripPrefix=1
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallbackcontroller?a=123
```
出现错误之后可以 fallback 错误的处理信息。此外，Hystrix断路器经常结合 Feign一起使用，还需要在Feign（客户端）进行熔断的配置。
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
		//根据serverResponse响应判断Access-Control-Allow-Origin
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
如上是报错部分的代码，这段代码的功能是基于cors的配置，处理给定的请求。首先判断是否为CORS的请求，是则直接返回true；否则判断响应中的头部Access-Control-Allow-Origin是否为空（Access-Control-Allow-Origin是HTML5中定义的一种解决资源跨域的策略。他是通过服务器端返回带有Access-Control-Allow-Origin标识的Response header，用来解决资源的跨域权限问题，表示接受哪些域名的请求）；否则
基于Origin、Host、Forwarded、X-Forwarded-Proto、X-Forwarded-Host、X-Forwarded-Port等头部，校验请求是否同源。

对于非简单请求，CORS机制跨域会首先进行 preflight（一个 OPTIONS 请求）， 该请求成功后才会发送真正的请求。 这一设计旨在确保服务器对 CORS 标准知情，以保护不支持CORS的旧服务器。

到这一步，会判断CORS的配置是否为空，如果为空，且不是一个preflight请求，则返回true，否则返回false；再下一步进入CORS的配置不为空的处理逻辑，此处略过。

这里我们拓展一下，浏览器将CORS请求分为两类：简单请求（simple request）和非简单请求（not-simple-request），简单请求浏览器不会预检，而非简单请求会预检。

这两种方式怎么区分？同时满足下列三大条件，就属于简单请求，否则属于非简单请求

1. 请求方式只能是：GET、POST、HEAD
2. HTTP请求头限制这几种字段：Accept、Accept-Language、Content-Language、Content-Type、Last-Event-ID 
3. Content-type只能取：application/x-www-form-urlencoded、multipart/form-data、text/plain。

对于简单请求，浏览器直接请求，会在请求头信息中，增加一个origin字段，来说明本次请求来自哪个源（协议+域名+端口）。服务器根据这个值，来决定是否同意该请求，服务器返回的响应会多几个头信息字段，如下所示：

1. Access-Control-Allow-Origin：该字段是必须的，* 表示接受任意域名的请求，还可以指定域名
2. Access-Control-Allow-Credentials：该字段可选，是个布尔值，表示是否可以携带cookie，（注意：如果Access-Control-Allow-Origin字段设置*，此字段设为true无效）
3. Access-Control-Allow-Headers：该字段可选，里面可以获取Cache-Control、Content-Type、Expires等，如果想要拿到其他字段，就可以在这个字段中指定。

上面的头信息中，三个与CORS请求相关，都是以Access-Control-开头。

非简单请求是对那种对服务器有特殊要求的请求，比如请求方式是PUT或者DELETE，或者Content-Type字段类型是application/json。都会在正式通信之前，增加一次HTTP请求，称之为预检。浏览器会先询问服务器，当前网页所在域名是否在服务器的许可名单之中，服务器允许之后，浏览器会发出正式的XMLHttpRequest请求，否则会报错。

回顾我们的业务场景，来自客户端的请求，到达网关后将会转发到具体服务，此时对应的服务是down的状态，返回的响应结果为空。我们在网关没有任何的CORS配置，因此按照上述的CORS处理逻辑，返回的结果为false。

当目标服务的状态是正常的，请求得到相应，CORS处理是正常的；因此，出错的根源在于，当我们的请求头中携带Origin时，目标服务的不可用将会导致如上的错误，这显然不是我们想要的结果。
## 解决思路
对于上述问题，围绕CORS的处理，我们有如下几种解决思路。
### 移除请求的头部`Origin`
移除请求的头部`Origin`，从CORS处理的逻辑得知，当该请求不是一个CORS请求（即不包含头部Origin），处理的过程就结束，这样可以避免后续的检查。

修改配置如下：

```
      routes:
      - id: hytstrix_route
        uri: lb://user
        order: 6000
        predicates:
        - Path=/user/**
        filters:
        - StripPrefix=1
        - RemoveRequestHeader=Origin
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallbackcontroller?a=123
```
再次发送请求，无论是GET还是POST，携带头部Origin都可以正常fallback。
### CORS配置
我们还可以增加CORS的过滤器，手动增加响应的头部信息。

```java
@Bean
    public WebFilter corsFilter() {
        return (ServerWebExchange ctx, WebFilterChain chain) -> {
            ServerHttpRequest request = ctx.getRequest();
            if (CorsUtils.isCorsRequest(request)) {
                ServerHttpResponse response = ctx.getResponse();
                HttpHeaders headers = response.getHeaders();
                headers.add("Access-Control-Allow-Origin", ALLOWED_ORIGIN);
                headers.add("Access-Control-Allow-Methods", ALLOWED_METHODS);
                headers.add("Access-Control-Max-Age", MAX_AGE);
                headers.add("Access-Control-Allow-Headers",ALLOWED_HEADERS);
                if (request.getMethod() == HttpMethod.OPTIONS) {
                    response.setStatusCode(HttpStatus.OK);
                    return Mono.empty();
                }
            }
            return chain.filter(ctx);
        };
    }
```
### Spring Cloud Gateway版本升级
自`2.0.1.RELEASE`版本开始，Spring Cloud Gateway提供了全局cors的配置：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
```
通过如上配置，可以实现与上一小节相同的功能。
## 小结
本文主要讲了Hystrix过滤器在网关中的应用时遇到的问题，通过错误信息，debug进源码寻找问题的根源。之后我们分析了问题，并根据问题的根源提出了几种可行的解决方案。

#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](http://image.blueskykong.com/wechat-public-code.jpg)
#### 参考
1. [Hystrix filter throw exception with post request](https://github.com/spring-cloud/spring-cloud-gateway/issues/593)
2. [http跨域时的options请求](https://www.jianshu.com/p/5cf82f092201)


