---
layout:     post
title:      Chapter 14. Understanding Kubernetes internals
subtitle:   
date:       2021-03-24
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

通过学习前面的篇章，我们已经可以很熟悉的使用 K8s 了。在此基础上，我们进一步深入 K8s 内部，了解内部是如何工作的。

## Understanding the architecture

在了解 K8s 内部如何工作之前，我们先回顾一下 K8s 的组成架构。在第一章中我们已经介绍到，K8s 分为两个部分：Control Plane 和 Worker Node。其中，Control Plane 主要负责控制和管理整个集群，由 etcd、API Server、Scheduler 和 Controller Manager 四部分构成；真正运行容器任务的是 Worker Node，它是由 Kubelet、Kube-Proxy 和 Container Runtime 组成。

除了 Control Plane 和 Worker Node，K8s 一般也会包含一些附加组件来完成额外的功能。例如：
+ K8s DNS Server
+ Dashboard
+ Ingress Controller
+ Heapster (之后介绍)
+ Container Network Interface network plugin (之后介绍)

### The distributed nature of Kubernetes components

上面提到的组件都以单独的进程运行，组件之间的依赖关系如下图所示。

![img](/img/post/K8sInternal/post_11.1.png)

为了获取 K8s 提供的所有功能，要让所有组件都正常运行。可以通过以下命令查看 Control Plane 组件的状态。

```
$ kubectl get componentstatuses
```

如上图所示，系统组件只与 API Server 通信，它们之间不直接通信。例如，其他组件不能直接与 etcd 通信，而是通过访问 API Server 到 etcd 修改集群状态。

尽管 Worker Node 上的组件要运行在同一个 Node，但 Control Plane 组件可以拆分到多个服务器上，每个组件可以有多个实例在运行，以确保高可用性。要注意的是，etcd 和 API Server 的多个实例可以同时处于活动状态并执行操作，而 Scheduler 和 Controller Manager 的实例在给定时间只有一个能处于活动状态，其他需待机。

Control Plane 组件要么直接部署在系统中，要么以 Pod 的方式运行。因此，Kubelet 也被部署在 Master Node。Kubelet 是始终作为系统组件运行的唯一组件，它将所有的其他组件以 Pod 形式运行。

### How Kubernetes uses etcd

我们创建的所有对象都要持久化存储在某个位置，以方便 API Server 重启或失效时也能保存创建对象的描述文件。为此，K8s 使用 etcd，这是一个分布式的一致 key-value 存储，可以同时运行多个实例来保证可用性和性能。

etcd 是保存集群信息和元数据的唯一存储，只可以通过 API Server 来访问。其他所有组件只能通过 API Server 读取或写入数据到 etcd，这样设计有很多好处，包括鲁棒的乐观锁设计和验证功能，把实际存储抽象出来方便将来轻松的替换。

数据以 key-value 形式保存在 etcd 中，每个 key 要么是一个包含其它 key 的目录，要么是一个对应 value 的真实 key。etcd 把所有的数据保存在 */registry* 目录下，我们可以通过如下命令查看，可以看到，Pod 的描述文件以 JSON 格式保存在 etcd 中。etcd 以等级化的方式保存所有资源。

```bash
$ etcdctl ls /registry
/registry/configmaps
/registry/daemonsets
/registry/deployments
/registry/events
/registry/namespaces
/registry/pods

$ etcdctl ls /registry/pods
/registry/pods/default
/registry/pods/kube-system

$ etcdctl ls /registry/pods/default
/registry/pods/default/kubia-159041347-xk0vc /registry/pods/default/kubia-159041347-wt6ga /registry/pods/default/kubia-159041347-hp2o5

$ etcdctl get /registry/pods/default/kubia-159041347-wt6ga
{"kind":"Pod","apiVersion":"v1","metadata":{"name":"kubia-159041347-wt6ga","namespace":"default","selfLink":...
```

