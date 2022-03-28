---
title: Spring Security Oauth
date: 2022-03-28 09:43:12
categories: Spring Cloud
tags: 
---

<a href="{% post_path 'Spring/Spring Security' %}">Spring Security</a> 是一个强大的可高度定制的认证和授权框架，对于Spring应用来说它是一套Web安全标准。SpringSecurity注重于为Java应用提供认证和授权功能，像所有的Spring项目一样，它对自定义需求具有强大的扩展性。

如果要做一个认证登录模块, 那么我们就无法绕过JWT的概念. **JWT** 是将用户登录信息存储在了用户电脑中, 而不再是服务器里. 如果项目中用户量庞大. 并且分为电脑端和移动端. 那么 JWT 是必不可少的.

但是在企业级项目中，我们更倾向于<a href="{% post_path 'Distributed System/Oauth2' %}">Oauth2</a>的使用。他保证了我们内部系统的登陆认证，也保证了外部系统的访问控制。

## **授权模式**

### Authorization Mode

---

- **Authorization Code**<a href="{% post_path 'Distributed System/Oauth2' %}#授权码">授权码</a>：正宗的OAuth2的授权模式，客户端先将用户导向认证服务器，登录后获取授权码，然后进行授权，最后根据授权码获取访问令牌；
- Implicit (隐式授权模式) ：适用于无应用服务器的项目，和授权码模式相比，取消了获取授权码的过程，直接获取访问令牌；
- **Resource Owner Password Credentials<a href="{% post_path 'Distributed System/Oauth2' %}#密码式">密码式</a>：客户端直接向用户获取用户名和密码，之后向认证服务器获取访问令牌；
- Client Credentials (客户端模式) ：客户端直接通过客户端认证 (比如client_id和client_secret) 从认证服务器获取访问令牌。

## **授权服务配置适配器**

### AuthorizationServerConfigurerAdapter

---

- configure(ClientDetailsServiceConfigurer clients): 配置客户端的授权模式 , 客户端ID , 权限范围 , 权限类型 , token有效期等.
- configure(AuthorizationServerEndpointsConfigurer endpoints): 配置具体的授权模式规则 , 如JWT 密码模式

## **过滤器链**

### WebSecurityConfigurerAdapter(ResourceServerConfigurerAdapter)

---

- WebSecurityConfigurerAdapter 是 Spring security默认情况下的http配置，而 ResourceServerConfigurerAdapter 是Spring security oauth2 默认情况下的http配置。ResourceServerConfigurerAdapter 的优先级更高一些
- **同样的资源地址，只会在优先级最高的 ResourceServerConfigurerAdapter 中生效，如果 WebSecurityConfigurerAdapter 也配置了这个资源地址，则不会生效**
	- configure(HttpSecurity httpSecurity)：用于配置需要拦截的url路径、jwt过滤器及出异常后的处理器；
	- configure(AuthenticationManagerBuilder auth)：用于配置UserDetailsService及PasswordEncoder；
	- RestfulAccessDeniedHandler ：当用户没有访问权限时的处理器，用于返回JSON格式的处理结果
	- RestAuthenticationEntryPoint ：当未登录或token失效时，返回JSON格式的结果
	- UserDetailsService:SpringSecurity定义的核心接口，用于根据用户名获取用户信息，需要自行实现；
	- UserDetails：SpringSecurity定义用于封装用户信息的类（主要是用户信息和权限），需要自行实现；
	- PasswordEncoder：SpringSecurity定义的用于对密码进行编码及比对的接口，目前使用的是BCryptPasswordEncoder；
	- JwtAuthenticationTokenFilter：在用户名和密码校验前添加的过滤器，如果有jwt的token，会自行根据token信息进行登录。

## **过滤链**

### SecurityWebFilterChain

---

- SpringSecurityFilterChain 作为 SpringSecurity 的核心过滤器链在整个认证授权过程中起着举足轻重的地位，每个请求到来，都会经过该过滤器链
- 如果你想自己实现每一个自定义的未授权 , 未认证 , 白名单等策略 , 可以定制该过滤器链

## **授权管理器**

### ReactiveAuthorizationManager

---

- WebFlux授权管理器 , 可以当成是拦截器的一种 , 重写方法对授权做自定义处理 , 比如白名单 , 跨域等.
- **因为Spring Cloud Gateway使用是基于WebFlux与Netty开发的，所以与传统的Servlet方式不同**

## 密码式登陆

Token相关的接口都应该采用Basic base64的请求头，业务相关的接口都应该采用Bearer token的请求头

### **认证逻辑**

