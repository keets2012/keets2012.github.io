---
title: 认证鉴权与API权限控制在微服务架构中的设计与实现：授权码模式
date: 2018-4-2
categories: Security
tags:
  - Spring Security
  - OAuth2
abbrlink: 23760
---

引言： 之前系列文章《认证鉴权与API权限控制在微服务架构中的设计与实现》，前面文章已经将认证鉴权与API权限控制的流程和主要细节讲解完。由于有些同学想了解下授权码模式，本文特地补充讲解。

## 授权码类型介绍
授权码类型(authorization code)通过重定向的方式让资源所有者直接与授权服务器进行交互来进行授权，避免了资源所有者信息泄漏给客户端，是功能最完整、流程最严密的授权类型，但是需要客户端必须能与资源所有者的代理(通常是Web浏览器)进行交互，和可从授权服务器中接受请求(重定向给予授权码)，授权流程如下：


     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)

1. 客户端引导资源所有者的用户代理到授权服务器的endpoint，一般通过重定向的方式。客户端提交的信息应包含客户端标识(client identifier)、请求范围(requested scope)、本地状态(local state)和用于返回授权码的重定向地址(redirection URI)
2. 授权服务器认证资源所有者(通过用户代理)，并确认资源所有者允许还是拒绝客户端的访问请求
3. 如果资源所有者授予客户端访问权限，授权服务器通过重定向用户代理的方式回调客户端提供的重定向地址，并在重定向地址中添加授权码和客户端先前提供的任何本地状态
4. 客户端携带上一步获得的授权码向授权服务器请求访问令牌。在这一步中授权码和客户端都要被授权服务器进行认证。客户端需要提交用于获取授权码的重定向地址
5. 授权服务器对客户端进行身份验证，和认证授权码，确保接收到的重定向地址与第三步中用于的获取授权码的重定向地址相匹配。如果有效，返回访问令牌，可能会有刷新令牌(Refresh Token)

## 快速入门

### Spring-Securiy 配置
由于授权码模式需要登录用户给请求access_token的客户端授权，所以auth-server需要添加Spring-Security的相关配置用于引导用户进行登录。

在原来的基础上，进行Spring-Securiy相关配置，允许用户进行表单登录：

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    CustomLogoutHandler customLogoutHandler;


    @Override
    protected void configure(HttpSecurity http) throws Exception {


        http.csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .and()
                .requestMatchers().antMatchers("/**")
                .and().authorizeRequests()
                .antMatchers("/**").permitAll()
                .anyRequest().authenticated()
                .and().formLogin()
                .permitAll()
                .and().logout()
                .logoutUrl("/logout")
                .clearAuthentication(true)
                .logoutSuccessHandler(new HttpStatusReturningLogoutSuccessHandler())
                .addLogoutHandler(customLogoutHandler);

    }

}

```

同时需要把`ResourceServerConfig`中的资源服务器中的对于登出端口的处理迁移到`WebSecurityConfig`中，注释掉`ResourceServerConfig`的`HttpSecurity`配置：

```java
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

//    @Override
//    public void configure(HttpSecurity http) throws Exception {
//        http.csrf().disable()
//                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
//                .and()
//                .requestMatchers().antMatchers("/**")
//                .and().authorizeRequests()
//                .antMatchers("/**").permitAll()
//                .anyRequest().authenticated()
//                .and().logout()
//                .logoutUrl("/logout")
//                .clearAuthentication(true)
//                .logoutSuccessHandler(new HttpStatusReturningLogoutSuccessHandler())
//                .addLogoutHandler(customLogoutHandler());
//
//        //http.antMatcher("/api/**").addFilterAt(customSecurityFilter(), FilterSecurityInterceptor.class);
//
//    }

 /*   @Bean
    public CustomSecurityFilter customSecurityFilter() {
        return new CustomSecurityFilter();
    }
*/
.....
}

```

### AuthenticationProvider
由于用户表单登录的认证过程可能有所不同，为此再添加一个`CustomSecurityAuthenticationProvider`，基本上与`CustomAuthenticationProvider`一致，只是忽略对client客户端的认证和处理。

```java
@Component
public class CustomSecurityAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private UserClient userClient;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password;

        Map map;

        password = (String) authentication.getCredentials();
        //如果你是调用user服务，这边不用注掉
        //map = userClient.checkUsernameAndPassword(getUserServicePostObject(username, password, type));
        map = checkUsernameAndPassword(getUserServicePostObject(username, password));


        String userId = (String) map.get("userId");
        if (StringUtils.isBlank(userId)) {
            String errorCode = (String) map.get("code");
            throw new BadCredentialsException(errorCode);
        }
        CustomUserDetails customUserDetails = buildCustomUserDetails(username, password, userId);
        return new CustomAuthenticationToken(customUserDetails);
    }

    private CustomUserDetails buildCustomUserDetails(String username, String password, String userId) {
        CustomUserDetails customUserDetails = new CustomUserDetails.CustomUserDetailsBuilder()
                .withUserId(userId)
                .withPassword(password)
                .withUsername(username)
                .withClientId("for Security")
                .build();
        return customUserDetails;
    }

    private Map<String, String> getUserServicePostObject(String username, String password) {
        Map<String, String> requestParam = new HashMap<String, String>();
        requestParam.put("userName", username);
        requestParam.put("password", password);
        return requestParam;
    }

    //模拟调用user服务的方法
    private Map checkUsernameAndPassword(Map map) {

        //checkUsernameAndPassword
        Map ret = new HashMap();
        ret.put("userId", UUID.randomUUID().toString());

        return ret;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }

}
```

在`AuthenticationManagerConfig`添加`CustomSecurityAuthenticationProvider`配置：

```java
@Configuration
public class AuthenticationManagerConfig extends GlobalAuthenticationConfigurerAdapter {

