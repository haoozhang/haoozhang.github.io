---
layout:     post
title:      Chapter 11. Accessing Metadata
subtitle:   Accessing pod metadata and other resources from applications
date:       2020-10-07
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

应用经常要访问它运行的环境信息，包括它自己的信息和集群中其他组件的信息。我们可以学习了 Kubernetes 通过环境变量和 DNS 发现服务，但其他信息不知道如何访问。本章我们首先学习把某个 Pod 和 container 的元数据传递给 container，然后学习 container 中的应用如何访问 Kubernetes API Server 以获得集群的资源信息。

### Passing metadata through the Downward API

在上一篇章中我们学习了如何通过环境变量、ConfigMap 和 Secret 给应用传递配置数据，这些数据都是自己设置的或者在 Pod 被创建之前就已经确定的。但是如何获取其他还不确定的信息呢？比如 Pod 的 IP 和名字、运行 Node 的名字等等。

Kubernetes 提供了 Downward API 来解决这个问题。它允许我们通过环境变量或者文件的形式传递 Pod 的元数据和环境信息。如下图所示。

![img](/img/post/post_pass_a_configmapentry_as_env.png)

#### Understanding the available metadata

Download API 允许我们把 Pod 的元数据暴露给运行在这个 Pod 的进程。当前可以传递的数据主要有以下：

+ Pod 名字；
+ Pod IP；
+ Pod 所在 Namespace；
+ 运行 Pod 的 Node 名字；
+ 运行 Pod 的 Service Account 名字；
+ 每个 container 的 CPU、内存需求；
+ 每个 container 的 CPI、内存限制；
+ Pod 的 label；
+ Pod 的 annotations；

上面大部分信息我们都接触到了，不熟悉的信息之后篇章我们会详细学习。大多数信息都可以通过 *环境变量* 或 *downwardAPI volume* 传递给 container，但 Pod 的 label 和 annotations 只可以通过 volume 传递。

#### Exposing metadata through environment variables

我们先学习通过环境变量传递 Pod 和 container 的元数据。我们先创建一个单 container Pod。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers: 
  - name: main
    image: luksa/kubia
    resources:
      requests:     # CPU和Memory的需求
        cpu: 15m
        memory: 100Ki
      limits:   # CPU和Memory的上限
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:    # 没有指定一个绝对值，而是引用了Pod描述文件的metadata.name
        fieldRef:
          fieldPath: metadata.name  
    - name: POD_NAMESPACE 
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:    # 通过resourceFieldRef引用容器的CPU和Memory需求
        resourceFieldRef: 
          resource: requests.cpu 
          divisor: 1m  # 定义divisor表明需要的单位值
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
      resourceFieldRef: 
        resource: limits.memory 
        divisor: 1Ki
```

当我们创建的 Pod 开始运行时，它就会查询 Pod 描述文件中定义的所有环境变量。下图展示了环境变量及其值的来源。

![img](/img/post/post_metadata_env.png)

在创建 Pod 后，我们可以使用 *kubectl exec* 命令查看容器内的所有环境变量。运行在这个容器中的所有进程都可以访问并使用这些环境变量。

```
$ kubectl exec downward env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=downward
POD_NAME=downward
POD_NAMESPACE=default
POD_IP=172.17.0.4
NODE_NAME=minikube
SERVICE_ACCOUNT=default
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBIA_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT_HTTPS=443
....
```

#### Passing metadata through file in DownwardAPI Volume

第二种方式是通过文件来暴露 Pod 的元数据。我们可以定义一个 downwardAPI Volume 并把它挂载在容器上。但是 Pod 的 label 和 annotations 只能通过 downwardAPI 来暴露。

我们修改上面的 Pod 描述来使用 volume。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:       # label和annotations通过downward API 暴露
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers: 
  - name: main
    image: luksa/kubia
    resources:
      requests:     # CPU和Memory的需求
        cpu: 15m
        memory: 100Ki
      limits:       # CPU和Memory的上限
        cpu: 100m
        memory: 4Mi
    volumeMounts:   # 在/etc/downward目录下挂载downward volume
    - name: downward 
      mountPath: /etc/downward
  volumes:
  - name: downward  # 定义名为downward的downwardAPI volume
    downwardAPI:
      items:
      - path: "podName"   # metadata.name 写到 podName 文件
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels" 
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores" 
        resourceFieldRef:   # 容器main的requests.cpu写到上面的文件
          containerName: main 
          resource: requests.cpu 
          divisor: 1m
      - path: "containerMemoryLimitBytes" 
        resourceFieldRef:
          containerName: main 
          resource: limits.memory 
          divisor: 1
```

下图展示了 Pod 描述文件的 downwardAPI Volume。

![img](/img/post/post_metadata_downward.png)

我们可以查看 */etc/downward* 目录下的文件。

```bash
$ kubectl exec downward ls  /etc/downward 
annotations
labels
podName
podNamespace
....
$ kubectl exec downward cat /etc/downward/labels
foo="bar"
$ kubectl exec downward cat /etc/downward/annotations
key1="value1"
key2="multi\nline\nvalue\n"
kubernetes.io/config.seen="2021-03-18T12:32:13.196520100Z"
kubernetes.io/config.source="api"
```

可以看到，每个 label/annotation 都以 key/value 对的形式分行展示。我们之前提到过，在 Pod 运行过程中我们也可以修改 label 和 annotations，此时 Kubernetes 就会更新文件，让 Pod 一直看到最新的数据。这也就解释了，为什么 label 和 annotations 只能通过 downwardAPI 来传递了，因为环境变量在 Pod 运行过程中不能再修改。

最后再总结一下通过 downwardAPI 暴露 container-level metadata。当 downwardAPI Volume 暴露 container-level 的 metadata 时，要指定 container 的名字。这是因为 Volume 是 Pod 层面的对象，不是容器层面的对象。

使用 Volumes 暴露容器的资源比通过环境变量稍微复杂。但好处是，Volume可以把一个容器的资源信息暴露给同一个Pod的另一个容器；而环境变量的方式只能传递自己的资源限制和需求。

### Talking to the Kubernetes API server



参考自：
1. Kuberneter in Action by Marko Luksa.

