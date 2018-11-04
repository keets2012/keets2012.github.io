---
title: Spring Cloud OpenFeign集成Protocol Buffer
date: 2018-10-6
categories: 微服务
tags:
  - Spring Cloud
  - RPC
  - REST
abbrlink: 26066
---
> 本文作者张天，著有《Spring Cloud 微服务架构进阶》一书。

## 背景
&emsp;在之前的文章中，我们介绍过基于Spring Cloud微服务架构，其中，微服务实例之间的交互方式一般为RESTful HTTP请求或RPC调用。Spring Cloud已经为开发者提供了专门用于RESTful HTTP请求处理的OpenFeign组件，但是并没有相关的RPC调用组件。今天，我们就要定制OpenFeign的编解码器，使用Google的Protocol Buffer编码，让它拥有RPC调用的数据传输和转换效率高的优点。

&emsp;OpenFeign是一个声明式RESTful HTTP请求客户端，它使得编写Web服务客户端更加方便和快捷。它有较强的定制性，可以根据自己的需求来对它的各个方面进行定制，比如说编解码器，服务路由解析和负载均衡。

&emsp;而Protocol Buffer 是Google的一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

&emsp;OpenFeign默认使用`HttpUrlConnection`进行网络请求的发送; 相关实现代码在`DefaultFeignLoadBalancedConfiguration` 的`Client.Default`。而其使用的编解码器默认为jackson2，默认配置为`HttpMessageConvertersAutoConfiguration`。

&emsp;Protocol Buffer的编解码效率要远高于jackson2，在微服务实例频频通信的场景下，使用Protocol Buffer编解码时会少占用系统资源，并且效率较高。具体详见这个对比对比各种序列化和反序列化框架的性能的文档，https://github.com/eishay/jvm-serializers/wiki。
![jvm平台编解码效率示意图](https://upload-images.jianshu.io/upload_images/623378-0631a2353cfc9827.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 客户端集成Protocol Buffer
&emsp;开发人员可以使用自定义配置类对OpenFeign进行定制，提供OpenFeign所需要的编解码组件实例，从而替代默认的组件实例，达到定制化的目的。自定义的配置类如下所示。

``` java
@Configuration
public class ProtoFeignConfiguration {
    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverterObjectFactory;
    @Bean
    public ProtobufHttpMessageConverter protobufHttpMessageConverter() {
        return new ProtobufHttpMessageConverter();
    }

    @Bean
    public Encoder springEncoder() {
        return new SpringEncoder(this.messageConverterObjectFactory);
    }

    @Bean
    public Decoder springDecoder() {
        return new ResponseEntityDecoder(new SpringDecoder(this.messageConverterObjectFactory));
    }
}
```
&emsp;其中`ProtobufHttpMessageConverter`是`HttpMessageConverters `的Protobuf的实现类，负责使用Protocol Buffer进行网络请求和响应的编解码。而`SpringEncoder`和`ResponseEntityDecoder `是OpenFeign中的编解码器实现类。

&emsp;下面，我们来看一下OpenFeign中发送网络请求的接口定义。`@FeignClient`中配置了`ProtoFeignConfiguration`为自定义配置类。

```
@FeignClient(name = "user", configuration = ProtoFeignConfiguration.class)
public interface UserClient {
    @RequestMapping(value = "/info", method = RequestMethod.GET,
            consumes = "application/x-protobuf", produces = "application/x-protobuf")
    UserDTO getUserInfo(@RequestParam("id") Long id);
}
```
&emsp;其中，`UserDTO`是使用Protocol Buffer的maven插件自动生成的。需要注意的是，必须将`@RequestMapping`的`consumes`和`produces`属性设置为`application/x-protobuf`，表示网络请求和响应的编码格式必须是Protobuf，否则可能会接收到406的错误响应码。

&emsp;下面是proto文件中的数据格式定义，其中java_package是表明生成文件的目标文件夹。该文件中定义了UserDTO数据格式，它包括ID，名称和主页URL三个属性。

```
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.remcarpediem.feignprotobuf.proto.dto";

package com.remcarpediem.feignprotobuf.proto.dto;

message UserDTO {
    int32 id = 1;
    string name = 2;
    string url = 3;
}
```
&emsp;在pom文件中配置build属性，使用Protocol Buffer的maven插件可以自动根据proto文件生成Java代码。每个配置项都在代码中有对应的解释。

``` xml
<build>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.5.0</version>
                <extensions>true</extensions>
                <configuration>
                    <!--默认值-->
                    <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                    <!--默认值-->
                    <!--<outputDirectory>${project.build.directory}/generated-sources/protobuf/java</outputDirectory>-->
                    <outputDirectory>${project.build.sourceDirectory}</outputDirectory>
                    <!--设置是否在生成java文件之前清空outputDirectory的文件，默认值为true，设置为false时也会覆盖同名文件-->
                    <clearOutputDirectory>false</clearOutputDirectory>
                    <!--默认值-->
                    <temporaryProtoFileDirectory>${project.build.directory}/protoc-dependencies</temporaryProtoFileDirectory>
                    <!--更多配置信息可以查看https://www.xolstice.org/protobuf-maven-plugin/compile-mojo.html-->
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>test-compile</goal>
                        </goals>
                        <!--也可以设置成局部变量，执行compile或test-compile时才执行-->
                        <!--<configuration>-->
                        <!--<protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>-->
                        <!--<outputDirectory>${project.build.directory}/generated-sources/protobuf/java</outputDirectory>-->
                        <!--<temporaryProtoFileDirectory>${project.build.directory}/protoc-dependencies</temporaryProtoFileDirectory>-->
                        <!--</configuration>-->
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
&emsp;然后运行Protocol Buffer的maven插件可以自动生成相关的数据类。

## 服务端
&emsp;然后是服务端对于Protocol Buffer的集成。我们也需要使用自定义配置类将`ProtobufHttpMessageConverter`设置为系统默认的编解码器，如下述代码所示。

``` java
@Configuration
public class Conf {
    @Bean
    ProtobufHttpMessageConverter protobufHttpMessageConverter() {
        return new ProtobufHttpMessageConverter();
    }
}
```
&emsp;然后定义Controller的关于user的info接口。返回UserDTO实例作为网络请求的返回值。`ProtobufHttpMessageConverter`会自动将其转换为Protocol Buffer的数据格式进行传输。

``` java
@RestController
public class UserController {
    private String host = "http://blog.com/user/";
    @GetMapping("/info")
    public UserDTO getUserInfo(@RequestParam("id") Long id) {
        return UserDTO.newBuilder().
                setId(id).setName("Tom").
                setUrl(host + "Tom").build();
    }
}
```
&emsp;本文的源码地址： GitHub：https://github.com/ztelur/feign-protobuf

## 总结
&emsp;欲了解更详细的实现原理和细节，大家可以关注笔者出版的《Spring Cloud 微服务架构进阶》，本书中对Spring Cloud Finchley.RELEASE版本的各个主要组件进行原理讲解和实战应用，里边也有关于OpenFeign的原理和实现的详细解析。更多的介绍见[Spring Cloud 微服务架构进阶](http://blueskykong.com/2018/09/30/ms-book/)。
 ![封面](http://image.blueskykong.com/%E5%B0%81%E9%9D%A2-jd.jpg)
 **《Spring Cloud 微服务架构进阶》预售地址：https://item.jd.com/12453340.html**