```Java
**# 在实际的开发中，我们不一定需要Basic的方式来对client进行解密验证方式，这只是pig自己的方式。但是的确蛮好用**
**# 请求头是Authorization=Basic base64.encode(client_id:client_secret) **

// 用户填写账户密码，发送登录请求，后台登录 API 主动发起获取/oauth/token的请求。

// 我们可以在中间加入网关拦截，验证base64的解密后格式是否正确
ValidateCodeGatewayFilter.apply()

/**
* 过滤器拦住Basic请求
* Spring让他默认拦截/oauth/token
*/
BasicAuthenticationFilter{
  void doFilterInternal(){
    // 所以再进入BasicAuthenticationFilter过滤器中，将client_id与client_secret解析，变成UsernamePasswordAuthenticationToken对象
    UsernamePasswordAuthenticationToken authRequest = this.authenticationConverter.convert(request);
    
    // 再通过一系列调用去找数据库存的客户端信息和传入的客户端信息比较，如果client_id或client_secret是错的，就报错。
    Authentication authResult = this.authenticationManager.authenticate(authRequest);
    
    ProviderManager.authenticate()
    →AbstractUserDetailsAuthenticationProvider.authenticate()
    →AbstractUserDetailsAuthenticationProvider.retrieveUser()
    →DaoAuthenticationProvider.retrieveUser()
    →ClientDetailsUserDetailsService.loadUserByUsername()
    →配置的PigClientDetailsService.loadClientByClientId()
  }
}

/**
* 专门拦截/oauth/token请求
*/
ClientCredentialsTokenEndpointFilter extends AbstractAuthenticationProcessingFilter{
  
  Authentication attemptAuthentication(){
    // 根据clientId和clientSecret最终会去查询client信息
    this.getAuthenticationManager().authenticate(authRequest);
    -> DaoAuthenticationProvider.retrieveUser()
    -> ClientDetailsUserDetailsService.loadUserByUsername();
  }
}

/**
* oauth2.0 token请求接口
*/
TokenEndPoint{
  ResponseEntity<OAuth2AccessToken> postAccessToken(){
    // 获取client信息，验证请求，授权码和密码式都会进来
    
    // 做一些验证操作，内部会开始创建OAuth2AccessToken对象
    OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
    
    AbstractTokenGranter{
      OAuth2AccessToken grant(){
        // 通过 DefaultTokenServices.createAccessToken() 从tokenStore缓存实现PigCustomTokenServices类中创建OAuth2AccessToken对象，还可以用增强器tokenEnhancer加入附加信息，比如用户对象。
        // 存入缓存，返回对象
        tokenServices.createAccessToken(getOAuth2Authentication(client, tokenRequest))
      }
    }
    // 一系列操作结束后，返回给请求方token对象
  }
}
```


### **正常访问**

```Java
**# 请求头需要加入Authorization=Bearer token的模式供服务端验证**

*// 网关不做鉴权拦截，只转发，具体的鉴权操作交由对应的业务微服务处理。只要引入Security依赖，并读取Redis中的Token即可*

*// 区别于Auth微服务的AuthorizationServerTokenServices(PigCustomTokenServices)
// 我们需要一个通用的ResourceServerTokenServices（PigLocalResourceServerTokenServices）来处理当前业务微服务的令牌认证问题*

/**
* 请求被拦截，开始校验并转换成认证对象，给调用链使用
* feign调用也进来这，所以我们需要在feign调用的时候，把请求头带上
*/
OAuth2AuthenticationProcessingFilter{
  void doFilter(){
    // 从当前线程中抽取请求头的Token值
    // 这一步在feign调用的时候很正常，用于token传递
    Authentication authentication = tokenExtractor.extract(request);
    if (authentication == null) {
      // 输出点日志
    }else {
      // 很长的调用，主要是通过token获取认证对象
      Authentication authResult = authenticationManager.authenticate(authentication);
      -> OAuth2AuthenticationManager.authenticate(authentication)
      -> ResourceServerTokenServices.loadAuthentication(accessToken)
      -> PigLocalResourceServerTokenServices(自定义).loadAuthentication(accessToken)
      
      // 存入线程
      SecurityContextHolder.getContext().setAuthentication(authResult);
    }
  }
}

/**
* 权限拦截器,方法前拦截
*/
FilterSecurityInterceptor extends AbstractSecurityInterceptor{
}
```


### Token续期

```Java
**# 前端可以定时调用/oauth/check_token端点检测Token是否失效过期**

CheckTokenEndpoint{
  checkToken(@RequestParam("token") String value) {
    resourceServerTokenServices.loadAuthentication(token.getValue());
  }
}

**# 若即将失效，则刷新令牌/oauth/token?grant_type=refresh_token**
**# 如果想要用户请求时对令牌续期，需要在tokenServices#loadAuthentication中进行重写**
TokenEndPoint{
  ResponseEntity<OAuth2AccessToken> postAccessToken(){
  }
}
```


---

## 授权码登陆

### **认证逻辑**

