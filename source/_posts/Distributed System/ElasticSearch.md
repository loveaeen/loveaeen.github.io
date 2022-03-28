---
title: ElasticSearch
date: 2022-03-28 16:43:12
excerpt: Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。
categories: Distributed System
tags: Database
typora-root-url: ../../
---

## Documents

ElasticSearch是面向文档的, 这也就意味着索引和搜索数据的最小单位是**文档**. 文档有三个重要的属性

- 自我包含: 意思是文档中会同时包含字段及对应的值, 也就是同时包含 key:value.
- 层次型: 一个泛化概念,表示一个文档可以有多层次的结构体, 你可以想象json, 某个属性可以是一个实体.
- 灵活的结构: 在ElasticSearch中, 你可以预定义文档的具体结构类型, 也可以动态添加字段, ElasticSearch会根据你添加的数据, 创建最适合的字段类型.

尽管我们可以随意的新增或者忽略某个字段, 但是, 每个字段的类型非常重要, 比如一个**年龄**字段类型, 可以是字符串也可以是整形. 因为ElasticSearch会保存字段和类型之间的映射及其他的设置. 这种映射具体到每个映射的每种类型, 这也是为什么在elasticsearch中, 类型有时候也称为映射类型

## **T**ypes

类型是文档的逻辑容器,就像关系型数据库一样,表格是行的容器. 类型中对于字段的定义称**为映射**, 比如name映射为字符串类型. 我们说文档是无模式的 它们不需要拥有映射中所定义的所有字段, 比如新增一个字段, 那么ElasticSearch是怎么做的呢?

ElasticSearch会自动的将新字段加入映射, 但是这个字段的不确定它是什么类型, ElasticSearch就开始猜, 如果这个值是18, 那么ElasticSearch会认为它是整形. 但是ElasticSearch也可能猜不对, 所以最安全的方式就是提前定义好所需要的映射, 这点跟关系型数据库殊途同归了, 先定义好字段, 然后再使用, 别整什么幺蛾子.

## **Index**

索引在ElasticSearch中就是关系型数据库的**数据库**, 索引存储了映射类型的字段和其他设置. 然后它们被存储到了各个分片上了.

## **Alias**

当A索引和B索引同时被设置了alias后, 例如:`ali`, 对该别名进行查询时, 则可以直接查询到A索引和B索引的数据, ElasticSearch会自动帮我们整合数据. 但是我们无法直接对别名索引进行写操作. 如果想进行写操作, 只能指定具体的索引. 既然索引可以设置别名, 字段同样可以设置别名. 但是我觉得用的地方不是很多.

## **Shard**s

什么是分片? 简单来讲就是咱们存储在ElasticSearch中所有数据的文件块, 也就是数据的最小单元块。

ElasticSearch的核心就是对所有分片的分布, 索引, 负载, 路由等达到惊人的速度. 比如我们设置了一个索引中有两个主分片, 我们向里面插入10条数据. 那么这10条数据会尽可能均匀的存储在这两个分片中. **每个分片理论上不存储超过30G的数据**, 每一个分片对应着一个节点, 节点数≥分片数. 如果为了保证数据的容灾, 我们就要引入副本分片了

主分片是在索引创建时就固定不可更改的，而副本分片可以动态增减。

### How to write？

首先客户端发送请求, 找到某个节点后转发到协调节点, 然后根据数据id进行HASH路由找到具体的分片中进行写入. 当数据请求到分片中以后, 首先会存储在内存中(buffer和TransLog[^1]文件) .此时数据无法被查询到.

然后ElasticSearch的定时线程每隔1秒钟(**refresh**)将buffer数据刷入到段文件里, 同时建立好倒排索引, 并清空buffer. 这时的段文件并没有存储在磁盘里, 是放在了系统缓存(Os Cache)中.

ElasticSearch另外一个定时线程每隔5秒钟(**flush,fsync**)会将TransLog数据刷入磁盘. 所以在严格意义上来说, ElasticSearch会丢失5秒钟的数据

当TransLog文件大小达到一定的阈值后, 会触发Commit操作. 就是将Os Cache数据全部刷入磁盘. ElasticSearch也会有一个定时器每隔30分钟进行一次Commit操作

