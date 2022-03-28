---
title: Oauth2
date: 2022-03-28 10:43:12
excerpt: 简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。
categories: Distributed System
tags: 
typora-root-url: ../../
---

## 概念

------

简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。

令牌（token）与密码（password）的作用是一样的，都可以进入系统，但是有三点差异。

1. 令牌是短期的，到期会自动失效，用户自己无法修改。密码一般长期有效，用户不修改，就不会发生变化。
2. 令牌可以被数据所有者撤销，会立即失效。
3. 令牌有权限范围（scope），对于网络服务来说，只读令牌就比读写令牌更安全。密码一般是完整权限。

## 角色

------

- Resource owner (资源拥有者) ：拥有该资源的最终用户，他有访问资源的账号密码（微信用户）
- Resource server (资源服务器) ：拥有受保护资源的服务器，如果请求包含正确的访问令牌，可以访问资源（微信服务器）
- Client (客户端) ：访问资源的客户端，会使用访问令牌去获取资源服务器的资源，可以是浏览器、移动设备或者服务器（戒毒网站）
- Authentication server (认证服务器) ：用于认证用户的服务器，如果客户端认证通过，发放访问资源服务器的令牌（微信认证服务）

## 授权模式

### 授权码

#### 答疑

- **会不会有其他应用冒充第三方应用骗取授权？** ClientID 代表一个第三方应用的“用户名”，这项信息是可以完全公开的。但 ClientSecret 应当只有应用自己才知道，这个代表了第三方应用的“密码”。在第 5 步发放令牌时，调用者必须能够提供 ClientSecret 才能成功完成。只要第三方应用妥善保管好 ClientSecret，就没有人能够冒充它。
- **为什么要先发放授权码，再用授权码换令牌？** 这是因为客户端转向（通常就是一次 HTTP 302 重定向）对于用户是可见的，换而言之，授权码可能会暴露给用户以及用户机器上的其他程序，但由于用户并没有 ClientSecret，光有授权码也是无法换取到令牌的，所以避免了令牌在传输转向过程中被泄漏的风险。
- **为什么要设计一个时限较长的刷新令牌和时限较短的访问令牌？不能直接把访问令牌的时间调长吗？** 这是为了缓解 OAuth2 在实际应用中的一个主要缺陷，通常访问令牌一旦发放，除非超过了令牌中的有效期，否则很难（需要付出较大代价）有其他方式让它失效，所以访问令牌的时效性一般设计的比较短，譬如几个小时，如果还需要继续用，那就定期用刷新令牌去更新，授权服务器就可以在更新过程中决定是否还要继续给予授权。

授权码（authorization code）方式，指的是第三方应用先申请一个授权码，然后再用该码获取令牌。这种方式是最常用的流程，安全性也最高，它适用于那些有后端的 Web 应用。

1. A 网站提供一个链接，用户点击后就会跳转到 B 网站，授权用户数据给 A 网站使用。下面就是 A 网站跳转 B 网站的一个示意链接。 ***在这一步请求时, 如果检测到scope参数, 一般会弹出具体的授权页面, 用户确定授权才会往下走, 这也是为什么需要authorize, 而不是直接请求token的原因***

```Java
// response_type参数表示要求返回授权码（code）
// client_id参数让 B 知道是谁在请求, 一个应用一个id
// redirect_uri参数是 B 接受或拒绝请求后的跳转网址

**// scope：参数表示要求的授权范围（这里是只读）, 也可以不传
// scope：如果传了, 服务端需要检查去数据库该应用的scope是否与之吻合, 不符合直接返回错误信息
// scope：如果不传, 则表示静默授权, 不返回任何权限scope, 只表明登录了, 所以也不能获取任何信息, 当然你也可以去查询该用户的所有scope, 后续返回给他**
**// secret：一开始不传是因为后面有重定向，重定向是可感知的，有可能被其他网站拿到**
https://b.com/oauth/authorize?
  response_type=code&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
```

1. 用户跳转后，B 网站会要求用户登录，然后询问是否同意给予 A 网站授权。用户表示同意，这时 B 网站就会跳回redirect_uri参数指定的网址。跳转时，会传回一个授权码，就像下面这样。

```Java
// code参数就是授权码, 需要设置过期时间, 可以放在redis里面
// 正常使用一次以后, 就要被销毁, 不允许第二次使用
https://a.com/callback?code=AUTHORIZATION_CODE
```

1. A 网站拿到授权码以后，就可以在后端，向 B 网站请求令牌。

