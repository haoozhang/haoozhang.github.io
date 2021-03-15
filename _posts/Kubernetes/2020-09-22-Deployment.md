---
layout:     post
title:      Chapter 12. Deployments
subtitle:   updating applications declaratively
date:       2020-09-22
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

前两篇我们学习了各种部署和管理 Pod 的资源，你可能觉得 ReplicationController 或 ReplicaSet 是管理一群 Pod 的理想选择，但事实并非如此。他们在某些方面表现不佳，所以我们要引出另一种新的资源 — Deployment。

### 更新运行在 Pod 中的应用

我们从如何更新 Pod 中的应用程序说起。假设我们现在有一系列 Pod 通过 ReplicationController 或 ReplicaSet 管理，它们通过一个 Service 暴露出服务供外部 client 访问（Service下篇具体讲）。如下图所示，展示了我们当前的应用部署。

![img](/img/post/post_deployment_1.png)

初始情况下，假设这些 Pod 运行应用的 v1 版本。然后此时我们想更新应用到 v2 版本，一共可有三种方式。

+ 先删除所有 v1 版本的 Pod，再新建对应数量的 v2 版本的 Pod。

![img](/img/post/post_update_delete_create.png)

+ 先新建相同数量的 v2 版本的 Pod，再删除所有的 v1 版本 Pod。

![img](/img/post/post_update_create_delete.png)

+ rolling update：新建一个 v2 版本 Pod，删除一个 v1 版本 Pod，逐步更新所有。

![img](/img/post/post_rollingupdate.png)

分别对应以下三种图示，看图一目了然。

### Rolling update with ReplicationController

假如你已经运行了一个名为 kubia-v1 的 ReplicationController 来管理 Pod，Pod 中运行 v1 版本的应用。现在你想升级到 kubia-v2，你只需要使用 **kubectl rolling-update** 命令，在参数中指定要替换哪一个 controller、给定一个新的 ReplicationContrller的名字、以及新版本应用的镜像文件。

```
kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2
```

更新过程如以下几张图所示。

![img](/img/post/post_rc_update1.png)

![img](/img/post/post_rc_update2.png)

![img](/img/post/post_rc_update3.png)

#### Why kubectl rolling-update is now obsolete

即便可以利用 **kubectl rolling-update** 命令来更新 ReplicationController 的 Pod 版本，但他仍有些许弊端。第一，你可以从上面的更新过程中发现，在更新过程中它会自动修改 Pod 的 label 和 ReplicationController 的 label selector。我们不希望自己创建的资源被 Kubernetes 任意修改。

第二，要注意的是，我们是用 **kubectl** 去完成 rolling update 的（client end），而非 Kubernetes 的 Master Node（server end），这就会带来一些问题。比如，当 kubectl 在更新过程中网络突然中断，那要更新的 Pod 和 ReplicationController 就处于某种不确定的中间状态。

第三，这种更新方法并不人性化。我们之前在 scale out pod 时说过，我们只需要告诉 Kubernetes 我们的理想状态，并不需要指定去具体做什么，它会自动帮我们完成。同样地，现在我们也希望只改变 Pod 的版本标签（从 v1 改为 v2），它就帮我们自动 rolling update。正因为这些原因，我们才引入新的部署管理 Pod 的资源，Deployment。

### Rolling update with Deployment

Deployment 是部署和更新 Pod 的 high level 资源。当创建一个 Deployment 时，背后会创建一个 ReplicaSet，它会直接创建和管理 Pod，Deployment 与 Pod 并不直接打交道。

![img](/img/post/post_deployment_replicaset.png)

你应该还记得上一篇中讲过 ReplicaSet 是 ReplicationController 的替代品。那你也许会疑问，为什么要引入一个更高级别的 Deployment 把更新这个问题复杂化呢？正如上一节中 ReplicationController 的更新过程所示，rolling update 需要两个 ReplicationController 相互协调，所以我们需要一种资源去协调这个过程，Deployment就负责这个。

#### 创建 Deployment

创建过程与其他资源的创建一样非常简单，我们通过 **kubectl create** 命令从下面的描述文件创建即可。

```yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
  spec:
    containers:
    - image: luksa/kubia:v1
      name: nodejs
```

如上，我们创建一个 Deployment，它部署和管理 3 个 Pod。你可以查看创建的 Deployment，也可以查看它创建的 ReplicaSet 和 Pod。

```
$ kubectl get deployments
$ kubectl get replicasets
$ kubectl get pods
```

#### 更新 Deployment