![img](/image/ElasticSearch/11.png)

### How to read？

那么既然我们存在了两个主分片, 当我们查询其中一条数据时, ElasticSearch如何帮我们找到具体在哪个分片中呢? 其实与写入操作是相似的, 也是需要根据id来进行HASH路由找到具体的分片, 如果该分片有备份分片的话, 则会默认采取round-robin随机轮询算法去找一个分片来查询.

## Segment file

段文件存储的是数据信息, 你可以理解为Mysql的[[数据页]], 并且会定期进行^^段合并^^操作

因ElasticSearch每秒定时器[1](((wZ4Vu7UW9))) 影响. 并且代码中也可能会手动调用刷新操作. 这就会导致短时间内段文件数量会暴增. 而每一个段文件都会消耗文件句柄, 内存和cpu运行周期. **更重要的是, 每个搜索请求都会轮询搜索每一个段, 所以段越多, 搜索越慢**

为了解决这种段文件越来越多的问题, ElasticSearch出现了**段合并**功能. 段合并会将那些旧的已删除的文档从文件系统中清除.

段合并操作不需要我们手动配置, 在进行搜索时会自动进行

## **Index** Templates

在实际开发中, ElasticSearch很大一部分工作是用来处理日志信息的, 比如某公司对于日志处理策略是以日期为名创建每天的日志索引. 并且每天的索引映射类型和配置信息都是一样的, 只是索引名称改变了. 如果手动的创建每天的索引, 将会是一件很麻烦的事情. 为了解决类似问题, ElasticSearch提供了预先定义的模板进行索引创建, 这个模板称作为Index Templates. 通过索引模板可以让类似的索引重用同一个模板

- 索引模版内容

```JSON
PUT _template/open_api_logs_template
{
  "index_patterns": ["open_api_logs.*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        // 逗号分割字符
        "comma": {
          "type": "pattern",
          "pattern": ","
        },
        "pinyin_analyzer": {
          "tokenizer": "hotline_pinyin"
        }
      },
      // 拼音分词器，需要自己下载一下给ES
      "tokenizer": {
        "hotline_pinyin": {
          "type": "pinyin",
          "keep_joined_full_pinyin": true,
          "keep_none_chinese_in_joined_full_pinyin": true,
          "keep_original": true
        }
      }
    }
  },
  "aliases": {
      "open_api_logs": {}
    },
  "mappings" : {
    "properties" : {
      "id": {
        "type": "keyword"
      },
      "loginName": {
        "type": "keyword"
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "search_pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer"
          }
        
        }
      },
      "type": {
        "type": "integer"
      },
      "methodName": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "search_pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer"
          }
        
        }
      },
      "parameters": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "search_pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer"
          }
        
        }
      },
      "ip": {
        "type": "keyword"
      },
      "result": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "search_pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer"
          }
        
        }
      },
      "startTime": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||yyyy/MM/dd HH:mm:ss"
      },
      "takeTime": {
        "type": "integer"
      }
    }
  }
}
```

## Rollover Index

滚动索引一般可以与**索引模板**结合使用, 实现按一定条件自动创建索引. 设置`Rollover` 满足条件后, 会自动新建索引, 将索引别名转向新索引. 当现有的索引太久或者太大时, 往往使用rollover index创建新索引

```Java
POST /logs_write/_rollover 
{
  "conditions": {
    // 如果索引创建超过了7天, 最大文档数到达了1000, 大小到达了5g, 则新建一个索引-000002
    "max_age":   "7d",
    "max_docs":  1000,
    "max_size":  "5gb"
  }
}
```

## **I**ndex Sorts

- ElasticSearch默认对数据的排序是写入排序, 意味着谁先写进来, 谁就排在最后面. 但是实际开发中, 我们都是对某字段做排序检索, 当我们指定某字段排序时, 在ElasticSearch底层是需要对所有倒排索引进行遍历, 然后对我们指定的某个字段做排序, 并取出前TopN的结果集.
- 如果数据量较小的情况, 性能无需担心, 但是当数据量特别大的情况下, 这种遍历过程就会耗费较长的时间. 于是, ElasticSearch为了解决这种问题, 推出了**索引排序**
- 顾名思义, 就是对索引里的数据做排序. 当我们配置了指定字段的排序后, 检索方式符合ElasticSearch推荐的方式, 比如不统计命中总数, 不统计最大值等, 那么我们就会触发Early Termination机制, 不会再遍历所有文档, 而是按照顺序找到N条数据

