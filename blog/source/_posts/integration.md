---
title: 微服务架构中整合网关、权限服务
date: 2017-12-10
categories: Security
tags:
- zuul
- OAuth2
- Spring Security
- gateway
---
前言：之前的文章有讲过微服务的权限系列和网关实现，都是孤立存在，本文将整合后端服务与网关、权限系统。安全权限部分的实现还讲解了基于前置验证的方式实现，但是由于与业务联系比较紧密，没有具体的示例。业务权限与业务联系非常密切，本次的整合项目将会把这部分的操作权限校验实现基于具体的业务服务。


## 1. 前文回顾与整合设计
在[认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/2017/10/19/security1/)系列文章中，讲解了在微服务架构中Auth系统的授权认证和鉴权。在[微服务网关](http://blueskykong.com/2017/11/13/gateway/)中，讲解了基于netflix-zuul组件实现的微服务网关。下面我们看一下这次整合的架构图。

![ms](http://ovcjgn2x0.bkt.clouddn.com/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E6%9D%83%E9%99%90%20%281%29.png "微服务架构权限")

整个流程分为两类：

- 用户尚未登录。客户端（web和移动端）发起登录请求，网关对于登录请求直接转发到auth服务，auth服务对用户身份信息进行校验（整合项目省略用户系统，读者可自行实现，直接硬编码返回用户信息），最终将身份合法的token返回给客户端。
- 用户已登录，请求其他服务。这种情况，客户端的请求到达网关，网关会调用auth系统进行请求身份合法性的验证，验证不通则直接拒绝，并返回401；如果通过验证，则转发到具体服务，服务经过过滤器，根据请求头部中的userId，获取该user的安全权限信息。利用切面，对该接口需要的权限进行校验，通过则proceed，否则返回403。

第一类其实比较简单，在讲解[认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/2017/10/19/security1/)就基本实现，现在要做的是与网关进行结合；第二类中，我们新建了一个后端服务，与网关、auth系统整合。

下面对整合项目涉及到的三个服务分别介绍。网关和auth服务的实现已经讲过，本文主要讲下这两个服务进行整合需要的改动，还有就是对于后端服务的主要实现进行讲解。

## 2. gateway实现
[微服务网关](http://blueskykong.com/2017/11/13/gateway/)已经基本介绍完了网关的实现，包括服务路由、几种过滤方式等。这一节将重点介绍实际应用时的整合。对于需要修改增强的地方如下：

- 区分暴露接口（即对外直接访问）和需要合法身份登录之后才能访问的接口
- 暴露接口直接放行，转发到具体服务，如登录、刷新token等
- 需要合法身份登录之后才能访问的接口，根据传入的Access token进行构造头部，头部主要包括userId等信息，可根据自己的实际业务在auth服务中进行设置。
- 最后，比较重要的一点，引入Spring Security的资源服务器配置，对于暴露接口设置permitAll()，其余接口进入身份合法性校验的流程，调用auth服务，如果通过则正常继续转发，否则抛出异常，返回401。

绘制的流程图如下：

![gwflow](http://ovcjgn2x0.bkt.clouddn.com/gwflow.jpg "网关路由流程图")

### 2.1 permitAll实现
对外暴露的接口可以直接访问，这可以依赖配置文件，而配置文件又可以通过配置中心进行动态更新，所以不用担心有hard-code的问题。
在配置文件中定义需要permitall的路径。

```yml
auth:
  permitall:
    -
      pattern: /login/**
    -
      pattern: /web/public/**
```
服务启动时，读入相应的Configuration，下面的配置属性读取以auth开头的配置。

```java
    @Bean
    @ConfigurationProperties(prefix = "auth")
    public PermitAllUrlProperties getPermitAllUrlProperties() {
        return new PermitAllUrlProperties();
    }
```

当然还需要有PermitAllUrlProperties对应的实体类，比较简单，不列出来了。

### 2.2 加强头部
Filter过滤器，它是Servlet技术中最实用的技术，Web开发人员通过Filter技术，对web服务器管理的所有web资源进行拦截。这边使用Filter进行头部增强，解析请求中的token，构造统一的头部信息，到了具体服务，可以利用头部中的userId进行操作权限获取与判断。




```java
public class HeaderEnhanceFilter implements Filter {

	//...

    @Autowired
    private PermitAllUrlProperties permitAllUrlProperties;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

	//主要的过滤方法
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        String authorization = ((HttpServletRequest) servletRequest).getHeader("Authorization");
        String requestURI = ((HttpServletRequest) servletRequest).getRequestURI();
        // test if request url is permit all , then remove authorization from header
        LOGGER.info(String.format("Enhance request URI : %s.", requestURI));
        //将isPermitAllUrl的请求进行传递
        if(isPermitAllUrl(requestURI) && isNotOAuthEndpoint(requestURI)) {
        	//移除头部，但不包括登录端点的头部
            HttpServletRequest resetRequest = removeValueFromRequestHeader((HttpServletRequest) servletRequest);
            filterChain.doFilter(resetRequest, servletResponse);
            return;
        }
        //判断是不是符合规范的头部
        if (StringUtils.isNotEmpty(authorization)) {
            if (isJwtBearerToken(authorization)) {
                try {
                    authorization = StringUtils.substringBetween(authorization, ".");
                    String decoded = new String(Base64.decodeBase64(authorization));

                    Map properties = new ObjectMapper().readValue(decoded, Map.class);
					//解析authorization中的token，构造USER_ID_IN_HEADER
                    String userId = (String) properties.get(SecurityConstants.USER_ID_IN_HEADER);

                    RequestContext.getCurrentContext().addZuulRequestHeader(SecurityConstants.USER_ID_IN_HEADER, userId);
                } catch (Exception e) {
                    LOGGER.error("Failed to customize header for the request", e);
                }
            }
        } else {
          //为了适配，设置匿名头部
            RequestContext.getCurrentContext().addZuulRequestHeader(SecurityConstants.USER_ID_IN_HEADER, ANONYMOUS_USER_ID);
        }

        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
    
    //...
    
}
```

上面代码列出了头部增强的基本处理流程，将isPermitAllUrl的请求进行直接传递，否则判断是不是符合规范的头部，然后解析authorization中的token，构造USER_ID_IN_HEADER。最后为了适配，设置匿名头部。   
需要注意的是，HeaderEnhanceFilter也要进行注册。Spring 提供了FilterRegistrationBean类，此类提供setOrder方法，可以为filter设置排序值，让spring在注册web filter之前排序后再依次注册。

### 2.3 资源服务器配置
利用资源服务器的配置，控制哪些是暴露端点不需要进行身份合法性的校验，直接路由转发，哪些是需要进行身份loadAuthentication，调用auth服务。

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

	//... 
	//配置permitAll的请求pattern，依赖于permitAllUrlProperties对象
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .requestMatchers().antMatchers("/**")
                .and()
                .authorizeRequests()
                .antMatchers(permitAllUrlProperties.getPermitallPatterns()).permitAll()
                .anyRequest().authenticated();
    }

	//通过自定义的CustomRemoteTokenServices，植入身份合法性的相关验证
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        CustomRemoteTokenServices resourceServerTokenServices = new CustomRemoteTokenServices();
        //...
        resources.tokenServices(resourceServerTokenServices);
    }
}
```

资源服务器的配置大家看了笔者之前的文章应该很熟悉，此处不过多重复讲了。关于`ResourceServerSecurityConfigurer`配置类，之前的安全系列文章已经讲过，`ResourceServerTokenServices`接口，当时我们也用到了，只不过用的是默认的`DefaultTokenServices`。这边通过自定义的`CustomRemoteTokenServices`，植入身份合法性的相关验证。

当然这个配置还要引入Spring Cloud Security oauth2的相应依赖。

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>
```


### 2.4 自定义RemoteTokenServices实现

`ResourceServerTokenServices`接口其中的一个实现是`RemoteTokenServices`。
>  Queries the /check_token endpoint to obtain the contents of an access token.
If the endpoint returns a 400 response, this indicates that the token is invalid.

`RemoteTokenServices`主要是查询auth服务的`/check_token`端点以获取一个token的校验结果。如果有错误，则说明token是不合法的。笔者这边的的`CustomRemoteTokenServices`实现就是沿用该思路。需要注意的是，笔者的项目基于Spring cloud，auth服务是多实例的，所以这边使用了Netflix Ribbon获取auth服务进行负载均衡。Spring Cloud Security添加如下默认配置，对应auth服务中的相应端点。

```yml
security:
  oauth2:
    client:
      accessTokenUri: /oauth/token
      clientId: gateway
      clientSecret: gateway
    resource:
      userInfoUri: /user
      token-info-uri: /oauth/check_token
```

至于具体的`CustomRemoteTokenServices`实现，可以参考上面讲的思路以及`RemoteTokenServices`，很简单，此处略去。


至此，网关服务的增强完成，下面看一下我们对auth服务和后端backend服务的实现。   
**强调一下，为什么头部传递的userId等信息需要在网关构造？读者可以自己思考一下，结合安全等方面，😆笔者暂时不给出答案。**

## 3. auth整合

auth服务的整合修改，其实没那么多，之前对于user、role以及permission之间的定义和关系没有给出实现，这部分的sql语句已经在auth.sql中。所以为了能给出一个完整的实例，笔者把这部分实现给补充了，主要就是user-role，role、role-permission的相应接口定义与实现，实现增删改查。   

读者要是想参考整合项目进行实际应用，这部分完全可以根据自己的业务进行增强，包括token的创建，其自定义的信息还可以在网关中进行统一处理，构造好之后传递给后端服务。

这边的接口只是列出了需要的几个，其他接口没写（因为懒。。）

这两个接口也是给backend项目用来获取相应的userId权限。

```java
//根据userId获取用户对应的权限
 @RequestMapping(method = RequestMethod.GET, value = "/api/userPermissions?userId={userId}",
            consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    List<Permission> getUserPermissions(@RequestParam("userId") String userId);

//根据userId获取用户对应的accessLevel（好像暂时没用到。。）
    @RequestMapping(method = RequestMethod.GET, value = "/api/userAccesses?userId={userId}",
            consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    List<UserAccess> getUserAccessList(@RequestParam("userId") String userId);
```


好了，这边的实现已经讲完了，具体见项目中的实现。

## 4. backend项目实现
本节是进行实现一个backend的实例，后端项目主要实现哪些功能呢？我们考虑一下，之前网关服务和auth服务所做的准备：

- 网关构造的头部userId（可能还有其他信息，这边只是示例），可以在backend获得
- 转发到backend服务的请求，都是经过身份合法性校验，或者是直接对外暴露的接口
- auth服务，提供根据userId进行获取相应的权限的接口

根据这些，笔者绘制了一个backend的通用流程图：

![bf](http://ovcjgn2x0.bkt.clouddn.com/backend%E6%B5%81%E7%A8%8B.png "backend流程图")

上面的流程图其实已经非常清晰了，首先经过filter过滤器，填充`SecurityContextHolder`的上下文。其次，通过切面来实现注解，是否需要进入切面表达式处理。不需要的话，直接执行接口内的方法；否则解析注解中需要的权限，判断是否有权限执行，有的话继续执行，否则返回403 forbidden。

### 4.1 filter过滤器
Filter过滤器，和上面网关使用一样，拦截客户的HttpServletRequest。

```java
public class AuthorizationFilter implements Filter {

    @Autowired
    private FeignAuthClient feignAuthClient;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        logger.info("过滤器正在执行...");
        // pass the request along the filter chain
        String userId = ((HttpServletRequest) servletRequest).getHeader(SecurityConstants.USER_ID_IN_HEADER);

        if (StringUtils.isNotEmpty(userId)) {
            UserContext userContext = new UserContext(UUID.fromString(userId));
            userContext.setAccessType(AccessType.ACCESS_TYPE_NORMAL);

            List<Permission> permissionList = feignAuthClient.getUserPermissions(userId);
            List<SimpleGrantedAuthority> authorityList = new ArrayList<>();
            for (Permission permission : permissionList) {
                SimpleGrantedAuthority authority = new SimpleGrantedAuthority();
                authority.setAuthority(permission.getPermission());
                authorityList.add(authority);
            }

            CustomAuthentication userAuth  = new CustomAuthentication();
            userAuth.setAuthorities(authorityList);
            userContext.setAuthorities(authorityList);
            userContext.setAuthentication(userAuth);
            SecurityContextHolder.setContext(userContext);
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }
    
	//...
}
```

上述代码主要实现了，根据请求头中的userId，利用feign client获取auth服务中的该user所具有的权限集合。之后构造了一个UserContext，UserContext是自定义的，实现了Spring Security的`UserDetails, SecurityContext`接口。

### 4.2 通过切面来实现@PreAuth注解
基于Spring的项目，使用Spring的AOP切面实现注解是比较方便的一件事，这边我们使用了自定义的注解`@PreAuth`

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface PreAuth {
    String value();
}
```

Target用于描述注解的使用范围，超出范围时编译失败,可以用在方法或者类上面。在运行时生效。不了解注解相关知识的，可以自行Google。

```java
@Component
@Aspect
public class AuthAspect {


    @Pointcut("@annotation(com.blueskykong.auth.demo.annotation.PreAuth)")
    private void cut() {
    }

    /**
     * 定制一个环绕通知，当想获得注解里面的属性，可以直接注入该注解
     *
     * @param joinPoint
     * @param preAuth
     */
    @Around("cut()&&@annotation(preAuth)")
    public Object record(ProceedingJoinPoint joinPoint, PreAuth preAuth) throws Throwable {
		//取出注解中的表达式
        String value = preAuth.value();
        //Spring EL 对value进行解析
        SecurityExpressionOperations operations = new CustomerSecurityExpressionRoot(SecurityContextHolder.getContext().getAuthentication());
        StandardEvaluationContext operationContext = new StandardEvaluationContext(operations);
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(value);
        //获取表达式判断的结果
        boolean result = expression.getValue(operationContext, boolean.class);
        if (result) {
        	//继续执行接口内的方法
            return joinPoint.proceed();
        }
        return "Forbidden";
    }
}
```

因为Aspect作用在bean上，所以先用Component把这个类添加到容器中。`@Pointcut`定义要拦截的注解。`@Around`定制一个环绕通知，当想获得注解里面的属性，可以直接注入该注解。切面表达式内主要实现了，利用Spring EL对value进行解析，将`SecurityContextHolder.getContext()`转换成标准的操作上下文，然后解析注解中的表达式，最后获取对表达式判断的结果。

```java
public class CustomerSecurityExpressionRoot extends SecurityExpressionRoot {

    public CustomerSecurityExpressionRoot(Authentication authentication) {
        super(authentication);
    }
}
```
`CustomerSecurityExpressionRoot`继承的是抽象类`SecurityExpressionRoot`，而我们用到的实际表达式是定义在`SecurityExpressionOperations`接口，`SecurityExpressionRoot`又实现了`SecurityExpressionOperations`接口。不过这里面的具体判断实现，Spring Security 调用的也是Spring EL。

### 4.3 controller接口

下面我们看看最终接口是怎么用上面实现的注解。

```java
    @RequestMapping(value = "/test", method = RequestMethod.GET)
    @PreAuth("hasAuthority('CREATE_COMPANY')") // 还可以定义很多表达式，如hasRole('Admin')
    public String test() {
        return "ok";
    }
```

`@PreAuth`中，可以定义的表达式很多，可以看`SecurityExpressionOperations`接口中的方法。目前笔者只是实现了`hasAuthority()`表达式，如果你想支持其他所有表达式，只需要构造相应的`SecurityContextHolder`即可。

### 4.4 为什么这样设计？
有些读者看了上面的设计，既然好多用到了Spring Security的工具类，肯定会问，为什么要引入这么复杂的工具类？   

其实很简单，首先因为`SecurityExpressionOperations`接口中定义的表达式足够多，且较为合理，能够覆盖我们在平时用到的大部分场景；其次，笔者之前的设计是直接在注解中指定所需权限，没有扩展性，且可读性差；最后，Spring Security 4 确实引入了`@PreAuthorize,@PostAuthorize`等注解，本来想用来着，自己尝试了一下，发现对于微服务架构这样的接口级别的操作权限校验不是很适合，十多个过滤器太过复杂，而且还涉及到的Principal、Credentials等信息，这些已经在auth系统实现了身份合法性校验。笔者认为这边的功能实现并不是很复杂，需要很轻量的实现，读者有兴趣可以试着这部分的实现封装成jar包或者Spring Boot的starter。

### 4.5 后期优化
优化的地方主要有两点：

- 现在的设计是，每次请求过来都会去调用auth服务获取该user相应的权限信息。而后端微服务数量有很多，没必要每个服务，或者说一个服务的多个服务实例，每次都去调用auth服务，笔者认为完全可以引入redis集群的缓存机制，在请求到达一个服务的某个实例时，首先去查询对应的user的缓存中的权限，如果没有再调用auth服务，最后写入redis缓存。当然，如果权限更新了，在auth服务肯定要delete相应的user权限缓存。
- 关于被拒绝的请求，在切面表达式中，直接返回了对象，笔者认为可以和response status 403进行绑定，定制返回对象的内容，返回的response更加友好。

## 5. 总结
如上，首先讲了整合的设计思路，主要包含三个服务：gateway、auth和backend demo。整合的项目，总体比较复杂，其中gateway服务扩充了好多内容，对于暴露的接口进行路由转发，这边引入了Spring Security 的starter，配置资源服务器对暴露的路径进行放行；对于其他接口需要调用auth服务进行身份合法性校验，保证到达backend的请求都是合法的或者公开的接口；auth服务在之前的基础上，补充了role、permission、user相应的接口，供外部调用；backend demo是新起的服务，实现了接口级别的操作权限的校验，主要用到了自定义注解和Spring AOP切面。

由于实现的细节实在有点多，本文限于篇幅，只对部分重要的实现进行列出与讲解。如果读者有兴趣实际的应用，可以根据实际的业务进行扩增一些信息，如auth授权的token、网关拦截请求构造的头部信息、注解支持的表达式等等。

可以优化的地方当然还有很多，整合项目中设计不合理的地方，各位同学可以多多提意见。

#### 推荐阅读
[系列文章：认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/categories/Security/) 

#### 源码

**网关、auth权限服务和backend服务的整合项目地址为：   
GitHub：https://github.com/keets2012/microservice-integration   
或者 码云：https://gitee.com/keets/microservice-integration**
