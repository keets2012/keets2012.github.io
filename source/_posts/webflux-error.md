---
title: Hystrix断路器在微服务网关中的应用（Spring Cloud Gateway）
categories: 微服务
tags:
  - 微服务
  - gateway
abbrlink: 56417
img: http://image.blueskykong.com/IMG_20180708_093833.jpg
date: 2018-12-11 00:00:00
---
# Spring Boot 2 Webflux的全局异常处理
本文首先将会回顾Spring 5之前的SpringMVC异常处理机制，然后主要讲解Spring Boot 2 Webflux的全局异常处理机制。

## SpringMVC的异常处理
Spring 统一异常处理有 3 种方式，分别为：

- 使用 `@ExceptionHandler` 注解
- 实现 `HandlerExceptionResolver` 接口
- 使用 `@controlleradvice` 注解

### 使用`@ExceptionHandler`注解
用于局部方法捕获，与抛出异常的方法处于同一个Controller类：

```java
@Controller
public class BuzController {

    @ExceptionHandler({NullPointerException.class})
    public String exception(NullPointerException e) {
        System.out.println(e.getMessage());
        e.printStackTrace();
        return "null pointer exception";
    }

    @RequestMapping("test")
    public void test() {
        throw new NullPointerException("出错了！");
    }
}
```
如上的代码实现，针对`BuzController`抛出的`NullPointerException`异常，将会捕获局部异常，返回指定的内容。

### 实现`HandlerExceptionResolver`接口
通过实现`HandlerExceptionResolver`接口，定义全局异常：

```java
@Component
public class CustomMvcExceptionHandler implements HandlerExceptionResolver {

    private ObjectMapper objectMapper;

    public CustomMvcExceptionHandler() {
        objectMapper = new ObjectMapper();
    }

    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
                                         Object o, Exception ex) {
        response.setStatus(200);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.setHeader("Cache-Control", "no-cache, must-revalidate");
        Map<String, Object> map = new HashMap<>();
        if (ex instanceof NullPointerException) {
            map.put("code", ResponseCode.NP_EXCEPTION);
        } else if (ex instanceof IndexOutOfBoundsException) {
            map.put("code", ResponseCode.INDEX_OUT_OF_BOUNDS_EXCEPTION);
        } else {
            map.put("code", ResponseCode.CATCH_EXCEPTION);
        }
        try {
            map.put("data", ex.getMessage());
            response.getWriter().write(objectMapper.writeValueAsString(map));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return new ModelAndView();
    }
}
```
如上为示例的使用方式，我们可以根据各种异常定制错误的响应。

### 使用`@controlleradvice`注解

```java
@ControllerAdvice
public class ExceptionController {
    @ExceptionHandler(RuntimeException.class)
    public ModelAndView handlerRuntimeException(RuntimeException ex) {
        if (ex instanceof MaxUploadSizeExceededException) {
            return new ModelAndView("error").addObject("msg", "文件太大！");
        }
        return new ModelAndView("error").addObject("msg", "未知错误：" + ex);
    }

    @ExceptionHandler(Exception.class)
    public ModelAndView handlerMaxUploadSizeExceededException(Exception ex) {
        if (ex != null) {
            return new ModelAndView("error").addObject("msg", ex);
        }

        return new ModelAndView("error").addObject("msg", "未知错误：" + ex);

    }
}
```
和第一种方式的区别在于，`ExceptionHandler`的定义和异常捕获可以扩展到全局。
## Spring 5 Webflux的异常处理
webflux支持mvc的注解，是一个非常便利的功能，相比较于RouteFunction，自动扫描注册比较省事。异常处理可以沿用ExceptionHandler。如下的全局异常处理对于RestController依然生效。

```java
@RestControllerAdvice
public class CustomExceptionHandler {
    private final Log logger = LogFactory.getLog(getClass());
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(code = HttpStatus.OK)
    public ErrorCode handleCustomException(Exception e) {
        logger.error(e.getMessage());
        return new ErrorCode("e","error" );
    }
}
```

### WebFlux示例

WebFlux提供了一套函数式接口，可以用来实现类似MVC的效果。我们先接触两个常用的。

Controller定义对Request的处理逻辑的方式，主要有方面：

- 方法定义处理逻辑；
- 然后用@RequestMapping注解定义好这个方法对什么样url进行响应。

在WebFlux的函数式开发模式中，我们用HandlerFunction和RouterFunction来实现上边这两点。

#### HandlerFunction
`HandlerFunction`相当于Controller中的具体处理方法，输入为请求，输出为装在Mono中的响应：

```java
    Mono<T> handle(ServerRequest var1);
```

在WebFlux中，请求和响应不再是WebMVC中的ServletRequest和ServletResponse，而是ServerRequest和ServerResponse。后者是在响应式编程中使用的接口，它们提供了对非阻塞和回压特性的支持，以及Http消息体与响应式类型Mono和Flux的转换方法。

```java
HandlerFunction<ServerResponse> timeFunction = 
        request -> ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).body(
            Mono.just("Now is " + new SimpleDateFormat("HH:mm:ss").format(new Date())), String.class);
```
如上定义了一个`TimeHandler`，根据请求的参数返回当前时间。

#### RouterFunction
`RouterFunction`，顾名思义，路由，相当于`@RequestMapping`，用来判断什么样的url映射到那个具体的`HandlerFunction`。输入为请求，输出为Mono中的`Handlerfunction`：

```java
Mono<HandlerFunction<T>> route(ServerRequest var1);
```

针对我们要对外提供的功能，我们定义一个Route。
```java
@Configuration
public class RouterConfig {
    private final TimeHandler timeHandler;

    @Autowired
    public RouterConfig(TimeHandler timeHandler) {
        this.timeHandler = timeHandler;
    }

    @Bean
    public RouterFunction<ServerResponse> timerRouter() {
        return route(GET("/time"), req -> timeHandler.getTime(req));  
    }
}
```
可以看到访问/time的GET请求，将会由`TimeHandler::getTime`处理。

### 功能级别处理异常

```java
@Component
public class TimeHandler {
    public Mono<ServerResponse> getTime(ServerRequest serverRequest) {
        String timeType = serverRequest.queryParam("time").orElse("Now");
        return getTimeByType(timeType).flatMap(s -> ServerResponse.ok()
                .contentType(MediaType.TEXT_PLAIN).syncBody(s))
                .onErrorResume(e -> Mono.just("Error: " + e.getMessage()).flatMap(s -> ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).syncBody(s)));
    }

    private Mono<String> getTimeByType(String timeType) {
        String type = Optional.ofNullable(timeType).orElse(
                "Now"
        );
        switch (type) {
            case "Now":
                return Mono.just("Now is " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            case "Today":
                return Mono.just("Today is " + new SimpleDateFormat("yyyy-MM-dd").format(new Date()));
            default:
                return Mono.empty();
        }
    }
}
```







