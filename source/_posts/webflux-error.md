---
title: Spring Boot 2 Webflux的全局异常处理
categories: 微服务
tags:
  - WebFlux
  - Spring
img: 'http://image.blueskykong.com/IMG_20180708_092949.jpg'
abbrlink: 56417
date: 2018-12-18 00:00:00
---

本文首先将会回顾Spring 5之前的SpringMVC异常处理机制，然后主要讲解Spring Boot 2 Webflux的全局异常处理机制。

## SpringMVC的异常处理
Spring 统一异常处理有 3 种方式，分别为：

- 使用 `@ExceptionHandler` 注解
- 实现 `HandlerExceptionResolver` 接口
- 使用 `@Controlleradvice` 注解

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
通过实现`HandlerExceptionResolver`接口，这里我们通过继承`SimpleMappingExceptionResolver`实现类（`HandlerExceptionResolver`实现，允许将异常类名称映射到视图名称，既可以是一组给定的handlers处理程序，也可以是`DispatcherServlet`中的所有handlers）定义全局异常：

```java
@Component
public class CustomMvcExceptionHandler extends SimpleMappingExceptionResolver {

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
@Component
public class TimeHandler {
    public Mono<ServerResponse> getTime(ServerRequest serverRequest) {
        String timeType = serverRequest.queryParam("type").get();
        //return ...
    }
}
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
如果我们在没有指定时间类型（type）的情况下调用相同的请求地址，例如/time，它将抛出异常。
Mono和Flux APIs内置了两个关键操作符，用于处理功能级别上的错误。

#### 使用onErrorResume处理错误
还可以使用onErrorResume处理错误，fallback方法定义如下：

```java
Mono<T> onErrorResume(Function<? super Throwable, ? extends Mono<? extends T>> fallback);
```
当出现错误时，我们使用fallback方法执行替代路径：

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
在如上的实现中，每当`getTimeByType()`抛出异常时，将会执行我们定义的`fallback`方法。除此之外，我们还可以捕获、包装和重新抛出异常，例如作为自定义业务异常：

```java
    public Mono<ServerResponse> getTime(ServerRequest serverRequest) {
        String timeType = serverRequest.queryParam("time").orElse("Now");
        return ServerResponse.ok()
                .body(getTimeByType(timeType)
                        .onErrorResume(e -> Mono.error(new ServerException(new ErrorCode(HttpStatus.BAD_REQUEST.value(),
                                "timeType is required", e.getMessage())))), String.class);
    }
```

#### 使用onErrorReturn处理错误
每当发生错误时，我们可以使用`onErrorReturn()`返回静态默认值：

```java
    public Mono<ServerResponse> getDate(ServerRequest serverRequest) {
        String timeType = serverRequest.queryParam("time").get();
        return getTimeByType(timeType)
                .onErrorReturn("Today is " + new SimpleDateFormat("yyyy-MM-dd").format(new Date()))
                .flatMap(s -> ServerResponse.ok()
                        .contentType(MediaType.TEXT_PLAIN).syncBody(s));
    }
```

### 全局异常处理
如上的配置是在方法的级别处理异常，如同对注解的Controller全局异常处理一样，WebFlux的函数式开发模式也可以进行全局异常处理。要做到这一点，我们只需要自定义全局错误响应属性，并且实现全局错误处理逻辑。

我们的处理程序抛出的异常将自动转换为HTTP状态和JSON错误正文。要自定义这些，我们可以简单地扩展`DefaultErrorAttributes`类并覆盖其`getErrorAttributes()`方法：

```java
@Component
public class GlobalErrorAttributes extends DefaultErrorAttributes {

    public GlobalErrorAttributes() {
        super(false);
    }

    @Override
    public Map<String, Object> getErrorAttributes(ServerRequest request, boolean includeStackTrace) {
        return assembleError(request);
    }

    private Map<String, Object> assembleError(ServerRequest request) {
        Map<String, Object> errorAttributes = new LinkedHashMap<>();
        Throwable error = getError(request);
        if (error instanceof ServerException) {
            errorAttributes.put("code", ((ServerException) error).getCode().getCode());
            errorAttributes.put("data", error.getMessage());
        } else {
            errorAttributes.put("code", HttpStatus.INTERNAL_SERVER_ERROR);
            errorAttributes.put("data", "INTERNAL SERVER ERROR");
        }
        return errorAttributes;
    }
    //...有省略
}
```
如上的实现中，我们对`ServerException`进行了特别处理，根据传入的`ErrorCode`对象构造对应的响应。

接下来，让我们实现全局错误处理程序。为此，Spring提供了一个方便的`AbstractErrorWebExceptionHandler`类，供我们在处理全局错误时进行扩展和实现：

```java
@Component
@Order(-2)
public class GlobalErrorWebExceptionHandler extends AbstractErrorWebExceptionHandler {

	//构造函数
    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(final ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
    }

    private Mono<ServerResponse> renderErrorResponse(final ServerRequest request) {

        final Map<String, Object> errorPropertiesMap = getErrorAttributes(request, true);

        return ServerResponse.status(HttpStatus.OK)
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .body(BodyInserters.fromObject(errorPropertiesMap));
    }
}
```
这里将全局错误处理程序的顺序设置为-2。这是为了让它比`@Order(-1)`注册的`DefaultErrorWebExceptionHandler`处理程序更高的优先级。

该errorAttributes对象将是我们在网络异常处理程序的构造函数传递一个的精确副本。理想情况下，这应该是我们自定义的Error Attributes类。然后，我们清楚地表明我们想要将所有错误处理请求路由到renderErrorResponse()方法。最后，我们获取错误属性并将它们插入服务器响应主体中。

然后，它会生成一个JSON响应，其中包含错误，HTTP状态和计算机客户端异常消息的详细信息。对于浏览器客户端，它有一个whitelabel错误处理程序，它以HTML格式呈现相同的数据。当然，这可以是定制的。

## 小结
本文首先讲了Spring 5之前的SpringMVC异常处理机制，SpringMVC统一异常处理有 3 种方式：使用 `@ExceptionHandler` 注解、实现 `HandlerExceptionResolver` 接口、使用 `@controlleradvice` 注解；然后通过WebFlux的函数式接口构建Web应用，讲解Spring Boot 2 Webflux的函数级别和全局异常处理机制（对于Spring WebMVC风格，基于注解的方式编写响应式的Web服务，仍然可以通过SpringMVC统一异常处理实现）。

注：本文后半部分基本翻译自**https://www.baeldung.com/spring-webflux-errors**


#### 订阅最新文章，欢迎关注我的公众号

![微信公众号](https://user-gold-cdn.xitu.io/2018/12/14/167ab4f74e513f68?w=430&h=430&f=jpeg&s=24511)

#### 参考
1. [Handling Errors in Spring WebFlux](https://www.baeldung.com/spring-webflux-errors)
2. [Spring WebFlux快速上手——响应式Spring的道法术器](https://blog.csdn.net/get_set/article/details/79480233)

