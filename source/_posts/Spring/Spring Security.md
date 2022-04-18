---
title: Spring Security
date: 2022-03-28 09:43:12
excerpt: Spring Security是一个强大的可高度定制的认证和授权框架
categories: Spring Cloud
hide: true
tags: 
---

## Session

### 创建

Spring Security没有像Shiro一样，创建一个自己掌控的Session出来， 而是默认交由Tomcat处理。虽然Security没有自己创建，但是给了我们可以实现的`SessionRegistry`，以及用于管理用户和Session的`SessionInformation`

`SessionManagementFilter`是专门用于管理Session状态的，如检测用户并发Session，失效Session的逻辑处理等

- expiredUrl：是用户并发会话过多，被挤下线后的重定向地址。
- invalidSessionUrl：Session超时的重定向地址

### 缓存

spring-session-data-redis依赖, 将Session进行分布式缓存

### 失效

Spring Security采用了事件广播在Session失效时，删除了对应的`SessionInformation`，我们可以在**`SessionRegistryImpl`**`#onApplicationEvent`进行深入的研究

### 登陆认证

```Java
/**
* 用户登录后, 交由 AbstractAuthenticationProcessingFilter 过滤器检测是否为登录地址
*/
AbstractAuthenticationProcessingFilter{
  void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    // 判断是否是配置好的验证登录请求
    if (!this.requiresAuthentication(request, response)) {
      chain.doFilter(request, response);
    } else {
      // 尝试身份验证
      authResult = this.attemptAuthentication(request, response);
    }
  }
  
}

/**
* 确定后默认再由 UsernamePasswordAuthenticationFilter进行用户获取认证
*/
UsernamePasswordAuthenticationFilter{
  Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
    return this.getAuthenticationManager().authenticate(authRequest);
  }
  
  // 然后 AbstractUserDetailsAuthenticationProvider#authenticate 
  // 会通过username和生成的Authentication对象生成一个UserDetails对象，并检查用户是否存在
  Authentication authenticate(Authentication authentication) throws AuthenticationException {
    String username = authentication.getPrincipal() == null ? "NONE_PROVIDED" : authentication.getName();
    // 去找缓存, 可以通过ehcache实现该类, 默认是NullUserCache空实现
    UserDetails user = this.userCache.getUserFromCache(username);
    if(user == null){
      // 没有缓存, 则通过DetailService的实现类去数据库查询, 也可以从缓存走
      user = this.retrieveUser(username, (UsernamePasswordAuthenticationToken)authentication){
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        return loadedUser;
      }
    }
  }
}

```


### 正常访问

```Java
/**
* 过滤器链的第二位
*/
SecurityContextPersistenceFilter{

  void doFilter(){
    // 获取session中的用户信息
    HttpSession session = request.getSession();
    HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request, response);
    SecurityContext contextBeforeChainExecution = this.repo.loadContext(holder);
    // 存入当前线程
    SecurityContextHolder.setContext(contextBeforeChainExecution);
  }
}

/**
* 异常过滤器
*/
ExceptionTranslationFilter{
  void doFilter(){
    // 后置过滤器，最终会走到FilterSecurityInterceptor
    try {
      chain.doFilter(request, response);
    }
    catch (Exception ex) {
      // 执行exceptionEntrypoint
      handleSpringSecurityException(request, response, chain, securityException);
    }
  }
}

/**
* 权限拦截器,方法前拦截
*/
FilterSecurityInterceptor extends AbstractSecurityInterceptor{
  
  void invoke(){
    // 前置, 检测访问请求是否是白名单，获取线程内认证对象并做权限校验，如@PreAuthorize
    super.beforeInvocation(filterInvocation);
    try {
      // 
      filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
    }
    finally {
      // 后置
      super.finallyInvocation(token);
    }
  }
}
```


### 用户缓存

`UserCache` 允许配置Ehcache缓存, 用于缓存 `UserDetails` 对象 , 在  CachingUserDetailsService 类中获取用户实体

感觉没啥用, 敲了断点发现只有登录的时候能用上该缓存.

## 鉴权管理

---

spring security的认证与鉴权过滤器都在`AbstractSecurityInterceptor`拦截器中. 其中鉴权部分交由`AbstractAccessDecisionManager`实现, 采用了投票机制进行鉴权.默认有三个实现类

- **AffirmativeBased(默认)** : 只要任一 AccessDecisionVoter 返回肯定的结果，便授予访问权限
- UnanimousBased : 只要任一 AccessDecisionVoter 返回失败的结果，便全体失败
- ConsensusBased : 少数服从多数授权访问决策方案

## 常用配置

---

