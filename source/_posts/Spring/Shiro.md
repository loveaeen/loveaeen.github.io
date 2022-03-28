---
title: Shiro
date: 2022-03-28 09:43:12
excerpt: Apache Shiro 是 Java 的一个安全框架。
categories: Spring Cloud
tags: 
---

|过滤器简称|相对应的java类|
|---|---|
|anon|org.apache.shiro.web.filter.authc.AnonymousFilter|
|user|org.apache.shiro.web.filter.authc.UserFilter|
|logout|org.apache.shiro.web.filter.authc.LogoutFilter|
|perms|org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter|
|authc|org.apache.shiro.web.filter.authc.FormAuthenticationFilter|
|authcBasic|org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter|
|port|org.apache.shiro.web.filter.authz.PortFilter|
|rest|org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter|
|roles|org.apache.shiro.web.filter.authz.RolesAuthorizationFilter|
|ssl|org.apache.shiro.web.filter.authz.SslFilter|



## 认证

---

### Session

从**`AbstractShiroFilter`**`#doFilterInternal`进入

`DefaultSecurityManager #createSubject` 负责返回一个Subject

内部的`#resolveSession`方法是获取Session信息, 主要走SessionDao方法获取, 可从缓存或数据库读取, 默认走`AbstractSessionDao`类, 所以从缓存走

内部的`#resolvePrincipals`中从Session获取principals信息, 并在**`AbstractShiroFilter`**类里进行线程绑定

#### 创建

Shiro创建Session交由`SessionFactory`负责, 我们可以实现该类, 创建一个自定义的Session, 比如有用户的一些属性和操作系统浏览器的属性.

然后在过滤器触发时, 将登录后的用户信息存入到该Session中

#### 缓存

其实在业务上如果需要Redis来管理Session，那么直接继承**`AbstractSessionDao`**, 该类有3个实现, 但是只有2个是有用的, 一个是cache存取 , 一个是 ConcurrentHashMap存取

- 如果只是在单机环境下需要做Session持久化（服务器重启保持Session在线），那么最好继承`EnterpriseCacheSessionDAO`来增加本地Session缓存以减少I/O开销，否则大量的doReadSession()调用会造成I/O甚至网络压力（单机环境下能省一点是一点嘛，集群就没办法省了）；
- 如果是在集群环境下做Session共享，千万不要继承EnterpriseCacheSessionDAO，会产生服务器间的本地Session缓存不同步问题，直接继承AbstractSessionDAO即可；

#### 失效

**`AbstractShiroFilter`**` #doFilterInternal`过滤器每一次会触发更新Session的操作`updateSessionLastAccessTime`, 最终会触发`SessionDao.update()`, 将更新后的session存入缓存

Shiro默认会采用定时线程检测session是否失效, 具体可查看定时器类**`ExecutorServiceSessionValidationScheduler`** 与失效session管理类**`AbstractValidatingSessionManager`**

研发人员可实现这两个类, 自己实现相关session失效后的业务逻辑

### 认证拦截

从统一入口**`OncePerRequestFilter`**儿子类**`AdviceFilter`**`#doFilterInternal`进入, 如果状态为true, 则调用`executeChain`做后续filter责任链处理, 并可以自定义`postHandle`方法

再交由孙子类`**PathMatchingFilte**``#preHandle #isFilterChainContinued `

接着孙孙子类**`AccessControlFilter`**`#onPreHandle` 进行权限认证鉴权

当访问非anon拦截时, 会交由UserFilter处理认证逻辑, 若认证失败则转到登录页面.

主要是在`filterChainDefinitionMap.put("/**", "user,kickout,onlineSession,syncOnlineSession")`配置

## 鉴权

---

当我们访问带有Shiro权限注释的方法时, 会被`AnnotationsAuthorizingMethodInterceptor`拦截到, 其内部会通过循环对多个权限注释的handler处理类进行鉴权处理, 如果权限不对, 则报错

多个handler最终还是会交由`AuthorizingSecurityManager`来做精细化的权限处理.

而在更内部则会调用**`AuthorizingRealm`**`#getAuthorizationInfo`方法, 该方法内部先从Cache获取用户实体, 如果获取不到, 则调用自定义方法从数据库拿

### 缓存

在鉴权的时候, **`AuthorizingRealm`**`#getAuthorizationInfo`会经过缓存

