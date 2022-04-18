---
title: Feign
date: 2022-03-28 09:43:12
excerpt: Feign是一个基于注解来生成HTTP请求，并且能自动处理请求返回的工具。
categories: Spring Cloud
hide: true
tags: 
---



SpringCloud feign 对Netflix feign进行了包装和增强，本文从源码角度梳理一个请求链路中参与的各个组件及其它们如何配合交互完成请求。组件包括feign、ribbon、hystrix、sleuth等

## 初始化

### @EnableFeignClients

该注解通过import方式，引入FeignClientsRegistrar。先注册默认配置，再注册所有的feignClient BeanDefinition。

扫描到所有@FeignClient的BeanDefinition后，包装成FeignClientFactoryBean，然后注册到spring上下文中。

### FeignClientFactoryBean

完成了调用前的所有准备工作，如FeignContext，Feign.builder的创建、判断是否需要负载均衡及设置及Feign的代理类的创建等。

Feign 的动态代理创建完成，并交由Spring容器管理。第二阶段的初始化阶段至此结束。http请求发生时，创建的代理对象最终聚合ribbon、hystrix、sleuth的功能完成整个调用链路的增强或跟踪。

## Log

- NONE：默认的，不显示任何日志；
- BASIC：仅记录请求方法、URL、响应状态码及执行时间；
- HEADERS：除了BASIC中定义的信息之外，还有请求和响应的头信息；
- FULL：除了HEADERS中定义的信息之外，还有请求和响应的正文及元数据。

```java
/**
 * Created by macro on 2019/9/5.
 */
@Configuration
public class FeignConfig {
  @Bean
  Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
  }
}

```


```yaml
logging:
  level:
  com.macro.cloud.service.UserService: debug

```


## Used Config

```yaml
feign:
  hystrix:
  enabled: true #在Feign中开启Hystrix
  compression:
  request:
    enabled: false #是否对请求进行GZIP压缩
    mime-types: text/xml,application/xml,application/json #指定压缩的请求数据类型
    min-request-size: 2048 #超过该大小的请求会被压缩
  response:
    enabled: false #是否对响应进行GZIP压缩
logging:
  level: #修改日志级别
  com.macro.cloud.service.UserService: debug

```


## 调用

在初始化的最后，创建了InvocationHandler，在请求发生时invoke()方法将被调用。如果引入了hystrix，那么就是调用HystrixInvocationHandler，该方法内部调用了HystrixCommand。

在run()方法中，调用对应的methodHandler的invoke()方法。在这里有获取URL，封装ribbon请求，负载均衡，并最终执行httpclient调用。

## 源码解读

[Feign 调用过程分析](https://www.jianshu.com/p/769081fe8d95)