```Java
http
  // 跨域
  .cors(AbstractHttpConfigurer::disable)
  // csrf攻击
  .csrf(AbstractHttpConfigurer::disable)
  // 完全放开frame页面
  .headers(h -> h.frameOptions().disable())
  // 异常处理配置 ExceptionTranslationFilter
  // 在过滤器
  .exceptionHandling(
          // 用来解决匿名用户访问无权限资源时的异常
          e->e.authenticationEntryPoint(customAuthenticationEntryPoint)
          // 用来解决认证过的用户访问无权限资源时的异常
                  .accessDeniedHandler(customAccessDeniedHandler)
  // session管理器
  ).sessionManagement(
  // 同一用户最大允许同时存在几个
  s->s.maximumSessions(10)
          // 管理所有用户登录的session
          .sessionRegistry(sessionRegistry())
          // session失效处理
          .expiredSessionStrategy(ssoSessionInformationExpiredStrategy)
  // 认证请求拦截策略, 以下都不拦截处理，但是会走过滤器
  // web.ignore的不拦截处理，是不走过滤器的，一般用于静态资源文件
).authorizeRequests(
  a-> a.antMatchers("/sso/*","/logout")
          .permitAll().anyRequest().authenticated()
  // 登陆失败的处理
).failureHandler(authenticationFailureHandler())
  // 退出登陆成功处理
.logoutSuccessHandler(logoutSuccessHandler())
  // 登录页面地址
.formLogin(
  f->f.loginPage(accessTokenUri)
        // 可以不配置,表示登录提交的地址,用于校验后端登录的url
        .loginProcessingUrl("/login")
          // 登录成功后回调
          .successHandler(customAuthenticationSuccessHandler)
// 在userfilter前执行validateFilter
).addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class)
// 在validateFilter前执行decryptFilter
.addFilterBefore(decryptParameterFilter, ValidateCodeFilter.class)
.apply(ssoAuthenticationSecurityConfig);

//禁用session管理
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);

/**
* ssoAuthenticationSecurityConfig extends SecurityConfigurerAdapter
* 可以在CAS中配置业务认证逻辑
*/

# **基于配置好的路径来进行认证的过滤器 AbstractAuthenticationProcessingFilter**
// 子类实现attemptAuthentication()来做逻辑处理
// 交由下面的AuthenticationManager来做认证处理

// 但是该抽象类中doFilter方法里有this.requiresAuthentication的调用, 里面是AntPathRequestMatcher类
// 该类默认记载了/login的url, 表示只有当用户请求地址与之相同时, 才会走下面的认证逻辑, 否则都会交由其他过滤器解决.
// 所以我们可以在子类构造中, 重写该地址, 将登陆后的请求地址放进去.*
AbstractAuthenticationProcessingFilter filter

// 用于处理身份验证的核心逻辑, 在CAS中可以根据authentication参数判断有没有被CAS中心认证过
// 最终需要返回认证对象
filter.setAuthenticationManager(providerManager);
// 认证成功后,Security会发布消息,用户可以订阅实现记录成功信息
providerManager.setAuthenticationEventPublisher();
// 业务模块认证通过后,可以根据savedRequest做跳转之类的
filter.setAuthenticationSuccessHandler(customAuthenticationSuccessHandler)
```


## JWT无状态

在认证的过程中，Spring Security会运行一个过滤器`SecurityContextPersistenceFilter`来存储请求的Security Context，这个上下文的存储是一个策略模式，但默认的是保存在HTTP Session中的`HttpSessionSecurityContextRepository`。现在我们设置了 create-session="stateless"，就会保存在`NullSecurityContextRepository`，里面没有任何session在上下文中保持。

既然没有为何还要调用这个空的filter？因为需要调用这个filter来保证每次请求完了`SecurityContextHolder`被清空了，下一次请求必须**re-authentication**。

### 登录认证

```Java
1. 可以继承UsernamePasswordAuthenticationFilter类，去通过UserDetailService获取用户。

2. 登录校验成功后由配置好的 http.successHandler(JwtLoginSuccessHandler)来返回Token给前端。
```


### 正常访问

```Java

// 禁用session管理
// 跳过 HttpSessionSecurityContextRepository, SessionManagementFilter, RequestCacheFilter. 
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
// 百度方法一般是继承该类
// 也有继承OncePerRequestFilter类的
JwtAuthorizationFilter extends BasicAuthenticationFilter
// 然后通过配置加入该过滤器
http.addFilter(new JwtAuthorizationFilter(authenticationManager()));

```


## 认证入口点AuthenticationEntryPoint

---

- 它在用户请求处理过程中遇到认证异常时，被`ExceptionTranslationFilter`或者`AuthenticationFailureHandler`用于开启特定认证方案的认证流程。
	- `Http403ForbiddenEntryPoint`: 设置响应状态字为403,并非触发一个真正的认证流程。通常在一个预验证(pre-authenticated authentication)已经得出结论需要拒绝用户请求的情况被用于拒绝用户请求。
	- `HttpStatusEntryPoint`: 设置特定的响应状态字，并非触发一个真正的认证流程。
	- `LoginUrlAuthenticationEntryPoint`: 根据配置计算出登录页面url,将用户重定向到该登录页面从而开始一个认证流程。
	- `BasicAuthenticationEntryPoint`: 对应标准Http Basic认证流程的触发动作，向响应写入状态字401和头部WWW-Authenticate:"Basic realm="xxx"触发标准Http Basic认证流程。
	- `DigestAuthenticationEntryPoint`: 对应标准Http Digest认证流程的触发动作，向响应写入状态字401和头部WWW-Authenticate:"Digest realm="xxx"触发标准Http Digest认证流程。
	- `DelegatingAuthenticationEntryPoint`: 这是一个代理，将认证任务委托给所代理的多个AuthenticationEntryPoint对象，其中一个被标记为缺省AuthenticationEntryPoint。

## **用户实体UserDetailService**

---

- Spring Security中进行身份验证的是 AuthenticationManager 接口，ProviderManager 是它的一个默认实现，但它并不用来处理身份认证，而是委托给配置好的 AuthenticationProvider ，每个 AuthenticationProvider 会轮流检查身份认证。检查后或者返回 Authentication 对象或者抛出异常。
- loadUserByUsername：通过用户名获取一个真实用户，返回一个 UserDetails 的实现类
- 密码验证由 AuthenticationProvider 的 DaoAuthenticationProvider#additionalAuthenticationChecks 处理，也可以自己定义
- UserDetail目前发现仅在登录时与Authentication做比较操作, 除此以外无他用
