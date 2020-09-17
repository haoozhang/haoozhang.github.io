---
layout:     post
title:      Chapter 2. Container and Docker Overview
subtitle:   
date:       2020-08-12
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

### 虚拟机与容器技术

相信很多人都用过虚拟机。虚拟机就是在你的操作系统里面装一个软件(VM Ware, VirtualBox, Parallel Desktop等)，然后通过这个软件模拟一台甚至多台“子电脑”出来。在“子电脑”里，你可以和正常电脑一样运行程序。“子电脑”和“子电脑”之间，是相互隔离的，互不影响。

虚拟机属于**虚拟化**技术。而 Docker 这样的容器技术，也是虚拟化技术，属于轻量级的虚拟化。\
虚拟机虽然可以隔离出很多“子电脑”，但占用空间更大，启动更慢；\
而容器技术恰好没有这些缺点。它不需要虚拟出整个操作系统，只需要虚拟一个小规模的环境，因此启动时间很快，几秒钟就能完成。而且，它对资源的利用率很高（一台主机可以同时运行几千个Docker容器）。此外，它占的空间很小，虚拟机一般要几GB到几十GB的空间，而容器只需要MB级甚至KB级。正因为如此，容器技术受到了热烈的欢迎和追捧，发展迅速。

![img](/img/post/post_vm_container.png)

### Docker 容器平台

容器技术的概念已经出现很久了，但真正让容器技术流行起来的还是 Docker 的出现。Docker 是第一个使得容器在不同机器之间容易移植部署的容器系统。它把应用程序、运行时的库和依赖、甚至整个OS文件系统都打包成一个 package，可以很方便地在其它运行 Docker 的机器上运行该程序。

这有点像通过在 VM 中安装一个操作系统来创建一个 VM 镜像。不同的是，VM Image 通过 VM 来实现 App 之间的隔离，而 Docker Image 是通过 Linux 容器技术实现相同级别的应用之间的隔离。另一个不同是一个 Docker Image 由几个 Layer 组成，这些 Layer 可以在不同的 Image 之间共享和重复使用。这就意味着，如果下载的一个镜像的某几个 Layer 已经由其他镜像下载到本地，那它只需下载剩余的几个 Layer 即可。

### Docker 相关的概念

要注意的是，Docker 不是镜像(Image)，也不是容器(Container)，它是打包发布、运行应用的一个平台。正如上述，它允许你把你的应用程序和它所有的环境变量（运行时依赖的库或访问的文件）一起打包，然后把这个 package 上传至一个中央仓库。这样，你就可以在别的运行 Docker 的机器上下载这个 package 运行它。

在这个场景下涉及到三个概念：
+ *Images*：Docker Image 是指你把应用程序和他的环境变量一起打包而成的一个东西。它包含你在运行过程中访问的文件系统以及其他元数据；
+ *Registries*：Docker Registry 是一个你存放和分享 docker image 的仓库，你可以把打包的 image 上传到仓库，然后在不同的用户和电脑上运行它。
+ *Containers*：Docker Containers 是指由一个 image 创建出来的 Linux 容器。一个运行的容器是一个进程，他与机器上的其他进程隔离，只可以访问到分配给他的有限资源。

### 构建、发布、运行 Docker 镜像

下图展示了三种概念以及它们之间的关系。开发者首先构建一个 Image 并把它发布到 Registry，这个 image 就可以被其他开发者通过 Registry 访问到，他们可以 pull 这个 Image 到自己的机器上运行。Docker 基于这个 Image 创建一个隔离的 Container 并运行 Image 中指定的可执行文件。

![img](/img/post/post_dockerFlow.png)

### 小结

这一篇我们介绍了虚拟化和容器技术，并引入 Docker 的相关概念和运作流程。有了这一篇的基本知识以后，下篇我们上手一下 Docker，看看如何打包发布一个应用镜像。

参考自：
1. Kuberneter in Action by Marko Luksa.

