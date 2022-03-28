---
title: ConfigServer动态配置
date: 2022-03-28 15:43:12
excerpt: 这是一篇简要描述ConfigServer动态配置的文章
categories: Spring Cloud
tags: 
typora-root-url: ../../
---

Spring Cloud Bus会向外提供一个http接口，即图中的/bus/refresh。我们将这个接口配置到远程的git的webhook上，当git上的文件内容发生变动时，就会自动调用/bus-refresh接口。Bus就会通知config-server，config-server会发布更新消息到消息总线的消息队列中，其他服务订阅到该消息就会信息刷新，从而实现整个微服务进行自动刷新。

**配置中心承担配置刷新的职责**

![Image](/image/ConfigServer%E5%8A%A8%E6%80%81%E9%85%8D%E7%BD%AE/28815AD5-AE1B-46D5-84DE-B3BD2A9B1862_2.jpeg)

1. 提交配置触发post调用客户端A的bus/refresh接口
2. 客户端A接收到请求从Server端更新配置并且发送给Spring Cloud Bus总线
3. Spring Cloud bus接到消息并通知给其它连接在总线上的客户端，所有总线上的客户端均能收到消息
4. 其它客户端接收到通知，请求Server端获取最新配置
5. 全部客户端均获取到最新的配置