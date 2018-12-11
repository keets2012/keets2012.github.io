---
title: Spring Boot Actuator详解与深入应用（三）：Prometheus+Grafana应用监控
categories: 微服务
tags:
  - Spring Boot
  - micro-service
abbrlink: 62823
img: http://image.blueskykong.com/grafana-5.jpg
date: 2018-11-25 00:00:00
---
> 《Spring Boot Actuator详解与深入应用》预计包括三篇，第一篇重点讲Spring Boot Actuator 1.x的应用与定制端点；第二篇将会对比Spring Boot Actuator 2.x 与1.x的区别，以及应用和定制2.x的端点；第三篇将会介绍Actuator metric指标与Prometheus和Grafana的使用结合。这部分内容很常用，且较为入门，欢迎大家的关注。

## 前文回顾
本文系《Spring Boot Actuator详解与深入应用》中的第三篇。在前两篇文章，我们主要讲了Spring Boot Actuator 1.x与 2.x 的应用与定制端点。相比于Actuator 1.x，基于Spring Boot 2.0的Actuator 2.x 在使用和定制方面有很大变化，对于Actuator的扩展也更加灵活。建议读者重点关注一下Actuator 2.x，关于Spring Boot 2.x流行的趋势是显而易见的。

Actuator提供端点将数据暴露出来，我们获取这些数据进行分析，但是仅仅这样，对于我们的分析并不能显得直观和方便。微服务架构中，拥有的微服务实例数量往往很庞大。数据可视化是我们一贯所期望的，开源项目：Spring Boot Admin提供了对Spring boot的Actuator Endpoint UI界面支持，同时也提供了对 Spring cloud的一些支持。本文将会对比首先介绍Spring Boot Admin的使用，然后重点介绍Spring Boot 2.x 中的应用监控：Actuator + Prometheus + Grafana。

## 应用Spring Boot Admin
Spring Boot Admin是一个Web应用程序，用于管理和监视Spring Boot应用程序。每个应用程序都被视为客户端并注册到管理服务器。实现的原理则是基于Spring Boot Actuator提供的端点。

这一部分，我们将描述配置Spring Boot Admin服务器以及应用程序如何成为客户端的步骤。

### Admin Server

#### 引入依赖
首先，我们需要引入相关的依赖：

```xml
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-server</artifactId>
            <version>2.0.1</version>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-server-ui</artifactId>
            <version>2.0.1</version>
        </dependency>
```
分别引入了admin-server和admin-server-ui。

#### 配置Admin Server
应用程序的入口增加注解`@EnableAdminServer`将会开启服务。

```yaml
server:
  port: 8088

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
```

#### Admin UI
打开http://localhost:8088/，我们可以看到如下的界面：

