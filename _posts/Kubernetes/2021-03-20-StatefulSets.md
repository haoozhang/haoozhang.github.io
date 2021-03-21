---
layout:     post
title:      Chapter 13. StatefulSets
subtitle:   deploying replicated stateful applications
date:       2021-03-20
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

到目前为止，我们已经学会运行单实例的Pod、复制的多实例的Pods、以及使用持久化存储的 Pod。例如，我们可以运行多个复制的 web-server Pods 和一个单实例的 database Pod，其中 database Pod 使用简单的 Pod Volume 或者 Persistent Volume 提供持久化存储。但是我们可以使用 ReplicaSet 复制 database Pod 吗？

### Replicating stateful pods

ReplicaSets 资源是从一个 Pod Template 中创建多个 Pod 实例。这些复制的 Pod 除了 name 和 IP 地址不同，其他完全一样。所以，如果 Pod 包含一个引用 PersistentVolumeClaim 的 Volume，那么这个 ReplicaSets 创建的所有 Pod 都使用相同的 PVC，进而使用相同的 PV。如下图所示，因为 PVC 的引用是在 Pod Template 中，所以我们不能让一个 ReplicaSets 创建的多个 Pod 分别有自己的 PVC。

![img](/img/post/StatefulSet/post_samePVC.png)

#### 运行多个复制 Pod，每个 Pod 有自己独立的存储

第一种方式，我们可以手动创建 Pod，让每个 Pod 有自己的 PVC。但是因为没有 ReplicaSets 管理，我们需要全程手动管理，这显然不是一种好方法。

所以引入第二种方式，我们并不直接手动创建 Pod，而是创建多个 ReplicaSets，每个 ReplicaSets 管理一个 Pod。这样每个 ReplicaSets 的 Pod Template 就可以引用特定的一个 PVC。如下图所示。

![img](/img/post/StatefulSet/post_eachRS.png)

但这种方式也略显笨重，比如当 scale out Pod 时，我们必须创建额外的 ReplicaSets。

还有一种方式是使用相同的 PVC，但每个 Pod 的 volume 有不同的文件目录。我们不能通过 Pod Template 来配置每个 Pod 来选择哪个目录。但是可以让每个 Pod 自动创建或选择一个没有被其他 Pod 使用的目录。这种方式就要求 Pod 之间协调。

![img](/img/post/StatefulSet/post_eachDir.png)

#### 给每个 Pod 提供一个稳定的标识

除了存储，某些集群化的应用也要求每个实例要有一个稳定的标识。例如在分布式的集群应用环境中，每个成员要获得集群其他成员的 IP。但 Kubernetes 一旦调度了一个 Pod，那么这个 Pod 就是全新的 Pod，它有新 IP。所以这时候每个成员就要重新配置。

我们解决这个问题的一种方式是，可以为每个 Pod 单独创建一个 Service，这样就会有唯一的 IP。这有点像之前为每个 Pod 创建一个 ReplicaSets 的方式。这两项方案结合起来就形成了如下的效果。

![img](/img/post/StatefulSet/post_eachSvc.png)

但这个方案也解决不了问题。一个 Pod 它自己不能知道它被哪个 Service 暴露，所以它不能使用这个 IP 在其他 Pod 上自己注册。

### Understanding StatefulSets

为了运行多个不可替代的（即有自己的标识和持久存储）同类 Pod，K8s 提供了一个新资源：StatefulSets。

#### 比较 StatefulSets 和 ReplicaSets

我们先用 "pets vs. cattle" 的例子解释一下 Stateful Pod。我们趋向于把应用比作宠物，给每个实例命名并照顾它。更一般地，我们把应用看作牛，每头牛不需要额外的精力照顾。只要一头牛不健康了，我们就把他杀了替换为另外一头。

对于像牛这样的 stateless 应用，我们不关心它是否必须是这个，当一个实例销毁时，创建另一个实例来代替它即可。

但对于 stateful 应用，它更像一只宠物。当宠物死亡时，我们不会随便再买一只来代替它。为了代替它，我们必须找到一只面貌和行为非常像它的。那在应用环境中，就是说新创建的应用要和就应用有相同的标识和状态。

RS 和 RC 创建的 Pod 大多是 stateless，可以随时被新 Pod 代替。但 StatefulSet 创建的 Pod 是 stateful，这意味着当这个 Pod 被销毁时，新创建的 Pod 要有相同的名字、网络标识和状态。

