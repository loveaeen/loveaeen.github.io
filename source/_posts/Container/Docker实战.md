---
title: Docker实战
date: 2022-03-28 16:20:12
excerpt: 这是一篇自己日常研发过程中常用的命令，包含docker run参数与offline打包
categories: Container
tags: Docker
typora-root-url: ../../
---

## Nacos & Seata

```Bash
docker run --name nacos-quick -v /Users/hanzhiguo/docker/nacos/logs:/home/nacos/logs -v /Users/hanzhiguo/docker/nacos/conf:/home/nacos/conf -e MODE=standalone -p 8848:8848 --privileged=true --restart=always -d nacos/nacos-server:2.0.0

docker run --name seata-server \
        -v /Users/hanzhiguo/docker/seata/logs:/seata-server/logs -v /Users/hanzhiguo/docker/seata/conf:/seata-server/resources \
        -p 8091:8091 \
        -e SEATA_IP=172.16.237.63 \
        -d seataio/seata-server:1.4.2
```

## ElasticSearch & Kibana

```Bash
# 创建自定义的网络(用于连接到连接到同一网络的其他服务(例如Kibana))
docker network create somenetwork 

# es 暴露的端口很多
# es 运行占用的内存超级大
# es 的数据一般需要放置到安全目录！挂载
# 启动 elasticsearch
docker run -d --name elasticsearch --net somenetwork -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch

# 查看docker状态
docker stats

# 限制ES内存 修改配置文件 -e 环境配置修改
docker run -d --name elasticsearch --net somenetwork -p 9200:9200 -p 9300:9300 -v /Users/hanzhiguo/docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml  -e "discovery.type=single-node" -e ES\_JAVA\_OPTS="-Xms64M -Xmx512M" elasticsearch:6.4.3

# 运行 Kibana
docker run -d --name kibana --net somenetwork -p 5601:5601 kibana
```

## Redis

```Bash
docker run \
-p 6379:6379 \
-v /Users/hanzhiguo/docker/redis/data:/data \
-v /Users/hanzhiguo/docker/redis/conf/redis.conf:/etc/redis/redis.conf \
--privileged=true \
--name myredis \
-d redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
```

## loveaeen/Centos7

```Bash
# docker run
docker run -itd --privileged --hostname=master --name centos7-master loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio1 --name centos7-minio1 loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio2 --name centos7-minio2 loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio3 --name centos7-minio3 loveaeen/centos-docker:1.0 /usr/sbin/init

# modify hostname
# hostnamectl set-hostname master
```

## Ubuntu & Tomcat8 & JDK8

```Bash
# 创建dockerfile文件
FROM         ubuntu:14.10
MAINTAINER    linx

# 把java与tomcat添加到容器中
ADD jdk-8u261-linux-x64.tar.gz /usr/local/
ADD apache-tomcat-8.5.57.tar.gz /usr/local/

# 配置java与tomcat环境变量
ENV JAVA\_HOME /usr/local/jdk1.8.0\_261
ENV CLASSPATH $JAVA\_HOME/lib/dt.jar:$JAVA\_HOME/lib/tools.jar
ENV CATALINA\_HOME /usr/local/apache-tomcat-8.5.57
ENV CATALINA\_BASE /usr/local/apache-tomcat-8.5.57
ENV PATH $PATH:$JAVA\_HOME/bin:$CATALINA\_HOME/lib:$CATALINA\_HOME/bin

# 容器运行时监听的端口
EXPOSE  8080

# 使用dockerfile
docker build -t name .

# tomcat启动并挂载
docker run -i -t -p 8084:8080 -v /Users/hanzhiguo/docker/tomcat:/usr/local/apache-tomcat-8.5.57/webapps --name shys\_tomcat shys\_tomcat
```

## Mysql

```Bash
docker run --name mysql -p 3306:3306 -e MYSQL\_ROOT\_PASSWORD=root -d  mysql --lower\_case\_table\_names=1

# 若mysql版本为8以上。有些工具无法连接 需要修改用户密码的验证方式
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql\_native\_password BY 'yourpassword'
```

## Prometheus & Grafana监控Springboot & Mysql

