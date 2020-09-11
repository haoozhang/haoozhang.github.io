---
layout:     post
title:      First Steps With Kubernetes
subtitle:   
date:       2020-08-15
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

我们已经学习了如何把应用程序打包成Docker镜像，并上传至Docker Hub以分享给他人使用。本文我们要做的是把打包的镜像部署运行到Kubernetes集群上，而非直接在Docker中运行。在此之前，我们首先需要建立一个Kubernetes集群。

### 建立本地的单节点集群Minikube

建立一个真正意义上的多节点Kubernetes集群并非易事，这涉及到多个分布的物理机和虚拟机，并且要配置合适的网络路由以保证之间的通信。当然现在各大云计算服务商提供了云上的Kubernetes集群平台，比如Google Kubernetes
Engine、Amazon Elastic Kubernetes Service、Azure Kubernetes Service等等，有兴趣的可以了解一下。本文中我们在本地建立Minikube，它在你电脑中的虚拟机上运行一个单节点的 Kubernetes 集群。

安装Minikube的详细步骤请参考[官方文档](https://kubernetes.io/zh/docs/tasks/tools/install-minikube/)。这里只简单地罗列一下安装所需的工具和步骤。

1. 检查系统是否支持虚拟化技术，不支持的可下载Virtual Box解决。
2. 安装Kubectl，它是和Kubernetes交互的客户端，可参考[这里](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)来安装。
3. 安装Minikube，可通过命令安装或直接下载安装，具体参考上面的官方文档。

要确认 Minikube 已成功安装，可以运行以下命令来启动本地 Kubernetes 集群：

```
$ minikube start
```

如果国内无法直接连接 k8s.gcr.io，推荐使用阿里云镜像仓库，在 minikube start 中添加 --image-repository 参数。

```
$ minikube start --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

若要为 minikube start 设置 --vm-driver，在下面提到 \<driver-name> 的地方， 用小写字母输入你安装的 VM 驱动程序的名称。 [这里](https://kubernetes.io/zh/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)列举了 --vm-driver 值的完整列表。

```
$ minikube start --vm-driver=<driver-name>
```

一旦 minikube start 完成，你可以运行下面的命令来检查集群的状态：

```
$ minikube status
```

如果你的集群正在运行，minikube status 的输出结果应该类似于这样：

```
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### 在Minikube上部署应用

由于这是第一次上手Kubernetes，我们选择用最简单的方式在集群上运行一个应用。但要注意的是，正常情况下我们应该准备一个.yaml文件，它描述了我们要如何部署应用的所有配置。

在Kubernetes上运行应用的最简单方式是 **kubectl run**命令。通过一条命令，它可以自动帮我们创建所需要的所有对象，我们暂时不用关注.yaml文件和各种对象。这里我们选择Docker Hub上已有的docker镜像演示，如下。

```
$ kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1
kubectl run --generator=run/v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
replicationcontroller/kubia created
```

**--image=luksa/kubia** 参数指定了我们想运行的镜像。**--port=8080**表示应用监听的端口为8080。**--generator**参数让Kubernetes创建了一个Replication Controller帮我们部署应用，这个对象我们之后再了解。可以看到一个warning说这条命令其实已经过时了，所以建议正式场景下不要再使用了，这里我们只做演示。

现在我们的应用其实已经部署在Kubernetes上了。为了验证这一点，我们可以使用下面的命令查看。

```
$ kubectl get pods
NAME          READY    STATUS    RESTARTS   AGE
kubia-4jfyf   1/1      Running   0          17m
```

这里我们又引入了一个名为 Pod 的对象。它是 Kubernetes 上最小的计算单元，有独立的IP、hostname等资源，一个Pod上可能运行多个容器，现在只需要知道这一些就好。通过查看Pod，我们可以看到kubia这个镜像的状态是running，代表部署成功已正常运行。如果状态显示为ContainerCreating或者是Pending，表示正在创建容器或在等待队列中排队，请稍等一会再查看。

### 部署命令背后

我们只输入了一条运行某个镜像的命令，Kubernetes就帮我们创建了Replication Controller对象（可以通过 **kubectl get rc** 查看）来部署镜像到某个Pod上。为了更好的理解这背后发生了什么，请看下图。

![img](/img/post/post_kubectl_run1.png)

我们首先把要部署的docker镜像上传到Docker Hub上，或者直接使用别人的镜像。当我们输入 **kubectl run** 的命令时，kubectl会向Kubernetes API Server发送一条REST HTTP请求来创建一个Replication Controller。然后RC会创建一个Pod，并由Kubernetes Master节点中的Scheduler将其调度到某个合适的Worker node。当这个Worker node上的kubelet发现该节点被调度了一个Pod，就会调用Docker从Docker Hub拉取运行的镜像。下载完镜像后，Docker会在这个Pod中创建容器并运行它。自此，我们可以通过**kubectl get pods**看到状态为running的Pod。

### 访问部署成功的应用

如何访问在Kubernetes上部署的应用呢？你可能会想到，利用Pod的IP地址我去访问。但这个IP是集群内部的地址，而且Pod的生命周期是不确定，它可能会被调度到其他node，也可能被销毁，这样就访问不到它了。

为了访问Pod上运行的应用，Kubernetes引入了 Service 的概念。它可以组织一系列运行相同应用的Pod，向外统一提供一个稳定的IP地址，这个地址不会随着某个Pod的调度和销毁而改变。因此，我们可以让Pod向外暴露一个 Service，我们通过Service访问应用，如下图所示。

![img](/img/post/post_service1.png)

首先，创建一个名为kubia-http、类型为NodePort的Service。

```
$ kubectl expose rc kubia --type=NodePort --name kubia-http
service "kubia-http" exposed
```

通过 **kubectl get service**可以查看。

```
$ kubectl get svc
kubernetes   ClusterIP  10.96.0.1      <none>   443/TCP          5d21h
kubia-http   NodePort   10.105.72.159  <none>   8080:31603/TCP   3s
```

因为我们使用的是Minikube本地集群，不能向外暴露一个公网的IP。我们可以通过 **minikube service**命令来暴露一个本地的IP地址。

```
$ minikube service kubia-http --url
 Starting tunnel for service kubia-http.
|-----------|------------|-------------|------------------------|
| NAMESPACE |    NAME    | TARGET PORT |          URL           |
|-----------|------------|-------------|------------------------|
| default   | kubia-http |             | http://127.0.0.1:53750 |
http://127.0.0.1:53750
```

我们可以通过如上述出的url来访问service，运行Pod上的应用。

```
$ curl http://127.0.0.1:53750
You’ve hit kubia-4jfyf
```

### 应用 Scale Out

现在我们已经部署运行并成功访问到了Kubernetes中的应用。接下来我们再展示一下Kubernetes的自动管理能力。之前我们也提到了Kubernetes的出发点就是作为一套方便Developer和Deployer的自动化管理工具，所以这里我们测试一下它如何水平扩展(scale out/in)。

此时的Pod是通过Replication Controller管理的，我们先查看一下它。

```
$ kubectl get replicationcontrollers
NAME    DESIRED     CURRENT     AGE
kubia   1           1           17m
```

上面展示了当前有一个名为kubia的Replication Controller，**DESIRED**表示我们想要的Pod数量，**CURRENT**表示了我们当前存在的实际数量。现在我们想要把Pod数量扩展为3。

```
$ kubectl scale rc kubia --replicas=3
replicationcontroller "kubia" scaled
```

通过以上命令，我们就把这个Replication Controller管理的Pod数量设置为3了。注意到我们只是设置了预想中的Pod数目，并没有具体的指定Kubernetes要做的动作。这也是它的自动化管理能力的一个体现，他会自己决定要做什么操作来实现我们想要达到的目标。

回到我们要做的事情上来，现在我们再来查看一下Replication Controller的数量。

```
$ kubectl get rc
NAME    DESIRED     CURRENT     READY   AGE
kubia   3           3           2       17m
```

可以看到Kubernetes正在scale out Pod的数目，当前有2个已经准备就绪了。再等一会儿，你就可以看到scale out完成。我们可以看一下当前的Pod。

```
$ kubectl get pods
NAME READY STATUS RESTARTS AGE
kubia-hczji 1/1 Running 0 7s
kubia-iq9y6 0/1 Running 0 7s
kubia-4jfyf 1/1 Running 0 18m
```

因为我们现在有多个运行的Pod，我们可以在测试运行一下应用。和上面一样，我们通过**minikube service**命令暴露一个url，然后访问它。

```
$ minikube service kubia-http --url
 Starting tunnel for service kubia-http.
|-----------|------------|-------------|------------------------|
| NAMESPACE |    NAME    | TARGET PORT |          URL           |
|-----------|------------|-------------|------------------------|
| default   | kubia-http |             | http://127.0.0.1:64544 |
http://127.0.0.1:64544
$ curl http://127.0.0.1:64544
You’ve hit kubia-hczji
$ curl http://127.0.0.1:64544
You’ve hit kubia-iq9y6
$ curl http://127.0.0.1:64544
You’ve hit kubia-iq9y6
$ curl http://127.0.0.1:64544
You’ve hit kubia-4jfyf
```

可以看到发送的请求被随机分配到某个Pod上，这就是Service帮我们做的。当只有一个Pod时，service会一直把请求分配到这个Pod，当存在多个运行相同应用的Pod时，service会随机的分配请求。下面的示意图可以与上面的图对比，帮助我们理解发生了什么。

![img](/img/post/post_service_multiple_pods1.png)

### Kubernetes dashboard

最后我们介绍一下kubernetes的可视化面板。目前我们一直是通过kubectl命令行来与kubernetes交互的，当然Kubernetes也提供了图形化的界面来查看和管理集群。你可以在命令行中运行以下命令，来在浏览器中打开dashboard。

```
$ minikube dashboard
```

默认浏览器会自动打开，你可以看到如下类似的界面。

![img](/img/post/post_minikube_dashboard1.png)

以上就是本文的全部内容。我们主要学习了如何在本地计算机上安装Minikube集群，然后在其上部署运行应用。在这之后我们了解了 scale out pod的数目，以及Kubernetes提供的dashboard。

参考自：
1. Kuberneter in Action by Marko Luksa.

