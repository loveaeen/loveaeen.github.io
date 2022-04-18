---
title: Mybatis
date: 2022-03-28 09:43:12
excerpt: MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。
categories: Spring Cloud
hide: true
tags: 
- Database
- ORM
typora-root-url: ../../
---

## Primary components

- Configuration       
	MyBatis所有的配置信息都维持在Configuration对象之中。**包含拦截器的动态代理功能**
	- `SqlSessionFactoryBuilder`对象通过该配置生成`SqlSessionFactory`, `SqlSessionFactory.openSession()`得到`SqlSession`对象
	- Spring集成版本中, 不再采用Builder生成, 而是SqlSessionFactoryBean的工厂bean方式注入到Spring中
- SqlSession
作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能。
- MappedStatement  
MappedStatement是维护了某一条<select|update|delete|insert>节点的封装对象。
- Executor
MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护。
- SqlSource           
负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回。

![](image/Mybatis/mybatis-y-arch-1.png "")

- BoundSql            
表示动态生成的SQL语句以及相应的参数信息。
- StatementHandler
封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。
- ParameterHandler
负责对用户传递的参数转换成JDBC Statement 所需要的参数。
- ResultSetHandler   
负责将JDBC返回的ResultSet结果集对象转换成List类型的集合。
- TypeHandler         
负责java数据类型和jdbc数据类型之间的映射和转换。
- DefaultObjectFactory
在返回查询结果时, 会调用该类的create方法创建java对象出来, 可以实现该方法自定义新的create方法构造出指定值的对象出来

## 源码解读

[Java 全栈知识体系-Mybatis详解](https://pdai.tech/md/framework/orm-mybatis/mybatis-overview.html)



