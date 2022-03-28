---
title: SSO单点登录
date: 2022-03-28 11:43:12
excerpt: 单点登录(SingleSignOn，SSO)，就是通过用户的一次性鉴别登录。
categories: Distributed System
tags: 
---

## 同域名方式

1. **使用共享Cookie来解决Token共享问题。**

   所谓共享Cookie，就是Cookie设置为顶域名，然后就可以在二级域名下的共享，举个例子：写在父域名stp.com下的Cookie，[在s1.stp.com](http://xn--s1-rl8c.stp.com)、s2.stp.com等子域名都是可以共享访问的。

2. **使用Redis来解决Session共享问题。**

   Cookie的问题解决了，我们再来看看session的问题。我们在sso系统登录了，这时再访问app1，Cookie也带到了app1的服务端（Server），app1的服务端怎么找到这个Cookie对应的Session呢？这里就要把 3 个系统的Session共享，采用Redis来实现。

### Shiro Session模式

Shiro标准模式是基于Session来做的，SessionID存储在Cookie里。也就意味着如果我们多个系统都采用Shiro框架认证。

那么当我们删除了Redis的Session后，所有系统都会全部注销。

### Sa-Token

用户登录成功，会创建多个Session对象（其实就是Map集合之类的）存储到Redis里面，再返回浏览器一个Token标识，后续请求服务器检验该Cookie中的Token。

该Token里面存储了用户信息，是否登录等。

用户访问其他系统时，网关拦截校验Cookie中的Token是否正确，是否登录，然后再交由后续登录方法给用户加权限信息。

当用户点击某系统的注销后，先删除Cookie，再删除Redis里的Session信息。

### JWT

JWT存入到Cookie里面，A系统首先登录，然后JWT存入到顶级域名的Cookie里面，B系统网关拦截后，重定向到SSO服务器，服务器获取Cookie是否存在并验证，成功后返回给B系统。

## 不同域的CAS方式

> Spring Security有与[CAS](https://github.com/apereo/cas)集成的便捷写法, 研发者只需要配置地址, 不需要关心如何统一认证.

CAS （Central Authentication Service）中央认证服务

1. A系统登录并重定向SSO 首先，用户想要访问系统A www.java3y.com受限的资源(比如说购物车功能，购物车功能需要登录后才能访问)，系统Awww.java3y.com发现用户并没有登录，于是重定向到sso认证中心，并将自己的地址作为参数。请求的地址如下：

   http://www.sso.com?service=www.java3y.com

2. SSO进行认证并设置SSO域的Token，并重定向给A系统 sso认证中心发现用户未登录，将用户引导至登录页面，用户进行输入用户名和密码进行登录，用户与认证中心建立全局会话（生成一份Token，写到Cookie中，保存在浏览器上）

   随后，认证中心重定向回系统A，并把Token携带过去给系统A，重定向的地址如下：

   http://www.java3y.com?token=xxxxxxx

3. A系统携带Token去SSO验证，并确认登录 接着，系统A去sso认证中心验证这个Token是否正确，如果正确，则系统A和用户建立局部会话（创建Session）。到此，系统A和用户已经是登录状态了。

4. 用户登录B系统，重定向到SSO，SSO检测Cookie里是否存在Token并验证 此时，用户想要访问系统Bwww.java4y.com受限的资源(比如说订单功能，订单功能需要登录后才能访问)，系统B www.java4y.com发现用户并没有登录，于是重定向到sso认证中心，并将自己的地址作为参数。请求的地址如下：

   http://www.sso.com?service=www.java4y.com

5. SSO发现用户已登录，将Token重定向B系统 注意，因为之前用户与认证中心www.sso.com已经建立了全局会话（当时已经把Cookie保存到浏览器上了），所以这次系统B重定向到认证中心www.sso.com是可以带上Cookie的。 认证中心根据带过来的Cookie发现已经与用户建立了全局会话了，认证中心重定向回系统B，并把Token携带过去给系统B，重定向的地址如下：

   http://www.java4y.com?token=xxxxxxx

6. B系统携带Token去SSO验证，并确认登录 接着，系统B去sso认证中心验证这个Token是否正确，如果正确，则系统B和用户建立局部会话（创建Session）。到此，系统B和用户已经是登录状态了。

### 票据模式

在CAS标准中，用户登录认证的操作如下。

1. 用户访问A系统，重定向SSO，并携带当前系统地址。
2. SSO进行登录，并生成 key = CASTGC, value = ${TGC}的Cookie，以及key = ${TGC}, value = ${TGT}的Session。
3. SSO分发ST给A系统，A系统携带ST去SSO验证，验证通过，A系统正常登录。
4. 用户访问B系统，重定向SSO
5. SSO发现Cookie中有CASTGC的Cookie，获取到TGC的值去Session查找TGT是否存在，找到的话就说明登录过了。
6. SSO分发ST给B系统，B系统携带ST去SSO验证，验证通过，B系统正常登录。

### 一次注销，全端下线

SSO在被重定向过来时，将业务系统的项目地址存入到用户相关的Set集合里。

当某个系统点击注销按钮，将该请求发送至SSO中，让SSO遍历Set集合对地址后追加注销字符串，进行逐一下线操作，然后再注销SSO。

### 本地服务

#### 认证

1. 用户访问/business/index , 因为没登录, 被配置的`authenticationEntryPoint`发送SSO登录请求重定向至/sso/code?redirectUrl
2. SSO服务的默认未认证拦截器拦截到该请求, 重定向至SSO登录页面.
3. 用户登录成功后, 交由研发人员实现SSO服务中的successHandler类进行处理, `authenticationDetailsSource.buildDetails(request)`设置details信息到Authentication里面, 记录日志等, 并根据savedRequest重定向到业务的index 或者 SSO的index
4. 再次访问/business/index , 还是因为没登录, 被配置的`authenticationEntryPoint`发送SSO登录请求重定向至/sso/code?redirectUrl
5. 这一次SSO通过已经存在的session发现认证了, 不再拦截, 进入code方法, 查询具体的客户端信息, 根据session中的用户信息与查询的客户端信息存入map, key=code, value=authentication, 再根据redirectUrl返回重定向至业务的post登录请求(/ssoLogin)
6. 这一次请求地址是/ssoLogin, 该地址是子类构造器中定义的, 所以被`AbstractAuthenticationProcessingFilter`过滤器匹配到. 根据code参数, new一个Authentication, 并authenticationDetailsSource.buildDetails(request)设置details信息到Authentication里面, 并创建一个当前session并传入. 检查code参数是否存在, 不存在就重新去获取code, 存在就进入自定义认证逻辑`SsoAuthenticationProvider`
7. 该认证方法中进行远程调用/sso/verify方法, 通过code参数从map中删除该key并获取authentication对象, 然后返回user信息对象, 里面包含SSO的sessionid, 将他也放入authentication对象里面, 然后返回authentication **在这一步中, 如果项目之间允许共享Redis, 则不需要将用户信息通过http的方式同步**
8. 进入业务的登录成功successHandler类进行处理,将SSO的sessionid与当前业务的sessionid绑定放入域中, 获取savedRequest并重定向

#### 登出

1. 用户调用当前业务的登出方法, 该方法重定向到SSO登出方法
2. SSO登出方法根据配置好的所有业务模块信息, 循环获取所有的登出地址, 传入SSO的sessionId并进行调用
3. 每一个业务登出方法处理都需要根据SSO的sessionid找到当前业务的sessionId, 然后注销.