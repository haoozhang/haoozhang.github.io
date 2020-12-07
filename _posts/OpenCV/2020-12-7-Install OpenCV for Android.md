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

之前介绍了如何搭建 OpenCV for Java 环境方便PC端开发，现在我们要开发 Android 应用，所以要在 Android Studio 中搭建 OpenCV 环境。

1. 下载[官网](https://developer.android.com/studio)安装 Android Studio；

2. 在[官网](https://opencv.org/releases/)下载OpenCV Android sdk；

3. 新建一个 Android Project；

4. File -> New -> Import Module，添加 Android Sdk -> sdk 下面的 java 目录；

5. File -> Project Structure -> app -> Dependencies，点击+号 -> module dependencies，添加刚才导入的OpenCV module；

6. 视图切换到 Project，修改OpenCV目录下的build.gradle文件，将minSdkVersion、compileSdkVersion、targetSdkVersion修改为和app下的build.gradle文件一样的版本号；

7. 拷贝OpenCV android sdk目录下的sdk—native—libs文件夹拷贝到Android Project目录下，具体到app—src—main下面，并更改名字为jniLibs；

8. 若出现下图所示的错误，则查看app和OpenCV目录下的AndroidManifest.xml文件，必然存在一处有写targetSdkVersion和minSdkVersion仍为4步骤中未修改的，删除即可。

![img](/img/post/opencvMinSdk.png)

Done！

参考自：
1. [OpenCV for Android打开相机](https://blog.csdn.net/linshuhe1/article/details/51202799)
