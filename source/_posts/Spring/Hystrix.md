---
title: Hystrix
date: 2022-03-28 09:43:12
excerpt: Hystrix具备服务降级、服务熔断、线程隔离、请求缓存、请求合并及服务监控等强大功能。
categories: Spring Cloud
tags: 
---

> 在微服务架构中，服务与服务之间通过远程调用的方式进行通信，一旦某个被调用的服务发生了故障，其依赖服务也会发生故障，此时就会发生故障的蔓延，最终导致系统瘫痪。Hystrix实现了断路器模式，当某个服务发生故障时，通过断路器的监控，给调用方返回一个错误响应，而不是长时间的等待，这样就不会使得调用方由于长时间得不到响应而占用线程，从而防止故障的蔓延。Hystrix具备服务降级、服务熔断、线程隔离、请求缓存、请求合并及服务监控等强大功能。


#### Used Parameters

- fallbackMethod：指定服务降级处理方法；
- ignoreExceptions：忽略某些异常，不发生服务降级；
- commandKey：命令名称，用于区分不同的命令；
- groupKey：分组名称，Hystrix会根据不同的分组来统计命令的告警及仪表盘信息；
- threadPoolKey：线程池名称，用于划分线程池。

### 资源隔离

![](https://res.craft.do/user/full/2c016c68-2ea0-5513-42ea-5813d216a654/doc/64F8465D-ECA0-4FC5-8B4D-195A84183BDB/AE24F0F6-D141-4924-8387-28C3C3D9AF70_2 "image")

**适用场景**：

- **线程池技术**，适合绝大多数场景，比如说我们对依赖服务的网络请求的调用和访问、需要对调用的 timeout 进行控制（捕捉 timeout 超时异常）。
- **信号量技术**，适合说你的访问不是对外部依赖的访问，而是对内部的一些比较复杂的业务逻辑的访问，并且系统内部的代码，其实不涉及任何的网络请求，那么只要做信号量的普通限流就可以了，因为不需要去捕获 timeout 类似的问题。

### Request Cache

在一次请求上下文中，如果有多个 command，参数都是一样的，调用的接口也是一样的，而结果可以认为也是一样的。那么这个时候，我们可以让第一个 command 执行返回的结果缓存在内存中，然后这个请求上下文后续的其它对这个依赖的调用全部从内存中取出缓存结果就可以了。

这样的话，好处在于不用在一次请求上下文中反复多次执行一样的 command，**避免重复执行网络请求，提升整个请求的性能**。

```java
@CacheResult(cacheKeyMethod = "getCacheKey")
@HystrixCommand(fallbackMethod = "getDefaultUser", commandKey = "getUserCache")
    public CommonResult getUserCache(Long id) {
    LOGGER.info("getUserCache id:{}", id);
    return restTemplate.getForObject(userServiceUrl + "/user/{1}", CommonResult.class, id);
}

/**
 * 为缓存生成key的方法
 */
public String getCacheKey(Long id) {
    return String.valueOf(id);
}
```


### HystrixCollapser 请求合并

- batchMethod：用于设置请求合并的方法；
- collapserProperties：请求合并属性，用于控制实例属性，有很多；
- timerDelayInMilliseconds：collapserProperties中的属性，用于控制每隔多少时间合并一次请求；
1. 在UserHystrixController中添加testCollapser方法，这里我们先进行两次服务调用，再间隔200ms以后进行第三次服务调用：

```java
@GetMapping("/testCollapser")
public CommonResult testCollapser() throws ExecutionException, InterruptedException {
  Future<User> future1 = userService.getUserFuture(1L);
  Future<User> future2 = userService.getUserFuture(2L);
  future1.get();
  future2.get();
  ThreadUtil.safeSleep(200);
  Future<User> future3 = userService.getUserFuture(3L);
  future3.get();
  return new CommonResult("操作成功", 200);
}

```


1. 使用@HystrixCollapser实现请求合并，所有对getUserFuture的的多次调用都会转化为对getUserByIds的单次调用：

```java
@HystrixCollapser(batchMethod = "getUserByIds",collapserProperties = {
  @HystrixProperty(name = "timerDelayInMilliseconds", value = "100")
})
public Future<User> getUserFuture(Long id) {
  return new AsyncResult<User>(){
  @Override
  public User invoke() {
    CommonResult commonResult = restTemplate.getForObject(userServiceUrl + "/user/{1}", CommonResult.class, id);
    Map data = (Map) commonResult.getData();
    User user = BeanUtil.mapToBean(data,User.class,true);
    LOGGER.info("getUserById username:{}", user.getUsername());
    return user;
    }
  };
}

@HystrixCommand
public List<User> getUserByIds(List<Long> ids) {
  LOGGER.info("getUserByIds:{}", ids);
  CommonResult commonResult = restTemplate.getForObject(userServiceUrl + "/user/getUserByIds?ids={1}", CommonResult.class, CollUtil.join(ids,","));
  return (List<User>) commonResult.getData();
}

```


### Used Config

```yaml
hystrix:
  command: #用于控制HystrixCommand的行为
  default:
    execution:
    isolation:
      strategy: THREAD #控制HystrixCommand的隔离策略，THREAD->线程池隔离策略(默认)，SEMAPHORE->信号量隔离策略
      thread:
      timeoutInMilliseconds: 1000 #配置HystrixCommand执行的超时时间，执行超过该时间会进行服务降级处理
      interruptOnTimeout: true #配置HystrixCommand执行超时的时候是否要中断
      interruptOnCancel: true #配置HystrixCommand执行被取消的时候是否要中断
      timeout:
      enabled: true #配置HystrixCommand的执行是否启用超时时间
      semaphore:
      maxConcurrentRequests: 10 #当使用信号量隔离策略时，用来控制并发量的大小，超过该并发量的请求会被拒绝
    fallback:
    enabled: true #用于控制是否启用服务降级
    circuitBreaker: #用于控制HystrixCircuitBreaker的行为
    enabled: true #用于控制断路器是否跟踪健康状况以及熔断请求
    requestVolumeThreshold: 20 #超过该请求数的请求会被拒绝
    forceOpen: false #强制打开断路器，拒绝所有请求
    forceClosed: false #强制关闭断路器，接收所有请求
    requestCache:
    enabled: true #用于控制是否开启请求缓存
  collapser: #用于控制HystrixCollapser的执行行为
  default:
    maxRequestsInBatch: 100 #控制一次合并请求合并的最大请求数
    timerDelayinMilliseconds: 10 #控制多少毫秒内的请求会被合并成一个
    requestCache:
    enabled: true #控制合并请求是否开启缓存
  threadpool: #用于控制HystrixCommand执行所在线程池的行为
  default:
    coreSize: 10 #线程池的核心线程数
    maximumSize: 10 #线程池的最大线程数，超过该线程数的请求会被拒绝
    maxQueueSize: -1 #用于设置线程池的最大队列大小，-1采用SynchronousQueue，其他正数采用LinkedBlockingQueue
    queueSizeRejectionThreshold: 5 #用于设置线程池队列的拒绝阀值，由于LinkedBlockingQueue不能动态改版大小，使用时需要用该参数来控制线程数

```


### DashBoard

Hystrix提供了Hystrix Dashboard来实时监控HystrixCommand方法的执行情况。 Hystrix Dashboard可以有效地反映出每个Hystrix实例的运行情况，帮助我们快速发现系统中的问题，从而采取对应措施。

![](https://res.craft.do/user/full/2c016c68-2ea0-5513-42ea-5813d216a654/doc/64F8465D-ECA0-4FC5-8B4D-195A84183BDB/7ED0A8F2-1BE6-4284-B09D-A44390CCE9F6_2 "image")



