---
layout:     post
title:      Windows 下 JDK 安装与环境配置
subtitle:   
date:       2020-12-8
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Java
    - Windows
---

### 下载 JDK 安装包

在[官网](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html)下载适合的 JDK 安装包，例如 Java SE Development Kit 8u271

![img](/img/post/jdk.png)

### 环境变量配置

1. 右键点击 **此电脑 - 属性** 选项。

![img](/img/post/shuxing.png)

2. 选择 **高级系统设置** 选项。

![img](/img/post/gaojixitongshezhi.png)

3. 弹出的窗口中点击环境变量按钮

![img](/img/post/envvariable.png)

4. 点击系统变量下的新建按钮，创建名为“JAVA_HOME”的环境变量，值为jdk的安装路径

![img](/img/post/newEnv.png)

5. 编辑Path变量，添加如下信息：%JAVA_HOME\bin

![img](/img/post/path.png)

6. 新建 **classpath** 的环境变量，变量值如下

![img](/img/post/classpath.png)

### 检查是否安装成功

配置好环境变量后，在 cmd 中检查是否安装正确，输入 **java -version**，如果能输出 Java 版本号，证明安装成功。

参考自：
1. [路径规划之 A* 算法](https://paul.pub/a-star-algorithm/)
