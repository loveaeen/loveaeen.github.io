---
title: Rancher
date: 2022-03-28 16:45:12
excerpt: Rancher 对于采用容器的团队来说是一个完整的软件栈。它解决了跨任何基础设施管理多个Kubernetes集群的操作和安全挑战，同时为 DevOps 团队提供运行集装箱式工作负载的集成工具。
categories: Container
tags: Kubernetes
typora-root-url: ../../
---

## [Why Rancher?](https://docs.rancher.cn/docs/rancher2.5/overview/_index)

Rancher 对于采用容器的团队来说是一个完整的软件栈。它解决了跨任何基础设施管理多个Kubernetes集群的操作和安全挑战，同时为 DevOps 团队提供运行集装箱式工作负载的集成工具。

## K8S Environment

### Build Virtual Machines

创建虚拟机模拟生产环境的多台主机。

```Bash
# docker run --network=host -p 8080:8080
docker run -itd --privileged --hostname=master --name centos7-master loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio1 --name centos7-minio1 loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio2 --name centos7-minio2 loveaeen/centos-docker:1.0 /usr/sbin/init

docker run -itd --privileged --hostname=minio3 --name centos7-minio3 loveaeen/centos-docker:1.0 /usr/sbin/init

# modify hostname
# hostnamectl set-hostname master
```

### Network File System

NFS（Network File System）即网络文件系统，它允许网络中的计算机之间通过网络共享资源。**将NFS主机分享的目录，挂载到本地客户端当中，本地NFS的客户端应用可以透明地读写位于远端NFS服务器上的文件，在客户端端看起来，就像访问本地文件一样。**

Rancher有方便的PV(Persistent volume)配置，允许开发人员将容器内文件进行远程挂载。

```Bash
docker run -itd --privileged --hostname=nfs --name centos7-minio3 loveaeen/centos-docker:1.0 /usr/sbin/init
```

### Run Rancher

启动Rancher，并新建集群，运行他的docker命令，创建好主节点与工作节点，管理kubernetes。

```Bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 -e CATTLE_SYSTEM_DEFAULT_REGISTRY=registry.cn-hangzhou.aliyuncs.com -e CATTLE_BOOTSTRAP_PASSWORD=admin --privileged rancher/rancher:v2.6.0
```

## K3S Environment

### Master

使用`docker`方式在主节点上安装k3s server服务

```Bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - --docker
```

### Worker

使用`docker`方式在工作节点上安装k3s agent服务

```Bash
# k3s_token 可以在master节点的/var/lib/rancher/k3s/server/node-token找到

curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://110.42.229.50:6443 K3S_TOKEN=K105e759bc78f55939faf9fd8c6018f2df53390e2572ef18d80180b71af65455895::server:992cc950fe1898069d48ff826912bcc6 sh -
```

### Rancher UI

```Bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 -e CATTLE_SYSTEM_DEFAULT_REGISTRY=registry.cn-hangzhou.aliyuncs.com -e CATTLE_BOOTSTRAP_PASSWORD=admin --privileged rancher/rancher:v2.6.0
```

## Spring Cloud

### Step

1. 在自己的微服务中添加`docker-compose.yml`，创建好每一个微服务的构建命令。
2. 每一个微服务有自己的`Dockerfile`，主要选择好/target/jar文件并配置相关参数，在`maven install`后可以直接执行compose命令做构建。
3. **注意：所有ip名称写对应的容器名字，让docker帮我们管理容器的互相访问。**
4. 将所有镜像构建后，上传至私有服务器Harbor中，这样可以方便的放在内网中部署。
5. Rancher从Harbor中拉取镜像并创建Pod，在一些对外访问的Pod中升级端口映射或添加Ingress组件。如Nacos、Nginx。

**PLEASE SEE VIDEO** : https://www.bilibili.com/video/BV1XJ411D7Qu?p=9&spm_id_from=pageDriver