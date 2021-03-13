---
layout:     post
title:      Chapter 1. Kubernetes Overview
subtitle:   
date:       2020-08-12
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

### 单体架构转变为微服务

以前，大多数的软件应用都是**单体架构**，它们要么以一个进程运行，要么以几个服务器的少数几个进程运行，这种架构的应用至今仍然大量存在。单体架构存在的问题是 **发布周期长，更新相对不频繁**。因为整个应用程序作为一个整体，Dev 团队修改一部分，要把整个应用全部打包交给 Ops 团队再重新部署发布。当部署这个应用的服务器硬件出现问题时，Ops 团队要手动的迁移整个应用程序，工作量很大。

如今，这种庞大的单体架构应用逐渐地被拆分为较小的、独立运行的组件，这些组件被称为**微服务**(microservice)。微服务之间相互解耦，所以可以 **分别独立的部署、发布、更新、扩展** ，如下图所示。这种快速的迭代更新方式在当今的需求快速变化的时代显得尤为重要。

![img](/img/post/post_microservice.png)

但是，随着一个应用里微服务的数量逐渐增多，**部署和运维的工作量线性增加**，如何把这些微服务配置部署到数据中心来让整个应用顺利运行变得越来越困难。这里的难点包括某个微服务应该被调度到哪台服务器执行、当服务器 crash 时微服务该如何迁移，以尽可能提高资源的利用率，降低硬件成本。人为的配置和调度显然是不可能的。我们需要一个自动化工具，它帮我们完成自动调度微服务组件到某台合适的服务器、自动配置部署、持续地监视组件状态以及出错时及时处理等工作。这就是 Kubernetes 的由来。

### Kubernetes 架构

Kubernetes 把底层的硬件基础设备抽象为一个计算资源池暴露给用户。它允许用户部署和运行微服务组件，而无需了解底层微服务被调度到了哪台服务器。当通过 Kubernetes 部署一个多组件的应用程序时，它自动为每个组件选一个合适的服务器部署在上面，并使该组件能被其他组件发现和与其他组件通信，来保证整个应用程序的顺利运行。下图展示了 Kubernetes 系统的角色示意，可以看到它将应用程序和服务器节点解耦，用户只需与Kubernetes交互。

[多说一句] 在 Kubernetes 出现之前，公司也逐渐意识到，让 Dev 团队参与到应用的部署和运维会更好，这种模式被称为 DevOps。而现在通过抽象底层硬件、只向上层暴露一个部署运行 App的平台，Kubernetes 可以让 Dev 团队直接参与全生命周期的应用部署，而无需 Ops 团队的协助，可称为 NoOps。

![img](/img/post/post_k8sRole.png)

上图中可以看到 Kubernetes 被标为 master，其实 Kubernetes 由一个 master 节点和一系列 worker 节点组成。用户先将部署任务提交到 master 节点，然后 master 节点调度部署到 worker 节点。更具体地讲，
+ master node: 保存 Kubernetes 的 control panel，用于控制和管理整个 Kubernetes 系统；
+ worker node: 运行部署的应用程序；

![img](/img/post/post_k8sStructure.png)

如上图所示，master节点主要有以下组件：
+ Kubernetes API Server: 用户交互的门户，它与其他 control panel 组件通信；
+ Scheduler: 调度应用到某个 worker 节点；
+ Controller Manager: 执行集群级别的功能，例如跟踪 worker 节点的状态，节点错误的处理等；
+ etcd: 存储集群配置数据的分布式数据库；

运行应用程序的worker节点主要有以下组件：
+ Container Runtime: 运行的容器，可以由 Docker 镜像构建；
+ Kubelet: 和 API Server 通信，管理运行在这个 node 上的容器，确保容器处于健康的运行状态。
+ Kube-proxy: 节点的网络代理，维护节点上的网络规则，负责网络流量的负载均衡；

### Kubernetes 运行流程

下图展示了应用如何部署到 Kubernetes 上。用户先将要部署的四种应用打包上传至 DockerHub，方便稍后 Kubernetes pull image，涉及到 Docker 和 Image 的下篇再讲。然后用户将 App Descriptor 提交至 Kubernetes 的 master 节点，可以看到 App Descriptor 包含四种应用，被分为三组，其实是指三个 Pod，关于 Pod 的概念之后再讲。master 节点调度特定数量的 Pod 到对应的 worker 节点，然后 worker 节点开始从 Docker Hub 拉取容器镜像，运行容器。

![img](/img/post/post_k8sRunApp.png)

### 理解使用Kubernetes的好处

1. 简化应用的部署。因为 Kubernetes 已将所有的 Worker Node 抽象为一个部署平台，所以应用开发者可以自己部署应用，而无需了解集群中 Server 的具体信息。当然，如果在某种情况下确实需要将应用部署在某特定要求的节点上，例如，需要将某个应用部署在有 SSD 的节点上，我们就可以告诉 Kubernetes 为该应用选择一个配有 SSD 的节点，至于如何选、选择哪一个就留给 K8S 自己完成。我们留在之后详细介绍。

2. 资源利用更高。Kubernetes 将 App 和底层资源解耦，当告诉它运行某个应用时，它会基于应用的描述为其选择最合适的节点。并且，当节点故障时，它会自动迁移应用到另一个合适的节点。

3. 健康核查和自动恢复。随着集群机器逐渐增多，节点故障就会更加频繁。Kubernetes 会监视运行的应用和节点，当发生故障时，会重新调度。这就极大解放了 Ops 团队。

4. 自动放缩。使用Kubernetes管理已部署的应用程序，还意味着 Ops 团队无需不断监视各个应用程序的负载以应对突然的负载高峰。如上所述，可以告诉 Kubernetes 监视每个应用程序使用的资源，并不断调整每个应用程序的运行实例数。

### 小结

本篇我们主要从应用程序的架构角度讲了 Kubernetes 的由来，然后介绍了它的基本架构和大体的运行流程。因为 Kubernetes 与容器分不开，下篇我们介绍容器的概念。

参考自：
1. [Kubernetes 是什么?](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/)
2. [Kubernetes 组件](https://kubernetes.io/zh/docs/concepts/overview/components/)
3. [Kubernetes, 2020 快速入门](https://zhuanlan.zhihu.com/p/100644716)
4. 《Kuberneter in Action》 by Marko Luksa.

