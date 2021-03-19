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

我们已经学习了 DownwardAPI 传递 Pod 和 container 的 metadata 到运行在其中的进程。它只暴露了自己 Pod 的一部分数据，但有时我们需要知道集群中其他 Pod 的资源，DownwardAPI 不能满足我们的需求。我们需要直接和 API server 交互。

![img](/img/post/post_apiserver.png)

在真正学习 Pod 中的应用如何访问 API server 之前，我们先探索一下 server 的 REST endpoints。

#### Exploring the Kubernetes REST API

如果我们要开发访问 Kubernetes API 的应用， 我们要首先了解 API。
我们可以通过 *kubectl cluster-info* 命令获得它的 URL。

```
$ kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:59498
```

可以看到 server 使用 HTTPS 连接要求认证，如果我们直接 curl，会显示未认证。幸运的是，我们可以通过 *kubectl proxy* 命令运行一个代理来访问 server。代理服务器接收 HTTP 连接并把它代理到 API server，所以我们绕过了认证。

```bash
$ curl https://127.0.0.1:59498 -k
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",    # 未认证
  "details": {
    
  },
  "code": 403
}

$ kubectl proxy   # 运行代理
Starting to serve on 127.0.0.1:8001

$ curl http://127.0.0.1:8001  # 访问代理
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",

```

我们通过连接 proxy 访问到了 API server，它返回了一系列路径，这些路径对应于我们创建资源时的 API group 和版本。例如，Pod、Service 都是 apiVersion:v1。

我们探索一下 Job 资源的 API，它对应于 api/batch/v1。

```bash
$ curl http://127.0.0.1:8001/apis/batch/v1
{
  "kind": "APIResourceList",  # 资源列表
  "apiVersion": "v1",
  "groupVersion": "batch/v1", # 对应batch/v1
  "resources": [
    {
      "name": "jobs",   # Job 资源
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [    # Job资源可执行的动作
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "mudhfqk/qZY="
    },
    {
      "name": "jobs/status",  # 一个特定的REST endpoints用于修改状态
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [  
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

为了得到集群中 Job 的列表，我们可以 GET *apis/batch/v1/jobs* 路径。

```bash
$ curl http://127.0.0.1:8001/apis/batch/v1/jobs
{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "selfLink": "/apis/batch/v1/jobs",
    "resourceVersion": "141150"
  },
  "items": []   # 目前集群中没有 Job 资源
}
```

假如我们已经创建了 Job 资源，items 域会显示出 Job 的 metadata。

如果我们要获取某一个 Job，我们可以通过指定 namespace 和 name 来获取。例如，

```bash
$ curl http://localhost:8001/apis/batch/v1/namespaces/default/jobs/my-job
```

#### Talking to the API server from within a pod

上面展示了通过 *kubectl proxy* 访问 API server。现在我们学习在 Pod 中访问 API server。我们要注意以下三点：

+ 找到 API server 的位置；
+ 确保是访问 API server，而非冒充的；
+ 和 server 认证；

##### Running Pod to communicate with API server

我们先创建一个和 API server 交互的 Pod，然后使用 *kubectl exec* 命令在容器中运行一个 shell，在 shell 中我们尝试访问 API server。Pod 描述文件如下所示。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers: 
  - name: main
    image: tutum/curl   # 这个镜像中包含了curl命令
    command: ["sleep", "9999999"]  # 容器不做任何操作
```

在容器中运行 shell 命令。

```
$ kubectl exec -it curl bash
root@curl:/#
```

##### Finding Api server's Address

因为一个名为 kubernetes 的 Service 指向 API server，所以我们可以找到 API server 的 IP 和 Port。

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   5d19h
```

我们学习 Service 时也了解到，每个 Service 的信息都配置到了环境变量。所以我们可以访问环境变量来找到 IP 地址和 Port。

```bash
root@curl:/# env | grep KUBERNETES_SERVICE
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
```

我们也记得通过 DNS 得到每个 Service。

```
root@curl:/# curl https://kubernetes
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: http://curl.haxx.se/docs/sslcerts.html
....
```

即便我们可以通过 '-k' 选项跳过 SSL 的检测，但我们在真实场景中还是要检测证书，防止被中间攻击。

##### 认证 server 的身份

在之前介绍 Secret 时，我们看到有一个自动创建的 *default-token-xyz* Secret，它被挂载在每个容器上的 */var/run/secrets/kubernetes.io/serviceaccount/* 目录，我们可以看看它的内容。

```
root@curl:/# ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt	namespace  token
```

可以看到有三个文件，我们先关注 *ca.crt*。它包含一个 CA 的证书，我们用
*curl --cacret* 命令指定 CA 证书。

```
root@curl:/# curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes
```

发现仍然未认证。这是因为目前 client 已经信任 API server 了，但 API server 不信任 client，不允许授权访问。

为了方便，我们可以把 *ca.crt* 设置为环境变量，之后就可以不用每次 *--cacert* 直接访问了。

```
root@curl:/# export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
root@curl:/# curl https://kubernetes
```

我们需要让 server 信任 client。此时我们用到 *default-token-xyz* Secret 的 token。我们先把它设置为环境变量，然后访问。

```
root@curl:/# TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
root@curl:/# curl -H "Authorization: Bearer $TOKEN" https://kubernetes
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
....
```

如果你正在使用开启了 RABC 的 Kubernetes 集群，那么上面的命令会显示错误：

```
system:serviceaccount:default:default cannot get path
```

仍然未被授权。这里最简单的方法是赋予所有的 service-account 集群管理员的超级权限。但这种方式非常危险，只能在测试环境。

```
kubectl create clusterrolebinding permissive-binding 
\ --clusterrole=cluster-admin 
\ --group=system:serviceaccounts
```

##### 列出所有 Pod

这里我们列出和这个 Pod 相同 Namespace 的所有 Pod。我们可以通过之前学过的 downwardAPI 暴露 Pod 的 Namespace。但 *default-token-xyz* Secret 已经有 namespace 文件。我们直接使用。

```
root@curl:/# NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
root@curl:/# curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods
```

##### 总结 Pod 访问 Kubernetes API

+ 应用使用 *ca.crt* 认证 server 的证书；
+ 应用发送 *Authorization: Bearer $TOKEN* 认证自己；
+ 当 CRUD Pod 命名空间的对象时，*namespace* 文件要传给 API server；

总结如下图所示。

![img](/img/post/post_default-token-secret.png)

#### Simplifying API server communication with ambassador containers

处理证书和认证 token 太麻烦了。我们是否可以简化 Pod 访问 API server 的流程？之前我们可以通过 kubectl proxy 简化访问 API server，在 Pod 中我们也可以这样。

假设我们有一个应用需要访问 API server，我们可以在主容器旁边创建另一个代理容器。主容器的应用通过 HTTP（注意，不用 HTTPS）访问代理容器，让代理容器自己管理与 API server 的 HTTPS 连接。如下图所示。

![img](/img/post/post_proxy_container.png)

我们创建一个新 Pod，其中不知包括一个 curl 容器，还包括一个代理容器。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-proxy
spec:
  containers: 
  - name: main
    image: tutum/curl   # 这个镜像中包含了curl命令
    command: ["sleep", "9999999"] 
  - name: proxy
    image: luksa/kubectl-proxy:1.6.2
```

