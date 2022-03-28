---
title: Sentinel
date: 2022-03-28 09:43:12
excerpt: Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
categories: Spring Cloud
tags:
---

> 随着微服务的流行，服务和服务之间的稳定性变得越来越重要。 Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。


### Caption

- resource：资源名称；
- limitApp：来源应用；
- grade：阈值类型，0表示线程数，1表示QPS；
- count：单机阈值；
- strategy：流控模式，0表示直接，1表示关联，2表示链路；
- controlBehavior：流控效果，0表示快速失败，1表示Warm Up，2表示排队等待；
- clusterMode：是否集群。

Sentinel具有如下特性:

- 丰富的应用场景：承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀，可以实时熔断下游不可用应用；
- 完备的实时监控：同时提供实时的监控功能。可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况；
- 广泛的开源生态：提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合；
- 完善的 SPI 扩展点：提供简单易用、完善的 SPI 扩展点。您可以通过实现扩展点，快速的定制逻辑。

### 实际使用

```java
/**
 * 限流功能
 * Created by macro on 2019/11/7.
 */
@RestController
@RequestMapping("/rateLimit")
public class RateLimitController {

  /**
   * 按资源名称限流，需要指定限流处理逻辑
   */
  @GetMapping("/byResource")
  @SentinelResource(value = "byResource",blockHandler = "handleException")
  public CommonResult byResource() {
    return new CommonResult("按资源名称限流", 200);
  }

  /**
   * 按URL限流，有默认的限流处理逻辑
   */
  @GetMapping("/byUrl")
  @SentinelResource(value = "byUrl",blockHandler = "handleException")
  public CommonResult byUrl() {
    return new CommonResult("按url限流", 200);
  }

  public CommonResult handleException(BlockException exception){
    return new CommonResult(exception.getClass().getCanonicalName(),200);
  }

}

```


### Nacos 存储规则

```yaml
spring:
  cloud:
  sentinel:
    datasource:
    ds1:
      nacos:
      server-addr: localhost:8848
      dataId: ${spring.application.name}-sentinel
      groupId: DEFAULT_GROUP
      data-type: json
      rule-type: flow


```


```json
[
  {
    "resource": "/rateLimit/byUrl",
    "limitApp": "default",
    "grade": 1,
    "count": 1,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  }
]

```

