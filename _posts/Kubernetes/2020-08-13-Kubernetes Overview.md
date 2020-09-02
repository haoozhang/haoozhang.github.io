---
layout:     post
title:      Kubernetes Overview
subtitle:   
date:       2020-08-13
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

### 单体架构转变为微服务

以前，大多数的软件应用都是**单体架构**，它们或以一个进程运行，或以几个服务器的少数几个进程运行，这种架构的应用至今仍然大量存在。单体架构存在的问题是发布周期长，更新相对不频繁。因为整个应用程序作为一个整体，developer团队修改一个部分，要把整个应用全部打包交给deploy团队，deploy团队再重新部署发布。当部署这个应用的服务器硬件出现问题时，deploy团队要手动的迁移整个应用程序，工作量很大。

如今，这种庞大的单体架构应用逐渐地被拆分为较小的、独立运行的组件，这些组件被称为**微服务**(microservice)。微服务之间相互解耦，所以可以分别独立的部署、发布、更新、扩展，如下图所示。这种快速的迭代更新方式在当今的需求快速变化的时代也显得尤为重要。

![img](/img/post/post_microservice.png)

但是，随着一个应用里微服务的数量逐渐增多，增加了部署和运维的工作量，如何把这些微服务配置部署到数据中心来让整个应用运行变得越来越困难。这里的难点在于某个微服务应该被调度到哪台服务器执行、当服务器crash时微服务该如何迁移，以尽可能提高资源的利用率，降低硬件成本。人为的配置和调度显然是不可能的。我们需要一个自动化工具，它帮我们完成自动调度微服务组件到某台合适的服务器、自动配置部署、持续地监视组件状态以及出错时的及时处理等工作。这就是Kubernetes的由来。

### Kubernetes架构

Kubernetes把底层的硬件基础设备抽象为一个计算资源池暴露给用户。它允许用户部署和运行微服务组件，而无需了解底层微服务被调度到了哪台服务器。当通过Kubernetes部署一个多组件的应用程序时，它自动为每个组件选一个合适的服务器部署在上面，并使该组件能被其他组件发现和与其他组件通信，来保证整个应用程序的顺利运行。下图展示了Kubernetes系统的角色示意，可以看到它将应用程序和服务器节点解耦，用户只需与Kubernetes交互。

![img](/img/post/post_k8sRole.png)

上图中可以看到Kubernetes被标为master，其实Kubernetes由一个master节点和一系列worker节点组成。用户先将部署任务提交到master节点，然后master节点调度部署到worker节点。更具体地讲，
+ master node: 保存Kubernetes的control panel，用于控制和管理整个Kubernetes系统；
+ worker node: 运行部署的应用程序；

![img](/img/post/post_k8sArchitecture.png)

如上图所示，master节点主要有以下组件：
+ Kubernetes API Server: 用户交互的门户，它与其他control panel组件通信；
+ Scheduler: 调度应用到某个worker节点；
+ Controller Manager: 执行集群级别的功能，例如跟踪worker节点的状态，节点错误的处理等；
+ etcd: 存储集群配置数据的分布式数据库；

运行应用程序的worker节点主要有以下组件：
+ Docker: 运行容器；
+ Kubelet: 和API Server通信，管理运行在这个node上的容器，确保容器处于健康的运行状态。
+ Kube-proxy: 节点的网络代理，维护节点上的网络规则，负责网络流量的负载均衡；

### Kubernetes运行流程

下图展示了应用如何部署到Kubernetes上。用户先将要部署的四种应用打包上传至DockerHub，方便Kubernetes稍后pull image，涉及到Docker和Image的下篇再讲。然后用户将App Descriptor提交至Kubernetes的master节点，可以看到App Descriptor包含四种应用，被分为三组，其实是指三个Pod，关于Pod的概念之后再讲。master节点调度特定数量的Pod到对应的worker节点，然后worker节点开始从Docker Hub拉取容器镜像，运行容器。

![img](/img/post/post_k8sRunApp.png)

参考自：
1. [Kubernetes 是什么?](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/)
2. [Kubernetes 组件](https://kubernetes.io/zh/docs/concepts/overview/components/)
3. [Kubernetes, 2020 快速入门](https://zhuanlan.zhihu.com/p/100644716)
4. Kuberneter in Action by Marko Luksa.

