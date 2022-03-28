---
title: Harbor
date: 2022-03-28 16:30:12
excerpt: Harbor是一个开源的注册中心，它通过政策和以角色为基础的存取控制保护工件，确保图像被扫描并且没有漏洞，并且标记镜像为可信的。
categories: Container
tags: 
- Docker
- kubernetes
typora-root-url: ../../
---

## What is Harbor?

> Harbor is an open source registry that secures artifacts with policies and role-based access control, ensures(确保) images are scanned and free from vulnerabilities(脆弱性), and signs images as trusted. Harbor, a CNCF Graduated project, delivers compliance, performance, and interoperability to help you consistently and securely manage artifacts across cloud native compute platforms like Kubernetes and Docker.

Harbor是一个开源的注册中心，它通过政策和以角色为基础的存取控制保护工件，确保图像被扫描并且没有漏洞，并且标记镜像为可信的。Harbor 是 CNCF 的一个毕业项目，它提供了遵从性、性能和互操作性，以帮助您在像 Kubernetes 和 Docker 这样的云本地计算平台上稳定和安全地管理工件。

主要用于为企业管理私有镜像，由Docker部署并运行。

## [Installation](https://github.com/goharbor/harbor/releases)

- <font color='gray'>**Online installer**: The online installer downloads the Harbor images from Docker hub. For this reason, the installer is very small in size.</font>

  在线下载：在线安装程序从 Docker hub 下载 Harbor 镜像。因此，安装程序的体积非常小。

- <font color='gray'>**Offline installer**: Use the offline installer if the host to which are are deploying Harbor does not have a connection to the Internet. The offline installer contains pre-built images, so it is larger than the online installer.</font>

  离线下载：如果正在部署 Harbor 的主机没有连接到 Internet，请使用离线下载。离线下载包含预先构建的镜像，因此它比联机安装程序大。

```Bash
# github install offline installer.tgz
# then scp remote linux
scp harbor-offline-installer-v2.4.1.tgz root@110.42.229.50:/root/harbor-offline-installer-v2.4.1.tgz

# tar
tar -zxvf harbor-offline-installer-v2.4.1.tgz

# config yml & modify **hostname to ip**
cd /harbor
vim harbor.yml
./install.sh

# http request
[http://110.42.229.50/harbor](http://110.42.229.50/harbor)
```

## Docker Integrate(集成)

好像也不用。为了让harbor感知到docker，所以需要让docker重启，再用compose启动harbor。

```Bash
# restart docker
# systemctl restart docker

# cd /harbor
# docker-compose start

# docker login & push image
```