创建 Pod 之后，我们在主容器中执行 shell。因为有两个容器，所以我们要用 *-c* 参数指定容器。

```
$ kubectl exec -it curl-with-ambassador -c main bash
```

然后我们通过代理容器访问 API server。默认情况下 *kubectl proxy* 绑定在 8001 端口。因为两个容器共享一个 Pod 的网络接口，所以我们直接 *curl http://localhost:8001*。

```
$ root@curl-with-proxy:/# curl http://localhost:8001
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
....
```

成功！可以通过代理容器直接访问 API server。下图展示了命令背后的操作流程。


![img](/img/post/post_curl_proxy_container.png)


#### Using client libraries to talk to the API server

如果我们只需要简单的 API server 访问操作，那我们可以使用代理容器来完成。但要是做更复杂的操作，建议使用 Kubernetes 已有的 API client libraries。

目前，Kubernetes 官方提供了两个 API client libraries。

+ Golang client - https://github.com/kubernetes/client-go
+ Python - https://github.com/kubernetes-incubator/client-python

除了官方支持的以外，也有用户贡献的 client libraries。以下列出了 Java 的两种，这些 client liarbries 都支持 HTTPS 认证，所以就不需要用代理容器了。

+ Java client by Fabric8 — https://github.com/fabric8io/kubernetes-client
+ Java client by Amdatu—https://bitbucket.org/amdatulabs/amdatu-kubernetes

下面是一个使用 Fabric8 Java Client 与 Kubernetes 交互的代码示例。

```java
import java.util.Arrays;
import io.fabric8.kubernetes.api.model.Pod;
import io.fabric8.kubernetes.api.model.PodList;
import io.fabric8.kubernetes.client.DefaultKubernetesClient; import io.fabric8.kubernetes.client.KubernetesClient;

public class Test {
  public static void main(String[] args) throws Exception {
    KubernetesClient client = new DefaultKubernetesClient();

    // list pods in the default namespace
    PodList pods = client.pods().inNamespace("default").list(); 
    pods.getItems().stream()
      .forEach(s -> System.out.println("Found pod: " + s.getMetadata().getName()));
    
    // create a pod
    System.out.println("Creating a pod");
    Pod pod = client.pods().inNamespace("default")
      .createNew() 
      .withNewMetadata()
        .withName("programmatically-created-pod") .endMetadata()
      .withNewSpec()
        .addNewContainer()
        .withName("main")
        .withImage("busybox") 
        .withCommand(Arrays.asList("sleep", "99999"))
        .endContainer() 
      .endSpec() 
      .done();
    System.out.println("Created pod: " + pod);
  
    // edit the pod (add a label to it) 
    client.pods().inNamespace("default")
      .withName("programmatically-created-pod") 
      .edit()
      .editMetadata()
        .addToLabels("foo", "bar") 
      .endMetadata()
      .done();
    System.out.println("Added label foo=bar to pod");
    
    System.out.println("Waiting 1 minute before deleting pod..."); 
    Thread.sleep(60000);
  
    // delete the pod 
    client.pods().inNamespace("default")
      .withName("programmatically-created-pod")
      .delete(); 
    System.out.println("Deleted the pod");
  }
}
```

### 总结

本篇我们学习了运行在 Pod 的应用如何获取该 Pod、其他 Pod、集群中其他组件的元数据。具体地：

+ 一个 Pod 的 name、namespace 等 metadata 通过环境变量或 downwardAPI Volume 文件的方式暴露给容器的应用；
+ CPU 和 Memory 的 request 和 limit 传递给应用；
+ Pod 使用 downwardAPI 获取最新 metadata (such as labels, annotations)；
+ 通过 *kubectl proxy* 访问 API server；
+ 通过 *get svc*、环境变量和 DNS 的方式查找 API server 的 IP 地址；
+ 运行在 Pod 的应用验证 API server 和 自己；
+ 使用代理容器简化访问 API server 的流程；
+ 使用现有的 API client 与 Kubernetes 交互；

参考自：
1. Kuberneter in Action by Marko Luksa.

