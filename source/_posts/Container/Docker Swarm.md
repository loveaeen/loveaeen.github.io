---
title: Docker Swarm
date: 2022-03-28 16:30:12
excerpt: 与独立容器相比，群服务的一个关键优势是，您可以修改服务的配置，包括它所连接的网络和卷，而无需手动重新启动服务。
categories: Container
tags: Docker
typora-root-url: ../../
---

## [What is a swarm?](https://docs.docker.com/engine/swarm/key-concepts/#what-is-a-swarm)

<font color='gray'>The cluster management and orchestration features embedded in the Docker Engine are built using swarmkit. Swarmkit is a separate project which implements Docker’s orchestration layer and is used directly within Docker.</font>

Docker 引擎中嵌入的集群管理和编排特性是使用群集构建的。Swarmkit 是一个独立的项目，它实现了 Docker 的编排层，并直接在 Docker 中使用。

<font color='gray'>A swarm consists of multiple Docker hosts which run in swarm mode and act as managers (to manage membership and delegation) and workers (which run swarm services). A given Docker host can be a manager, a worker, or perform both roles. When you create a service, you define its optimal state (number of replicas, network and storage resources available to it, ports the service exposes to the outside world, and more). Docker works to maintain that desired state. For instance, if a worker node becomes unavailable, Docker schedules that node’s tasks on other nodes. A task is a running container which is part of a swarm service and managed by a swarm manager, as opposed to a standalone container.</font>

一个群体由多个 Docker 主机组成，它们以群体模式运行，充当管理者(管理成员资格和授权)和工作者(运行群体服务)。给定的 Docker 主机可以是管理者、工作者，或者同时执行这两个角色。当您创建一个服务时，您将定义它的最佳状态(副本的数量、可用的网络和存储资源、服务向外部世界公开的端口等)。多克尔工作是为了保持所需的状态。例如，如果一个工作节点变得不可用，Docker 将该节点的任务调度到其他节点上。任务是一个正在运行的容器，它是群服务的一部分，由群管理器管理，而不是一个独立的容器。

<font color='gray'>One of the key advantages of swarm services over standalone containers is that you can modify a service’s configuration, including the networks and volumes it is connected to, without the need to manually restart the service. Docker will update the configuration, stop the service tasks with the out of date configuration, and create new ones matching the desired configuration.</font>

与独立容器相比，群服务的一个关键优势是，您可以修改服务的配置，包括它所连接的网络和卷，而无需手动重新启动服务。Docker 将更新配置，使用过期配置停止服务任务，并创建与所需配置匹配的新任务。

<font color='gray'>When Docker is running in swarm mode, you can still run standalone containers on any of the Docker hosts participating in the swarm, as well as swarm services. A key difference between standalone containers and swarm services is that only swarm managers can manage a swarm, while standalone containers can be started on any daemon. Docker daemons can participate in a swarm as managers, workers, or both.</font>

当 Docker 在群模式下运行时，你仍然可以在任何参与群的 Docker 主机上运行独立的容器，以及群服务。独立容器和群服务的一个关键区别是，只有群管理器可以管理群，而独立容器可以在任何守护进程上启动。多克尔守护进程可以作为管理者、工作者或两者都参与到群中。

<font color='gray'>In the same way that you can use Docker Compose to define and run containers, you can define and run Swarm service stacks.</font>

与使用 dockercompose 定义和运行容器一样，您可以定义和运行 Swarm 服务堆栈。

<font color='gray'>Keep reading for details about concepts relating to Docker swarm services, including nodes, services, tasks, and load balancing.</font>

继续阅读关于 Docker 群服务相关概念的详细信息，包括节点、服务、任务和负载平衡。