和 RS 一样，StatefulSet 有 Replica Conut 来决定运行多少个 Pod；也提供 Pod Template 创建 Pod。但是创建的 Pod 不是精确复制的，每个 Pod 都有自己 volume 和稳定的标识。

#### 提供稳定的网络标识

StatefulSet 创建的每个 Pod 都有一个顺序的 index，用于生成名字、hostname和稳定的存储。Pod 的名字由 StatefulSet 的名字和 index 构成。

![img](/img/post/StatefulSet/post_ssPodName.png)

在 RS 中，每个 Pod 都是一样的，所以访问 Service 时随机选择一个。但在 StatefulSet 中，每个 Pod 有不同的状态。有时候，我们要通过 hostname 访问特定的某个 Pod。因此，StatefulSet 要创建一个对应的 headless Service，来为每个 Pod 提供真正的网络标识。例如，假设一个 default 命名空间有名为 foo 的管理 Service，其中一个 Pod 为 A-0，我么就可以通过 FQDN a-0.foo.default.svc.cluster.local 访问这个 Pod。

当一个 StatefulSet 管理的 Pod 因为某种原因（node fail / delete manually）销毁时，StatefulSet 会创建一个有相同 name 和 hostname 的 Pod。新 Pod 可能不在原来的 Node 上，这与之前的一致。即便 Pod 被调度到别的 Node，依然可以通过原来的 hostname 访问。

![img](/img/post/StatefulSet/post_replace_ssPod.png)

Scale out Pod 时按照 index 顺序给新 Pod 命名。Scale in Pod 时由高 index 依次减少。

![img](/img/post/StatefulSet/post_scalein_ssPod.png)


注意 scale in Pod 时一次只允许减少一个，因为某些 stateful 应用不能处理快速 scale in 问题，例如，两个 Node 分别存储一条数据的副本，当这个 Node fail 被销毁时，该数据就丢失了。当有 Pod 不健康时，也不允许 scale in Pod 操作，因为可能会同时失去两个 Pod。

#### 提供特定的存储

每个 stateful Pod 要有自己特定的存储。StatefulSet 提供了 Volume Claim Template 来为每个 Pod 实例绑定一个 PVC。

![img](/img/post/StatefulSet/post_PVCTemplate.png)

Scale out Pod 会导致增加 PVC 的数量。但是 scale in Pod 不会删除对应的 PVC。因为删除 PVC 后，对应的 PV 中的内容一定会被回收或者删除，这对于 stateful 应用不能接受。PVC 会在再次 scale up 后重订绑定到 Pod 上。

![img](/img/post/StatefulSet/post_PVCT_rebind.png)

#### 理解 StatefulSet 的含义

当 K8s 察觉到一个 stateful Pod 销毁时，它就会创建一个相同 name、hostname、storage 的新 Pod。但如果 K8s 不能确定 Pod 的状态，创建的新 Pod 和旧 Pod 一起运行，此时两个相同标识的进程在写相同的文件。

所以 K8s 要确保不能有上述情况发生，也即 StatefulSet 必须保证一个标识的 Pod 实例 at-most-one。

### 使用 StatefulSet

我们通过实际部署来学习一下如何使用 StatefulSet。

#### 创建 app 和容器镜像

这里， 我们使用 *luksa/kubia-pet* 镜像。这个应用可以接收 POST 和 GET 请求。当接收到 POST 请求时，它把请求体中的数据写在 */var/data/kubia.txt* 文件中。当接收到 GET 请求时，它返回储存的数据。

#### 通过 StatefulSet 部署

为了部署应用，我们需要创建以下类型的资源：

+ PersistentVolumes 用于存储数据；当集群不支持动态配置时要自己创建；
+ 一个管理 Service；
+ StatefulSet 对象；

##### 创建 PV

首先创建 PV。这里我们要把 stateful Pod scale out 到三个，所以我们至少要创建三个 PV。如果使用 Minikube，我们可以创建 hostPath Volume；如果使用 Google Kubernetes，我们按照如下命令创建：

```
$ gcloud compute disks create --size=1GiB --zone=europe-west1-b pv-a 
$ gcloud compute disks create --size=1GiB --zone=europe-west1-b pv-b 
$ gcloud compute disks create --size=1GiB --zone=europe-west1-b pv-c
```

然后从以下文件中创建 PV。

```yml
kind: List 
apiVersion: v1 
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-a 
  spec:
    capacity:
      storage: 1Mi
    accessModes:
    - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    gcePersistentDisk:
        pdName: pv-a
        fsType: nfs4 
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-b
....
```

