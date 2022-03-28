---
title: ShardingJDBC
date: 2022-03-28 09:43:12
excerpt: ShardingJDBC 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。
categories: Spring Cloud
tags: Database
---

### [分片算法](https://shardingsphere.apache.org/document/current/cn/dev-manual/sharding/)

- 精准分片 StandardShardingStrategy
- 范围分片 RangeShardingAlgorithm
- 行表达式分片 InlineShardingAlgorithm
- 复合分片 ComplexShardingStrategy
- Hint分片 HintShardingStrategy
- 不分片 NoneShardingStrategy

### 如果SQL中没有分片键，但是代码中有怎么办

#### 动机

通过解析 SQL 语句提取分片键列与值并进行分片是 Apache ShardingSphere 对 SQL 零侵入的实现方式。若 SQL 语句中没有分片条件，则无法进行分片，需要全路由。

在一些应用场景中，分片条件并不存在于 SQL，而存在于外部业务逻辑。因此需要提供一种通过外部指定分片结果的方式，在 Apache ShardingSphere 中叫做 Hint。

#### 机制

Apache ShardingSphere 使用 `ThreadLocal` 管理分片键值。可以通过编程的方式向 `HintManager` 中添加分片条件，该分片条件仅在当前线程内生效。

除了通过编程的方式使用强制分片路由，Apache ShardingSphere 还计划通过 SQL 中的特殊注释的方式引用 Hint，使开发者可以采用更加透明的方式使用该功能。

指定了强制分片路由的 SQL 将会无视原有的分片逻辑，直接路由至指定的真实数据节点。

## 高可用方案



## 主从复制原理

#### 强制分片路由 Hint

```java
HintManager hintManager = HintManager.getInstance() ;
hintManager.setMasterRouteOnly();
```


比如戒毒系统，更新后需要立即刷新列表。如果采用了主从复制技术的话，可能读取从库会导致暂时性的缺少数据。

那么我们就可以在更新的方法中添加该代码，代码下方进行更新和查询操作，那么自然就会去读取主库数据。


#### 等GTID 方案

MySQL 提供了一条基于 GTID 的命令，用于在从节点上执行，等待从库同步到了对应的 GTID（binlog文件中会包含 GTID），或者超时返回。

select wait_for_executed_gtid_set(gtid_set, timeout);

MySQL 在执行完事务后，会将该事务的 GTID 会给客户端，然后客户端可以使用该命令去要执行读操作的从库中执行，等待该 GTID，等待成功后，再执行读操作；如果等待超时，则去主库执行读操作，或者再换一个从库执行上述流程。

举个例子，原来要执行读操作的 SQL 和添加了前缀的 SQL 如下所示：

```sql
SELECT * FROM city; SET @maxscale_secret_variable=(SELECT CASE WHEN WAIT_FOR_EXECUTED_GTID_SET('232-1-1', 10) = 0 THEN 1 ELSE (SELECT 1 FROM INFORMATION_SCHEMA.ENGINES) END); SELECT * FROM city;
```

当 WAIT_FOR_EXECUTED_GTID_SET 执行失败后，原 SQL 就不会再执行，而是将该 SQL 去主节点执行。

#### 缓存主库查询次数

概念很简单, 用户新增或修改后, 往Redis中插入一条数据primaryRead:userid为1, 若多次插入, 则进行累加.

当用户调用查询接口时, 判断用户的查询缓存次数是否存在, 并减一操作后走主库查询. 若不存在或减一后小于0则走从库查询

