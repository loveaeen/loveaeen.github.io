---
title: Hikari
date: 2022-03-28 09:43:12
excerpt: Hikari是Spring在2.0之后的默认jdbc连接工具, 相比于常用的Druid, 他的连接速度加快, 字节码精简, 并发读写效率提高。
categories: Spring Cloud
tags: Database
---

Hikari是Spring在2.0之后的默认jdbc连接工具, 相比于常用的Druid, 他的连接速度加快, 字节码精简, 并发读写效率提高

但是Druid有强大的Sql监控特性以及额外的扩展功能

## 主要构件

- HikariDataSource
是一切的入口, 用于获取连接的主要类
- HikariConfig
配置类
- HikariPool
持有连接池, 延时maxlifetime任务, 定时keepalive清理任务的控制类
- ConcurrentBag
连接池, 内部持有了连接对象的集合, 用于获取 , 删除, 增加连接
- PoolEntry
具体的连接对象, 持有实际的物理连接Connection
- PoolBase
主要用于生产一个DriverDataSource类, 给new PoolEntry()时使用

## 源码解读

[HikariCP源码流程](https://juejin.cn/post/6986812265357901860)



