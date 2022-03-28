---
title: Sleuth
date: 2022-03-28 09:43:12
excerpt: Spring Cloud Sleuth 是分布式系统中跟踪服务间调用的工具，它可以直观地展示出一次请求的调用过程。
categories: Spring Cloud
tags: 
---

> Spring Cloud Sleuth 是分布式系统中跟踪服务间调用的工具，它可以直观地展示出一次请求的调用过程，本文将对其用法进行详细介绍。


### Zipkin

Zipkin是Twitter的一个开源项目，可以用来获取和分析Spring Cloud Sleuth 中产生的请求链路跟踪日志，它提供了Web界面来帮助我们直观地查看请求链路跟踪信息。

Zipkin是一个单独的服务，Spring项目引入依赖后，再下载Zipkin的jar后启动。

Zipkin默认记录只是存储在内存中，Zipkin关闭则全部丢失。

- 在user-service和ribbon-service中添加相关依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```


- 修改application.yml文件，配置收集日志的zipkin-server访问地址：

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      probability: 0.1 #设置Sleuth的抽样收集概率
```


### ElasticSearch存储

```PowerShell
# STORAGE_TYPE：表示存储类型 ES_HOSTS：表示ES的访问地址
java -jar zipkin-server-2.12.9-exec.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=localhost:9200 
```


### Used Config

```CSS
1、使用rabbitmq+elasticsearch的启动方式

java -jar zipkin.jar --zipkin.collector.rabbitmq.addresses=62.234.66.186 --zipkin.collector.rabbitmq.username=root --zipkin.collector.rabbitmq.password=123456 --zipkin.storage.type=elasticsearch --zipkin.storage.elasticsearch.host=http://62.234.66.186:9200 --zipkin.storage.elasticsearch.http-logging=BASIC

2、使用rabbitmq+mysql启动方式

java -jar zipkin.jar --zipkin.collector.rabbitmq.addresses=62.234.66.186 --zipkin.collector.rabbitmq.username=root --zipkin.collector.rabbitmq.password=123456 --zipkin.storage.type=mysql --zipkin.storage.mysql.host=62.234.66.186 --zipkin.storage.mysql.port=3306 --zipkin.storage.mysql.username=root --zipkin.storage.mysql.password=root --zipkin.storage.mysql.db=psych

3、使用activemq+mysql启动方式

java -jar zipkin.jar --zipkin.collector.activemq.url=tcp://62.234.66.186:61616 --zipkin.collector.activemq.username=admin --zipkin.collector.activemq.password=admin --zipkin.storage.type=mysql --zipkin.storage.mysql.host=62.234.66.186 --zipkin.storage.mysql.port=3306 --zipkin.storage.mysql.username=root --zipkin.storage.mysql.password=root --zipkin.storage.mysql.db=psych

4、使用activemq+elasticsearch方式雷同，只需要将数据源改为elasticsearch即可
```

