---
title: Spring Boot Actuator详解与深入应用（一）：Actuator 1.x
categories: 微服务
tags:
  - Spring Boot
  - micro-service
abbrlink: 24246
date: 2018-11-16 16:06:59
---

> 《Spring Boot Actuator详解与深入应用》预计包括三篇，第一篇重点讲Spring Boot Actuator 1.x的应用与定制端点；第二篇将会对比Spring Boot Actuator 2.x 与1.x的区别，以及应用和定制2.x的端点；第三篇将会介绍Actuator metric指标与Prometheus和Grafana的使用结合。这部分内容很常用，且较为入门，欢迎大家的关注。

## Actuator是什么

Spring Boot Actuator提供了生产上经常用到的功能（如健康检查，审计，指标收集，HTTP跟踪等），帮助我们监控和管理Spring Boot应用程序。这些功能都可以通过JMX或HTTP端点访问。

通过引入相关的依赖，即可监控我们的应用程序，收集指标、了解流量或数据库的状态变得很简单。该库的主要好处是我们可以获得生产级工具，而无需自己实际实现这些功能。与大多数Spring模块一样，我们可以通过多种方式轻松配置或扩展它。

Actuator还可以与外部应用监控系统集成，如Prometheus，Graphite，DataDog，Influx，Wavefront，New Relic等等。 这些系统为您提供出色的仪表板，图形，分析和警报，以帮助我们在一个统一界面监控和管理应用服务。

本文将会介绍Spring Boot Actuator 1.x 包括其中的端点（HTTP端点）、配置管理以及扩展和自定义端点。

## 快速开始
引入如下的依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

## Spring Boot Actuator 1.x 
在1.x中，Actuator遵循读写模型，这意味着我们可以从中读取信息或写入信息。我们可以检索指标或我们的应用程序的健康状况，当然我们也可以优雅地终止我们的应用程序或更改我们的日志配置。Actuator通过Spring MVC暴露其HTTP端点。

### 端点
当引入的Actuator的版本为1.x时，启动应用服务，可以控制台输出如下的端点信息：

