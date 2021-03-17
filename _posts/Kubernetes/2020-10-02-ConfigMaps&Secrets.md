---
layout:     post
title:      Chapter 10. ConfigMaps&Secrets
subtitle:   configuring applications
date:       2020-10-02
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

这一章我们学习当在 Kubernetes 中运行应用时，如何给它传递配置数据。

### 配置容器化的应用

在正式介绍如何向 Kubernetes 中运行的应用传递配置数据之前，我们先看看容器化的应用是如何配置的。

除了直接把配置硬编码到代码中，一般是通过 *命令行参数* 来配置应用。随着配置选项逐渐增多，可以把参数写到一个 *配置文件* 中。

还有一种传递配置选项的方式是通过 *环境变量*。应用不读取配置文件和命令行参数，而是查询一个特定的环境变量的值。

环境变量的方式比命令行参数的方式更受欢迎。一是因为在 Docker 容器中使用配置文件有些麻烦，要么是通过把配置文件写到容器镜像中（类似于硬编码，每次修改都要 rebuild image），要么是通过挂载包含配置文件的 Volume（容器启动之前要先写到Volume）。二是因为每个访问镜像的人都可以看到配置，一个保密的内容可能会暴露。

配置容器化应用一般有这三种方式：

+ 传递命令行参数；
+ 通过 Volume 挂载配置文件到容器中；
+ 设置环境变量；

接下来我们先介绍这几种方式。

### 向容器传递命令行参数

到目前为止，我们创建的容器都是运行镜像中定义的默认命令。但是 Kubernetes 也允许我们在 Pod 的容器定义时覆盖命令。接下来看看如何实现。

#### Docker中定义命令和参数

首先我们解释一下，在容器中执行的命令是有两部分组成：*command* 和 *arguments*。在一个 Dockfile 中，两个指令定义这两部分：

+ *ENTRYPOINT*：定义了容器启动时执行的命令；
+ *CMD*：制定了传递给 *ENTRYPOINT* 的参数；

即便你也可以使用 *CMD* 来指定容器启动时执行的命令，但还是建议通过 *ENTRYPOINT* 来指定命令，用 *CMD* 指定默认的参数。因为这样的话就可以直接不指定参数运行镜像：

```bash
$ docker run <image>
```

或者指定额外的参数，来覆盖在 Dockerfile 中 *CMD* 设置的参数：

```bash
$ docker run <image> <arguments>
```

另外，*ENTRYPOINT* 执行的指令都支持两种格式：

+ shell 格式：例如，ENTRYPOINT node app.js
+ exec 格式：例如，ENTRYPOINT ["node", "app.js"]

两者的区别是指定的命令是否在 shell 中被调用执行。在 Chapter 3 的 Dockfile 中我们使用了 *exec* 格式，这样直接运行进程，不是在 shell 中。

```
ENTRYPOINT ["java", "-jar","app.jar"]
```

我们来修改一个脚本和镜像，让它循环的时间间隔可以配置。我们添加一个 *INTERVAL* 变量并用第一个命令行参数初始它，如下所示。

```bash
#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1   # INTERVAL用第一个参数初始化
echo Configured to generate new fortune every $INTERVAL seconds    # echo打印信息
while :
do
    echo $(date) Hit
    sleep $INTERVAL
done
```

我们已经修改了这个脚本。现在我们要修改 Dockfile 使得它用 *exec* 格式的指令，并使用 *CMD* 把 *INTERVAL* 初始化为 10。如下所示。

```bash
FROM ubuntu
COPY ./loop.sh bin/loop.sh 
ENTRYPOINT ["bin/loop.sh"]  # exec格式
CMD ["10"]  # 默认参数为10
```

现在我们构建镜像，并上传到 DockerHub。

```
$ docker build -t zhaodockerhub/loop .
$ docker tag zhaodockerhub/loop zhaodockerhub/loop:args
$ docker push zhaodockerhub/loop:args
```

现在我们运行测试。

```bash
$ docker run -it zhaodockerhub/loop:args  # 默认参数
Configured to generate new fortune every 10 seconds
Wed Mar 17 11:45:23 UTC 2021 Hit
Wed Mar 17 11:45:33 UTC 2021 Hit
Wed Mar 17 11:45:43 UTC 2021 Hit

(press Ctrl+C)

$ docker run -it zhaodockerhub/loop:args 5  # 覆盖CMD参数
Configured to generate new fortune every 5 seconds
Wed Mar 17 11:46:02 UTC 2021 Hit
Wed Mar 17 11:46:07 UTC 2021 Hit
Wed Mar 17 11:46:12 UTC 2021 Hit
```

#### Kubernetes中覆盖命令和参数

在 Kubernetes 中，当我们描述一个 container 时，我们可以选择覆盖 *ENTRYPOINT* 和 *CMD*。如下所示，我们通过设置描述文件中 container 的 *command* 和 *args* 属性。

```
kind: Pod
spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

绝大多数情况下我们都只是自定义参数，一般不轻易修改命令。

为了运行有自定义时间间隔的 Pod，我们创建以下 YAML 文件。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: fortuneloop
spec:
  containers:
  - image: zhaodockerhub/loop:args
    args: ["3"]  # 添加了 args 数组自定义参数
    name: fortune 
```

如果我们有多个自定义参数，则可以按照以下格式。其中字符串无需用双引号，但数字需要用双引号。

```
args:
- foo
- bar
- "15"
```

用 *kubectl create* 命令创建上述 Pod 后，等 Pod 为运行状态时，可查看它的 log。可以看到按照 Pod 中设置的每隔 3 秒打印。

```
$kubectl logs fortuneloop
Configured to generate new fortune every 3 seconds
Wed Mar 17 11:48:04 UTC 2021 Hit
Wed Mar 17 11:48:07 UTC 2021 Hit
Wed Mar 17 11:48:10 UTC 2021 Hit
Wed Mar 17 11:48:13 UTC 2021 Hit
Wed Mar 17 11:48:16 UTC 2021 Hit
Wed Mar 17 11:48:19 UTC 2021 Hit
```

### 为容器设置环境变量


参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

