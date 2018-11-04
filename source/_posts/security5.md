---
title: 认证鉴权与API权限控制在微服务架构中的设计与实现：升级
date: 2018-9-5
categories: Security
tags:
  - Spring Security
  - OAuth2
abbrlink: 60325
---
## 概述

在之前的系列文章[认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/categories/Security/)中，我们有四篇文章讲解了微服务下的认证鉴权与API权限控制的实现。当时基于的Spring Cloud版本为`Dalston.SR4`，当前最新的Spring Cloud版本为`Finchley.SR1`，对应的Spring Boot也升级到了`2.0.x`。Spring Cloud版本为`Finchley`和Spring Boot2.0相对之前的版本有较大的变化，至于具体的changes，请参见官网。本次会将项目升级到最新版本，下面具体介绍其中的变化。与使用之前的版本，请切换到`1.0-RELEASE`。

## 升级依赖

将Spring Boot的依赖升级为`2.0.4.RELEASE`。

```yaml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
</parent>
```
升级dependencyManagement中的spring-cloud依赖为`Finchley.RELEASE`。

```yaml
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
删除`spring-cloud-starter-oauth2`依赖，只留下`spring-cloud-starter-security`依赖。

## 工具升级

### flyway
我们在项目中，引入了flyway的依赖，用以初始化数据库的增量脚本，具体可以参见[数据库版本管理工具Flyway应用](http://blueskykong.com/2018/08/15/flyway/)。

### docker容器
为了更加简便的体验本项目，笔者在项目中提供了docker compose脚本。在本地安装好docker compose的情况下，进入项目根目录执行`docker-compose up`命令。

![](http://image.blueskykong.com/docker-security.jpg)

即可启动我们所需要的mysql和redis。

### Mybatis和HikariCP
在Spring Boot 2.0.X版本中，选择了HikariCP作为默认数据库连接池。所以我们并不需要额外配置DataSource。

Mybatis的mapper和config-location配置也通过配置文件的形式，因此`DatasourceConfig`大大简化。
### application.yml

```yml
spring:
  flyway:
    baseline-on-migrate: true
    locations: classpath:db
  datasource:
    hikari:
      connection-test-query: SELECT 1
      minimum-idle: 1
      maximum-pool-size: 5
      pool-name: dbcp1
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/auth?autoReconnect=true&useSSL=false
    username: ${AUTH_DB_PWD:root}
    password: ${AUTH_DB_USER:_123456_}
#    schema[0]: classpath:/auth.sql
#    initialization-mode: ALWAYS
    type: com.zaxxer.hikari.HikariDataSource
  redis:
    database: 0
    host: localhost
    port: 6379

