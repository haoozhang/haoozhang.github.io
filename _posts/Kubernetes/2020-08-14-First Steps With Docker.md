---
layout:     post
title:      Chapter 3. First Steps With Docker
subtitle:   
date:       2020-08-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Kubernetes
---

之前我们已经了解到 Kubernetes 中的应用程序是通过打包成容器镜像运行的，所以我们先学习一下 Docker 的基本操作和使用，包括如何创建、运行、分享一个容器镜像等。

### 安装 Docker 和运行 "Hello World" 镜像

安装 Docker 请参考[官方文档](https://docs.docker.com/get-docker/)，我是在 Mac 和 Windows 上安装的，只需要按照文档安装一个 Docker Desktop 就好了。安装完成后，你可以打印一下 Docker 的版本来验证是否安装成功，如下所示。

```
$ docker --version
Docker version 19.03.12, build 48a66213fe
```
然后你就可以正常的使用 Docker 了，例如你可以拉取和运行 Docker Hub (Docker 的中央仓库)中已有的 Docker 镜像。我们选择一个名为 busybox 的镜像，它是把许多 UNIX 标准的命令行工具(echo, ls等)集成起来的一个可执行文件，我们可以利用它来输出 Hello World。

你不需要下载和安装任何东西，只需要在命令行窗口输入一条简单的命令指定要运行的 Docker 镜像即可，如下所示。

```
$ docker run busybox echo "Hello world"
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
9c075fe2c773: Pull complete
Digest: sha256:c3dbcbbf6261c620d133312aee9e858b45e1b686efbcead7b34d9aae58a37378
Status: Downloaded newer image for busybox:latest
Hello world
```

是不是很神奇！你并没有下载和安装任何东西，也没有配置任何环境变量，只需要一条命令就运行了一个存储在 Docker Hub（其实下载到了本地）的应用程序。当然这里的应用程序只是输出一个 "Hello World"，但它也可以是其他更复杂的有环境依赖的应用。这也就是Docker的核心理念：Build once，Run anywhere。

我们回过头再看看上面命令的执行过程。*docker run*是运行 Docker 镜像的命令，*busybox* 是我们指定要运行的镜像，后面的 *echo "Hello world"* 是我们指定镜像要运行的指令，这是可选的，只针对于这个镜像。
当我们按下回车键，可以看到 Docker 先在本地寻找这个名为 *busybox* 的镜像，发现本地不存在后从中央仓库 pull 这个镜像，当 pull 成功后会进行一次sha256校验，应该是校验拉取的镜像文件有无受损，这个我们不用在意。最后输出 "Hello World" 的指令。Docker 的运行流程如下图所示。

![img](/img/post/post_dockerRun.png)

最后再说明一点，可以看到上面 Docker 从 Docker Hub 拉取镜像时输出的信息是 *latest: Pulling from library/busybox*，这个 latest 其实是指这个镜像的 tag，默认的选择是 latest ，当然你也可以指定这个镜像的其他 tag 来运行，命令如下所示。这个 tag 可以理解为用来区分一个镜像的不同版本。

```
$ docker run <image>:<tag>
```

### 创建一个 Hello World App

通过运行 Docker Hub 中的 Hello World 镜像，我们对 Docker 有了一个基本的印象。接下来我们看看如何构建一个自己的 Docker 镜像。我们以构建一个自己简单的 Hello World 镜像为例，首先我们写一个 Hello World 的应用代码。如果你不想写代码，这个项目的源码可以从[这里](https://github.com/haozhangms/Kubernetes_blog/tree/master/FirstStepsWithDocker)获得。

```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```

因为我是用 Maven 构建的项目，要在 pom.xml 文件中 build 位置指定 main class，以方便后续打包成 package。对于其他方式（CLI，Ant，Gradle）构建项目的同学，请参考[这里](https://stackoverflow.com/questions/9689793/cant-execute-jar-file-no-main-manifest-attribute)。

```
<build>
    <plugins>
        <plugin>
            <!-- Build an executable JAR -->
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.1.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>lib/<classpathPrefix>
                        <mainClass>HelloWorld</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后，我们利用 Maven 将应用程序打包，打包完成后可以看到在 target 文件夹下生成一个 .jar 文件。

```
$ mvn clean package
```

### 写 Dockerfile

为了将上面打包的 jar 包构建为一个镜像，我们通常需要一个 Dockerfile 来完成构建镜像的工作。Dockerfile 文件有一系列构建镜像的指令组成，每一条指令构建一个 Layer，因此每条指令的内容就是描述该层 Layer 应当如何构建。\
本文中使用的 Dockerfile 比较简单，如果想要了解更多 Dockerfile 的知识，可参考[这里](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)。

```
FROM openjdk:11-jdk-slim

COPY ./target/*.jar app.jar
ENTRYPOINT ["java", "-jar","app.jar"]
```

*FROM* 指定构建的新镜像是来自于哪个基础镜像；*COPY* 拷贝文件或目录到镜像中，这里我们把刚才生成的 jar 包文件拷贝到镜像根目录下；*ENTRYPOINT* 指定启动容器时执行的 Shell 命令，这里我们指定要执行 *java -jar app.jar*。

### 构建和运行镜像

现在我们可以利用 Dockerfile 构建一个 HelloWorld 镜像了，只需要运行以下命令即可。

```
docker build -t zhaodockerhub/helloworld .
```

*docker build* 命令格式与 *docker run* 相似，是构建镜像的命令。*-t* 是指定镜像的名字和 tag，这里默认 tag 为 latest。别忘了最后空格之后还有一个点，用来指定当前要构建镜像的目录。

这里注意一点，镜像名字我选择了\<Docker Hub username>/\<image name>。这是因为之后上传到 Docker Hub 时要具体到你的 Docker Hub 用户名，现在指定之后就不用重新 tag 了。当然你也可以之后运行 *docker tag* 命令重新指定镜像名字。

当构建完成后，可以运行 *docker images* 查看镜像，可以看到本地已经存在一个构建的镜像了。我们可以按照之前运行镜像的方式运行该镜像，如下所示。

```
$ docker images
REPOSITORY                   TAG      IMAGE ID        CREATED             SIZE
zhaodockerhub/helloworld     latest   b29259e0be42    20 minutes ago      402MB

$ docker run zhaodockerhub/helloworld
Hello World
```

如果要停止运行镜像，可以使用如下命令。

```
$ docker stop <image name>
```

如果要删除镜像，可以运行以下命令。

```
$ docker rm <image name>
```

### 分享镜像

现在你构建的镜像还只是存储在本地，只能在本地的机器上运行。正如之前说到的，我们可以把自己构建的镜像上传到一个中央仓库（DockerHub），分享给别的用户。首先，你需要在 DockerHub 上注册一个账户。然后你就可以往自己的仓库下 push 镜像了。

Docker Hub上传镜像时要求镜像名称为\<DockerHub username>/\<image name>，如果不是，要先重命名你的镜像。

```
$ docker tag <image name> <DockerHub username>/<image name>
```

其实 *docker tag* 并没有重命名该镜像，而是创建了这个镜像的一个副本，并命名为新的名称，你可以运行 *docker images* 查看。现在你可以通过以下命令上传你的镜像。

```
$ docker push <DockerHub username>/<image name>
```

上传结束后，你也可以在 DockerHub 上看到刚刚分享的镜像。这样别人或者你在别处就可以使用刚刚构建的镜像，而无需安装任何其他环境。

### 小结

本篇我们学习了 Docker 的简单使用，包括构建、运行、发布一个镜像。在基本了解和使用 Docker 之后，下一篇我们开始 Kubernetes 的初步上手入门。

参考自：
1. 《Kuberneter in Action》 by Marko Luksa.

