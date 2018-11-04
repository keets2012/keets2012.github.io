---
title: 基于可靠消息方案的分布式事务（四）：接入Lottor服务
date: 2018-8-8
categories: 分布式事务
tags:
  - 分布式
  - 事务
  - micro-service
abbrlink: 27575
---
在[上一篇](http://blueskykong.com/2018/08/05/lottor-readme/)文章中，通过Lottor Sample介绍了快速体验分布式事务Lottor。本文将会介绍如何将微服务中的生产方和消费方服务接入Lottor。

## 场景描述

- 生产方：User服务
- 消费方：Auth服务
- 事务管理方：Lottor Server

Lottor-Samples中的场景为：客户端调用User服务创建一个用户，用户服务的user表中增加了一条用户记录。除此之外，还会调用Auth服务创建该用户对应的角色和权限信息。
![](http://image.blueskykong.com/事务流程.jpg)

我们通过上面的请求流程图入手，介绍接入Lottor服务。当您启动好docker-compose中的组件时，会创建好两个服务对应的user和auth数据库。其中User和Auth服务所需要的初始化数据已经准备好，放在各自的classpath下，服务在启动时会自动初始化数据库，所需要的预置数据（如角色、权限信息）也放在sql文件中。

## Lottor客户端API
Lottor Client中提供了一个`ExternalNettyService`接口，用以发送三类消息到Lottor Server：

- 预提交消息
- 确认提交消息
- 消费完成消息

```
public interface ExternalNettyService {

    /**
     * pre-commit msgs
     *
     * @param preCommitMsgs
     */
    public Boolean preSend(List<TransactionMsg> preCommitMsgs);

    /**
     * confirm msgs
     *
     * @param success
     */
    public void postSend(Boolean success, Object message);

    /**
     * msgs after consuming
     *
     * @param msg
     * @param success
     */
    public void consumedSend(TransactionMsg msg, Boolean success);
}
```
预发送`#preSend`的入参为预提交的消息列表，一个生产者可能有对应的多个消费者；确认提交`#postSend`的入参为生产方本地事务执行的状态，如果失败，第二个参数记录异常信息；`#consumedSend`为消费方消费成功的发送的异步消息，第一个入参为其接收到的事务消息，第二个为消费的状态。

## 事务消息TransactionMsg

```java
public class TransactionMsg implements Serializable {
    /**
     * 用于消息的追溯
     */
    private String groupId;

    /**
     * 事务消息id
     */
    @NonNull
    private String subTaskId;

    /**
     * 源服务，即调用发起方
     */
    private String source;

    /**
     * 目标方服务
     */
    private String target;

    /**
     * 执行的方法，适配成枚举
     */
    private String method;

    /**
     * 参数，即要传递的内容，可以为null
     */
    private Object args;

    /**
     * 创建时间
     */
    private Long createTime = Timestamp.valueOf(DateUtils.getCurrentDateTime()).getTime();

    /**
     * 操作结果信息
     */
    private String message;

    /**
     * 更新时间
     */
    private Long updateTime;

    /**
     * 是否消费，默认为否
     *
     * {@linkplain com.blueskykong.lottor.common.enums.ConsumedStatus}
     */
    private int consumed = ConsumedStatus.UNCONSUMED.getStatus();
	 
	 ...
}
```
在构建事务消息时，事务消息id、源服务、目标服务、目标方法和目标方法的传参args都是必不可少的。消费方消费完之后，将会设置consumed的状态，出现异常将会设置异常message信息。
## 生产方-User服务

创建用户时，需要创建对应的角色。生产方接入分为三步：

- 发送预提交消息
- 执行本地事务
- 发送确认提交的消息

### 引入依赖
首先，需要引入Lottor客户端的依赖：

```xml
    <dependency>
        <groupId>com.blueskykong</groupId>
        <artifactId>lottor-starter</artifactId>
        <version>2.0.0-SNAPSHOT</version>
    </dependency>
```

### 发起调用
在`UserService`中定义了创建用户的方法，我们需要在执行本地事务之前，构造事务消息并预发送到Lottor Server（对应流程图中的步骤1）。如果遇到预发送失败，则直接停止本地事务的执行。如果本地事务执行成功（对应步骤3），则发送confirm消息，否则发送回滚消息到Lottor Server（对应步骤4）。

```java
@Service
public class UserServiceImpl implements UserService {

    private static final Logger LOGGER = LoggerFactory.getLogger(UserServiceImpl.class);

	 //注入ExternalNettyService
    @Autowired
    private ExternalNettyService nettyService;

    @Autowired
    private UserMapper userMapper;

    @Override
    @Transactional
    public Boolean createUser(UserEntity userEntity, StateEnum flag) {
        UserRoleDTO userRoleDTO = new UserRoleDTO(RoleEnum.ADMIN, userEntity.getId());
		 //构造消费方的TransactionMsg
        TransactionMsg transactionMsg = new TransactionMsg.Builder()
                .setSource(ServiceNameEnum.TEST_USER.getServiceName())
                .setTarget(ServiceNameEnum.TEST_AUTH.getServiceName())
                .setMethod(MethodNameEnum.AUTH_ROLE.getMethod())
                .setSubTaskId(IdWorkerUtils.getInstance().createUUID())
                .setArgs(userRoleDTO)
                .build();

        if (flag == StateEnum.CONSUME_FAIL) {
            userRoleDTO.setUserId(null);
            transactionMsg.setArgs(userRoleDTO);
        }

        //发送预处理消息
        if (!nettyService.preSend(Collections.singletonList(transactionMsg))) {
            return false;//预发送失败，本地事务停止执行
        }

        //local transaction本地事务
        try {
            LOGGER.debug("执行本地事务！");
            if (flag != StateEnum.PRODUCE_FAIL) {
                userMapper.saveUser(userEntity);
            } else {
                userMapper.saveUserFailure(userEntity);
            }
        } catch (Exception e) {
        	  //本地事务异常，发送回滚消息
            nettyService.postSend(false, e.getMessage());
            LOGGER.error("执行本地事务失败，cause is 【{}】", e.getLocalizedMessage());
            return false;
        }
        //发送确认消息
        nettyService.postSend(true, null);
        return true;
    }

}
```
代码如上所示，实现不是很复杂。本地事务执行前，必然已经成功发送了预提交消息，当本地事务执行成功，Lottor Client将会记录本地事务执行的状态，避免异步发送的确认消息的丢失，便于后续的Lottor Server回查。

### 配置文件

```yml
lottor:
  enabled: true
  core:
    cache: true  
    cache-type: redis
    tx-redis-config:
      host-name: localhost
      port: 6379
    serializer: kryo
    netty-serializer: kryo
    tx-manager-id: lottor

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user?autoReconnect=true&useSSL=false
    continue-on-error: false
    initialize: true
    max-active: 50
    max-idle: 10
    max-wait: 10000
    min-evictable-idle-time-millis: 60000
    min-idle: 8
    name: dbcp1
    test-on-borrow: false
    test-on-return: false
    test-while-idle: false
    time-between-eviction-runs-millis: 5000
    username: root
    password: _123456_
    schema[0]: classpath:/user.sql
```
如上为User服务的部分配置文件，`lottor.enabled: true`开启Lottor 客户端服务。cache 开启本地缓存记录。cache-type指定了本地事务记录的缓存方式，可以为redis或者MongoDB。serializer为序列化和反序列化方式。tx-manager-id为对应的Lottor Server的服务名。
## Lottor Server
多个微服务的接入，对Lottor Server其实没什么侵入性。这里需要注意的是，`TransactionMsg`中设置的`source`和`target`字段来源于lottor-common中的`com.blueskykong.lottor.common.enums.ServiceNameEnum`：

```java
public enum ServiceNameEnum {
    TEST_USER("user", "tx-user"),
    TEST_AUTH("auth", "tx-auth");
	//服务名
    String serviceName;
	//消息中间件的topic
    String topic;
    
    ...
}
```
消息中间件的topic是在服务名的基础上，加上`tx-`前缀。消费方在设置订阅的topic时，需要按照这样的规则命名。Lottor Server完成的步骤为上面流程图中的2（成功收到预提交消息）和5（发送事务消息到指定的消费方），除此之外，还会定时轮询异常状态的事务组和事务消息。
## 消费方-Auth服务
### 引入依赖

```xml
    <dependency>
        <groupId>com.blueskykong</groupId>
        <artifactId>lottor-starter</artifactId>
        <version>2.0.0-SNAPSHOT</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-stream</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    </dependency>
```
引入了Lottor客户端starter，spring-cloud-stream用于消费方接收来自Lottor Server的事务消息。
### topic监听

```java
@Component
@EnableBinding({TestSink.class})
public class ListenerStream extends InitStreamHandler {
    private static final Logger LOGGER = LoggerFactory.getLogger(ListenerStream.class);

    @Autowired
    private RoleUserService roleUserService;

    @Autowired
    public ListenerStream(ExternalNettyService nettyService, ObjectSerializer objectSerializer) {
        super(nettyService, objectSerializer);
    }

    @StreamListener(TestSink.INPUT)
    public void processSMS(Message message) {
        //解析接收到的TransactionMsg
        process(init(message));
    }

    @Transactional
    public void process(TransactionMsg message) {
        try {
            if (Objects.nonNull(message)) {
                LOGGER.info("===============consume notification message: =======================" + message.toString());
                if (StringUtils.isNotBlank(message.getMethod())) {
                    MethodNameEnum method = MethodNameEnum.fromString(message.getMethod());
                    LOGGER.info(message.getMethod());
                    //根据目标方法进行处理，因为一个服务可以对应多个生产方，有多个目标方法
                    switch (method) {
                        case AUTH_ROLE:
                            UserRoleDTO userRoleDTO = (UserRoleDTO) message.getArgs();
                            RoleEntity roleEntity = roleUserService.getRole(userRoleDTO.getRoleEnum().getName());
                            String roleId = "";
                            if (Objects.nonNull(roleEntity)) {
                                roleId = roleEntity.getId();
                            }
                            roleUserService.saveRoleUser(new UserRole(UUID.randomUUID().toString(), userRoleDTO.getUserId(), roleId));
                            LOGGER.info("matched case {}", MethodNameEnum.AUTH_ROLE);

                            break;
                        default:
                            LOGGER.warn("no matched consumer case!");
                            message.setMessage("no matched consumer case!");
                            nettyService.consumedSend(message, false);
                            return;
                    }
                }
            }
        } catch (Exception e) {
        	  //处理异常，发送消费失败的消息
            LOGGER.error(e.getLocalizedMessage());
            message.setMessage(e.getLocalizedMessage());
            nettyService.consumedSend(message, false);
            return;
        }
        //成功消费
        nettyService.consumedSend(message, true);
        return;
    }
}
```
消费方监听指定的topic（如上实现中，为test-input中指定的topic，spring-cloud-stream更加简便调用的接口），解析接收到的TransactionMsg。根据目标方法进行处理，因为一个服务可以对应多个生产方，有多个目标方法。执行本地事务时，Auth会根据TransactionMsg中提供的args作为入参执行指定的方法（对应步骤7），最后向Lottor Server发送消费的结果（对应步骤8）。
### 配置文件

```yml
---
spring:
  cloud:
    stream:
      bindings:
        test-input:
          group: testGroup
          content-type: application/x-java-object;type=com.blueskykong.lottor.common.entity.TransactionMsgAdapter
          destination: tx-auth
          binder: rabbit1
      binders:
        rabbit1:
          type: rabbit
          environment:
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
                virtual-host: /

---
lottor:
  enabled: true
  core:
    cache: true
    cache-type: redis
    tx-redis-config:
      host-name: localhost
      port: 6379
    serializer: kryo
    netty-serializer: kryo
    tx-manager-id: lottor
```
配置和User服务的差别在于增加了spring-cloud-stream的配置，配置了rabbitmq的相关信息，监听的topic为tx-auth。

## 小结
本文主要通过User和Auth的示例服务讲解了如何接入Lottor客户端。生产方构造涉及的事务消息，首先预发送事务消息到Lottor Server，成功预提交之后便执行本地事务；本地事务执行完则异步发送确认消息（可能成功，也可能失败）。Lottor Server根据接收到的确认消息决定是否将对应的事务组消息发送到对应的消费方。Lottor Server还会定时轮询异常状态的事务组和事务消息，以防因为异步的确认消息发送失败。消费方收到事务消息之后，将会根据目标方法执行对应的处理操作，最后将消费结果异步回写到Lottor Server。

### 推荐阅读

[基于可靠消息方案的分布式事务](http://blueskykong.com/categories/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1/)

**Lottor项目地址：https://github.com/keets2012/Lottor**