K8s 的前身 Borg 只使用一个集中化的存储保存集群状态，并允许多个 Control Plane 组件都可直接访问存储。所以这些组件都需要实现相同的乐观锁机制来处理并发冲突。K8s要求只能通过 API Server 间接访问，因为只需要在一处实现乐观锁，保证了数据一致性。

多个 etcd 实例构成的的分布式系统为了保证一致性，使用了 Raft 一致性算法。每个节点的状态要么是多数节点投票的当前状态，要么是之前同意的状态之一。etcd 通常部署奇数个节点，这是因为 Raft 是一种基于多节点投票选举机制的算法，只有超过半数 (N/2+1) 节点在线才能提供服务。举例来说，3节点集群需要2个以上节点在线，5节点集群需要3个以上节点在线，等等。对于偶数节点的集群，2节点集群需要2节点同时在线，4节点集群需要3节点在线。

### What the API Server does

API Server 是被其他组件访问的核心组件，例如 *kubectl* 向 API Server 发送 CURD 请求查询或修改集群状态。除了提供在 etcd 存储对象的一致性以外，API Server 也验证这些存储的对象，不合语法的对象不会被存储；它也处理乐观锁机制，防止资源对象被并发修改。

以 *kubectl create* 为例，当 *kubectl* 提交了一个资源的描述文件后，下图展示了 API Server 内部的运行流程。

![img](/img/post/K8sInternal/post_11.3.png)

首先，API Server 通过一个或多个 *认证插件* 认证发送请求的 client，直到认证出发送的 client，client 信息可以从 HTPP 请求头或证书中提取出来，API Server 接着利用 *授权插件* 判断请求的 client 是否有权限执行要求的操作。如果请求是 create、update、delete 资源对象，则会被传送到 *Admission Control*。它针对于修改资源的场景，主要负责初始化资源描述中未明确指定的描述、修改描述中未提到的其他相关资源。（如果请求是 Read 操作，则不会被传送到 Admission Control 插件）

Admission Control 插件包括：

+ AlwaysPullImages：Pod 的 imagePullPolicy 覆盖为 always，保证 Pod 每次部署时都 pull image；
+ ServiceAccount：未明确指定时，Pod 应用默认的 Service Account；
+ NamespaceLifecycle：防止在正在删除的命名空间以及不存在的命名空间中创建 Pod；
+ ResourceQuota：确保 Pod 最多使用分配给这个 Namespace 的 CPU 和 Memory；

API Server 就完成上述的工作，它不会去创建 Pod，也不会管理 Service 的 Endpoints，这些都是 Controller Manager 中的 controllers 做的。但是 Control Plane 可以要求当资源发生变化时被通知到，client 可以通过连接到 API Server 的 HTTP 连接观察到这些变化。下图展示了具体的流程。

![img](/img/post/K8sInternal/post_11.4.png)

*kubectl* 也支持监听资源变化。我们可以通过 *--watch* 参数查看变化过程。

```
$ kubectl get pods --watch
```

### Understanding the Scheduler

我们已经知道，一般不会指定 Pod 在哪个 Node 上运行，调度的任务留给 Scheduler 来完成。Scheduler 并不会直接指定哪个 Node 运行 Pod，它是通过 API Server 来修改 Pod 的描述定义，然后 API Server 通知对应 Node 的 Kubelet，一旦 Kubelet 收到 Pod 被调度到该 Node，它就会创建 Pod 并运行容器。

为 Pod 选择最佳 Node 并不简单。最简单的调度方法是随机选择一个 Node，不用关注这个 Node 是否已经运行了 Pod。另一方面，可以使用机器学习的方法预测接下来的时间段内要调度什么类型的 Pod，以实现硬件最大化利用。K8s 的默认调度方法介于两者之间。

选择一个 Node 的方法可以分为两部分，首先，获取一个可以调度这个 Pod 的 Node 列表。然后，按照优先级排列这些 Node，选择一个最合适的。如下图所示。

