---
layout:     post
title:      Container and Docker Overview
subtitle:   
date:       2020-08-13
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

### 虚拟机与容器技术

相信很多人都用过虚拟机。虚拟机就是在你的操作系统里面装一个软件(VM Ware, VirtualBox, Parallel Desktop等)，然后通过这个软件模拟一台甚至多台“子电脑”出来。在“子电脑”里，你可以和正常电脑一样运行程序。“子电脑”和“子电脑”之间，是相互隔离的，互不影响。

虚拟机属于虚拟化技术。而Docker这样的容器技术，也是虚拟化技术，属于轻量级的虚拟化。虚拟机虽然可以隔离出很多“子电脑”，但占用空间更大，启动更慢；\
而容器技术恰好没有这些缺点。它不需要虚拟出整个操作系统，只需要虚拟一个小规模的环境，因此启动时间很快，几秒钟就能完成。而且，它对资源的利用率很高（一台主机可以同时运行几千个Docker容器）。此外，它占的空间很小，虚拟机一般要几GB到几十GB的空间，而容器只需要MB级甚至KB级。正因为如此，容器技术受到了热烈的欢迎和追捧，发展迅速。

![img](/img/post/post_VMvsContainer.png)

### Docker容器平台

容器技术的概念已经出现很久了，但真正让容器技术流行起来的还是Docker的出现。Docker是第一个使得容器在不同机器之间移植部署的容器系统。它把应用程序、运行时的库和依赖、甚至整个OS文件系统都打包成一个package，可以很方便地在其它运行Docker的机器上运行该程序。

这有点像通过在VM中安装一个操作系统来创建一个VM镜像。不同的是，VM Image通过VM来实现App之间的隔离，而Docker Image是通过Linux容器技术实现相同级别的应用之间的隔离。另一个不同是一个Docker Image由几个Layer组成，这些Layer可以在不同的Image之间共享和重复使用。这就意味着，如果下载的一个镜像的某几个Layer已经由其他镜像下载到本地，那它只需下载剩余的几个Layer即可。

### Docker相关的概念

要注意的是，Docker不是镜像(Image)，也不是容器(Container)，它是打包发布、运行应用的一个平台。正如上述，它允许你把你的应用程序和它所有的环境变量（运行时依赖的库或访问的文件）一起打包，然后把这个package上传至一个中央仓库。这样，你就可以在别的运行Docker的机器上下载这个package运行它。

在这个场景下涉及到三个概念：
+ *Images*：Docker Image是指你把应用程序和他的环境变量一起打包而成的一个东西。它包含你在运行过程中访问的文件系统以及其他元数据；
+ *Registries*：Docker Registry是一个你存放和分享docker image的仓库，你可以把打包的image上传到仓库，然后在不同的用户和电脑上运行它。
+ *Containers*：Docker Containers是指由一个image创建出来的Linux容器。一个运行的容器是一个进程，他与机器上的其他进程隔离，只可以访问到分配给他的有限资源。

### 构建、发布、运行Docker镜像

下图展示了三种概念以及它们之间的关系。开发者首先构建一个Image并把它发布到Registry，这个image就可以被其他开发者通过Registry访问到，他们可以pull这个Image到自己的机器上运行。Docker基于这个Image创建一个隔离的Container并运行Image中指定的可执行文件。

![img](/img/post/post_dockerFlow.png)

参考自：
1. Kuberneter in Action by Marko Luksa.

