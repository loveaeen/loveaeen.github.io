---
title: GateWay
date: 2022-03-28 09:43:12
excerpt: Spring Cloud Gateway 旨在提供一种简单而有效的方式来路由到 API，并为它们提供横切关注点。
categories: Spring Cloud
tags: 
---

### 过滤器

#### GlobalFilter

全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过GatewayFilterAdapter包装成GatewayFilterChain可识别的过滤器，它为请求业务以及路由的URI转换为真实业务服务的请求地址的核心过滤器，不需要配置，系统初始化时加载，并作用在每个路由上。

- **可以实现此过滤器来完成鉴权功能, 比如ruoyi-cloud项目的AuthFilter**
	```Java
	/**
	 * 网关鉴权
	 * 
	 * @author ruoyi
	 */
	@Component
	public class AuthFilter implements GlobalFilter, Ordered
	{
	    private static final Logger log = LoggerFactory.getLogger(AuthFilter.class);
	
	    // 排除过滤的 uri 地址，nacos自行添加
	    @Autowired
	    private IgnoreWhiteProperties ignoreWhite;
	
	    @Autowired
	    private RedisService redisService;
	
	
	    @Override
	    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)
	    {
	        ServerHttpRequest request = exchange.getRequest();
	        ServerHttpRequest.Builder mutate = request.mutate();
	
	        String url = request.getURI().getPath();
	        // 跳过不需要验证的路径
	        if (StringUtils.matches(url, ignoreWhite.getWhites()))
	        {
	            return chain.filter(exchange);
	        }
	        String token = getToken(request);
	        if (StringUtils.isEmpty(token))
	        {
	            return unauthorizedResponse(exchange, "令牌不能为空");
	        }
	        Claims claims = JwtUtils.parseToken(token);
	        if (claims == null)
	        {
	            return unauthorizedResponse(exchange, "令牌已过期或验证不正确！");
	        }
	        String userkey = JwtUtils.getUserKey(claims);
	        boolean islogin = redisService.hasKey(getTokenKey(userkey));
	        if (!islogin)
	        {
	            return unauthorizedResponse(exchange, "登录状态已过期");
	        }
	        String userid = JwtUtils.getUserId(claims);
	        String username = JwtUtils.getUserName(claims);
	        if (StringUtils.isEmpty(userid) || StringUtils.isEmpty(username))
	        {
	            return unauthorizedResponse(exchange, "令牌验证失败");
	        }
	
	        // 设置用户信息到请求
	        addHeader(mutate, SecurityConstants.USER_KEY, userkey);
	        addHeader(mutate, SecurityConstants.DETAILS_USER_ID, userid);
	        addHeader(mutate, SecurityConstants.DETAILS_USERNAME, username);
	        // 内部请求来源参数清除
	        removeHeader(mutate, SecurityConstants.FROM_SOURCE);
	        return chain.filter(exchange.mutate().request(mutate.build()).build());
	    }
	
	    private void addHeader(ServerHttpRequest.Builder mutate, String name, Object value)
	    {
	        if (value == null)
	        {
	            return;
	        }
	        String valueStr = value.toString();
	        String valueEncode = ServletUtils.urlEncode(valueStr);
	        mutate.header(name, valueEncode);
	    }
	
	    private void removeHeader(ServerHttpRequest.Builder mutate, String name)
	    {
	        mutate.headers(httpHeaders -> httpHeaders.remove(name)).build();
	    }
	
	    private Mono<Void> unauthorizedResponse(ServerWebExchange exchange, String msg)
	    {
	        log.error("[鉴权异常处理]请求路径:{}", exchange.getRequest().getPath());
	        return ServletUtils.webFluxResponseWriter(exchange.getResponse(), msg, HttpStatus.UNAUTHORIZED);
	    }
	
	    /**
	     * 获取缓存key
	     */
	    private String getTokenKey(String token)
	    {
	        return CacheConstants.LOGIN_TOKEN_KEY + token;
	    }
	
	    /**
	     * 获取请求token
	     */
	    private String getToken(ServerHttpRequest request)
	    {
	        String token = request.getHeaders().getFirst(TokenConstants.AUTHENTICATION);
	        // 如果前端设置了令牌前缀，则裁剪掉前缀
	        if (StringUtils.isNotEmpty(token) && token.startsWith(TokenConstants.PREFIX))
	        {
	            token = token.replaceFirst(TokenConstants.PREFIX, StringUtils.EMPTY);
	        }
	        return token;
	    }
	
	    @Override
	    public int getOrder()
	    {
	        return -200;
	    }
	}
	```
	

