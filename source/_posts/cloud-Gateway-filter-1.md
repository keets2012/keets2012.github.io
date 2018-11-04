---
title: Spring Cloud Gateway中的过滤器工厂：重试过滤器
categories: 微服务
tags:
  - gateway
abbrlink: 23988
date: 2018-04-25 00:00:00
---
Spring Cloud Gateway基于Spring Boot 2，是Spring Cloud的全新项目，该项目提供了一个构建在Spring 生态之上的API网关。本文基于的Spring Cloud版本为`Finchley M9`，Spring Cloud Gateway对应的版本为`2.0.0.RC1`。

[Spring Cloud Gateway入门](http://blueskykong.com/2018/03/10/cloud-Gateway-1/)一文介绍了全新的Spring Cloud Gateway的一些基础应用。本文将会介绍Spring Cloud Gateway重试过滤器。

## 过滤器
`GatewayFilter`网关过滤器用于拦截和链式处理web请求，可以实现横切的、与应用无关的需求，比如安全、访问超时的设定等等。

```java
public interface GatewayFilter {
	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

接口中定义了唯一的方法`#filter`，处理web请求，并且可以通过给定的过滤器链传递到下一个过滤器。该接口有多个实现类，下面看一下该接口的类图。

![GatewayFilter](http://ovcjgn2x0.bkt.clouddn.com/GatewayFilter.png "GatewayFilter类图")
从类图可以看到，`GatewayFilter`有两个实现类，但是在源码中寻找该接口的用法会发现，在`GatewayFilterFactory`实现类中有内部匿名类，实际是返回了一个 `GatewayFilter` 内部实现类。

Spring Cloud Gateway提供了很多种类的过滤器工厂，网关过滤器有近二十个实现类，总得说来可以分为七类：Header、Parameter、Path、Status、Redirect跳转、Hystrix熔断和RateLimiter限流等。

## 重试过滤器
### 请求的重试
当转发到代理服务时，遇到指定的服务端Error，如httpStatus为500时，我们可以设定重试几次。除了对指定的异常重试之外，还可以指定请求的方法，GET或POST。

实验场景涉及到：网关服务和用户服务。客户端请求经过网关，请求用户服务的API接口，遇到指定的异常时，进行重试。

### 项目准备
示例启动两个服务：Gateway-Server和user-Server。模拟的场景是，客户端请求后端服务，网关提供后端服务的统一入口。后端的服务都注册到服务发现Consul（搭建zk，Eureka都可以，笔者比较习惯使用consul）。网关通过负载均衡转发到具体的后端服务。

#### 用户服务
用户服务注册到Consul上，并提供一个接口/test。

#### 网关服务
引入网关的依赖，并进行相应配置。上一章已经讲过，这里不重复列出代码，具体见源码。

### 服务改造
#### 网关服务
网关服务中，新增一个路由的定义`retry_java`，请求的判定是路径以`/test`为前缀的请求，并将请求转发到user服务。当遇到内部服务错误（状态码为500）时，设定重试的次数为2。当然该路由也可以通过网关服务的配置文件，效果是一样的。

```java
    @Bean
    public RouteLocator retryRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("retry_java", r -> r.path("/test/**")
                        .filters(f -> f.stripPrefix(1)
                                .retry(config -> config.setRetries(2).setStatuses(HttpStatus.INTERNAL_SERVER_ERROR)))
			.uri("lb://user"))
		.build();
    }
```

#### 用户服务
用户服务增加一个API接口，请求中传入参数key和count。

```java
    ConcurrentHashMap<String, AtomicInteger> map = new ConcurrentHashMap<>();

    @GetMapping("/exception")
    public String testException(@RequestParam("key") String key, @RequestParam(name = "count", defaultValue = "3") int count) {
        AtomicInteger num = map.computeIfAbsent(key, s -> new AtomicInteger());
        int i = num.incrementAndGet();
        log.warn("Retry count: "+i);
        if (i < count) {
            throw new RuntimeException("temporarily broken");
        }
        return String.valueOf(i);
    }
```
这里主要是为了能配置网关请求次数的演示，count是指定的重试次数，默认为3，第一次和第二次都会抛出运行时异常（状态码为500），变量 i 是key对应的值，初始为0，每重试一次，i 会递增，直到 i 大于等于count的值。

#### 测试结果
根据上面的实现，我们访问的地址为`http://localhost:9090/test/exception?key=abc&count=2`。
按照用户服务实现的逻辑，用户服务将会重试一次即可成功。用户服务的控制台日志信息如下：

```
Retry count: 1

java.lang.IllegalArgumentException: temporarily broken] with root cause
...
Retry count: 2
```

![请求的响应结果](http://ovcjgn2x0.bkt.clouddn.com/retry-result.jpg "请求的响应结果")

控制台的信息和最后的响应结果可以看出，请求的重试执行成功。

## 小结
本文在[Spring Cloud Gateway入门](http://blueskykong.com/2018/03/10/cloud-Gateway-1/)的基础上，介绍了Spring Cloud Gateway的过滤器相关概念，并具体介绍了其中的一个过滤器工厂：`RetryGatewayFilterFactory`。当转发到代理服务时，遇到指定的服务端Error，如httpStatus为500时，我们可以设定重试几次，应用重试过滤器。Spring Cloud Gateway提供了很多过滤器工厂的实现，后面文章将会介绍其中比较重要的过滤器，敬请关注。

### 源码地址
https://github.com/keets2012/Spring-Cloud_Samples

### 参考
Spring Cloud Gateway官方文档