### 1. 需要在springboot项目中加入依赖

```Bash
&lt;!-- 为prometheus暴露采集端点 --&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-starter-actuator&lt;/artifactId&gt;
        &lt;/dependency&gt;

        &lt;!-- prometheus支持 --&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;io.micrometer&lt;/groupId&gt;
            &lt;artifactId&gt;micrometer-registry-prometheus&lt;/artifactId&gt;
        &lt;/dependency&gt;
```

### 2. yml文件中开放地址

```YAML
management:
  endpoints:
  web:
    exposure:
      include: "*"
```

### 3.创建一个prometheus.yml文件

```YAML
# 全局配置
lobal:
 scrape_interval:     15s # 多久 收集 一次数据
 evaluation_interval: 15s # 多久评估一次 规则
 scrape_timeout:      10s   # 每次 收集数据的 超时时间

 监控预警配置.未启用
lerting:
 alertmanagers:
 - static_configs:
   - targets:
     # - alertmanager:9093

 # 规则文件, 可以使用通配符. 未启用
ule_files:
 # - "first_rules.yml"
 # - "second_rules.yml"

 prometheus服务配置
crape_configs:
 - job_name: 'prometheus'
   static_configs:
   - targets: ['localhost:9090']

 SpringBoot应用配置
 - job_name: 'SpringBootPrometheus'
   scrape_interval: 5s
   metrics_path: '/actuator/prometheus'
   static_configs:
     - targets: ['192.168.1.145:81']

 mysql配置
 - job_name: 'mysql'
   static_configs:
     - targets: ['192.168.1.145:9104']
```

### 4.docker启动prometheus(通过配置文件挂载)及grafana

```Bash
# 需要先创建一个配置文件。用于挂载
docker run  -d \
  -p 9090:9090 \
  --name prometheus \
  -v /Users/hanzhiguo/docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
  prom/prometheus

# 启动Grafana
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v /Users/hanzhiguo/docker/grafana-storage:/var/lib/grafana \
  grafana/grafana
```

### 5.安装mysql-exporter监控mysql

```Bash
# 创建一个监控账户
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'XXXXXXXX' WITH MAX\_USER\_CONNECTIONS 10;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
# 启动mysql-exporter
docker run -d \
  -p 9104:9104 \
  -e DATA\_SOURCE\_NAME="exporter:exporter@(192.168.1.145:3306)/" \
  prom/mysqld-exporter

# 结束后使用mrbird的mysql监控json
```

## BackUp & Transfer

### BackUp

- 运行一个 Ubuntu 的容器，挂载mongo容器的所有 volume，映射宿主机的 backup 目录到容器里面的 /backup 目录，然后运行 tar 命令将data目录下的所有文件打包到backup.tar中 `docker run --rm --volumes-from mongo -v d:/backup:/backup ubuntu tar cvf /backup/backup.tar /data/`
- 这样宿主机的backup目录就有了mongo容器的备份压缩包了

### Transfer

- 运行一个 ubuntu 容器，挂载 mongo 容器的所有 volumes，然后读取 /backup 目录中的备份文件，解压到 /data/ 目录 `docker run --rm --volumes-from mongo -v d:/backup:/backup ubuntu bash -c "cd /data/ && tar xvf /backup/backup.tar --strip 1"`
- strip 1 表示解压时去掉前面1层目录，因为压缩时包含了绝对路径

## Push & Pull

```Bash
# docker login your account
# input your loginname & password
docker login

# tag the image
docker tag image:1.0 loveaee/image:1.0

# push your images to Docker Hub
docker push loveaee/image:1.0

# delete local image
docker rmi imageId

# pull image from docker hub
docker pull loveaee/image:1.0
```

## Offline

在内网模式下，我们可以让Docker接入内网仓库harbor来管理所有的镜像文件。

```Bash
vim /etc/docker/daemon.json

"insecure-registries": ["172.17.0.3"]

systemctl restart docker
```

也可以使用ftp的方式来使用离线版镜像。

```
# save image to disk
# -o : Write to a file
docker save image -o image.tar

# load image from disk
# -i : Read from tar archive file
docker load -i image.tar
```