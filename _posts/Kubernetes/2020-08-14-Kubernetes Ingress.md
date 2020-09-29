---
layout:     post
title:      Kubernetes Ingress
subtitle:   
date:       2020-08-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

### Ingress是什么

Ingress公开和定义了从集群外部到集群内部 Service 的 HTTP/HTTPS 路由规则。流量路由由 Ingress 资源上定义的规则控制。

### 一个最小的 Ingress 资源示例 (单服务 Ingress)

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
          backend: 
            serviceName: test 
            servicePort: 80 
```

+ 与所有其他 K8S 资源一样，Ingress 资源需要使用 apiVersion、kind 和 metadata 字段。 
+ 可选主机：此示例中未指定主机，因此该规则适用于指定 IP 地址的所有入站 HTTP 通信；如果提供了主机 (例如 foo.bar.com)，则规则适用于该主机。 
+ 路径列表 (例如，/testpath)，每个路径都有一个由 serviceName 和 servicePort 定义的关联后端。 
负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。 
+ 后端是服务名和端口的组合。与规则中主机和路径匹配的HTTP/HTTPS请求将发送到列出的后端。


### 简单分列

上面展示了单服务的 Ingress 示例，下面展示流量从单个 IP 地址路由到多个服务。

```java
foo.bar.com -> 178.91.123.132 -> /foo  service1:4200 
                                 /bar  service2:8080 
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

### 基于名称的虚拟托管
针对多个主机名的 HTTP 流量路由到同一 IP 地址上

```java
foo.bar.com --|                 |-> foo.bar.com service1:80 

              | 178.91.123.132  | 

bar.foo.com --|                 |-> bar.foo.com service2:80
``` 

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

### 查看和更新Ingress 
```java
kubectl describe ingress test 
kubectl edit ingress test 
```

### TLS

通过指定一个包含TLS私钥和证书的Secret对象，我们可以通过TLS访问Ingress。Ingress资源只支持单个TLS端口443，并且只能作用于外部流入Service的流量，Service到Pod的流量以HTTP格式转发，也即官方所说的TLS termination at the ingress point。

 TLS Secret 必须包含名为 tls.crt 和 tls.key 的键名。例如：

 ```
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
 ```

然后在Ingress中引用此 Secret 将会告诉 Ingress 控制器使用 TLS 加密从客户端到Service的通道。

```yml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
spec:
  tls:
  - hosts:
    - foo.bar.com
    secretName: testsecret-tls
  rules: 
    - host: foo.bar.com 
      http: 
        paths: 
        - path: /foo 
          backend: 
            serviceName: service1 
            servicePort: 8080 
        - path: /bar 
          backend: 
            serviceName: service2 
            servicePort: 8080
```

参考自：
1. [Kubernetes Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)


