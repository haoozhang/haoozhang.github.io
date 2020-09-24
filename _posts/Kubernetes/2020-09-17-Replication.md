---
layout:     post
title:      Chapter 6. ReplicationController
subtitle:   deploying managed pods
date:       2020-09-17
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

上篇我们介绍了 Pod 的创建和使用，但是手动创建的 Pod 必须要我们自己管理它的生命周期。在实际应用中我们并不直接手动创建 Pod，而是通过 Kubernetes 中的某些资源间接创建，然后由它来自动监视和管理这些 Pod。

### Liveness 探针: Keep Pods healthy

本系列第一篇中我们说过，Kubernetes 是作为一套自动化管理工具被提出的。当我们创建一个 Pod 后，Kubernetes 会调度到某一个合适的 Node 上启动运行容器，然后 Node 上的 kubelet 会负责维护容器的运行状态。当 Pod 中的容器终止（比如主进程crash）时，kubelet会自动重启这个容器。

但是并不是所有的异常状态都能被 kubelet 捕捉到从而重启容器，比如容器可能由于陷入死循环或者死锁状态无法响应外部的请求，此时 kubelet 并不能捕捉到这种异常，因为容器仍然处于 running 状态。为了能够及时的处理这种情况，我们必须周期性的从外部检测 Pod 中应用的运行状态，由此引入 Liveness 探针。

如下创建一个 Liveness 探针，它告诉 Kubernetes 周期性的执行 HTTP 请求访问应用的8080端口，来判断应用是否正常运行。

```yaml
apiVersion: v1
kind: pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```

查看创建的 Liveness 探针。

```yaml
$ kubectl describe po kubia-liveness
Name: kubia-liveness
....
Liveness: http-get http://:8080/ delay=0s timeout=1s period=10s \#success=1 \#failure=3
....
```

可以看到上面的这一行除了我们定义的，还描述了探针的一些其他信息 (delay, timeout, period等)。**delay=0s** 表示容器一旦启动探针就立刻开始工作；**timeout** 被设置为 1s，表示应用必须在 1s 内响应结果；然后探针每 10s 探查一次（**period=10s**）；如果失败次数连续超过 3 次就重启容器（**failure=3**）。

我们也可以自定义上面的这些参数。例如我们想设置探针在容器启动 15s 后再探查，如下图所示。

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15
```

其实一般都建议要设置这个参数，因为容器刚启动可能没有准备好响应请求，探针立刻探查的话可能会导致重启。另外要注意的是，应该尽可能让探针轻量化，一般我们在程序中设置 **/health** 路径以供探测。如果在工程中包含了 **spring-boot-starter-actuator** 依赖，它会自动探测 **/actuator/health** 路径。

### 引入 Replication Controller

ReplicationController 是确保它管理的 Pod 能一直保持运行的一种资源。它会一直监视管理的 Pod，当 Pod 因为某种原因消失或者失败了，它会监测到这种情况并创建替代的 Pod。我们通过下面这张图了解它的作用。

![img](/img/post/post_intro_rc1.png)

上面这张图只展示了 ReplicationController 管理一个 Pod，当然它也可以管理多个 Pod。ReplicationController的工作其实就是确保它管理的一组 Pod 按照设定的数量运行。当 Pod 数量少了，它就再创建一个。Pod 数量多了，它就选择一个终止运行。工作流程如下图所示。

![img](/img/post/post_intro_rc2.png)

所以，它主要是由三个必要的部分组成，label selector用来选择它管理哪些 Pod；Replicas Count指定理想中应该运行 Pod 的数量；Pod Template被用来创建新的 Pod。

![img](/img/post/post_rc_parts.png)

这里要注意一点，**Pod Selector 和 Pod Template 的改变只作用于之后创建的新 Pod，不会影响已有的 Pod**。举个例子，如果改变了 Pod Selector，那么已有的 Pod 会被排除在外（而非终止删除），此时 Replication 发现自己管理的新的 Selector 下 Pod 数为0，就新创建 Replicas Count 数量的 Pod。如果改变了 Pod Template，此前创建的 Pod 并不受影响，在之后创建的 Pod 会改用新的 template。

### 创建一个ReplicationController

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```

然后用 **kubectl create -f kubia-rc.yaml** 命令创建。此时集群会创建一个名为 kubia 的 ReplicationController，它管理标签为 **app: kubia** 的Pod，Pod 数量设为3，如果需要创建 Pod，则利用 template 中描述的容器和镜像。

查看创建的 Pod。

```
$ kubectl get pods
NAME          READY   STATUS                RESTARTS    AGE
kubia-53thy   0/1     ContainerCreating     0           2s
kubia-k0xz6   0/1     ContainerCreating     0           2s
kubia-q3vkg   0/1     ContainerCreating     0           2s
```

### 手动删除一个 Pod

ReplicationController 会察觉到并新创建一个。

![img](/img/post/post_rc_del_pod.png)

```
$ kubectl delete pod kubia-53thy
pod "kubia-53thy" deleted
$ kubectl get pods
NAME          READY   STATUS                RESTARTS    AGE
kubia-53thy   0/1     Terminating           0           3m
kubia-k0xz6   0/1     Running               0           3m
kubia-q3vkg   0/1     Running               0           3m
kubia-oini2   0/1     ContainerCreating     0           2s
```

### 修改 Pod 的 label

ReplicationController 就不再监视这个 Pod，并新创建一个。

![img](/img/post/post_rc_change_label.png)

```
$ kubectl label pod kubia-dmdck app=foo --overwrite
pod "kubia-dmdck" labeled
$ kubectl get pods -L app
NAME          READY   STATUS                RESTARTS    AGE   APP
kubia-2qneh   0/1     ContainerCreating     0           2s    kubia
kubia-oini2   1/1     Running               0           20m   kubia
kubia-k0xz6   1/1     Running               0           20m   kubia
kubia-dmdck   1/1     Running               0           10m   foo
```

### 修改 Pod Template

正如之前所说，这次改变只会影响后续创建的 Pod。

![img](/img/post/post_rc_change_template.png)

### Scale out Pods

可以通过修改 kubia 的 Replicas 来水平扩展 Pods。具体地，通过 **kubectl edit rc kubia** 命令会打开 kubia 的描述文件，然后修改 Replicas 后保存即可。还有另一种方式直接通过命令水平扩展。

```
$ kubectl scale rc kubia --replicas=10
```

### 删除 ReplicationController

通过 **kubectl delete** 命令删除 ReplicationController 时默认删除它管理的 Pod，但是也可以选择通过 **--cascade=false** 参数只删除 ReplicationController 保留 Pod，即如下图所示。

![img](/img/post/post_del_rc.png)

```
$ kubectl delete rc kubia 
$ kubectl delete rc kubia --cascade=false
```

### 小结

本篇我们学习了 ReplicationController - 一个管理 Pod 运行状态的资源。但是这个资源已经逐渐过时，在之后的文章里我们介绍功能更先进的管理 Pod 的方式。

参考自：
1. Kuberneter in Action by Marko Luksa.

