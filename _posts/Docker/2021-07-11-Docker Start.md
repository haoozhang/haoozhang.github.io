---
layout:     post
title:      Docker Quick Start Guide
subtitle:   
date:       2021-07-11
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Docker
---

这是一篇快速上手 Docker 的 Quick Start Guide，主要包含以下内容：

+ Build an image, and run it as a container
+ Share images using Docker Hub
+ Deploy Docker ???
+ Run applications using Docker Compose

## Download and Install

这里假设已经下载安装了 Docker Desktop，如果没有，请从(官网)[https://www.docker.com/]下载安装适合的版本

## Start the tutorial

下载安装完 Docker Desktop 之后，打开命令行窗口并运行以下命令：

```
docker run -d -p 80:80 docker/getting-started
```

命令中各个参数的意义如下：
+ -d：后台运行容器
+ -p 80:80：host 的 80 端口映射到 container 的 80 端口
+ docker/getting-started：运行的容器镜像

也可以合并单个字符的命令参数，以缩短完整的命令。例如以上命令可以写为：

```
docker run -dp 80:80 docker/getting-started
```

### Docker Dashboard

在继续深入之前，我们先了解一下 Docker Dashboard。它以图形化界面的方式列出了当前运行的所有容器，并提供了访问容器 log 的 shell 命令行，我们可以手动的管理容器的生命周期。如果打开 Docker Desktop，我们可以看到刚才运行的容器，如下图所示

![img](/img/post/Docker/dockerdesktop.png)

### What is a container

现在我们已经运行了一个 container，那么 container 到底是什么呢？简而言之，一个 container 就是在 host machine 上运行的单个进程，它与其它运行的进程相互隔离。这种隔离主要利用了 Linux 中的 namespace 和 cgroup 特征，Docker 使得这种隔离易于使用。

### What is a container image

当运行一个容器时，它使用了隔离的文件系统，这个隔离的文件系统就由 image 提供。其实不止文件系统，image 包含运行一个 container 的所有东西，包括依赖、环境配置等等。我们之后深入了解 image。

## Sample Application

在本文之后的篇幅，我们将运行和改进一个 todo application，关于它的源码实现不用担心，我们直接 clone 即可。

### Get the App

[Download the App contents](https://github.com/docker/getting-started/tree/master/app)，可以直接 pull 整个项目，也可以只下载这个压缩包并提取出其中的文件。

打开提取的代码文件，可以看到 package.json 和两个子目录 (src and spec)

![img](/img/post/Docker/src-code.png)

### Build app's Image

要想构建一个 container image，我们需要一个 Dockerfile，Dockerfile 是用来构建镜像的指令脚本，我们在与 package.json 文件同级的目录下创建 Dockerfile 文件，注意没有后缀文件名，文件内容如下：

```bash
 # syntax=docker/dockerfile:1
 FROM node:12-alpine
 RUN apk add --no-cache python g++ make
 WORKDIR /app
 COPY . .
 RUN yarn install --production
 CMD ["node", "src/index.js"]
```

之后通过终端窗口进入到 Dockerfile 的目录，输入构建镜像的命令

```
docker build -t getting-started .
```

你可能从输出信息中注意到正在下载很多 layer，这是因为我们在 Dockerfile 中指定从 node:12-alpine 镜像开始构建我们的 image，但是 local machine 上并没有这些 layer，所以需要下载。等下载完成之后，我们通过 COPY 命令把应用复制进去，然后使用 yarn 命令安装应用所需的依赖，CMD 指定了我们从镜像中运行容器的默认启动命令

-t 参数给构建的 image 指定了标签，给镜像一个 getting-started 名字，之后我们就可以通过 image name 指定镜像来运行容器

最后的 . 告诉 Docker 从当前目录查找 Dockerfile

### Run Container

现在我们已经有了 image，我们就可以使用 docker run 命令运行容器，命令中指定我们的镜像名

```
docker run -dp 3000:3000 getting-started
```

之前我们已经了解过 -dp，-d 参数使容器后台运行，-p 参数使容器的 3000 端口映射到 host machine 的 3000 端口，如果不映射端口，我们无法访问到应用

之后，我们就可以访问  http://localhost:3000 来访问这个应用，添加几个 item 看是否可以正常 work

![img](/img/post/Docker/todo-empty.png)

加上之前运行的容器，我们现在已经运行了两个容器，我们可以通过 docker dashboard 查看运行的容器列表

## Update Application

接下来我们有一个小需求，

### Update the source code

### Replace the old container

## Share application

### Create a repo on Docker Hub

### Push the image

### Run the image elsewhere

## 

### Container’s filesystem

### Container volumes

### 










版本控制分类

1、本地版本控制

许多人习惯用复制整个项目目录的方式来保存不同的版本，或许还会改名加上时间戳以示区别。这么做唯一的好处就是简单，但是特别容易犯错。 有时候会混淆所在的工作目录，一不小心会写错文件或者覆盖意想外的文件。

为了解决这个问题，人们很久以前就开发了许多种本地版本控制系统，大多都是采用某种简单的数据库来记录文件的历次更新差异。

![img](/img/post/Git/local_vc.png)

2、集中版本控制

如何让多个开发者协同工作？集中化的版本控制系统应运而生。这类版本控制工具都有一个单一的集中服务器，保存所有文件的修订版本。协同工作的人员都通过客户端连接到这台服务器，拉去最新文件或者提交文件更新。

![img](/img/post/Git/center_vc.png)

集中化版本控制工具已经发展非常成熟，典型的就是 SVN。但这样的版本控制系统最显而易见的缺点就是集中服务器的单点故障问题。一旦集中服务器宕机，那么谁都无法提交更新，也无法协同工作。假设一种更坏的情况，如果集中服务器的数据库磁盘损坏，那么整个项目的变更历史都将丢失。

3、分布式版本控制

于是分布式版本控制系统面世了，典型的就是 Git。在这类版本控制工具中，客户端并不只提取最新版本的文件快照，而是把代码仓库完整地克隆下来，包括完整的历史记录。这么一来，任何一处协同工作的服务器发生故障，都可以用其他任意一个克隆出来的本地仓库恢复。因为每一次的克隆操作，都是对代码仓库的完整备份。

![img](/img/post/Git/distribute_vc.png)

## What is Git

Git 是一种免费、开源的分布式版本控制系统。最初是由 Linux 之父 Linus Benedic Torvalds 开发，用于辅助 Linux 内核开发，是目前世界上最先进的分布式版本控制系统。

## Installing Git

在 Git 官网 (https://git-scm.com/downloads)，下载对应系统的安装包。

Tip：如果官网下载很慢，可以使用淘宝镜像下载：http://npm.taobao.org/mirrors/git-for-windows/

安装过程一路 Next 即可。接下来我们要配置 Git，Git 配置文件保存在本地，我们可以通过以下命令查看。

```bash
# 查看系统config
$ git config --system --list
# 查看当前用户（global）配置
$ git config --global --list
```

我们需要设置用户名和邮箱，用于提交记录时标识自己的身份。

```bash
$ git config --global user.name <your username>
$ git config --global user.email <your email>
```

## 工作区域

Git 本地有三个工作区域：工作目录 (WorkSpace)、暂存区(Stage)、本地仓库 (Local Repo)，再加上远程仓库 (Remote Repo) 一共有四个区域。这四个区域之间的关系如下图所示。

![img](/img/post/Git/repo.png)

+ WorkSpace: 即我们的项目代码目录；
+ Stage: 临时存放文件的改动，其实它是一个文件；
+ Local Repo: 本地仓库，安全地存放版本数据；
+ Remote Repo: 远程仓库，即托管代码的服务器，如 GitHub；

## 一般流程

使用 Git 的一般流程是，

1. 在 WorkSpace 中添加和修改文件，用到 git add；

2. 添加需要版本控制的文件到暂存区，用到 git commit；

3. 将暂存区中的文件提交到 Git 仓库，用到 git push；

## Git 项目搭建

### 本地仓库搭建

第一种方式是，在本地创建一个新目录，然后使用 Git 初始化。

```
$ git init
```

当执行完上面 Git 初始化命令后，可以看到目录中出现一个 .git 隐藏目录，该目录保存所有版本信息。

### 克隆远程仓库

另一种方式是，现在远程仓库创建一个 Repo，然后将其克隆到本地。

```
$ git clone <repo url>
```

这样克隆到本地的仓库就自动包含 Git 信息。

## 忽略文件

有时候我们不想把某些文件纳入版本控制中，比如数据库文件，临时文件，设计文件等。此时，我们可以在主目录下建立 .gitignore 文件，此文件有如下规则：

1. 文件中的空行或以 # 开始的行将被忽略。

2. 可使用 Linux 通配符。例如，'*' 代表任意多个字符，'?' 代表一个字符，'[abc]' 代表可选字符范围，'{string1,string2,...}' 代表可选的字符串等。

3. '!' 表示例外规则，该文件将不被忽略。

4. 以路径分隔符 '/' 开头，表示要忽略的文件在此目录下，而子目录中的文件不忽略。

5. 以路径分隔符 '/' 结尾，表示忽略该名称的子目录。

```bash
#为注释
*.txt        # 忽略所有 .txt结尾的文件,这样的话上传就不会被选中！
!lib.txt     # 但lib.txt除外
/temp        # 仅忽略项目根目录下的TODO文件,不包括其它目录temp
build/       # 忽略build/目录下的所有文件
doc/*.txt    # 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
```

## 分支

分支在 Git 中相对较难，分支相当于科幻电影里面的平行宇宙，如果两个平行宇宙互不干扰，那对现在的你也没啥影响。不过，在某个时间点，如果两个平行宇宙合并了，我们就需要处理一些问题了！
如果同一个文件在合并分支时都被修改了则会引起冲突，解决的办法是我们可以手动修改冲突文件，确定选择保留哪一方的修改后重新提交。

### Git 分支常用命令

```bash
# 列出所有本地分支
git branch

# 列出所有远程分支
git branch -r

# 新建一个分支，但依然停留在当前分支
git branch [branch-name]

# 新建一个分支，并切换到该分支
git checkout -b [branch]

# 合并指定分支到当前分支
$ git merge [branch]

# 删除分支
$ git branch -d [branch-name]

# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]
```

## 总结

本篇我们简单学习了 Git，主要从版本控制的由来、Git 的一般流程、分支管理方面介绍。Git 分支相对较难，在以后实际工作中多加练习应该会有更深的体会。


参考自：
1. [Git Documentation](https://git-scm.com/doc)