```Java
**# 如果是内部平台做Oauth2的逻辑，那就先登陆，再跳转到authorize接口**

/**
* 授权码模式专用授权请求
* oauth/authorize?
* response_type=code&
* client_id=CLIENT_ID&
* redirect_uri=CALLBACK_URL&
* scope=read
*/

@SessionAttributes
AuthorizationEndpoint{
  // 最终返回confirm页面，供用于确认授权
  // 可以自定义该页面
  ModelAndView authorize(Principal principal){
    // 因为类加入了Session注解，所以将model内容存储在了Session中
    // 后续confirm页面提交时，也会拿到这次的内容
    // 如果redis已经存在该clientId+userName的key，则省略confirm确认，直接返回重定向
  }
}

**# 如果是第三方调用的话，会在访问authorize接口的时候，因未登录被拦截器重定向到登陆页面，先登陆，然后再返回到authorize接口**

/**
* confirm页面请求通过，并携带了**user_oauth_approval**参数
*/
AuthorizationEndpoint{
  // 最终返回重定向地址，并拼接code参数传回
  View approveOrDeny(@RequestParam Map<String, String> approvalParameters, Map<String, ?> model, Principal principal){
    // 获取从authorize方法存储进session的内容
    // 进行一系列验证后，返回重定向地址+code
    // code默认放进ConcurrentHashMap里面，只要不用，就会一直存在
  }
}

/**
* 专门拦截/oauth/token请求
*/
ClientCredentialsTokenEndpointFilter extends AbstractAuthenticationProcessingFilter{
  
  Authentication attemptAuthentication(){
    // 根据clientId和clientSecret最终会去查询client信息
    this.getAuthenticationManager().authenticate(authRequest);
    -> DaoAuthenticationProvider.retrieveUser()
    -> ClientDetailsUserDetailsService.loadUserByUsername(username);
    -> ClientDetailsService.loadClientByClientId(username);
  }
}

/**
* oauth/token?
* client_id=CLIENT_ID&
* client_secret=CLIENT_SECRET&
* grant_type=authorization_code&
* code=AUTHORIZATION_CODE&
* redirect_uri=CALLBACK_URL
*/
TokenEndPoint{

  ResponseEntity<OAuth2AccessToken> postAccessToken(){
    OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
    -> AuthorizationServerEndpointsConfigurer.grant()
    -> CompositeTokenGranter.grant()
    // 删除之前的code并返回对应的认证对象，与token值绑定
    -> AuthorizationCodeTokenGranter.grant().getOAuth2Authentication(client, tokenRequest)
    // 令牌增强器也在这
    -> PigCustomTokenServices.createAccessToken(oAuth2Authentication)
    // 一系列操作结束后，返回给请求方token对象
  }
}
```


### 正常访问

与密码式相同

### Token失效

与密码式相同

---

## SSO

<a href="{% post_path 'Distributed System/SSO单点登录' %}">SSO单点登录</a>

以Pig项目为例，Auth服务配置了「Spring Security」「Spring Security Oauth2密码模式」「Spring Security Oauth2授权码模式」三种认证方式。

简单的Spring Security配置是为了在SSO单点登录时与Session结合，让Oauth2授权码模式能够记住登录状态。

Spring Security Oauth2密码模式是为了给项目内微服务认证使用。

---

## 拓展相关🌟🌟

### BearerTokenExtractor

该类是通过`OAuth2AuthenticationProcessingFilter`过滤器默认调用，专门处理用户访问请求，剥离Header中的`access_token`来确认用户令牌。我们可以实现该类，采用别的名称来替换`access_token`。比如掘金的就是`X-Legacy-Token`而非必须是 `Authorization`

也可以参考pig项目，增加注解AOP实现，对部分接口实现白名单策略，分别解决外部访问以及feign调用的问题。

Inner注解的使用 [https://www.yuque.com/pig4cloud/pig/hz5ppn](https://www.yuque.com/pig4cloud/pig/hz5ppn)

### OAuth2FeignRequestInterceptor

是为了解决Spring服务中Feign调用时，无法将Token通过Header传递给下游服务的问题。

该类需要程序员实现子类，在Feign调用前从线程中获取authentication对象并传递给父类做内部持有。主要方法是`accessTokenContextRelay.copyToken()`。

然后`OAuth2FeignRequestInterceptor`会根据内部持有的对象对下游服务发起携带Header的请求。下游服务可以从Header中获取Token值，并转换为authentication对象。

### Reference

Oauth2.0的数据库说明 [https://andaily.com/spring-oauth-server/db_table_description.html](https://andaily.com/spring-oauth-server/db_table_description.html)

Spring Oauth流程说明 [https://www.yuque.com/pig4cloud/pig/dqnyuc](https://www.yuque.com/pig4cloud/pig/dqnyuc)

分布式系统下的认证与授权 [https://www.bmpi.dev/dev/authentication-and-authorization-in-a-distributed-system/](https://www.bmpi.dev/dev/authentication-and-authorization-in-a-distributed-system/)

---



