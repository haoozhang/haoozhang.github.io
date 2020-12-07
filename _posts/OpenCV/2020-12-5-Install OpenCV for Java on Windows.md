---
layout:     post
title:      Install OpenCV for Java with IDEA on Windows
subtitle:   
date:       2020-12-5
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenCV
---

如果我们需要在 PC 端做一些计算机视觉开发的工作，推荐使用 OpenCV。这是一个是由 Intel 发起和参与的计算机视觉工具库，可用于实时图像处理、计算机视觉以及模式识别开发。本篇我们以 IDEA 开发工具介绍，如何在 Windows 系统下安装 OpenCV for Java 环境。

1. 配置 jdk 环境：可参考[这篇文章](https://www.cnblogs.com/cnwutianhao/p/5487758.html)。

2. 下载 IDEA：直接在[官网](https://www.jetbrains.com/zh-cn/idea/download/#section=windows)下载安装包，并根据提示安装即可。

3. 下载 OpenCV：下载对应版本的 Win Pack 安装包，解压缩到某个目录下，例如D盘；

4. 新建 IDEA 项目，打开 File -> Project Structure，在 Library 目录下添加 OpenCV 库。\
因为刚才解压缩到D盘，故库的位置在 D:\opencv\build\java\opencv-342.jar

5. 为避免 Core.NATIVE_LIBRARY 找不到 Java 库路径，打开 Run -> Edit Configuration，
    在 VM Option 一栏填写参数：-Djava.library.path=D:/opencv/build/java/x64

6. 在主类中添加如下静态代码块，使程序运行前提前加载好 OpenCV 库。

```java
static {
    System.loadLibrary(Core.NATIVE_LIBRARY_NAME);
}
```



参考自：
1. [Win10 Java 环境变量配置](https://www.cnblogs.com/cnwutianhao/p/5487758.html)