#### GatewayFilter

需要通过 `spring.cloud.routes.filters` 配置在**具体路由**下，只作用在当前路由上或通过spring.cloud.default-filters配置在全局，作用在所有路由上。

#### Hystrix GatewayFilter

Hystrix 过滤器允许你将断路器功能添加到**具体的网关路由**中，使你的服务免受级联故障的影响，并提供服务降级处理。

- 要开启断路器功能，我们需要在pom.xml中添加Hystrix的相关依赖：
	```xml
	<dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
	</dependency>
	```
	
- 然后添加相关服务降级的处理类：
	```java
	/**
	 * Created by macro on 2019/9/25.
	 */
	@RestController
	public class FallbackController {
	
	    @GetMapping("/fallback")
	    public Object fallback() {
	        Map<String,Object> result = new HashMap<>();
	        result.put("data",null);
	        result.put("message","Get request fallback!");
	        result.put("code",500);
	        return result;
	    }
	}
	```
	
- 在application-filter.yml中添加相关配置，当路由出错时会转发到服务降级处理的控制器上：
	```yaml
	spring:
	  cloud:
	    gateway:
	      routes:
	        - id: hystrix_route
	          uri: http://localhost:8201
	          predicates:
	            - Method=GET
	          filters:
	            - name: Hystrix
	              args:
	                name: fallbackcmd
	                fallbackUri: forward:/fallback
	```
	
- 关闭user-service，调用该地址进行测试：[http://localhost:9201/user/1](http://localhost:9201/user/1) ，发现已经返回了服务降级的处理信息。

![](http://www.macrozheng.com/images/springcloud_gateway_03.png "image")



### 异常拦截器

#### WebExceptionHandler

在该异常处理中, 可以对 `服务未找到` 或者`sentinel`的相关异常 (BlockException) 进行判断拦截, 并返回对应错误信息, 告知用户

具体可查看**ruoyi**项目的相关代码, 写的比较完全

```Java
/**
 * 网关统一异常处理
 *
 * @author ruoyi
 */
@Order(-1) //优先级一定要小于内置ResponseStatusExceptionHandler, 经过它处理的获取对应错误类的 响应码
@Configuration
public class GatewayExceptionHandler implements ErrorWebExceptionHandler
{
    private static final Logger log = LoggerFactory.getLogger(GatewayExceptionHandler.class);

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex)
    {
        ServerHttpResponse response = exchange.getResponse();

        if (exchange.getResponse().isCommitted())
        {
            return Mono.error(ex);
        }

        String msg;

        if (ex instanceof NotFoundException)
        {
            msg = "服务未找到";
        }
        else if (ex instanceof ResponseStatusException)
        {
            ResponseStatusException responseStatusException = (ResponseStatusException) ex;
            msg = responseStatusException.getMessage();
        }
        else
        {
            msg = "内部服务器错误";
        }

        log.error("[网关异常处理]请求路径:{},异常信息:{}", exchange.getRequest().getPath(), ex.getMessage());

        return ServletUtils.webFluxResponseWriter(response, msg);
    }
}

/**
 * 自定义限流异常处理
 *
 * @author ruoyi
 */
public class SentinelFallbackHandler implements WebExceptionHandler
{
    private Mono<Void> writeResponse(ServerResponse response, ServerWebExchange exchange)
    {
        return ServletUtils.webFluxResponseWriter(exchange.getResponse(), "请求超过最大数，请稍候再试");
    }

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex)
    {
        if (exchange.getResponse().isCommitted())
        {
            return Mono.error(ex);
        }
        if (!BlockException.isBlockException(ex))
        {
            return Mono.error(ex);
        }
        return handleBlockedRequest(exchange, ex).flatMap(response -> writeResponse(response, exchange));
    }

    private Mono<ServerResponse> handleBlockedRequest(ServerWebExchange exchange, Throwable throwable)
    {
        return GatewayCallbackManager.getBlockHandler().handleRequest(exchange, throwable);
    }
}

```



