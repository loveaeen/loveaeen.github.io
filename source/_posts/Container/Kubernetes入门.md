---
title: Kubernetes入门
date: 2022-03-28 16:35:12
excerpt: Kubernetes 是一个开源的容器编排引擎，用来对容器化应用进行自动化部署、 扩缩和管理。该项目托管在 CNCF。
categories: Container
tags: Kubernetes
typora-root-url: ../../
---

## 1. MASTER NODE

- [API SERVER](https://kubernetes.io/zh/docs/concepts/architecture/control-plane-node-communication/) : 你可以理解为网关. 当一个请求 (命令) 进入后, 首先就会请求到 API SERVER , 该组件也可以进行鉴权功能
- [Scheduler](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/kube-scheduler/) : 决定了你应该在哪台 Node 上进行 Pod 相关操作, 该组件只是去对应的 Node 中调度 Kubelet 组件, 比如你需要新增几个 Pod , 该组件就会根据 Node 目前资源使用量, 找到最不繁忙的一个 Node ,进行调度操作.
- [Controller Manager](https://kubernetes.io/zh/docs/concepts/architecture/cloud-controller/) : 管理集群健康状况, 比如某个 Pod 宕机/死去, 那么该组件检测到以后, 通知 Scheduler 选取一个适合的 Node , 然后调度对应的 Kubelet 组件进行资源重启
- [etcd](https://etcd.io/) : 键值存储集群状态的组件, 想象它是集群的大脑🧠 如果一个 Pod 新生或者死亡, 都会将信息存储在该组件中. 很多组件都会通过它来获取一些必要的信息

## **2. SLAVE NODE**

- [Container Runtime](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/) : 负责运行容器的软件, 一般来说, 大家都会使用 [[Docker]]

- [Kubelet](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/) : 实际管理一个 Pod 或者重启它

- [Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/) : 公开了从集群外部到集群内[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由. 流量路由由 Ingress 资源上定义的规则控制, 之后再发送至 Service

- [Service](https://kubernetes.io/zh/docs/concepts/services-networking/service/) : 提供 Pod 的服务发现, 公开网络服务

- [Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/) : 专门用于鉴权配置的组件, 比如配置 Mongodb 的账户和密码, 并在 Mongo 那边引用 Secret

- [ConfigMap](https://kubernetes.io/zh/docs/concepts/configuration/configmap/) : 用于放置一些公共配置的组件

- [StatefulSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/)：StatefulSet 是用来管理有状态的应用，例如数据库。前面我们部署的应用，都是不需要存储数据，不需要记住状态的，可以随意扩充副本，每个副本都是一样的，可替代的。而像数据库、Redis 这类有状态的，则不能随意扩充副本。StatefulSet 会固定每个 Pod 的名字

- Deployment

   : Pod 的声明式抽象层, 负责管理 Pod, 现在已经不会直接创建 Pod , 而是使用 Deployment 代替.

  - [Pod](https://kubernetes.io/zh/docs/concepts/workloads/pods/) : Kubernetes 中创建和管理的最小单元
  - [ReplicaSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) : 当你创建了一个 Deployment 后, 就会自动创建一个该组件出来, 反之删除也会一样. 主要用于维护 Pod 的数量

## **3. KUBECTL**

- Kubectl 命令行工具管理 Kubernetes 集群, 所有的命令实际都是调用 REST API , 都会走 API SERVER.

## **4. NAMESPACE**

- 命名空间可以隔离资源, 并且 Kubernetes 可以限制每一个命名空间的最大使用资源情况
- 可以使用 `brew install kubectx` 安装插件, 切换默认的命名空间

## **5. TOOLS**

- 5.1

  minikube

  - Minikube 将 Master 与 Slave 放置在一个虚拟 Node 中运行, 并附加基本的 Container Runtime(Docker)

- 5.2

  Helm

  - Helm 是查找、分享和使用软件构建 [Kubernetes](https://kubernetes.io/) 的最优方式

## 6. Sping Cloud & Kubernates

