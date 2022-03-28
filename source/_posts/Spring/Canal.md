---
title: Canal
date: 2022-03-28 09:43:12
excerpt: Canal是一个基于 MySQL 二进制日志的高性能数据同步系统。
categories: Spring Cloud
tags: CDC
---

## What is Canal?

Canal is a high performance data synchronization system based on MySQL binary log. Canal is widely used in Alibaba group (including [https://www.taobao.com](https://www.taobao.com)) to provide reliable low latency incremental data pipeline.

Canal是一个基于 MySQL 二进制日志的高性能数据同步系统。Canal 在阿里巴巴集团(包括 [https://www.taobao.com](https://www.taobao.com) )中被广泛应用，以提供可靠的低延迟增量数据流水线。

Canal Server is capable of parsing MySQL binlog and subscribe to the data change, while Canal Client can be implemented to broadcast the change to anywhere, e.g. database and Apache Kafka.

Canal Server能够解析 MySQL binlog 并订阅数据更改，而 Canal Client 可以实现将更改广播到任何地方，例如数据库和 Apache Kafka。

## Component

### Admin

canal-admin设计上是为canal提供整体配置管理、节点运维等面向运维的功能，提供相对友好的WebUI操作界面，方便更多用户快速和安全的操作。

### Adapter

canal-adapter是为了让研发人员方便快速的将增量数据发送到ElasticSearch等Nosql中，减少Java代码的编写。

### Deployer

canal-deployer是一个模仿Mysql备机的小型服务，用于获取数据库日志抓取数据。

## Use

### 1. Schedule

使用Springboot定时器，定时抓取Canal中的增量数据，并解析，然后发送到自己想去的地方。官方中也是使用此做法。

```
Get /knowledge/_search
{
  "query":{
    "term":{
      "searchTypeId.keyword":"3"
    }
  }
}
```

