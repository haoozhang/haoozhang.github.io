---
layout:     post
title:      Install OpenCV for Java with IDEA on Mac
subtitle:   
date:       2020-12-6
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenCV
    - Mac
---

之前介绍了在 Windows 环境下新建 OpenCV for Java 的 IDEA 项目，这里我们介绍在 Mac 环境下的 OpenCV for Java 搭建。因为官方没有直接提供 OpenCV 的 jar 包，所以需要我们手动编译生成。下面是详细步骤。

1. 下载安装 IDEA：此处省略不再赘述。

2. 安装OpenCV：根据官方教程 (见下图)，需要依次安装 Homebrew，Xcode Command Line Tools 以及 ant。然后我们修改 opencv formula，就可以编译安装 OpenCV了。

![img](/img/post/makeOpenCV.png)

这里要注意几点：

+ 下载过程中因为墙的原因可能会出现 unable to access github.com 的问题，尝试多装几次。
+ 编译过程比较慢，我大概安装了快1个小时，尤其是卡在 cmake 和 make 命令，要耐心等待。
+ 因为 OpenCV 版本更新的原因，现在可能默认安装的是 OpenCV 4.x 版本。如果我们想要安装 OpenCV 3.x 版本，我们查看 Homebrew 的 formula，发现还有 opencv@3 和 opencv@2 这两个 formula，所以我们修改 opencv@3 范式并安装它即可。opencv@2 同理。

3. IDEA 环境搭建：

+ Create a new project。

+ 在 File -> Project Structure -> Library 中添加 OpenCV 对应的库，默认在 /usr/local/Cellar 目录下。

+ 在主类中添加如下静态代码块，使程序运行前提前加载好 OpenCV 库。

```java
static {
    System.loadLibrary(Core.NATIVE_LIBRARY_NAME);
}
```

+ 在Run—Edit Configurations中，VM Options一栏添加JVM启动参数：\
-Djava.library.path=/usr/local/Cellar/opencv/3.4.2/share/OpenCV/java (写到jar包所在目录)

接下来就可以正常调用 OpenCV 中的库函数了！下面这个示例代码可用来测试。

```java
public static void main(String[] args) {
        Mat mat = Imgcodecs.imread("./test.png");
        Mat grayMat = new Mat();
        Imgproc.cvtColor(mat, grayMat, Imgproc.COLOR_BGR2YCrCb);
        Imgcodecs.imwrite("gray.png", grayMat);
}
```

参考自：
1. [OpenCV Java Tutorials](https://opencv-java-tutorials.readthedocs.io/en/latest/01-installing-opencv-for-java.html#set-up-opencv-for-java-in-other-ides-experimental)
