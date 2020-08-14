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
* Deployment：ReplicaSets + 部署/更新/扩展Pod，支持rolling update和rollback；(Rolling update解释：https://kubernetes.io/zh/docs/tutorials/kubernetes-basics/update/update-intro/)
* DaemonSet：确保一个Pod的copy运行在集群每个node上，集群可能扩展或收缩； 
* Ingress：声明从集群外部访问内部Service的路由规则； 
* CronJob：周期执行的定时任务； 
* CRD：Custom Resource Definition，定义新的资源类型 

#### Video series with Brendan Burns ()
1、Why should care about containers? \
easier to build and deploy, cloud repository to push and share 

2、How Kubernetes works?
 
3、How deployment work？ 
 
4、Understand serverless with Kubernetes 
 
5、How the Kubernetes scheduler works？ 
 
6、Setting up a Kubernetes build pipeline 

参考自：
1. [Kubernetes Learning Path](http://azure.microsoft.com/mediahandler/files/resourcefiles/kubernetes-learning-path/Kubernetes%20Learning%20Path%20version%201.0.pdf)


