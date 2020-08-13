---
layout:     post
title:      Kubernetes 入门
subtitle:   
date:       2020-08-13
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

#### Container

Container: 容器，集装箱；

集装箱运输能够获得成功，可以概括出如下几个特点： 

1. 可移植性：集装箱可以被任何类型支持的船舶使用 

2. 包容性：支持多种类型的货物，这些货物都可以被打包在集装箱内 

3. 标准大小：标准大小的集装箱可以被完美的组合在一起 

4. 共存：多个集装箱可以放到同一个船上 

5. 隔离：不同集装箱的货物间彼此隔离 

这些特点同样适用于软件领域的容器： 

1、可移植性：容器可以被任何类型支持的操作系统安装使用 

2、包容性：支持多种类型的软件，这些软件都可以被打包在容器内 

3、标准格式 

4、共存：多个容器可以运行在同一个物理机上

5、隔离：不同容器的软件间彼此隔离

![img](/img/post/post_container.jpg)

#### 微服务架构

容器技术使软件更加的模块化，引领软件由单体架构转变向微服务架构。如下图所示：

![img](/img/post/post_microservice.jpg)

容器和微服务化后，带来了一些好处，比如： 

1、模块间更加独立，可以独立的部署和发布，加快发布和更新速度 

2、运行环境隔离，可以为不同模块定制不同的运行环境 

但微服务化带来的问题是：

1、以前一个单独的服务被拆分成多个，增大了部署和运维工作量

2、由于不同模块的软件依赖、网络配置、资源需求等各不相同，如果都直接使用 docker cli 来进行管理和配置，无疑是一件困难且容易出错的事，不能管理大规模的容器系统，因此就需要容器编排系统来处理。

由此引入 k8s：容器的编排管理系统

#### Kubernetes 概念
从应用使用的角度介绍概念：
[计算资源] 以 docker 为例，部署一个应用需要将软件打包在 docker image 内，然后将 docker image 部署在容器中，容器提供运行应用所需要的资源。但不能直接部署一个容器在 k8s 上，因为 k8s 中可部署的最小单元是 Pod。 
Pod：k8s 的最小计算单元；包括一个或者多个容器，所有容器共享同样的独立 ip, hostname, 存储资源；是调度的最小单位，一个 pod 会放到一台宿主机上； 
[网络通信] 在微服务架构里，多个微服务间需要进行通信，当每个应用部署到 pod 上之后，这里存在两个挑战： 
Pod 间：不能使用写死的 ip，因为 Pod 可能会漂移到其他节点上，更换 ip
对外：希望使用所有能够提供服务的 Pod，同时支持负载均衡 
因此 k8s 提供了 Service 的概念，Service 定义了一个 Pod 的逻辑组，这些 Pod 提供相同的功能服务。Service 根据类型不同分为如下几类： 
ClusterIP Service：这是默认的 Service 类型，该类型 Service 对外暴露一个集群内部 ip，可以用于集群内部服务间的通信 
NodePort Service：该类型 Service 对外暴露集群上所有节点的固定端口，支持外部通过 <NodeIP>:<NodePort> 进行访问 
LoadBalancer：该类 Service 通过云平台提供的 load balancer 向外提供服务 
此外，还可以使用 Ingress 来向外暴露服务，Ingress 虽然不是一个 Service 的类型，但也可以充当你应用集群的入口，通过编写路由规则实现流量路由。 

#### K8S集群组件

K8S集群组件：由代表控制平面的组件和一组称为节点的机器组成

![img](/img/post/post_k8sarchitecture.png)

Kubelet: 集群中每个节点上运行的代理，它保证容器都运行在 Pod 中。接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。
Kube-proxy: 集群中每个节点上运行的网络代理，实现 Service  概念的一部分。维护节点上的网络规则，这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。 

参考自：
1. [递归公式的理解](https://leetcode-cn.com/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/solution/nan-dian-shi-di-gui-gong-shi-de-li-jie-by-piao-yi-/)