我们可以通过三条中间线分割 YAML 文件。这里我们用一种新的方式，定义一个 *List* 对象，把资源列为列表项。这里我们创建了三个 Persistent Volume。

##### 创建 Service

正如之前所述，我们要创建一个 headless Service。

```yml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None
  selector:
    app: kubia
  ports:
  - name: http
    port: 80
```

##### 创建 StatefulSet

```yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia  # service名字
  replicas: 2  # 创建2个Pod
  template:
    metadata:
      labels:   # 创建的Pod含有该label
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet 
        ports:
        - name: http
          containerPort: 8080 
        volumeMounts:
        - name: data    # Pod挂载名为data的volume到这个路径
          mountPath: /var/data
  volumeClaimTemplates:     # PVC 从这个模板创建，Pod自动引用
  - metadata: 
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

使用 *kubectl create* 命令创建之后，我们可以查看 Pod。

```
$ kubectl get po
NAME     READY  STATUS              RESTARTS    AGE 
kubia-0  1/1    Running             0           8s 
kubia-1  0/1    ContainerCreating   0           2s
```

可以看到，不同于之前 RC 和 RS 同时创建多个 Pod，这里第一个 Pod 创建成功之后第二个才开始创建。这是因为集群化应用中同时出现多个成员会对竞争条件敏感。因此安全起见，一个一个创建。

#### Playing with your pods

当 Pod 成功运行在集群中，我们不能通过创建的 headless Service 访问。我们可以像之前一样通过 *curl* 或 *port-forwarding* 直接访问。这里我们选择用 API SErver 做代理直接访问 Pod。假如我们要访问 *kubia-0* Pod，我们 curl 执行如下 URL。

```
<apiServerHost>:<port>/api/v1/namespaces/default/pods/kubia-0/proxy/<path>
```

但因为 API Server 要求 HTTPS 连接，所以我们可以通过 *kubectl proxy* 再做一层代理。

![img](/img/post/StatefulSet/post_APIServer_Proxy.png)

```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: No data posted yet
```

现在，我们发送一个 POST 请求。

```
$ curl -X POST -d "Hey there! This greeting was submitted to kubia-0." localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
Data stored on pod kubia-0
```

此时 Pod 中应该已经存储了数据。我们 GET 试试。

```
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: Hey there! This greeting was submitted to kubia-0.
```

我们再删除这个 Pod 让他重新调度，来看看保存的数据是否仍然存在。

![img](/img/post/StatefulSet/post_ssPo_reschedule.png)

当重新调度好 Pod 后，我们再发送 GET 请求。

```
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: Hey there! This greeting was submitted to kubia-0.
```

Scale down StatefulSet 然后再 Scale up 与上面的效果没有什么不同。这里就不展示了。不过需要注意的是，Scale up/down 操作是逐步进行的。当删除多个 Pod 时，当第一个完全删除了第二个才开始删除。

最后，我们也可以建一个正常的 non-headless Service 来代理 Pod。

```yml
apiVersion: v1
kind: Service
metadata:
  name: kubia-public 
spec:
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

之前介绍 service 时我们学习到，这是 ClusterIP Service，不是 NodePort和 LoadBalancer。所以只能从内部访问。但现在，我们可以用下面的 URL 通过 API Server 代理访问。

```bash
$ curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
You've hit kubia-1
Data stored on this pod: No data posted yet
```

### Discovering peers in a StatefulSet

之前我们举例子，集群应用要发现集群中的其他成员，也即 peer discovery。当然，我们可以通过访问 K8s API Server 来获取，但是 K8s 本身应该与应用程序无关。所以这不是一种理想的方法。我们可以通过 DNS 完成。

#### Implementing peer discovery through DNS

有一种名为 SRV 的 DNS 类型，它主要是指向一个提供特定服务 server 的 hostname 和 Port。K8s 创建了 SRV 记录，用来指向 headless Service 背后的 Pod。我们可以查看 stateful Pod 的 SRV 记录。

```
$ kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local
```

上面的命令运行一个名为 srvlookup 的一次性 Pod（*--restart=Never*），它一旦 terminate 就被删除（*--rm*），用来执行 *dig SRV kubia.default.svc.cluster.local* 命令。这个命令会展示出当前系统下指向 headless Service 背后 Pod 的两条 SRV 记录。所以我们可以用这个命令来 discover peer。

