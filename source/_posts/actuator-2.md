---
title: Spring Boot Actuator详解与深入应用（二）：Actuator 2.x
categories: 微服务
tags:
  - Spring Boot
  - micro-service
abbrlink: 34554
img: http://image.blueskykong.com/actuator2-health.jpg
date: 2018-11-19 16:06:59
---
> 《Spring Boot Actuator详解与深入应用》预计包括三篇，第一篇重点讲Spring Boot Actuator 1.x的应用与定制端点；第二篇将会对比Spring Boot Actuator 2.x 与1.x的区别，以及应用和定制2.x的端点；第三篇将会介绍Actuator metric指标与Prometheus和Grafana的使用结合。这部分内容很常用，且较为入门，欢迎大家的关注。

## 前文回顾
本文系《Spring Boot Actuator详解与深入应用》中的第二篇。在上一篇文章：[Spring Boot Actuator详解与深入应用（一）：Actuator 1.x](http://blueskykong.com/2018/11/16/actuator-1/)主要讲了Spring Boot Actuator 1.x的应用与定制端点。Spring Boot2.0的正式版已经发布有一段时间了，目前已经到了`2.1.0.RELEASE`。关于Spring Boot2.x的特性，在此不详细叙述了，但是其流行的趋势是显而易见的。

本文将会对比Spring Boot Actuator 2.x 与1.x的区别，以及应用和定制2.x的端点。重点介绍最新的2.x版本的Actuator。

## Actuator 2.x 
Actuator 2.x继续保持其基本功能，但简化其模型，扩展其功能并包含合适的默认值。首先，这个版本变得与特定框架解耦；此外，它通过将其与应用程序合并来简化其安全模型；最后，在各种变化中，有些变化是巨大的，这包括HTTP请求/响应以及提供的Java API。此外，最新版本支持CRUD模型，而不是旧的RW（读/写）模型。

在Actuator 1.x中，它与Spring MVC绑定，因此与Servlet API相关联。而在2.x中，Actuator定义了它的模型可插拔且可扩展，而不依赖于MVC。因此，通过这个新模型，我们可以像MVC一样使用WebFlux作为底层Web技术。此外，以后的框架可以通过实现特定的适配器来增加到这个模型中。在没有任何额外的代码的情况下，JMX仍然支持暴露端点。

## 快速开始
引入如下的依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
和使用的Spring Boot Actuator 1.x并没有太大的区别。

## Actuator 2.x的变化
不同于之前的Actuator 1.x，Actuator 2.x 的大多数端点默认被禁掉。
Actuator 2.x 中的默认端点增加了/actuator前缀。

默认暴露的两个端点为/actuator/health 和 /actuator/info。我们可以通过设置如下的属性：

```
management.endpoints.web.exposure.include=*
```
可以使得所有的端点暴露出来。此外，我们也可以列出需要暴露的端点或者排除某些端点。如：

```
management.endpoints.web.exposure.exclude=env,beans
```

### 默认端点
下面我们看一下可用的端点，他们大部分在1.x中已经存在。尽管如此，有些端点新增，有些被删除，有些被重构。

- /auditevents：同Actuator 1.x，还可以通过关键字进行过滤
- /beans：同Actuator 1.x，不可以过滤
- /conditions：返回服务中的自动配置项
- /configprops：允许我们获取`@ConfigurationProperties`的bean对象
- /env：返回当前的环境变量，我们也可以检索某个值
- /flyway：提供Flyway数据库迁移的详细情况
- /health：同Actuator 1.x
- /heapdump：返回应用服务使用地jvm堆dump信息
- /info：同Actuator 1.x
- /liquibase：类似于 /flyway，但是组件工具为Liquibase
- /logfile：返回应用的普通日志文件
- /loggers：允许我们查询和修改应用的日志等级
- /metrics：同Actuator 1.x
- /prometheus：返回与/metrics类似，与Prometheus server一起使用
- /scheduledtasks：返回应用的周期性任务
- /sessions：同Actuator 1.x
- /shutdown：同Actuator 1.x
- /threaddump：dump所依赖的jvm线程信息

### Actuator的端点安全
Actuator端点是敏感的，必须防止未经授权的访问。 如果应用程序中存在Spring Security，则默认情况下使用基于表单的HTTP基本身份验证来保护端点。使用Spring Security保护Actuator的端点访问。

#### 引入依赖

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

#### 安全配置
为了应用Actuator的安全规则，我们增加如下的配置：

```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(
  ServerHttpSecurity http) {
    return http.authorizeExchange()
      .pathMatchers("/actuator/**").permitAll()
      .anyExchange().authenticated()
      .and().build();
}
```
如上的配置使得所有访问/actuator开头的URL都必须是登录的状态。我们还可以使用更加细化的配置：

```java
@Configuration
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .requestMatchers(EndpointRequest.to(ShutdownEndpoint.class))
                        .hasRole("ACTUATOR_ADMIN")
                    .requestMatchers(EndpointRequest.toAnyEndpoint())
                        .permitAll()
                    .requestMatchers(PathRequest.toStaticResources().atCommonLocations())
                        .permitAll()
                    .antMatchers("/")
                        .permitAll()
                    .antMatchers("/**")
                        .authenticated()
                .and()
                .httpBasic();
    }
}
```
如上的配置主要实现了：

- 限定访问Shutdown端点的角色只能是ACTUATOR_ADMIN
- 允许访问其他所有的端点
- 允许访问静态资源
- 允许访问根目录'/'
- 所有的请求都要经过认证
- 允许http静态认证（可以使用任何形式的认证）

#### 测试
为了能够使用HTTP基本身份验证测试上述配置，可以添加默认的spring安全性用户：

```
spring:
  security:
    user:
      name: actuator
      password: actuator
      roles: ACTUATOR_ADMIN
```

### /health端点
与以前的版本一样，我们可以轻松添加自定义指标。创建自定义健康端点的抽象保持不变。与Spring Boot 1.x不同，`endpoints.<id> .sensitive`属性已被删除。/health端点公开的运行状况信息取决于：

```
management.endpoint.health.show-details
```

该属性可以使用以下值之一进行配置：

- never：不展示详细信息，up或者down的状态，默认配置
- when-authorized：详细信息将会展示给通过认证的用户。授权的角色可以通过`management.endpoint.health.roles`配置。
- always：暴露详细信息

/health端点有很多自动配置的健康指示器：如redis、rabbitmq等组件。

![](http://image.blueskykong.com/actuator2-health.jpg)
当如上的组件有一个状态异常，应用服务的整体状态即为down。我们也可以通过配置禁用某个组件的健康监测

```
management.health.mongo.enabled: false
```

或者禁用所有自动配置的健康指示器：

```
management.health.defaults.enabled: false
```

除此之外，还添加了新的接口`ReactiveHealthIndicator`以实现响应式运行状况检查。
#### 引入依赖

```xml
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-core</artifactId>
        </dependency>
```
#### 自定义reactive健康检查

```java
@Component
public class DownstreamServiceHealthIndicator implements ReactiveHealthIndicator {

    @Override
    public Mono<Health> health() {
        return checkDownstreamServiceHealth().onErrorResume(
                ex -> Mono.just(new Health.Builder().down(ex).build())
        );
    }

    private Mono<Health> checkDownstreamServiceHealth() {
        // we could use WebClient to check health reactively
        return Mono.just(new Health.Builder().up().build());
    }
}
```
健康指标的一个便利功能是我们可以将它们聚合为层次结构的一部分。 因此，按照上面的示例，我们可以将所有下游服务分组到下游服务类别下。只要每个嵌套服务都可以访问，此次访问就是健康的。

### /metrics端点
在Spring Boot 2.0中，有一个bean类型为`MeterRegistry`将会被自动配置，并且`MeterRegistry`已经包含在Actuator的依赖中。如下为我们获得的/metrics端点信息。

```
{
  "names": [
    "jvm.gc.pause",
    "jvm.buffer.memory.used",
    "jvm.memory.used",
    "jvm.buffer.count",
    // ...
  ]
}
```
可以看到，不同于1.x，我们已经看不到具体的指标信息，只是展示了一个指标列表。为了获取到某个指标的详细信息，我们可以请求具体的指标信息，如`/actuator/metrics/jvm.gc.pause`

```json
{
	"name": "jvm.gc.pause",
	"description": "Time spent in GC pause",
	"baseUnit": "seconds",
	"measurements": [{
		"statistic": "COUNT",
		"value": 2.0
	}, {
		"statistic": "TOTAL_TIME",
		"value": 0.07300000000000001
	}, {
		"statistic": "MAX",
		"value": 0.0
	}],
	"availableTags": [{
		"tag": "cause",
		"values": ["Metadata GC Threshold"]
	}, {
		"tag": "action",
		"values": ["end of minor GC", "end of major GC"]
	}]
}
```
我们可以看到，现在的指标要详细得多。不仅包括不同的值，还包括一些相关的元数据。

### /info端点
/info端点没有什么变化，我们可以通过maven或者gradle引入依赖，增加git的详细信息。

```xml
<dependency>
    <groupId>pl.project13.maven</groupId>
    <artifactId>git-commit-id-plugin</artifactId>
</dependency>
```
同样的，我们可以使用maven和gradle的插件，获取到构建的name，group和version（需要类路径下存在META-INF/build-info.properties文件）。

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```


### 自定义端点
我们可以自定义端点，Spring Boot 2已经更新了自定义端点的方法，下面我们定义一个可以查询、开启或者关闭features标志位的端点。

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {

    private Map<String, Feature> features = new ConcurrentHashMap<>();

    @ReadOperation
    public Map<String, Feature> features() {
        return features;
    }

    @ReadOperation
    public Feature feature(@Selector String name) {
        return features.get(name);
    }

    @WriteOperation
    public void configureFeature(@Selector String name, Feature feature) {
        features.put(name, feature);
    }

    @DeleteOperation
    public void deleteFeature(@Selector String name) {
        features.remove(name);
    }

    public static class Feature {
        private Boolean enabled;

		//...
    }

}
```
定义的端点路径由`@Endpoint`中的id属性决定，在如上的例子中，请求的端点地址为`/actuator/features`。并用如下的方法注解来定义操作：

- @ReadOperation：HTTP GET
- @WriteOperation：HTTP POST
- @DeleteOperation：HTTP DELETE

启动应用，可以看到控制台多了如下的日志输出：

```
[...].WebFluxEndpointHandlerMapping: Mapped "{[/actuator/features/{name}],
  methods=[GET],
  produces=[application/vnd.spring-boot.actuator.v2+json || application/json]}"
[...].WebFluxEndpointHandlerMapping : Mapped "{[/actuator/features],
  methods=[GET],
  produces=[application/vnd.spring-boot.actuator.v2+json || application/json]}"
[...].WebFluxEndpointHandlerMapping : Mapped "{[/actuator/features/{name}],
  methods=[POST],
  consumes=[application/vnd.spring-boot.actuator.v2+json || application/json]}"
[...].WebFluxEndpointHandlerMapping : Mapped "{[/actuator/features/{name}],
  methods=[DELETE]}"[...]
```
如上的日志展示了Webflux如何暴露我们的端点，至于切换到Spring MVC，我们只需要引入依赖即可，并不需要更改任何代码。

之前方法上的元数据信息（sensitive, enabled）都不在使用了，开启或禁用端点，使用`@Endpoint(id = “features”, enableByDefault = false)`。相比于旧的读写模型，我们可以使用`@DeleteOperation`定义DELETE操作。

### 扩展端点
我们还可以通过注解`@EndpointExtension`扩展事先定义好的端点，更精确的注解为：`@EndpointWebExtension`，`@EndpointJmxExtension`。

```
@Component
@EndpointWebExtension(endpoint = InfoEndpoint.class)
public class InfoWebEndpointExtension {
 
    private InfoEndpoint delegate;
 
    @Autowired
    public InfoWebEndpointExtension(InfoEndpoint delegate) {
        this.delegate = delegate;
    }
 
    @ReadOperation
    public WebEndpointResponse<Map> info() {
        Map<String, Object> info = this.delegate.info();
        Integer status = getStatus(info);
        return new WebEndpointResponse<>(info, status);
    }
 
    private Integer getStatus(Map<String, Object> info) {
        // return 5xx if this is a snapshot
        return 200;
    }
}
```

## 总结
本文主要讲了Actuator 2.x相关特性和使用，对比了与Actuator 1.x 在使用上的区别。Actuator 2.x不依赖于某个框架组件（如Spring MVC），做到了易于插拔和扩展。当我们想要切换到Webflux时，通过Actuator 2.x中的适配器，不需要更改任何代码即可实现。

**本文源码：https://github.com/keets2012/Spring-Boot-Samples/tree/master/spring-boot-actuator-2**

#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](http://image.blueskykong.com/wechat-public-code.jpg)

### 参考
1. [Actuator docs](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)
2. [Spring Boot Actuator](https://www.baeldung.com/spring-boot-actuators#boot-1x-actuator)



