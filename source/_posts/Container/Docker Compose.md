---
title: Docker Compose
date: 2022-03-28 16:20:12
excerpt: Docker Compose是用于定义和运行多容器Docker应用程序的工具。
categories: Container
tags: Docker
typora-root-url: ../../
---

我们可以通过`Dockerfile`文件让用户很方便的定义一个单独的应用容器。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况，或者开发一个Web应用，除了Web服务容器本身，还需要数据库服务容器、缓存容器，甚至还包括负载均衡容器等等。

`Docker Compose`是用于定义和运行多容器Docker应用程序的工具。通过Compose，您可以使用YAML文件来配置应用程序所需要的服务。然后使用一个命令，就可以通过YAML配置文件创建并启动所有服务。

### Samples

#### Solution

- 如果修改了jar文件后, 如果直接用docker-compose -f -d 直接重启容器的话, jar文件并不会改变, 我们需要使用-build命令进行重新构建才可以
- spring的yaml配置中, 不要再设置固定ip了, 因为docker-compose管理中是默认给容器设置动态ip的, 如果想用服务连接nacos的话, 我们需要在需要的服务容器中添加nacos的link, 然后在spring的yaml配置中ip修改为link名称即可

#### Reference

```YAML
version : '3.8'
services:
  ruoyi-nacos:
    container_name: ruoyi-nacos
    image: nacos/nacos-server
    build:
      context: ./nacos
    environment:
      - MODE=standalone
    volumes:
      - ./nacos/logs/:/home/nacos/logs
      - ./nacos/conf/application.properties:/home/nacos/conf/application.properties
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9849:9849"
    depends_on:
      - ruoyi-mysql
  ruoyi-mysql:
    container_name: ruoyi-mysql
    image: mysql:5.7
    build:
      context: ./mysql
    ports:
      - "3306:3306"
    volumes:
      - ./mysql/conf:/etc/mysql/conf.d
      - ./mysql/logs:/logs
      - ./mysql/data:/var/lib/mysql
    command: [
          'mysqld',
          '--innodb-buffer-pool-size=80M',
          '--character-set-server=utf8mb4',
          '--collation-server=utf8mb4_unicode_ci',
          '--default-time-zone=+8:00',
          '--lower-case-table-names=1'
        ]
    environment:
      MYSQL_DATABASE: 'ry-cloud'
      MYSQL_ROOT_PASSWORD: password
  ruoyi-redis:
    container_name: ruoyi-redis
    image: redis
    build:
      context: ./redis
    ports:
      - "6379:6379"
    volumes:
      - ./redis/conf/redis.conf:/home/ruoyi/redis/redis.conf
      - ./redis/data:/data
    command: redis-server /home/ruoyi/redis/redis.conf
  ruoyi-nginx:
    container_name: ruoyi-nginx
    image: nginx
    build:
      context: ./nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/html/dist:/home/ruoyi/projects/ruoyi-ui
      - ./nginx/conf/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/logs:/var/log/nginx
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - ruoyi-gateway
    links:
      - ruoyi-gateway
  ruoyi-gateway:
    container_name: ruoyi-gateway
    build:
      context: ./ruoyi/gateway
    ports:
      - "8080:8080"
    depends_on:
      - ruoyi-redis
    links:
      - ruoyi-redis
      - ruoyi-nacos
  ruoyi-auth:
    container_name: ruoyi-auth
    build:
      context: ./ruoyi/auth
    ports:
      - "9200:9200"
    depends_on:
      - ruoyi-redis
    links:
      - ruoyi-redis
  ruoyi-modules-system:
    container_name: ruoyi-modules-system
    build:
      context: ./ruoyi/modules/system
    ports:
      - "9201:9201"
    depends_on:
      - ruoyi-redis
      - ruoyi-mysql
    links:
      - ruoyi-redis
      - ruoyi-mysql
  ruoyi-modules-gen:
    container_name: ruoyi-modules-gen
    build:
      context: ./ruoyi/modules/gen
    ports:
      - "9202:9202"
    depends_on:
      - ruoyi-mysql
    links:
      - ruoyi-mysql
  ruoyi-modules-job:
    container_name: ruoyi-modules-job
    build:
      context: ./ruoyi/modules/job
    ports:
      - "9203:9203"
    depends_on:
      - ruoyi-mysql
    links:
      - ruoyi-mysql
  ruoyi-modules-file:
    container_name: ruoyi-modules-file
    build:
      context: ./ruoyi/modules/file
    ports:
      - "9300:9300"
    volumes:
    - ./ruoyi/uploadPath:/home/ruoyi/uploadPath
  ruoyi-visual-monitor:
    container_name: ruoyi-visual-monitor
    build:
      context: ./ruoyi/visual/monitor
    ports:
      - "9100:9100"
```