![img](/img/post/K8sInternal/post_11.5.png)

为了判断 Node 是否可以调度 Pod，Scheduler 通过以下一系列条件检查 Node：

+ Node 能否满足 Pod 的硬件需求；
+ Node 资源是否已经耗尽；
+ Pod 是否已经指定了调度的 Node；
+ Pod 描述定义中是否有 node selector 的 label；
+ Pod 是否要求绑定特定的主机端口，该端口是否已经在该 Node 上使用；
+ 如果 Pod 要求特定类型的 Volume，是否可以挂载在该 Node，是否已被占用；

通过以上检查后，可以获得一个 Node 子集，但这些 Node 要有一个优先级排列。例如，假设你有一个两节点集群，两个节点都可以被调度，但一个节点已经运行了10个 Pod，而另一个节点还没运行 Pod。很明显这时应该调度到第二个 Pod。但假设这两个节点集群是云提供商提供的，则最好将 Pod 调度到第一个节点，这样可以回收第二个节点以节省花费。

再举一个例子，假设一个应用运行多个 Pod 实例，那我们应该尽可能让它分布在多个节点上。如果都调度在一个节点上，该节点失效将导致整个应用不可访问。我们也可以定义某些规则来强制 Pod 分散调度或调度在一起。K8s 允许配置 Scheduler 来适应特定的需求，也允许在集群中运行多个 Scheduler，每个 Pod 可以指定特定的 Scheduler。

### Introducing the controllers running in the Controller Manager

正如之前所述，API Server 除了在 etcd 存储资源和通知资源变化外，不关注其它事情。Scheduler 只负责决定把 Pod 调度到哪一个 Node。所以需要另一种组件来确保系统的状态与资源描述中定义的理想状态一致，这是由 Controller Manager 中的 controllers 完成的。controllers 主要包括以下：

+ Replication Manager (a controller for ReplicationController resources) 
+ ReplicaSet, DaemonSet, and Job controllers
+ Deployment controller 
+ StatefulSet controller 
+ Node controller
+ Service controller
+ Endpoints controller
+ Namespace controller
+ PersistentVolume controller 
+ Others

大体上，controllers 都监听对应资源的变化，并对每个变化执行特定的操作。它们执行一个循环，协调真实状态与资源描述中定义的理想状态，并把新的真实状态写入资源的 *status* 位置。controllers 彼此之间不直接通信，它们都是与 API Server 连接，通过监听机制要求当对应的资源变化时被通知到。下图展示了 Replication Manager 的操作。

![img](/img/post/K8sInternal/post_11.6.png)

### What the Kubelet does

Kubelet 负责运行在 Worker Node 的一切东西。首先，它在 API Server 上创建一个 Node 资源来注册这个 Node，然后它需要持续监听 API Server，当有 Pod 被调度到这个 Node 上时，它就通过 container runtime 来启动 Pod 的容器。之后继续监听运行的容器，并向 API Server 报告容器状态。

Kubelet 同时负责运行容器的 Liveness Probe，当探查失败时会重启容器。最后，当 Pod 被 API Server 删除时，Kubelet 终止运行的容器。

除了从 API Server 拿到 Pod 的描述文件，Kubelet 也可以运行指定本地目录下的 Pod 描述文件，这主要是用来将 Control Plane 的容器版本运行为 Pod。之前我们说过，Control Plane 要么直接部署在系统中，要么以 Pod 方式运行。因此，可以讲 K8s 系统组件的 Pod 描述放在 Kubelet 的 manifest 目录让它运行和管理，如下图所示。

![img](/img/post/K8sInternal/post_11.6.png)

### The role of the Kubernetes Service Proxy

除了 Kubelet，每个 Worker Node 也运行一个 Kube-proxy，它用于确保 client 访问到 Service 的连接最后到达 Service 背后的某一个 Pod，当 Service 代理多个 Pod 时，会在这些 Pod 之间做负载均衡。



参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

