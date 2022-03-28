---
title: Nacos注册与配置
date: 2022-03-28 15:43:12
excerpt: 这是一篇简要描述Nacos注册与配置原理的文章
categories: Spring Cloud
tags: 
typora-root-url: ../../
---

## 注册

Nacos - NacosNamingService初始化中提到NacosNamingService初始化会初始化EventDispatcher、NamingProxy、BeatReactor、HostReactor。其中EventDispatcher已经说了，NamingProxy的定时任务主要是默认每30毫秒更新服务器地址、默认每5毫秒登录获取token等信息

HostReactor的建立任务包括每5秒检测是否开启故障转移，若是开启，则把文件数据读入serviceMap、天天把服务信息写入本地、检测本地缓存文件，若是没有则建立缓存文件、监听UDP请求

在restTemplates / feign采用服务名调用的时候, 进入`excute`的深层方法, 会走到`AbstractLoadBalancerRule`抽象类, nacos对他进行了实现, 并通过`HostReactor`获取所有的服务信息, 并进行负载均衡

## 配置

Nacos 服务端创建了相关的配置项后，客户端就可以进行监听了。客户端是通过一个定时任务来检查自己监听的配置项的数据的，**一旦服务端的数据发生变化时，客户端将会获取到最新的数据**，并将最新的数据保存在一个 CacheData 对象中，然后会重新计算 CacheData 的 md5 属性的值，此时就会对该 CacheData 所绑定的 Listener 触发 receiveConfigInfo 回调。

考虑到服务端故障的问题，客户端将最新数据获取后会保存在本地的 snapshot 文件中，以后会优先从文件中获取配置信息的值。

### 那么客户端如何感知到数据发生变化的呢？

我们知道客户端会有一个长轮训的任务去检查服务器端的配置是否发生了变化，如果发生了变更，那么客户端会拿到变更的 groupKey 再根据 groupKey 去获取配置项的最新值更新到本地的缓存以及文件中，那么这种每次都靠客户端去请求，那请求的时间间隔设置多少合适呢？

**长轮训的概念：**

客户端发起一个请求到服务端，服务端收到客户端的请求后，并不会立刻响应给客户端，而是先把这个请求hold住，然后服务端会在hold住的这段时间检查数据是否有更新，如果有，则响应给客户端，如果一直没有数据变更，则达到一定的时间（长轮训时间间隔）才返回。

![img](image/Nacos%E6%B3%A8%E5%86%8C%E4%B8%8E%E9%85%8D%E7%BD%AE/2CF311CF-60DC-4520-82CA-75FBC68679E2_2.jpeg)

timeout是在init这个方法中赋值的，默认情况下是30秒，可以通过configLongPollTimeout进行修改，但是源码中delayTime字段用于作为结束监听返回数据的时间间隔，所以等待的时间一般是略小于30秒。

![img](/image/Nacos%E6%B3%A8%E5%86%8C%E4%B8%8E%E9%85%8D%E7%BD%AE/B1C9C3BA-7663-4D94-A57C-E4ED013ED484_2.jpeg)

ClientLongPolling 被提交给 scheduler 执行之后，实际执行的内容可以拆分成以下四个步骤：

- 1.创建一个调度的任务，调度的延时时间为 29.5s
- 2.将该 ClientLongPolling 自身的实例添加到一个 allSubs 中去
- 3.延时时间到了之后，首先将该 ClientLongPolling 自身的实例从 allSubs 中移除
- 4.获取服务端中保存的对应客户端请求的 groupKeys 是否发生变更，将结果写入 response 返回给客户端

**在过程中总共分两种情况，第一种在监听过程中发生变化，第二种就是超时之后执行检测后返回**