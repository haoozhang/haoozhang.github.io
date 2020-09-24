---
layout:     post
title:      Chapter 11. ConfigMaps&Secrets
subtitle:   configuring applications
date:       2020-10-07
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

上篇我们介绍了 Pod 的创建和使用，但是手动创建的 Pod 必须要我们自己管理它的生命周期。在实际应用中我们并不直接手动创建 Pod，而是通过 Kubernetes 中的某些资源间接创建，然后由它来自动监视和管理这些 Pod。

### Keep Pods healthy - Liveness 探针

引入，创建，配置属性

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

可以看到上面的这一行除了我们定义的，还描述了探针的一些其他信息，诸如 delay, timeout, period。**delay=0s** 表示容器一旦启动探针就立刻开始工作；**timeout** 被设置为 1s，表示应用必须在 1s 内响应结果；然后探针每 10s 探查一次（**period=10s**）；如果失败次数连续超过 3 次就重启容器（**failure=3**）。

我们也可以自定义上面的这些参数。例如我们想设置探针在容器启动 15s 后再探查，如下图所示。

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15
```

其实一般都建议要设置这个参数，因为容器刚启动可能没有准备好响应请求，探针立刻探查的话可能会导致重启。另外要注意的是，应该尽可能让探针轻量化，一般我们在程序中设置 **/health** 路径以供探测。如果在工程中包含了 **spring-boot-starter-actuator** 依赖，它会自动探测 **/actuator/health** 路径。

### Replication Controller

引入（一张图）
创建
改变Pod  删除、修改pod的label
修改template，
水平扩展
删除

### ReplicaSets

对比
创建
更复杂的 label selector
删除

### DaemonSets

引入一张图
只在某些Node上运行

### Job

### CronJob


参考自：
1. Kuberneter in Action by Marko Luksa.