mybatis:
  mapper-locations: classpath:/mybatis/mapper/*Mapper.xml
  config-location: classpath:/mybatis/config.xml
```

## 配置类升级
### AuthenticationManagerConfig

弃用，由于循环依赖的问题，将`AuthenticationManager`的配置放置到`WebSecurityConfig`中。

### WebSecurityConfig

添加了来自`AuthenticationManagerConfig`的`AuthenticationManager`配置。

由于Spring Security5默认`PasswordEncoder`不是`NoOpPasswordEncoder`，需要手动指定。原来的auth项目中没有对密码进行加密，`NoOpPasswordEncoder`已经被废弃，只适合在测试环境中使用，本次我们使用`SCryptPasswordEncoder`密码加密器对密码进行加解密，更贴近产线的使用。其他的算法还有`Pbkdf2PasswordEncoder`和`BCryptPasswordEncoder`。

关于Scrpyt算法，可以肯定的是其很难被攻击。

> Scrpyt算法是由著名的FreeBSD黑客 Colin Percival为他的备份服务 Tarsnap开发的，当初的设计是为了降低CPU负荷，尽量少的依赖cpu计算，利用CPU闲置时间进行计算，因此scrypt不仅计算所需时间长，而且占用的内存也多，使得并行计算多个摘要异常困难，因此利用rainbow table进行暴力攻击更加困难。Scrpyt没有在生产环境中大规模应用，并且缺乏仔细的审察和广泛的函数库支持。所以Scrpyt一直没有推广开，但是由于其内存依赖的设计特别符合当时对抗专业矿机的设计，成为数字货币算法发展的一个主要应用方向。

而BCrypt相对出现的时间更久，也很安全。Spring Security中的`BCryptPasswordEncoder`方法采用SHA-256 + 随机盐 + 密钥对密码进行加密。SHA系列是Hash算法，不是加密算法，使用加密算法意味着可以解密（这个与编码/解码一样），但是采用Hash处理，其过程是不可逆的。

1. 加密(encode)：注册用户时，使用SHA-256+随机盐+密钥把用户输入的密码进行hash处理，得到密码的hash值，然后将其存入数据库中。
2. 密码匹配(matches)：用户登录时，密码匹配阶段并没有进行密码解密（因为密码经过Hash处理，是不可逆的），而是使用相同的算法把用户输入的密码进行hash处理，得到密码的hash值，然后将其与从数据库中查询到的密码hash值进行比较。如果两者相同，说明用户输入的密码正确。

关于怎么初始化密码呢，和注册用户的时候怎么给密码加密，我们可以在初始化密码时调用如下的方法：

```java
SCryptPasswordEncoder sCryptPasswordEncoder = new SCryptPasswordEncoder();
sCryptPasswordEncoder.encode("frontend");
```

此时需要对数据库中的`client_secret`进行修改，如把`frontend`修改为:

```
$e0801$65x9sjjnRPuKmqaFn3mICtPYnSWrjE7OB/pKzKTAI4ryhmVoa04cus+9sJcSAFKXZaJ8lcPO1I9H22TZk6EN4A==$o+ZWccaWXSA2t7TxE5VBRvz2W8psujU3RPPvejvNs4U=
```

并修改配置如下：

```java
    @Autowired
    CustomAuthenticationProvider customAuthenticationProvider;
    @Autowired
    CodeAuthenticationProvider codeAuthenticationProvider;

    @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(customAuthenticationProvider);
        auth.authenticationProvider(codeAuthenticationProvider);
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

   @Bean
    public PasswordEncoder passwordEncoder(){
        return new SCryptPasswordEncoder();
    }
```

### ResourceServerConfig

弃用，auth项目不启用资源服务器的功能。

### OAuth2Config

由于当前版本的`spring-boot-redis`中的`RedisConnection`缺少`#set`方法，直接使用`RedisTokenStore`会出现以下异常：

```java
java.lang.NoSuchMethodError: org.springframework.data.redis.connection.RedisConnection.set([B[B)V
```
因此自定义`CustomRedisTokenStore`类，与RedisTokenStore代码一致，只是将`RedisConnection#set`方法的调用替换为`RedisConnection#stringCommands#set`，如下所示：

```java
    conn.stringCommands().set(accessKey, serializedAccessToken);
    conn.stringCommands().set(authKey, serializedAuth);
    conn.stringCommands().set(authToAccessKey, serializedAccessToken);
```
完整代码见文末的GitHub地址。
## 结果验证
经过如上的升级改造，我们将验证如下的API端点：

- password模式获取token：/oauth/token?grant_type=password
- 刷新token：/oauth/token?grant_type=refresh_token&refresh_token=...
- 检验token：/oauth/check_token
- 登出：/logout
- 授权：/oauth/authorize
- 授权码模式获取token：/oauth/token?grant_type=authorization_code

结果就不展示了，都可以正常使用。

## 小结 
OAuth鉴权服务是微服务架构中的一个基础服务，项目公开之后得到了好多同学的关注，好多同学在加入QQ群之后也提出了自己关于这方面的疑惑或者建议，一起讨论和解决疑惑的地方。随着Spring Boot和Spring Cloud的版本升级，笔者也及时更新了本项目，希望能够帮到一些童鞋。笔者筹划的一本关于Spring Cloud应用的书籍，本月即将出版面世，其中关于Spring Cloud Security部分，有着详细的解析，各位同学可以支持一下正版。

**本文的源码地址：   
GitHub：https://github.com/keets2012/Auth-service   
码云： https://gitee.com/keets/Auth-Service**

### 相关阅读
[认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/categories/Security/)   

### 参考
[scrypt算法的前世今生（从零开始学区块链 192）](https://www.sohu.com/a/167016356_99901444)