```Java
PUT events
{
    "settings" : {
        "index" : {
            "sort.field" : "timestamp",
            "sort.order" : "desc" 
        }
    },
    "mappings": {
        "properties": {
            "timestamp": {
                "type": "date"
            }
        }
    }
}
```

## Optimization

1. `/etc/elasticsearch/jvm.options` - `-Xms32g -Xmx32g -Xx:+ExitOnOutOfMemoryError` - 这里建议，不管物理内存有多大，分配给Elasticsearch的只设成32G，同时后面如果出现OOM，进程直接退出 - 为什么说要把它改成这个呢？因为我们在使用一个集群时看到，有时因为聚合操作的原因，会导致某一台机器上的JAVA进程出现OOM，但是这个JVM进程还在，并没有退出，退出的话可以通过monit捕捉到，也可以进行重启。如果没有退出，而是一直挂在那的话，就不能提供正常的服务。 此外加上这个参数的话，需要升级一下JDK版本，JDK要求1.8.0_92, 从这个版本开始支持`ExitOnOutOfMemoryError`参数
2. `/etc/elasticsearch/elasticsearch.yml` - `bootstarp.memory_lock:true` 锁定内存，防止内存交换，若遇到错误，请[增加命令](((0CBLUtikE))) - `bootstrap.system_call_filter:false` 禁用seccomp
3. 参数优化 - `index.refresh_interval = 15`s 数据多久可以被查询到，控制buffer存入到段文件的速度。每秒写入量越大，该值也应更大。 - `index.translog.flush_threshold_size = 2g` 当内存中的translog达到2g，tanslog会执行刷盘

### Search

虽然ElasticSearch查询性能极佳, 但是合理的调整服务器与数据模型设计, 是我们区别于只会使用ElasticSearch查询的人群的关键.

#### 文件系统缓存

