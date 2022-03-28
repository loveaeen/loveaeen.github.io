---
title: K3S入门
date: 2022-03-28 16:35:12
excerpt: k3s 是史上最轻量级 Kubernetes.
categories: Container
tags: Kubernetes
typora-root-url: ../../
---

## [What is k3s？](https://docs.rancher.cn/docs/k3s/_index)

K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。

## Why is k3s？

使用内嵌轻量级数据库 SQLite 作为默认数据存储替代 etcd，当然 etcd 仍然是支持的。

内置了 local storage provider、service load balancer、helm controller、Traefik ingress controller，开箱即用。

所有 Kubernetes 控制平面组件如 api-server、scheduler 等封装成为一个精简二进制程序，控制平面只需要一个进程即可运行。

删除内置插件(比如 cloudprovider 插件和存储插件)。

减少外部依赖，操作系统只需要安装较新的内核以及支持 cgroup 即可，k3s 安装包已经包含了 containerd、Flannel、CoreDNS，非常方便地一键式安装，不需要额外安装 Docker、Flannel 等组件。

## Manual run k3s

### K3S Environment

## [AutoK3s](https://docs.rancher.cn/docs/k3s/autok3s/_index)

AutoK3s 是用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s 服务。

### Docker

```Bash
docker run -itd --restart=unless-stopped -p 8080:8080 cnrancher/autok3s:v0.4.5
```

### CLI

```Bash
curl -sS http://rancher-mirror.cnrancher.com/autok3s/install.sh  | INSTALL_AUTOK3S_MIRROR=cn sh
```