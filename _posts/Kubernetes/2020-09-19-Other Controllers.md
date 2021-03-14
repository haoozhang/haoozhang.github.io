---
layout:     post
title:      Chapter 7. Other Controllers
subtitle:   deploying managed pods
date:       2020-09-19
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

上一篇我们学习了 ReplicationController，这一篇我们介绍其他几种管理 Pod 的 Controller。

### ReplicaSets

即便我们可以通过 ReplicationController 来管理 Pod，但这种资源逐渐过时。官方推荐我们用 ReplicaSet 代替 ReplicationController。这两种资源非常相似，唯一不同的就是 ReplicaSet 的 Pod Selector 可以更复杂，它可以只根据 label 的 key 值选择 Pod。我们可以通过下面的例子具体了解。

在一般使用过程中我们并不手动创建 ReplicaSet，而是由 Deployment 资源来自动管理它。当然这里我们手动创建一个。

```yml
apiVersion: apps/v1beta2    # ReplicaSet属于v1beta2版本(不是v1)
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:    # 使用matchLabels选择符
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```

然后通过 **kubectl create** 命令从这个描述文件创建就好了。可以看到与上一篇中 ReplicationController 描述文件唯一不同的就是，RC 是用 selector 选择 Pod，而这里使用 selector.matchLabels。

当然会有更复杂的 Pod Selector，可以利用更复杂的 matchExpressions 属性，它必须包括一个key、一个operator、可能有一系列values。如下。

```yml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - kubia
```

operator 如下。

+ In - label 的 value 必须匹配指定的其中一个；
+ NotIn - label 的 value 一定不能匹配指定的任何一个；
+ Exists - Pod 必须包含指定的 key，这里不指定 values；
+ DoesNotExist - Pod 不能包含指定 key 的 label，这里不指定 values；

### DaemonSets

ReplicationController 和 ReplicaSet 都负责运行部署在 Kubernetes 中任意 Node 的若干个 Pod。但在某种情况下，我们需要在集群的每个 Node 上只运行一个这种类型的 Pod，比如在每个 Node 上运行一个 log collector 和 resource monitor，或者我们第一篇文章中说到的 kube-proxy 进程是要在每个 Node 上运行来保障 Service 工作的。

这时我们可以创建一个 DaemonSets 来实现，它与 ReplicaSet 很相似，但不同的是它在每个 Node 上都只部署一个 Pod。区别如下图所示。

![img](/img/post/post_daemonset.png)

除了在每个 Node 上运行某个 Pod，DaemonSets 也可以指定 Pod 在某些 Node 上运行，比如我们有时需要在配有 ssd 的 Node 上执行任务。这个可以通过 nodeSelector 实现。

![img](/img/post/post_ssd_monitor.png)

```yml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:   # Pod Template包括node selector，以选择特定的node
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

通过 *kubectl crate* 命令创建 DaemonSet。

```
$ kubectl create -f ssd-monitor-daemonset.yaml
daemonset "ssd-monitor" created
```

给集群中的节点添加 label。

```
$ kubectl label node minikube disk=ssd
node "minikube" labeled
```

查看 Pod 状态。

```
$ kubectl get po
NAME                READY     STATUS    RESTARTS   AGE
ssd-monitor-hgxwq   1/1       Running   0          35s
```

此时我们再把节点的label修改了。

```
$ kubectl label node minikube disk=hdd --overwrite
node "minikube" labeled
```

就会看到创建的 Pod 正在被销毁。

```
$ kubectl get po
NAME                READY     STATUS        RESTARTS   AGE
ssd-monitor-hgxwq   1/1       Terminating   0          4m
```

### Job

上面我们考虑的都是持续运行的 Pod。如果我们只需要运行一个可以正常结束的应用，我们可以通过创建一个 Job来实现，它允许运行一个进程正常结束后不会再重启容器的 Pod。在节点故障时，Job 管理的 Pod 会被重新调度到其他节点；在进程自身故障（例如进程返回error exit code）时，Job会被配置是否自动重启容器。

![img](/img/post/post_job.png)

Job 的定义如下面的描述文件所示。注意 restartPolicy 设置为 OnFailure，表示正常结束后不再重新启动，否则默认为 Always。

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:   # 不用指定Pod Seletor，它会基于template的label创建
      labels:
        app: batch-job
  spec:
    restartPolicy: OnFailure    # 不能使用默认的Always
    containers:
    - name: main
      image: luksa/batch-job    # 这个镜像会运行120秒然后停止
```
Job 管理的 Pod 执行结束后不会自动删除，因为要方便用户查看 log。

Job 也可以设置多次运行，可以在 spec.completions 属性设置次数。

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5    # 运行5次
  template:
  <same as above>
```

上面的 Job 是串行运行的，如果要并行运行，可以设置 spec.parallelism 属性。

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  parallelism: 2    # 最多2个Pod并行运行
  template:
  <same as above>
```

也可以在 Job 运行过程中修改并行数目。如下所示，我们将并行数目增加到3，那此时就会立即新建另一个 Pod，三个 Pod 一起运行。

```
$ kubectl scale job multi-completion-batch-job --replicas 3
```

最后，我们可以通过 **spec.activeDeadlineSeconds** 属性限制一个 Pod 完成 Job 的时间。如果在设定时间内完成，则正常完成结束。否则，系统会终止 Pod 并把任务标记为失败。在这里，也可以通过 spec.backoffLimit 属性配置在被标记为失败之前 Pod 可以尝试多少次，默认为6次。

### CronJob

当创建 Job 资源后，Job 会立即创建运行 Pod。许多任务也需要在将来的特定时间运行或特定间隔内周期运行。当我们需要运行周期性的任务时，我们可以创建 CronJob 资源。

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"
  jobTemplate:    # CronJob定时的创建这个job
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

上面的描述文件中 spec.schedule 属性指定每隔15分钟运行一次任务，这里用到了定时表达式，如果不熟悉可以Goole简单学习。

事实上，CronJob 是大概在指定的周期时间创建 Job，然后 Job 创建 Pod，所以 Job 和 Pod 的创建可能稍微迟一些，这种情况下可以通过 startingDeadlineSeconds 属性指定一个 deadline。

```yml
apiVersion: batch/v1beta1
kind: CronJob
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15
  ...
```

上面的描述文件中，假设任务在 10:30 被调度运行，如果它在 10:30:15 还没开始，系统就认定它失败。

### 小结

本篇我们介绍了其他的集中管理 Pod 的资源，主要有 ReplicaSet, DaemonSet, Job, CronJob，不同的资源对应于不同的需求和应用场景。

参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