    @Autowired
    CustomAuthenticationProvider customAuthenticationProvider;
    @Autowired
    CustomSecurityAuthenticationProvider securityAuthenticationProvider;

    @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(customAuthenticationProvider)
                .authenticationProvider(securityAuthenticationProvider);
    }

}
```
保证数据库中的请求客户端存在授权码的请求授权和具备回调地址，回调地址是用来接受授权码的。

![](http://ovcjgn2x0.bkt.clouddn.com/authority1.png)

### 测试使用
启动服务，浏览器访问地址`http://localhost:9091/oauth/authorize?response_type=code&client_id=frontend&
scope=all&redirect_uri=http://localhost:8080`。

重定向到登录界面，引导用户登录：

![](http://ovcjgn2x0.bkt.clouddn.com/authority2.png)

登录成功，授权客户端获取授权码。

![](http://ovcjgn2x0.bkt.clouddn.com/authority3.png)

授权之后，从回调地址中获取到授权码：

```json
http://localhost:8080/?code=7OglOJ
```

携带授权码获取对应的token：

![](http://ovcjgn2x0.bkt.clouddn.com/authority4.png)

![](http://ovcjgn2x0.bkt.clouddn.com/authority5.png)

## 源码详解

`AuthorizationServerTokenServices`是授权服务器中进行token操作的接口，提供了以下的三个接口：

```java
public interface AuthorizationServerTokenServices {

	// 生成与OAuth2认证绑定的access_token
	OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException;

	// 根据refresh_token刷新access_token
	OAuth2AccessToken refreshAccessToken(String refreshToken, TokenRequest tokenRequest)
			throws AuthenticationException;

	// 获取OAuth2认证的access_token，如果access_token存在的话
	OAuth2AccessToken getAccessToken(OAuth2Authentication authentication);

}

```
请注意，生成的token都是与授权的用户进行绑定的。


`AuthorizationServerTokenServices`接口的默认实现是`DefaultTokenServices`，注意token通过`TokenStore`进行保存管理。

生成token：

```java
//DefaultTokenServices
@Transactional
public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {
	// 从TokenStore获取access_token
	OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
	OAuth2RefreshToken refreshToken = null;
	if (existingAccessToken != null) {
		if (existingAccessToken.isExpired()) {
			// 如果access_token已经存在但是过期了
			// 删除对应的access_token和refresh_token
			if (existingAccessToken.getRefreshToken() != null) {
				refreshToken = existingAccessToken.getRefreshToken();
			  	tokenStore.removeRefreshToken(refreshToken);
			}
			tokenStore.removeAccessToken(existingAccessToken);
		}
		else {
			// 如果access_token已经存在并且没有过期
			// 重新保存一下防止authentication改变，并且返回该access_token
			tokenStore.storeAccessToken(existingAccessToken, authentication);
			return existingAccessToken;
		}
	}

	// 只有当refresh_token为null时，才重新创建一个新的refresh_token
	// 这样可以使持有过期access_token的客户端可以根据以前拿到refresh_token拿到重新创建的access_token
	// 因为创建的access_token需要绑定refresh_token
	if (refreshToken == null) {
		refreshToken = createRefreshToken(authentication);
	}else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
	 	// 如果refresh_token也有期限并且过期，重新创建
		ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
		if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
			refreshToken = createRefreshToken(authentication);
		}
	}
	// 绑定授权用户和refresh_token创建新的access_token
	OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
	// 将access_token与授权用户对应保存
	tokenStore.storeAccessToken(accessToken, authentication);
	// In case it was modified
	refreshToken = accessToken.getRefreshToken();
	if (refreshToken != null) {
		// 将refresh_token与授权用户对应保存
		tokenStore.storeRefreshToken(refreshToken, authentication);
	}
	return accessToken;
}
```

需要注意到，在创建token的过程中，会根据该授权用户去查询是否存在未过期的access_token，有就直接返回，没有的话才会重新创建新的access_token，同时也应该注意到是先创建refresh_token，再去创建access_token，这是为了防止持有过期的access_token能够通过refresh_token重新获得access_token，因为前后创建access_token绑定了同一个refresh_token。

`DefaultTokenServices`中刷新token的`refreshAccessToken()`以及获取token的`getAccessToken()`方法就留给读者们自己去查看，在此不介绍。

## 小结
 本文主要讲了授权码模式，在授权码模式需要用户登录之后进行授权才获取获取授权码，再携带授权码去向`TokenEndpoint`请求访问令牌，当然也可以在请求中设置`response_token=token`通过隐式类型直接获取到access_token。这里需要注意一个问题，在到达`AuthorizationEndpoint`端点时，并没有对客户端进行验证，但是必须要经过用户认证的请求才能被接受。

### 推荐阅读
[系列文章：认证鉴权与API权限控制在微服务架构中的设计与实现](http://blueskykong.com/categories/Security/) 
## 参考
[spring-security](https://docs.spring.io/spring-security/site/docs/5.0.3.RELEASE/reference/htmlsingle/)
 

