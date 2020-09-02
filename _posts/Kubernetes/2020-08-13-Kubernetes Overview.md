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


#### Container
**Container: 容器，集装箱;**\
集装箱运输能够获得成功，可以概括出如下几个特点： 
+ 可移植性：集装箱可以被任何类型支持的船舶使用
+ 包容性：支持多种类型的货物，这些货物都可以被打包在集装箱内
+ 标准大小：标准大小的集装箱可以被完美的组合在一起
+ 共存：多个集装箱可以放到同一个船上
+ 隔离：不同集装箱的货物间彼此隔离 

这些特点同样适用于软件领域的容器： 
+ 可移植性：容器可以被任何类型支持的操作系统安装使用 
+ 包容性：支持多种类型的软件，这些软件都可以被打包在容器内 
+ 标准格式 
+ 共存：多个容器可以运行在同一个物理机上
+ 隔离：不同容器的软件间彼此隔离

![img](/img/post/post_container.jpg)

#### 微服务架构
容器技术使软件更加的模块化，引领软件由单体架构转变向**微服务架构**。如下图所示：

![img](/img/post/post_microservice.jpg)

容器和微服务化后，带来了一些好处，比如： 
+ 模块间更加独立，可以独立的部署和发布，加快发布和更新速度 
+ 运行环境隔离，可以为不同模块定制不同的运行环境 

但微服务化带来的问题是：
+ 以前一个单独的服务被拆分成多个，增大了部署和运维工作量
+ 由于不同模块的软件依赖、网络配置、资源需求等各不相同，如果都直接使用 docker cli 来进行管理和配置，无疑是一件困难且容易出错的事，不能管理大规模的容器系统，因此就需要容器编排系统来处理。

由此引入 **k8s: 容器的编排管理系统**

#### Kubernetes 概念
从应用使用的角度介绍概念：\
[计算资源] 以 docker 为例，部署一个应用需要将软件打包在 docker image 内，然后将 docker image 部署在容器中，容器提供运行应用所需要的资源。但不能直接部署一个容器在 k8s 上，因为 k8s 中可部署的最小单元是 Pod。

**Pod**：k8s 的最小计算单元；包括一个或者多个容器，所有容器共享同样的独立 ip, hostname, 存储资源；是调度的最小单位，一个 pod 会放到一台宿主机上； 

[网络通信] 在微服务架构里，多个微服务间需要进行通信，当每个应用部署到 pod 上之后，这里存在两个挑战：
+ Pod 间：不能使用写死的 ip，因为 Pod 可能会漂移到其他节点上，更换 ip 
+ 对外：希望使用所有能够提供服务的 Pod，同时支持负载均衡 

因此 k8s 提供了 **Service** 的概念，Service 定义了一个 Pod 的逻辑组，这些 Pod 提供相同的功能服务。Service 根据类型不同分为如下几类： 
+ ClusterIP Service：这是默认的 Service 类型，该类型 Service 对外暴露一个集群内部 ip，可以用于集群内部服务间的通信 
+ NodePort Service：该类型 Service 对外暴露集群上所有节点的固定端口，支持外部通过 <NodeIP>:<NodePort> 进行访问 
+ LoadBalancer：该类 Service 通过云平台提供的 load balancer 向外提供服务 

此外，还可以使用 **Ingress** 来向外暴露服务，Ingress 虽然不是一个 Service 的类型，但也可以充当你应用集群的入口，通过编写路由规则实现流量路由。 

#### Kubernetes 集群组件

K8S集群组件：由代表控制平面的组件 (Master) 和一组称为节点的机器 (Workers) 组成

![img](/img/post/post_k8sArchitecture.png)

Kubelet: 集群中每个节点上运行的代理，它保证容器都运行在 Pod 中。接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。

Kube-proxy: 集群中每个节点上运行的网络代理，实现 Service  概念的一部分。维护节点上的网络规则，这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。 


参考自：
1. [Kubernetes 是什么?](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/)
2. [Kubernetes 组件](https://kubernetes.io/zh/docs/concepts/overview/components/)
3. [Kubernetes, 2020 快速入门](https://zhuanlan.zhihu.com/p/100644716)
4. Kuberneter in Action by Marko Luksa.