![](http://image.blueskykong.com/actuator-1.x.jpg)

我们介绍一下常用的endpoints：

- /health：显示应用程序运行状况信息（通过未经身份验证的连接访问时的简单“状态”或经过身份验证时的完整消息详细信息），它默认不敏感
- /info：显示应用程序信息，默认情况下不敏感
- /metrics：显示当前应用程序的“指标”信息，它默认也很敏感
- /trace：显示跟踪信息（默认情况下是最后几个HTTP请求）

有些端点默认并不会被开启，如/shutdown。
### 配置端点
我们可以自定义每个端点的属性，按照如下的格式：

```
endpoints.[endpoint name].[property to customize]
```
可以自定义的属性有如下三个：

- id，暴露的http端点地址
- enabled，是否开启
- sensitive，当为true时，需要认证之后才会通过http获取到敏感信息

我们在配置文件中增加如下的配置，将会定制/beans端点。

```
endpoints.beans.id=springbeans
endpoints.beans.sensitive=false
endpoints.beans.enabled=true
```
### /health端点
/health端点用于监控运行的服务实例状态，当服务实例下线或者因为其他的原因变得异常（如DB连接不上，磁盘缺少空间）时，将会及时通知运维人员。

在默认未授权的情况下，通过HTTP的方式仅会返回如下的简单信息：

```
{
	"status": "UP"
}
```
#### 获取详细的health信息
我们进行如下的配置：

```
endpoints:
  health:
    id: chealth
    sensitive: false

management.security.enabled: false
```
如上一小节所述，我们更改了/health端点的访问路径为/chealth，并将安全授权关闭。访问`http://localhost:8005/chealth`将会得到如下的结果。

```
{
	"status": "UP",
	"healthCheck": {
		"status": "UP"
	},
	"diskSpace": {
		"status": "UP",
		"total": 999995129856,
		"free": 762513104896,
		"threshold": 10485760
	}
}
```

#### 自定义health的信息
我们还可以定制实现health指示器。它可以收集特定于应用程序的任何类型的自定义运行状况数据，并通过/health端点访问到定义的信息。

```java
@Component
public class HealthCheck implements HealthIndicator {

    @Override
    public Health health() {
        int errorCode = check(); // perform some specific health check
        if (errorCode != 0) {
            return Health.down()
                    .withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }

    public int check() {
        // Our logic to check health
        return 0;
    }
}
```
实现`HealthIndicator`接口，并覆写其中的`health()`方法即可自定义我们的/health端点。
### /info端点
通过/info端点，我们可以为应用服务定义一些基本信息：

```
info.app.name=Spring Sample Application
info.app.description=This is my first spring boot application
info.app.version=1.0.0
```
我们在如上的配置中定义了服务名、描述和服务的版本号。
### /metrics端点
/metrics端点展示了OS、JVM和应用级别的指标信息。当开启之后，我们可以获取内存、堆、线程、线程池、类加载和HTTP等信息。

```
{
	"mem": 417304,
	"mem.free": 231678,
	"processors": 4,
	"instance.uptime": 248325,
	"uptime": 250921,
	"systemload.average": 1.9541015625,
	"heap.committed": 375296,
	"heap.init": 393216,
	"heap.used": 143617,
	"heap": 5592576,
	"nonheap.committed": 43104,
	"nonheap.init": 2496,
	"nonheap.used": 42010,
	"nonheap": 0,
	"threads.peak": 30,
	"threads.daemon": 18,
	"threads.totalStarted": 46,
	"threads": 20,
	"classes": 6020,
	"classes.loaded": 6020,
	"classes.unloaded": 0,
	"gc.ps_scavenge.count": 3,
	"gc.ps_scavenge.time": 35,
	"gc.ps_marksweep.count": 1,
	"gc.ps_marksweep.time": 29,
	"httpsessions.max": -1,
	"httpsessions.active": 0,
	"gauge.response.info": 38.0,
	"counter.status.200.info": 1
}
```
#### 定制metrics端点
为了收集自定义的metrics，Actuator支持单数值记录的功能，简单的增加/减少计数功能。如下的实现，我们将登录成功和失败的次数作为自定义指标记录下来。

```java
@Service
public class LoginServiceImpl implements LoginService {
    private final CounterService counterService;

    @Autowired
    public LoginServiceImpl(CounterService counterService) {
        this.counterService = counterService;
    }

    @Override
    public Boolean login(String userName, char[] password) {
        boolean success;
        if (userName.equals("admin") && "secret".toCharArray().equals(password)) {
            counterService.increment("counter.login.success");
            success = true;
        } else {
            counterService.increment("counter.login.failure");
            success = false;
        }
        return success;
    }
}
```
再次访问/metrics，发现多了如下的指标信息。登录尝试和其他安全相关事件在Actuator中可用作审计事件。

```
{
	"gauge.response.metrics": 2.0,
	"gauge.response.test": 3.0,
	"gauge.response.star-star.favicon.ico": 1.0,
	"counter.status.200.star-star.favicon.ico": 10,
	"counter.status.200.test": 6,
	"counter.login.failure": 6,
	"counter.status.200.metrics": 4
}
```


### 自定义端点
除了使用Spring Boot Actuator提供的端点，我们也可以定义一个全新的端点。

首先，我们需要实现`Endpoint`接口：

```java
@Component
public class CustomEndpoint implements Endpoint<List<String>> {
    @Override
    public String getId() {
        return "custom";
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    @Override
    public boolean isSensitive() {
        return false;
    }

    @Override
    public List<String> invoke() {
        // Custom logic to build the output
        List<String> messages = new ArrayList<String>();
        messages.add("This is message 1");
        messages.add("This is message 2");
        return messages;
    }
}
```
`getId()`方法用于匹配访问这个端点，当我们访问/custom时，将会调用`invoke()`我们自定义的逻辑。
另外两个方法，用于设置是否开启和是否为敏感的端点。

```
[ "This is message 1", "This is message 2" ]
```

### 进一步定制
出于安全考虑，我们可能选择通过非标准端口暴露Actuator端点。通过management.port属性来配置它。

另外，正如我们已经提到的那样，在1.x. Actuator基于Spring Security配置自己的安全模型，但独立于应用程序的其余部分。

因此，我们可以更改management.address属性以限制可以通过网络访问端点的位置：

```yml
#port used to expose actuator
management.port=8081 
 
#CIDR allowed to hit actuator
management.address=127.0.0.1 
```
此外，除了/info端点，其他所有的端点默认都是敏感的，如果引入了Spring Security，我们通过在配置文件中定义这些安全的属性（username, password, role）来确保内置端点的安全。

## 总结
Spring Boot Actuator为我们的应用服务在生产环境提供了很多开箱即用的功能。本文主要讲解了Spring Boot Actuator 1.x的深入使用。我们既可以使用内置的端点（如/health，/info等），可以在这些端点的基础进行扩展和定制，还可以自定义全新的端点，在使用方式上显得非常灵活。

**本文源码：https://github.com/keets2012/Spring-Boot-Samples/tree/master/spring-boot-actuator**

### 参考
1. [Actuator docs](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)
2. [Spring Boot Actuator](https://www.baeldung.com/spring-boot-actuators#boot-1x-actuator)