你可以通过以下命令修改 Pod Template 的 image，当然你也可以通过 **kubectl edit deployment** 命令直接编辑 Deployment 的描述文件。

```
$ kubectl set image deployment kubia nodejs=luksa/kubia:v2
```

当镜像被修改后，Deployment就会开启 rolling update 过程。具体地，它会创建另一个 ReplicaSet，然后逐步 scale out，同时原来的 ReplicaSet scale down 为 0。这一系列步骤都是 Kubernets 的 controller panel 完成的。

![img](/img/post/post_deployment_update.png)

#### 回退 Deployment

在执行完 rolling update过程之后，旧的 RelicaSet 被 scale down to 0，但它别没有被删除，而是继续保存在 Deployment 中。这使得可以在某种场景下（比如，更新到新版本之后发现有bug）我们回退 Deployment 的版本。

```
$ kubectl rollout undo deployment kubia
```

我们可以查看 Deployment 的更新历史。如果是通过 **kubectl set image** 命令更新的，可以看到对应的 CHANGE-CAUSE；如果是通过 **kubectl edit deployment** 更新的，输出的 CHANGE-CAUSE 域为空。

```
$ kubectl rollout history deployment kubia
```

我们也可以选择回退到某一个具体的版本。

```
$ kubectl rollout undo deployment kubia --to-revision=1
```

#### Controlling the rate of the rollout

Rolling Update 的工作流程是，先创建一个新 Pod，当它变为 available 时，删除一个旧 Pod，此过程一直持续到替换掉所有旧 Pod。新 Pod 创建和旧 Pod 的删除是可以通过如下两个属性来配置的。

```
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
```

通俗点讲，maxSurge 属性指定在 rolling update 过程中可以超出 Desired Count 的 Pod 数目。maxUnavailable 属性指定了  rolling update 过程中相对于 Desired Count 有多少 Pod 可以暂时不能 available。具体的解释见下图。

![img](/img/post/post_deployment_maxsurge_maxunavailable.png)

下面两个示例展示了不同 maxSurge 和 maxUnavailable 配置的 rolling update 过程。

![img](/img/post/post_deployment_maxsurge.png)

![img](/img/post/post_deployment_maxunavailable.png)

#### Pausing the rollout process

如果你担心新版本的 Pod 可能有 bug，如果一股脑地把所有旧 Pod 全部升级会crash，你也可以先升级一个 Pod 看看它是否可以正常工作，如果可以正常工作了，再升级剩余的 Pod。这个想法也可以通过 Deployment 来实现。我们可以暂停 Deployment 的 rolling update 过程。

```
$ kubectl set image deployment kubia nodejs=luksa/kubia:v4
deployment "kubia" image updated

$ kubectl rollout pause deployment kubia
deployment "kubia" paused
```

此时 Deployment 应该建立了一个新 Pod，但是其他所有的旧 Pod 仍在继续运行。所以流向这个 Service 的部分请求会被定向到新 Pod。我们可以观察新 Pod 可否正常响应请求，来决定继续 rolling update，还是回退到旧版本。一旦你觉得新 Pod 可以稳定正常工作，你可以恢复它。

```
$ kubectl rollout resume deployment kubia
deployment "kubia" resumed
```

#### Blocking rollouts of bad versions

最后，我们再介绍一个 Deployment 资源的属性。minReadySeconds 属性指定了一个 Pod 在被视为 available 之前应该正常运行多长时间，在被视为 available 之前，rolling update 过程不会继续。这里的 Pod 稳定运行是指所有的 readiness 探针都要返回成功。

如下配置了一个描述文件，将 image 的版本升级到 v3。我们重点关注 minReadySeconds 和 readinessProbe 属性。可以看到，我们要求新创建的 Pod 要稳定运行 10s 才能继续下一个 rolling update。

```yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v3
        name: nodejs
        readinessProbe:
          periodSeconds: 1
          httpGet:
            path: /
            port: 8080
```

这次我们通过 **kubectl apply -f** 命令更新 Deplyoment。\
假如 v3 版本创建的新 Pod 不能稳定运行工作，那么流向这个 Service 的请求就不会重定向到新 Pod。因为，此时 Kubernetes 中的状态如下图所示。

![img](/img/post/post_deployment_blocked.png)

通过这样的配置，我们就防止了 Deployment 更新到一个有 bug 的版本。因为 rolling update 不会再继续了，我们要做的就是终止更新过程，让它回退到原来的版本。

```
$ kubectl rollout undo deployment kubia
deployment "kubia" rolled back
```

参考自：
1. Kuberneter in Action by Marko Luksa.