```Java
**// client_id参数让 B 知道是谁在请求, 一个应用一个id
// client_secret参数用来让B确认A的身份, 是应用的保密秘钥, 可以让服务器分发一个加密秘钥, 然后服务器有解密的算法**

**// 有些应用在让第三方请求时, 会有一个前置接口, 让第三方公司输入应用名称等信息, 然后返回给client_id和client_secret**
**// 如果是自己的应用, 那么自己约定好这两个信息, 让客户端及服务端都能知晓就可以了**

// grant_type参数的值是AUTHORIZATION_CODE, 表示采用的授权方式是授权码
// code参数是上一步拿到的授权码
// redirect_uri参数是令牌颁发后的回调网址
https://b.com/oauth/token?
 client_id=CLIENT_ID&
 client_secret=CLIENT_SECRET&
 grant_type=authorization_code&
 code=AUTHORIZATION_CODE&
 redirect_uri=CALLBACK_URL
```

1. B 网站收到请求以后，就会颁发令牌。具体做法是向redirect_uri指定的网址，发送一段 JSON 数据。

```JSON
{    
  "access_token":"ONWVJzfOv1XidEdeBAdqPxYGL60toIzhzHRwp1ySPHMEKWFbp5FIp3FuaQ3g",
  "token_type":"bearer",
  "expires_in":2592000,
  "refresh_token":"REFRESH_TOKEN",
  "scope":"read", // Set集合
  "uid":100101,
  "info":{...}
}
// 或者
{
  "access_token":"ONWVJzfOv1XidEdeBAdqPxYGL60toIzhzHRwp1ySPHMEKWFbp5FIp3FuaQ3g",
  "refresh_token":"SshGOiUl4w9UIePixwQzvXjlnddMi52xgviM8S4RPm26m1bBgJInpSXC4AND",
  "expires_in":7199,
  "refresh_expires_in":2591999, // 刷新令牌的失效时间
  "client_id":1001,
  "uid":10001,
  "scope":"", // Set集合
  "openid":"gr_SwoIN0MC1ewxHX_vfCW3BothWDZMMtx__" // JWT
}
```

### 密码式

如果你高度信任某个应用，**RFC 6749** 也允许用户把用户名和密码，直接告诉该应用。该应用就使用你的密码，申请令牌，这种方式称为"密码式"（password）。

1. A 网站要求用户提供 B 网站的用户名和密码。拿到以后，A 就直接向 B 请求令牌。

```Java
// client_id参数让 B 知道是谁在请求, 一个应用一个id
// client_secret参数用来让B确认A的身份, 是应用的保密秘钥, 自己配一个
// grant_type参数的值是password, 表示采用的授权方式是密码式
// username和password是 B 的用户名和密码。
https://oauth.b.com/oauth/token?
  grant_type=password&
  username=USERNAME&
  password=PASSWORD&
  client_secret=CLIENT_SECRET&
  client_id=CLIENT_ID&
  scope=scope
```

1. B 网站验证身份通过后，直接给出令牌。注意，这时不需要跳转，而是把令牌放在 JSON 数据里面，作为 HTTP 回应，A 因此拿到令牌。

这种方式需要用户给出自己的用户名/密码，显然风险很大，因此只适用于其他授权方式都无法采用的情况，而且必须是用户高度信任的应用。

### **隐藏式**

有些 Web 应用是纯前端应用，没有后端。这时就不能用上面的方式了，必须将令牌储存在前端。**RFC 6749 **就规定了第二种方式，允许直接向前端颁发令牌。这种方式没有授权码这个中间步骤，所以称为（授权码）"隐藏式"（implicit）。

1. A 网站提供一个链接，要求用户跳转到 B 网站，授权用户数据给 A 网站使用。

```Http
https://b.com/oauth/authorize?
  response_type=token&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
上面 URL 中，`response_type`参数为`token`，表示要求直接返回令牌。
```

1. 用户跳转到 B 网站，登录后同意给予 A 网站授权。这时，B 网站就会跳回redirect_uri参数指定的跳转网址，并且把令牌作为 URL 参数，传给 A 网站。

```Http
https://a.com/callback#token=ACCESS_TOKEN
上面 URL 中，`token`参数就是令牌，A 网站因此直接在前端拿到令牌。
```

### **凭证式**

最后一种方式是凭证式（client credentials），适用于没有前端的命令行应用，即在命令行下请求令牌。

1. A 应用在命令行向 B 发出请求。

```Http
https://oauth.b.com/token?
  grant_type=client_credentials&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
上面 URL 中，`grant_type`参数等于`client_credentials`表示采用凭证式，`client_id`和`client_secret`用来让 B 确认 A 的身份。
```