- Elasticsearch 高度依赖操作系统的 `filesystem cache` 来提升其数据读取效率，如果我们将大量的系统内存分配给 JVM 进程(例如将堆内存设置的很大), 那么将导致操作系统无法得到足够的内存来作为 `filesystem cache`, 这会严重的影响 Elasticsearch 的性能.
- 我们知道现代操作系统中的内存都是按页分配的，操作系统在把某些数据从外存读取到内存中的页面上时，会把紧邻在被读取数据后面的几个页面的数据也读到内存中，这叫做**文件预读**，文件预读会极大的提升硬盘的顺序读取速度。除此之外，数据在被读取到内存中之后，即使用户已经使用完毕，操作系统也不会立即把这些数据所占用的内存给释放掉，这些数据会暂时的保存在内存中，如果此时用户开始读取硬盘上的同一块数据，操作系统会立即从内存中把该数据返回给用户而不需要再去操作硬盘进行数据读取操作，这部分暂时留存在内存中的数据就叫做**页缓存** ([page cache](https://en.wikipedia.org/wiki/Page_cache))。

#### 数据预热

- 每当机器重启时, `filesystem cache`就会被清空, OS将索引的热点数据加载进来是需要花费时间的. 所以我们可以增加`index.store.preload` 告知OS, 优先将这些数据加载进入

#### 冷热分离思想

- 就是将冷数据与热数据进行分离, 让热数据被默认搜索, 而冷数据需要选取时间节点后才允许搜索到. 如果要进行这项功能的话, 单机情况可以采用[索引别名](((H7wp71XmU)))的方式, 而集群的话需要采用[[数据分层]]功能

#### 文档结构设计

- 既然ElasticSearch是搜索引擎, 那么我们就不要将非查询字段添加进来了, 一方面会让文件增大, 一方面也会让索引查询变慢.

#### 分页性能优化

- ElasticSearch的分页效率不高, 按照实际开发中的那种方式分页的话, 初始速度还可以, 但是越到后面速度越慢. 所以ElasticSearch有其他的推荐分页方式
  - 1. `scroll api` 快照下拉滚动方式. 这个方式有时间限制, 超出该时间会报错
  - 1. `search after` 通过上一页的结果获取下一页数据

## Ingest Pipeline

### 概念

默认情况下，每个节点都是Ingest Node，可以通过node.ingest=true|false来设置

目前最常用的就是在doc中追加插入时间, 采用pipeline的方式配置, 这样可以有效的在kibana进行时间查询.

### 功能

ingest node具有预处理，可拦截Index或者Bulk API的请求. 对数据进行转化和加工，并重新返回给Index或者Bulk API的功能

### 组件

Pipeline 对通过的数据按照顺序进行加工 Processor 对一些加工的行为进行了封装

## Cluster

![img](/image/ElasticSearch/iShot2021-09-03%2013.52.52.png)

### Summary(摘要)

节点(`node`)是你运行的Elasticsearch实例。一个集群(`cluster`)是一组具有相同cluster.name的节点集合，他们协同工作，共享数据并提供故障转移和扩展功能，当有新的节点加入或者删除节点，集群就会感知到并平衡数据。集群中一个节点会被选举为主节点(master),它用来管理集群中的一些变更，例如新建或删除索引、增加或移除节点等;当然一个节点也可以组成一个集群。

当你的集群扩容或缩小，Elasticsearch将会自动在你的节点间迁移分片，以使集群保持平衡。

当我们指定了某个索引有主分片以及副本分片时，在集群中，该索引会自动的将各个分片有效分配到不同节点中，避免了单点故障的风险。

当主节点宕机，集群会自动选举某从节点为主节点，并将主分片从故障节点转移。此时的集群状态是Yellow，需要重启故障节点才会Green。

### Enviroment & Config

集群配置中最重要的两项是`node.name`与`network.host`，每个节点都必须不同。其中`node.name`是节点名称主要是在Elasticsearch自己的日志加以区分每一个节点信息。 `discovery.zen.ping.unicast.hosts`是集群中的节点信息，可以使用IP地址、可以使用主机名(必须可以解析)。

```
vim /etc/elasticsearch/elasticsearch.yml

cluster.name: my-els                               # 集群名称,设置成一样的才能放在一个集群中。
node.name: els-node1                               # 节点名称，仅仅是描述名称，用于在日志中区分

path.data: /opt/elasticsearch/data                 # 数据的默认存放路径
path.logs: /opt/elasticsearch/log                  # 日志的默认存放路径

network.host: 192.168.60.201                        # 当前节点的IP地址
http.port: 9200                                    # 对外提供服务的端口，9300为集群服务的端口
#添加如下内容
#culster transport port
transport.tcp.port: 9300
transport.tcp.compress: true

discovery.zen.ping.unicast.hosts: ["192.168.60.201", "192.168.60.202","192.168.60.203"]       
# 集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析

discovery.zen.minimum_master_nodes: 2              # 为了避免脑裂，集群节点数最少为 半数+1
```

### UserAdd

```
# 创建用户组
groupadd es
# 创建用户并添加至用户组
useradd es -g es 
# 更改用户密码（输入 123123）
passwd es

chown -R es:es  /etc/share/elasticsearch/
chown -R es:es  /usr/share/elasticsearch/
chown -R es:es  /var/log/elasticsearch/      # 以上操作都是为了赋予es用户操作权限

# 需切换为es用户
su es
# 启动服务（当前的路径为：/usr/share/elasticsearch/）
./bin/elasticsearch -p /tmp/elasticsearch-pid -d
```

### Head

head插件是一个ES集群的web前端工具，它提供可视化的页面方便用户查看节点信息，对ES进行各种操作，如查询、删除、浏览索引等。

## Problems

- LocalDateTime：请使用String类型的日期时间格式化，或者使用long类型存储。其他解决方式都太麻烦了。
- 采用上方String存入的话，kibana设置browser，会存在kibana的时间查询有问题，改成UTC模式的话，需要选择精确的时间范围，不能用默认的now()来确定当前时间。

[^1]: 该文件实时记录所有的修改索引操作确保数据不丢失, 提供索引恢复.