---
title: 微服务网关Zuul迁移到Spring Cloud Gateway
date: 2018-9-20
categories: Security
tags:
  - zuul
  - OAuth2
  - Spring Security
  - gateway
abbrlink: 40293
---
## 背景
在之前的文章中，我们介绍过微服务网关[Spring Cloud Netflix Zuul](http://blueskykong.com/2017/11/13/gateway/)，前段时间有两篇文章专门介绍了Spring Cloud的全新项目[Spring Cloud Gateway](http://blueskykong.com/2018/03/10/cloud-Gateway-1/)，以及其中的[过滤器工厂](http://blueskykong.com/2018/04/25/cloud-Gateway-filter-1/)。本文将会介绍将微服务网关由Zuul迁移到Spring Cloud Gateway。

Spring Cloud Netflix Zuul是由Netflix开源的API网关，在微服务架构下，网关作为对外的门户，实现动态路由、监控、授权、安全、调度等功能。

Zuul基于servlet 2.5（使用3.x），使用阻塞API。 它不支持任何长连接，如websockets。而Gateway建立在Spring Framework 5，Project Reactor和Spring Boot 2之上，使用非阻塞API。 比较完美地支持异步非阻塞编程，先前的Spring系大多是同步阻塞的编程模式，使用thread-per-request处理模型。即使在Spring MVC Controller方法上加@Async注解或返回DeferredResult、Callable类型的结果，其实仍只是把方法的同步调用封装成执行任务放到线程池的任务队列中，还是thread-per-request模型。Gateway 中Websockets得到支持，并且由于它与Spring紧密集成，所以将会是一个更好的开发体验。

在一个微服务集成的项目中[microservice-integration](https://github.com/keets2012/microservice-integration)，我们整合了包括网关、auth权限服务和backend服务。提供了一套微服务架构下，网关服务路由、鉴权和授权认证的项目案例。整个项目的架构图如下：

![](http://image.blueskykong.com/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E6%9D%83%E9%99%90-aoho%20.png)

具体参见：[微服务架构中整合网关、权限服务](http://blueskykong.com/2017/12/10/integration/)。本文将以该项目中的Zuul网关升级作为示例。

## Zuul网关
在该项目中，Zuul网关的主要功能为路由转发、鉴权授权和安全访问等功能。

Zuul中，很容易配置动态路由转发，如：

```yaml
zuul:
  ribbon:
    eager-load:
      enabled: true     #zuul饥饿加载
  host:
    maxTotalConnections: 200
    maxPerRouteConnections: 20
  routes:
    user:
      path: /user/**
      ignoredPatterns: /consul
      serviceId: user
      sensitiveHeaders: Cookie,Set-Cookie
```

默认情况下，Zuul在请求路由时，会过滤HTTP请求头信息中的一些敏感信息，这里我们不过多介绍。

网关中还配置了请求的鉴权，结合Auth服务，通过Zuul自带的Pre过滤器可以实现该功能。当然还可以利用Post过滤器对请求结果进行适配和修改等操作。

除此之外，还可以配置限流过滤器和断路器，下文中将会增加实现这部分功能。

## 迁移到Spring Cloud Gateway
笔者新建了一个`gateway-enhanced`的项目，因为变化很大，不适合在之前的`gateway`项目基础上修改。实现的主要功能如下：路由转发、权重路由、断路器、限流、鉴权和黑白名单等。本文基于主要实现如下的三方面功能：

- 路由断言
- 过滤器（包括全局过滤器，如断路器、限流等）
- 全局鉴权
- 路由配置
- CORS

### 依赖
本文采用的Spring Cloud Gateway版本为`2.0.0.RELEASE`。增加的主要依赖如下，具体的细节可以参见Github上的项目。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
            <!--<version>2.0.1.RELEASE</version>-->
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-gateway-webflux</artifactId>
        </dependency>
    </dependencies>
        
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>        

```

### 路由断言
Spring Cloud Gateway对于路由断言、过滤器和路由的定义，同时支持配置文件的shortcut和Fluent API。我们将以在本项目中实际使用的功能进行讲解。

路由断言在网关进行转发请求之前进行判断路由的具体服务，通常可以根据请求的路径、请求体、请求方式（GET/POST）、请求地址、请求时间、请求的HOST等信息。我们主要用到的是基于请求路径的方式，如下：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: service_to_web
        uri: lb://authdemo
        predicates:
        - Path=/demo/**
```
我们定义了一个名为`service_to_web`的路由，将请求路径以`/demo/**`的请求都转发到authdemo服务实例。

我们在本项目中路由断言的需求并不复杂，下面介绍通过Fluent API配置的其他路由断言：

```java
    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(r -> r.host("**.changeuri.org").and().header("X-Next-Url")
                        .uri("http://blueskykong.com"))
                .route(r -> r.host("**.changeuri.org").and().query("url")
                        .uri("http://blueskykong.com"))
                .build();
    }
```

在如上的路由定义中，我们配置了以及请求HOST、请求头部和请求的参数。在一个路由定义中，可以配置多个断言，采取与或非的关系判断。

以上增加的配置仅作为扩展，读者可以根据自己的需要进行配置相应的断言。
### 过滤器
过滤器分为全局过滤器和局部过滤器。我们通过实现`GlobalFilter`、`GatewayFilter`接口，自定义过滤器。

#### 全局过滤器
本项目中，我们配置了如下的全局过滤器：

- 基于令牌桶的限流过滤器
- 基于漏桶算法的限流过滤器
- 全局断路器
- 全局鉴权过滤器

定义全局过滤器，可以通过在配置文件中，增加`spring.cloud.gateway.default-filters`，或者实现`GlobalFilter`接口。

1.基于令牌桶的限流过滤器

随着时间流逝，系统会按恒定 1/QPS 时间间隔（如果 QPS=100，则间隔是 10ms）往桶里加入 Token，如果桶已经满了就不再加了。每个请求来临时，会拿走一个 Token，如果没有 Token 可拿了，就阻塞或者拒绝服务。

令牌桶的另外一个好处是可以方便的改变速度。一旦需要提高速率，则按需提高放入桶中的令牌的速率。一般会定时（比如 100 毫秒）往桶中增加一定数量的令牌，有些变种算法则实时的计算应该增加的令牌的数量。

在Spring Cloud Gateway中提供了默认的实现，我们需要引入redis的依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
        </dependency>
```
并进行如下的配置：

```yml
spring:
  redis:
    host: localhost
    password: pwd
    port: 6378
  cloud:
    default-filters:
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@remoteAddrKeyResolver}"
          rate-limiter: "#{@customRateLimiter}"   # token
```

注意到，在配置中使用了两个SpEL表达式，分别定义限流键和限流的配置。因此，我们需要在实现中增加如下的配置：

```java
    @Bean(name = "customRateLimiter")
    public RedisRateLimiter myRateLimiter(GatewayLimitProperties gatewayLimitProperties) {
        GatewayLimitProperties.RedisRate redisRate = gatewayLimitProperties.getRedisRate();
        if (Objects.isNull(redisRate)) {
            throw new ServerException(ErrorCodes.PROPERTY_NOT_INITIAL);
        }
        return new RedisRateLimiter(redisRate.getReplenishRate(), redisRate.getBurstCapacity());
    }
    
        @Bean(name = RemoteAddrKeyResolver.BEAN_NAME)
    public RemoteAddrKeyResolver remoteAddrKeyResolver() {
        return new RemoteAddrKeyResolver();
    }

```
在如上的实现中，初始化好`RedisRateLimiter`和`RemoteAddrKeyResolver`两个Bean实例，`RedisRateLimiter`是定义在Gateway中的redis限流属性；而`RemoteAddrKeyResolver`使我们自定义的，基于请求的地址作为限流键。如下为该限流键的定义：

```java
public class RemoteAddrKeyResolver implements KeyResolver {
    private static final Logger LOGGER = LoggerFactory.getLogger(RemoteAddrKeyResolver.class);

    public static final String BEAN_NAME = "remoteAddrKeyResolver";

    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        LOGGER.debug("token limit for ip: {} ", exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
        return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    }

}
```
`RemoteAddrKeyResolver`实现了`KeyResolver`接口，覆写其中定义的接口，返回值为请求中的地址。

如上，即实现了基于令牌桶算法的链路过滤器，具体细节不再展开。

2.基于漏桶算法的限流过滤器

漏桶（Leaky Bucket）算法思路很简单，水（请求）先进入到漏桶里，漏桶以一定的速度出水（接口有响应速率），当水流入速度过大会直接溢出（访问频率超过接口响应速率），然后就拒绝请求，可以看出漏桶算法能强行限制数据的传输速率。

这部分实现读者参见GitHub项目以及文末配套的书，此处略过。

3.全局断路器

关于Hystrix断路器，是一种服务容错的保护措施。`断路器`本身是一种开关装置，用于在电路上保护线路过载，当线路中有发生短路状况时，`断路器`能够及时的切断故障电路，防止发生过载、起火等情况。

微服务架构中，断路器模式的作用也是类似的，当某个服务单元发生故障之后，通过断路器的故障监控，直接切断原来的主逻辑调用。关于断路器的更多资料和Hystrix实现原理，读者可以参考文末配套的书。

这里需要引入`spring-cloud-starter-netflix-hystrix`依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
            <optional>true</optional>
        </dependency>
```

并增加如下的配置：

```yml
      default-filters:
      - name: Hystrix
        args:
          name: fallbackcmd
          fallbackUri: forward:/fallbackcontroller
```
如上的配置，将会使用`HystrixCommand`打包剩余的过滤器，并命名为`fallbackcmd`，我们还配置了可选的参数`fallbackUri`，降级逻辑被调用，请求将会被转发到URI为`/fallbackcontroller`的控制器处理。定义降级处理如下：

```java
    @RequestMapping(value = "/fallbackcontroller")
    public Map<String, String> fallBackController() {
        Map<String, String> res = new HashMap();
        res.put("code", "-100");
        res.put("data", "service not available");
        return res;
    }
```

4.全局鉴权过滤器

我们通过自定义一个全局过滤器实现，对请求合法性的鉴权。具体功能不再赘述了，通过实现`GlobalFilter`接口，区别的是Webflux传入的是`ServerWebExchange`，通过判断是不是外部接口（外部接口不需要登录鉴权），执行之前实现的处理逻辑。


```java
public class AuthorizationFilter implements GlobalFilter, Ordered {

	//....

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        if (predicate(exchange)) {
            request = headerEnhanceFilter.doFilter(request);
            String accessToken = extractHeaderToken(request);

            customRemoteTokenServices.loadAuthentication(accessToken);
            LOGGER.info("success auth token and permission!");
        }

        return chain.filter(exchange);
    }
	//提出头部的token
    protected String extractHeaderToken(ServerHttpRequest request) {
        List<String> headers = request.getHeaders().get("Authorization");
        if (Objects.nonNull(headers) && headers.size() > 0) { // typically there is only one (most servers enforce that)
            String value = headers.get(0);
            if ((value.toLowerCase().startsWith(OAuth2AccessToken.BEARER_TYPE.toLowerCase()))) {
                String authHeaderValue = value.substring(OAuth2AccessToken.BEARER_TYPE.length()).trim();
                // Add this here for the auth details later. Would be better to change the signature of this method.
                int commaIndex = authHeaderValue.indexOf(',');
                if (commaIndex > 0) {
                    authHeaderValue = authHeaderValue.substring(0, commaIndex);
                }
                return authHeaderValue;
            }
        }

        return null;
    }

}
```
定义好全局过滤器之后，只需要配置一下即可：

```java
    @Bean
    public AuthorizationFilter authorizationFilter(CustomRemoteTokenServices customRemoteTokenServices,
                                                   HeaderEnhanceFilter headerEnhanceFilter,
                                                   PermitAllUrlProperties permitAllUrlProperties) {
        return new AuthorizationFilter(customRemoteTokenServices, headerEnhanceFilter, permitAllUrlProperties);
    }
```

#### 局部过滤器
我们常用的局部过滤器有增减请求和相应头部、增减请求的路径等多种过滤器。我们这里用到的是去除请求的指定前缀，这部分前缀只是用户网关进行路由判断，在转发到具体服务时，需要去除前缀：

```yaml
      - id: service_to_user
        uri: lb://user
        order: 8000
        predicates:
        - Path=/user/**
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
        - StripPrefix=1
```
还可以通过Fluent API，如下：

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
除了设置前缀过滤器外，我们还设置了重试过滤器，可以参见：[Spring Cloud Gateway中的过滤器工厂：重试过滤器](http://blueskykong.com/2018/04/25/cloud-Gateway-filter-1/)

### 路由配置
路由定义在上面的示例中已经有列出，可以通过配置文件和定义`RouteLocator`的对象。这里需要注意的是，配置中的`uri`属性，可以是具体的服务地址（IP+端口号），也可以是通过服务发现加上负载均衡定义的：`lb://user`，表示转发到user的服务实例。当然这需要我们进行一些配置。

引入服务发现的依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```
网关中开启`spring.cloud.gateway.discovery.locator.enabled=true`即可。

### CORS配置
在Spring 5 Webflux中，配置CORS，可以通过自定义`WebFilter`实现：

```java
    private static final String ALLOWED_HEADERS = "x-requested-with, authorization, Content-Type, Authorization, credential, X-XSRF-TOKEN";
    private static final String ALLOWED_METHODS = "GET, PUT, POST, DELETE, OPTIONS";
    private static final String ALLOWED_ORIGIN = "*";
    private static final String MAX_AGE = "3600";

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
上述代码实现比较简单，读者根据实际的需要配置`ALLOWED_ORIGIN`等参数。

## 总结

在高并发和潜在的高延迟场景下，网关要实现高性能高吞吐量的一个基本要求是全链路异步，不要阻塞线程。Zuul网关采用同步阻塞模式不符合要求。

Spring Cloud Gateway基于Webflux，比较完美地支持异步非阻塞编程，很多功能实现起来比较方便。Spring5必须使用java 8，函数式编程就是java8重要的特点之一，而WebFlux支持函数式编程来定义路由端点处理请求。

通过如上的实现，我们将网关从Zuul迁移到了Spring Cloud Gateway。在Gateway中定义了丰富的路由断言和过滤器，通过配置文件或者Fluent API可以直接调用和使用，非常方便。在性能上，也是胜于之前的Zuul网关。

欲了解更详细的实现原理和细节，大家可以关注笔者本月底即将出版的《Spring Cloud 微服务架构进阶》，本书中对Spring Cloud `Finchley.RELEASE`版本的各个主要组件进行原理讲解和实战应用，网关则是基于最新的Spring Cloud Gateway。

![Spring Cloud 微服务架构进阶](http://image.blueskykong.com/sc-depth.jpg)

**本文的源码地址：   
GitHub：https://github.com/keets2012/microservice-integration
或者 码云：https://gitee.com/keets/microservice-integration**

### 参考

[Spring Cloud Gateway（限流）](https://windmt.com/2018/05/09/spring-cloud-15-spring-cloud-gateway-ratelimiter/)