```
....
kubia-0.kubia.default.svc.cluster.local. 30 IN A 172.17.0.4 
kubia-1.kubia.default.svc.cluster.local. 30 IN A 172.17.0.6
....
```

下图展示了当应用接收到一个 GET 请求的运行流程。先执行 SRV 查询，然后向查到的每个 Pod 发送 GET 请求，然后返回所有存储的数据。我们使用作者已经打包好的 *kubia-pet-peers-image* 镜像。

![img](/img/post/StatefulSet/post_srv.png)

#### Updating a StatefulSet

现在我们更新创建的 StatefulSet，把 Replicas Count 改为 3，并把镜像改为 *luksa/kubia-pet-peers*。通过查看 Pod，我们可以看到新创建了一个 Pod。

```
$ kubectl get po
NAME      READY     STATUS              RESTARTS   AGE
kubia-0   1/1       Running             0          20m
kubia-1   1/1       Running             0          20m
kubia-2   0/1       ContainerCreating   0          3s
```

新创建的 Pod 运行新镜像。但在 K8s 1.7版本之前，不会像 Deployment 一样自动检测到镜像变化并 rolling update。所以我们要手动删除前两个 Pod 让它重建。1.7之后，StatefulSet 可以自动检测变化并执行 rolling update。

#### 测试

我们先向 POST 几条数据。

```
$ curl -X POST -d "The sun is shining" localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/ 
Data stored on pod kubia-1
$ curl -X POST -d "The weather is sweet" localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/ 
Data stored on pod kubia-0
```

然后我们读取数据。可以看到通过正常的 Service 访问 hit 到 kubia-2 Pod，拿到了所有数据。

```
$ curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
You've hit kubia-2
Data stored on each cluster node:
- kubia-0.kubia.default.svc.cluster.local: The weather is sweet 
- kubia-1.kubia.default.svc.cluster.local: The sun is shining
- kubia-2.kubia.default.svc.cluster.local: No data posted yet
```

### StatefulSets 如何处理 Node failure

在之前 "理解 StatefulSet 含义" 一节，我们说到 K8s 必须保证不能同时出现两个相同标识和存储的 Pod 运行在集群中，也即 at-most-one。这一节我们看看当一个 Node 断连时 StatefulSet 如何处理。

#### Node 从网络中断连

我们通过关闭 Node 的 *eth0* 网络接口模拟 Node 断连。

```
$ sudo ifconfig eth0 down
```

当断连之后，运行在 Node 上的 Kubelet 就不能连到 K8s Master Node，control panel 就无法得知 Node 和 Pod 的运行状态。一会儿之后，control panel 就把 Node 标记为 notReady，把运行在该 Node 上的 Pod 标记为 unknown。当 Pod 在几分钟内仍然 unknown，那 control panel 就自动删除 Pod 资源来驱逐该 Pod。如果 Kubelet 能看到，那它就会终止 Node 上的 Pod。但此时因为我们已经断连，Kubelet 看不到 control panel 的状态。

#### Deleting the pod manually

为了能继续运行三个正常的 Pod，我们手动删除 Pod。

```
$ kubectl delete po kubia-0
pod "kubia-0" deleted
```

但我们查看会发现，Pod 仍然存在。

```
$ kubectl get po
NAME      READY     STATUS    RESTARTS   AGE
kubia-0   1/1       Unknown   0          15m       
kubia-1   1/1       Running   0          14m 
kubia-2   1/1       Running   0          13m
```

这是因为，在我们删除之前，control panel 已经把它标记为 deletion，一旦等到 Kubelet 通知它 Pod container 被终止，它就会删除。但在我们的情况下，因为 Node 断连，Kubelet 永远不会通知到。所以我们手动强制删除，这里要用到 *--force* 和 *--grace-period 0* 两个参数。

```
$ kubectl delete po kubia-0 --force --grace-period 0
```

再次查看 Pod，会发现已经重新创建。

```
$ kubectl get po
NAME      READY     STATUS              RESTARTS   AGE
kubia-0   1/1       ContainerCreating   0          15s      
kubia-1   1/1       Running             0          14m 
kubia-2   1/1       Running             0          13m
```

### 总结

这一章我们学习了使用 StatefulSet 部署 stateful apps。

+ 复制的 Pod 分配独立的存储；
+ 为 Pod 提供稳定的标识；
+ 创建 StatefulSet 和对应的 headless governing Service；
+ scale and update StatefulSet；
+ 通过 DNS 发现集群成员；
+ 强制删除 stateful Pod；

参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

