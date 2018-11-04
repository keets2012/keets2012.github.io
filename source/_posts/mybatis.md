---
title: Spring Boot集成MyBatis实现通用Mapper
date: 2018-8-22
categories: 微服务
tags:
  - Spring Boot
  - Mybatis
abbrlink: 57919
---
## MyBatis
关于MyBatis，大部分人都很熟悉。MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

不管是DDD（Domain Driven Design，领域驱动建模）还是分层架构的风格，都会涉及到对数据库持久层的操作，本文将会讲解Spring Boot集成MyBatis如何实现通用Mapper。

## Spring Boot集成MyBatis

### 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.1</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>
```

可以看到如上关于Mybatis引入了`mybatis-spring-boot-starter`，由Mybatis提供的starter。

### 数据库配置
在application.yml中增加如下配置：

```yml
spring:
  datasource:
    hikari:
      connection-test-query: SELECT 1
      minimum-idle: 1
      maximum-pool-size: 5
      pool-name: dbcp1
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test?autoReconnect=true&useSSL=false&useUnicode=true&characterEncoding=utf-8
    username: user
    password: pwd
    type: com.zaxxer.hikari.HikariDataSource
    schema[0]: classpath:/init.sql
    initialize: true
```
可以看到，我们配置了hikari和数据库的基本信息。在应用服务启动时，会自动初始化classpath下的sql脚本。

```sql
CREATE TABLE IF NOT EXISTS `test` (
  `id` bigint(20) unsigned NOT NULL,
  `local_name` varchar(128) NOT NULL ,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
在sql脚本中，我们创建了一张`test`表。

到这里，后面我们一般需要配置Mybatis映射的xml文件和实体类的路径。根据mybatis generator 自动生成代码。包括`XXMapper.java`,`XXEntity.java`, `XXMapper.xml`。这里我们就不演示了，直接进入下一步的通用Mapper实现。

## 通用Mapper的使用

### 引入依赖

```xml
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper</artifactId>
            <version>3.4.0</version>
        </dependency>
```
通用Mapper的作者[abel533](https://github.com/abel533/Mapper)，有兴趣可阅读源码。

### 配置通用Mapper

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import tk.mybatis.spring.mapper.MapperScannerConfigurer;

import java.util.Properties;

@Configuration
public class MyBatisMapperScannerConfig {
    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
        mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        mapperScannerConfigurer.setBasePackage("com.blueskykong.mybatis.dao");//扫描该路径下的dao
        Properties properties = new Properties();
        properties.setProperty("mappers", "com.blueskykong.mybatis.config.BaseDao");//通用dao
        properties.setProperty("notEmpty", "false");
        properties.setProperty("IDENTITY", "MYSQL");
        mapperScannerConfigurer.setProperties(properties);
        return mapperScannerConfigurer;
    }
}
```
在配置中，设定了指定路径下的dao，并指定了通用dao。需要注意的是，`MapperScannerConfigurer`来自于`tk.mybatis.spring.mapper`包下。

### BaseDao

```java
import tk.mybatis.mapper.common.Mapper;
import tk.mybatis.mapper.common.MySqlMapper;

public interface BaseDao<T> extends Mapper<T>,MySqlMapper<T>{

}
```
通用Mapper接口，其他接口继承该接口即可。

### 创建实体
我们需要添加`test`表对应的实体。

```java
@Data
@Table(name = "test")
@AllArgsConstructor
@NoArgsConstructor
public class TestModel {

    @Id
    @Column(name = "id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String localName;
}
```
其中，`@Table(name = "test")`注解指定了该实体对应的数据库表名。

### 配置文件

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```
为了更好地映射Java实体和数据库字段，我们指定下划线驼峰法的映射配置。

### TestDao编写

```java
public interface TestDao extends BaseDao<TestModel> {


    @Insert("insert into test(id, local_name) values(#{id}, #{localName})")
    Integer insertTestModel(TestModel testModel);
}
```
`TestDao`继承自`BaseDao`，并指定了泛型为对应的`TestModel`。`TestDao`包含继承的方法，如：

```java
    int deleteByPrimaryKey(Integer userId);

    int insert(User record);

    int insertSelective(User record);

    User selectByPrimaryKey(Integer userId);

    int updateByPrimaryKeySelective(User record);

    int updateByPrimaryKey(User record);
```
还可以自定义一些方法，我们在上面自定义了一个`insertTestModel`方法。

### Service层和控制层

本文略过这两层，比较简单，读者可以参见本文对应的源码地址。

### 结果验证

我们在插入一条数据之后，查询对应的实体。对应执行的结果也都是成功，可以看到控制台的如下日志信息：

```
c.b.mybatis.dao.TestDao.insertTestModel  : ==>  Preparing: insert into test(id, local_name) values(?, ?) 
c.b.mybatis.dao.TestDao.insertTestModel  : ==> Parameters: 5953(Integer), testName(String)
c.b.mybatis.dao.TestDao.insertTestModel  : <==    Updates: 1
c.b.m.dao.TestDao.selectByPrimaryKey     : ==>  Preparing: SELECT id,local_name FROM test WHERE id = ? 
c.b.m.dao.TestDao.selectByPrimaryKey     : ==> Parameters: 5953(Integer)
c.b.m.dao.TestDao.selectByPrimaryKey     : <==      Total: 1
```
Spring Boot集成MyBatis实现通用Mapper到此就大功告成。

## 小结
MyBatis是持久层非常常用的组件，Spring Boot倡导约定优于配置，特别是很多xml的配置。当然还有很多同学使用Spring Data。相比而言，我觉得MyBatis的SQL比Spring Data更加灵活，至于具体比较不在此讨论。

**本文对应的源码地址：   
https://github.com/keets2012/Spring-Boot-Samples/tree/master/mybatis-demo**
#### 参考

1. [abel533/Mapper](https://github.com/abel533/Mapper/wiki/4.1.mappergenerator)
2. [配置Spring Boot集成MyBatis、通用Mapper、Quartz、PageHelper
](https://www.jianshu.com/p/cf43176067d8)



