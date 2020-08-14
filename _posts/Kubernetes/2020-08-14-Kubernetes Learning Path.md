---
layout:     post
title:      Kubernetes Learning Path
subtitle:   
date:       2020-08-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

#### A Kubernetes Story (基本概念)

* Pod：K8S内部运行容器的基本单元，每个Pod运行一个或多个容器，可以设置容器的环境变量、存储及其他信息； 
* ReplicaSet：确保以预定的副本数量运行一组配置相同的Pod； 
* Secret：储存token、password等非公开的信息，当附在Pod上时自动解码； 
* Deployment：ReplicaSets + 部署/更新/扩展Pod，支持rolling update和rollback； 
Rolling update解释：https://kubernetes.io/zh/docs/tutorials/kubernetes-basics/update/update-intro/ 
* DaemonSet：确保一个Pod的copy运行在集群每个node上，集群可能扩展或收缩； 
* Ingress：声明从集群外部访问内部Service的路由规则； 
* CronJob：周期执行的定时任务； 
* CRD：CustomResourceDefinition，定义新的资源类型 

#### Video series with Brendan Burns ()
* Why should care about containers? 
easier to build and deploy, cloud repository to push and share 
* How Kubernetes works?
 
 
 
 
* How deployment work？ 
 
* Understand serverless with Kubernetes 
 
* How the Kubernetes scheduler works？ 
 
* Setting up a Kubernetes build pipeline 

```java
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress 
metadata: 
    name: test-ingress 
    annotations: 
        nginx.ingress.kubernetes.io/rewrite-target: / 
spec: 
    rules: 
    - http: 
        paths: 
        - path: /testpath 
            pathType: Prefix 
            backend: 
                serviceName: test 
                servicePort: 80 
```

+ 与所有其他 K8S 资源一样，Ingress 资源需要使用 apiVersion、kind 和 metadata 字段。 
+ 可选主机：此示例中未指定主机，因此该规则适用于指定 IP 地址的所有入站 HTTP 通信；如果提供了主机 (例如 foo.bar.com)，则规则适用于该主机。 
+ 路径列表 (例如，/testpath)，每个路径都有一个由 serviceName 和 servicePort 定义的关联后端。 
负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。 
+ 后端是服务名和端口的组合。 与规则中主机和路径匹配的HTTP/HTTPS请求将发送到列出的后端。


#### 简单分列

上面展示了单服务的 Ingress 示例，下面展示流量从单个 IP 地址路由到多个服务。

```java
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200 

                                                        / bar    service2:8080 
```

```java
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata: 
    name: simple-fanout-example 
    annotations: 
        nginx.ingress.kubernetes.io/rewrite-target: / 
spec: 
    rules: 
    - host: foo.bar.com 
    http: 
        paths: 
        - path: /foo 
            backend: 
                serviceName: service1 
                servicePort: 4200 
        - path: /bar 
            backend: 
                serviceName: service2 
                servicePort: 8080
```

#### 基于名称的虚拟托管：针对多个主机名的 HTTP 流量路由到同一 IP 地址上

foo.bar.com --|                           |-> foo.bar.com service1:80 

                       | 178.91.123.132  | 

bar.foo.com --|                           |-> bar.foo.com service2:80 

```java
apiVersion: networking.k8s.io/v1beta1 

kind: Ingress 

metadata: 

    name: name-virtual-host-ingress 

spec: 

    rules: 

    - host: foo.bar.com 

    http: 

        paths: 

         - backend: 

            serviceName: service1 

            servicePort: 80 

    - host: bar.foo.com 

    http: 

        paths: 

         - backend: 

            serviceName: service2 

            servicePort: 80 
```

#### 查看和更新Ingress 
kubectl describe ingress test 
kubectl edit ingress test 

参考自：
1. [递归公式的理解](https://leetcode-cn.com/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/solution/nan-dian-shi-di-gui-gong-shi-de-li-jie-by-piao-yi-/)