![](http://image.blueskykong.com/admin-server.jpg)

下面我们注册客户端服务到Admin Server上。
### Admin Client

#### 引入依赖

```xml
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-client</artifactId>
            <version>2.0.1</version>
        </dependency>
```
在各个服务中，增加admin client的依赖。
#### 配置Admin Client

```yaml
server:
  port: 8081

spring:
  application:
    name: admin-client

---
spring.boot.admin.client.url: "http://localhost:8088"
management.endpoints.web.exposure.include: "*"
---
management:
  endpoint:
    health:
      show-details: always
      defaults:
        enabled: true

eureka.client.serviceUrl.defaultZone: http://localhost:8761/eureka/
```
我们指定了Admin Server的地址`http://localhost:8088`。暴露出所有的Actuator的端点，并且显示健康检查的详细信息。

通过如上的配置，即完成了一个简单的Admin Server和Client的搭建。
### UI界面
当我们的客户端注册到Admin Server之后，打开Admin的界面，可以看到如下的信息截图：

![](http://image.blueskykong.com/admin-client.jpg)

![](http://image.blueskykong.com/admin-client-preface.jpg)

![](http://image.blueskykong.com/admin-client-preface-2.jpg)

![](http://image.blueskykong.com/admin-loggers.jpg)
上图为loggers界面的截图，我们在上一篇介绍过，Actuator 2.x是CRUD模型，所以我们这里也可以更改日志的等级。
### 使用服务发现
如上的示例中，我们配置Admin Server是通过指定URL。在微服务集群中，服务一般都会注册到服务发现与注册组件中，我们可以引入现有的组件，如Eureka、Consul等，轻松实现客户端注册。

实现此处略过，感兴趣的读者可以参见GitHub源码：https://github.com/keets2012/Spring-Boot-Samples/tree/master/spring-boot-admin-2

## 使用Prometheus与Grafana监控
通过Actuator收集的各种指标信息，存储到Prometheus统计，Grafana则提供了一个友好的界面展示。下面我们将介绍如何整合这三个组件，进行应用服务与系统的监控。

### 暴露Prometheus端点

我们在之前的基础上，增加micrometer的依赖：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

并将端点暴露，增加如下配置：

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m
        descriptions: true
  web:
    server:
      auto-time-requests: true
  endpoints:
    prometheus:
      id: springmetrics
    web:
      exposure:
        include: health,info,env,prometheus,metrics,httptrace,threaddump,heapdump,springmetrics

server:
  port: 8082
```
### Prometheus介绍
Prometheus 是 Cloud Native Computing Foundation 项目之一，是一个系统和服务监控系统。它的工作方式是被监控的服务需要公开一个Prometheus端点，这端点是一个HTTP接口，该接口公开了度量的列表和当前的值，然后Prometheus应用从此接口定时拉取数据，一般可以存放在时序数据库中，然后通过可视化的Dashboard(如Grafana)进行数据展示。

Prometheus特性：

- 多维度数据模型（由度量名称和键/值维度集定义的时间序列）
- 灵活的查询语言 来利用这种维度
- 不依赖分布式存储；单个服务器节点是自治的
- 时间序列采集通过HTTP上的 pull model 发生
- 推送时间序列 通过中间网关得到支持
- 通过服务发现或静态配置来发现目标
- 多种模式的图形和仪表盘支持
- 支持分级和水平federation

支持的prometheus metrics，如Counter，Gauge，Histogram和Summary等。需要注意的是counter只能增不能减，适用于服务请求量，用户访问数等统计，但是如果需要统计有增有减的指标需要用Gauge。支持的exporter很多，可以方便的监控很多应用，同时也可以自定义开发非官方提供的exporter。

#### 安装Prometheus
可以通过压缩直接安装，笔者偷懒直接采用docker镜像。

```
docker run -it  -p 9090:9090 -v /etc/spring-boot-samples/spring-boot-actuator-prometheus/src/main/resources/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

配置Prometheus文件如下：

```yaml
# Prometheus全局配置项
global:
  scrape_interval:     15s # 设定抓取数据的周期，默认为1min
  evaluation_interval: 15s # 设定更新rules文件的周期，默认为1min
  scrape_timeout: 15s # 设定抓取数据的超时时间，默认为10s
  external_labels: # 额外的属性，会添加到拉取得数据并存到数据库中
   monitor: 'codelab_monitor'


# Alertmanager配置
alerting:
 alertmanagers:
 - static_configs:
   - targets: ["localhost:9093"] # 设定alertmanager和prometheus交互的接口，即alertmanager监听的ip地址和端口

# rule配置，首次读取默认加载，之后根据evaluation_interval设定的周期加载
rule_files:
 - "alertmanager_rules.yml"
 - "prometheus_rules.yml"

# scape配置
scrape_configs:
- job_name: 'prometheus' # job_name默认写入timeseries的labels中，可以用于查询使用
  scrape_interval: 15s # 抓取周期，默认采用global配置
  static_configs: # 静态配置
  - targets: ['localdns:9090'] # prometheus所要抓取数据的地址，即instance实例项

- job_name: 'example-random'
  static_configs:
  - targets: ['localhost:8082']
```
如上为prometheus的配置文件，8082端口为上一小节启动的应用的端口。prometheus的端口为9090。此外还配置了Alertmanager，用于email等类型的告警。

![](http://image.blueskykong.com/prometheus-query-2.jpg)

![](http://image.blueskykong.com/prometheus.jpg)
### Grafana介绍

Grafana是一个开源的Dashboard展示工具，可以支持很多主流数据源，包括时序性的和非时序性的。其提供的展示配置以及可扩展性能满足绝大部分时间序列数据展示需求，是一个比较优秀的工具。支持的数据源包括：prometheus、zabbix、elasticsearch、mysql和openTSDB等。

#### 安装
Grafana同样是使用docker镜像安装，执行如下的启动命令：

```
docker run -d -p 3000:3000 grafana/grafana
```
并增加prometheus的数据源，如下：

**配置数据源：**
![](http://image.blueskykong.com/grafana-config.jpg)

**配置Dashboard：**

![](http://image.blueskykong.com/grafana-config-dashboard.jpg)

#### 统计界面
经过如上的配置，我们可以看到如下图所示的统计图：

![](http://image.blueskykong.com/grafana-5.jpg)

当然我们还可以安装插件，如下所示:

![](http://image.blueskykong.com/grafana-2.jpg)

![](http://image.blueskykong.com/grafana-zabbix.jpg)

![](http://image.blueskykong.com/grafana-loggers.jpg)

上图展示了笔者配置的一些插件，如zabbix、k8s等，配置较为简单，不在此处一一列出。

此部分设计的代码和监本，参见：https://github.com/keets2012/Spring-Boot-Samples/tree/master/spring-boot-actuator-prometheus
## 总结
本文较为简单，主要讲了结合Actuator的监控应用，首先讲了Spring Boot Admin的应用并给出示例程序；然后讲了使用prometheus+grafana进行集成监控。总的来说，后一种方式功能更为强大，更为专业，支持更丰富的数据源，读者可以自行尝试。

#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](http://image.blueskykong.com/wechat-public-code.jpg)

#### 参考
1. [grafana docs](http://docs.grafana.org/installation/docker/)
2. [prometheus+grafana+springboot2监控集成配置](https://www.callicoder.com/spring-boot-actuator-metrics-monitoring-dashboard-prometheus-grafana/)
