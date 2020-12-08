---
layout:     post
title:      Install OpenCV for Android
subtitle:   
date:       2020-12-7
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenCV
    - Android
---

之前介绍了如何搭建 OpenCV for Java 环境方便 PC 端开发。假如我们要开发 Android 应用，所以要在 Android Studio 中搭建 OpenCV 环境。

1. 在[官网](https://developer.android.com/studio)下载安装 Android Studio；

2. 在[官网](https://opencv.org/releases/)下载 OpenCV Android sdk；

3. 新建一个 Android Project；

4. File -> New -> Import Module，添加 Android Sdk -> sdk 下面的 java 目录；

5. File -> Project Structure -> app -> Dependencies，点击+号 -> module dependencies，添加刚才导入的OpenCV module；

6. 视图切换到 Project，修改 OpenCV 目录下的 build.gradle 文件，将 minSdkVersion、compileSdkVersion、targetSdkVersion 修改为和 app 下的 build.gradle 文件一样的版本号；

7. 拷贝 OpenCV android sdk 目录下的 sdk—native—libs 文件夹拷贝到 Android Project 目录下，具体到 app—src—main 路径下，并更改名字为 jniLibs；

8. 若出现下图所示的错误，则查看app和OpenCV目录下的 AndroidManifest.xml 文件，必然存在一处有写 targetSdkVersion 和 minSdkVersion 仍为未修改的，删除即可。

![img](/img/post/opencvMinSdk.png)

Done！

参考自：
1. [OpenCV for Android打开相机](https://blog.csdn.net/linshuhe1/article/details/51202799)