1. B 网站验证通过以后，直接返回令牌。

   这种方式给出的令牌，是针对第三方应用的，而不是针对用户的，即有可能多个用户共享同一个令牌。

## 令牌的使用

------

A 网站拿到令牌以后，就可以向 B 网站的 API 请求数据了。

此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个Authorization字段，令牌就放在这个字段里面。

```Bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
"https://api.b.com"
```

上面命令中，ACCESS_TOKEN就是拿到的令牌。

## 更新令牌

------

令牌的有效期到了，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。OAuth 2.0 允许用户自动更新令牌。

具体方法是，B 网站颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据，另一个用于获取新的令牌（refresh token 字段）。令牌到期前，用户使用 refresh token 发一个请求，去更新令牌。

B 网站验证通过以后，就会颁发新的令牌。

```Java
// grant_type参数为refresh_token表示要求更新令牌
// client_id参数和client_secret参数用于确认身份
// refresh_token参数就是用于更新令牌的令牌。
https://b.com/oauth/token?
  grant_type=refresh_token&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET&
  refresh_token=REFRESH_TOKEN
```

## 第三方登录原理

------

所谓第三方登录，实质就是 OAuth 授权。用户想要登录 A 网站，A 网站让用户提供第三方网站的数据，证明自己的身份。获取第三方网站的身份数据，就需要 OAuth 授权。

举例来说，A 网站允许 GitHub 登录，背后就是下面的流程。

```text
**Authorize**
1. A 网站让用户跳转到 GitHub。

2. GitHub 要求用户登录，然后询问"A 网站要求获得 xx 权限，你是否同意？"

3. 用户同意，GitHub 就会重定向回 A 网站，同时发回一个授权码。

**Token**
4. A 网站使用授权码，向 GitHub 请求令牌。

5. GitHub 返回令牌.

6. A 网站使用令牌，向 GitHub 请求用户数据。
```

### 应用登记

一个应用要求 OAuth 授权，必须先到对方网站登记，让对方知道是谁在请求。

所以，你要先去 GitHub 登记一下。当然，我已经登记过了，你使用我的登记信息也可以，但为了完整走一遍流程，还是建议大家自己登记。这是免费的。

访问这个网址，填写登记表。

![img](/image/Oauth2/WX20220328-114828@2x.png)

应用的名称随便填，主页 URL 填写`http://localhost:8080`。

跳转网址填写 `http://localhost:8080/oauth/redirect`。

提交表单以后，GitHub 应该会返回客户端 ID（client ID）和客户端密钥（client secret），这就是应用的身份识别码。

### 跳转授权

在你自己搭建的Web网站中，对Github进行授权访问。

```Http
https://github.com/login/oauth/authorize?
  client_id=7e015d8ce32370079895&
  redirect_uri=http://localhost:8080/oauth/redirect
```

这个 URL 指向 GitHub 的 OAuth 授权网址，带有两个参数：`client_id`告诉 GitHub 谁在请求，`redirect_uri`是稍后跳转回来的网址。

用户点击到了 GitHub，GitHub 会要求用户登录，确保是本人在操作。

### 获取授权码

登录后，GitHub 询问用户，该应用正在请求数据，你是否同意授权。

![img](/image/Oauth2/WX20220328-114914@2x.png)

用户同意授权， GitHub 就会跳转到`redirect_uri`指定的跳转网址，并且带上授权码，跳转回来的 URL 就是下面的样子。

```Http
http://localhost:8080/oauth/redirect?
  code=859310e7cecc9196f4af
```

后端收到这个请求以后，就拿到了授权码（code参数）。

### 获取令牌

后端使用这个授权码，向 GitHub 请求令牌。

```Http
https://github.com/login/oauth/access_token?
    client_id=${clientID}&
    client_secret=${clientSecret}&
    code=${requestToken}
```

上面代码中，GitHub 的令牌接口`https://github.com/login/oauth/access_token`需要提供三个参数。

```Http
client_id：客户端的 ID

client_secret：客户端的密钥

code：授权码
```

作为回应，GitHub 会返回一段 JSON 数据，里面包含了令牌accessToken。

### API数据

有了令牌以后，就可以向 API 请求数据了。

```Http
https://api.github.com/user,
  headers: {
    accept: 'application/json',
    Authorization: 'Bearer ${accessToken}'
  }
```

上面代码中，GitHub API 的地址是`https://api.github.com/user`，请求的时候必须在 HTTP 头信息里面带上令牌`Authorization: token 361507da`。

然后，就可以拿到用户数据，得到用户的身份。