---
layout:     post
title:      Install Mac OS Catalina (10.15) without App Store
subtitle:   
date:       2020-12-21
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Mac
---

![img](/img/post/catalina.png)

最近苹果官方将 Mac 系统升级到了 BigSur (11.0)，以适应全新搭载的 M1 芯片。我升级之后发现部分库还不能支持新系统，比如 OpenCV3 for Java 暂时还不可以在 11.0 系统上源码编译，所以我决定从 BigSur 降级到 Catalina。

### 备份文件

首先，要备份电脑上的个人文件。我选择将所有要保存的内容都拷贝在了移动硬盘。当然，Time Machine 应该也是一种不错的方式，有兴趣的同学可以一试，我还没有试过。

### 下载镜像

要想降级到 Catalina，我们要有系统的安装镜像。之前好像是可以在 App Store 上找到 Catalina Installer，但苹果发布新系统之后就隐藏了旧版。所以我们不能在官网上找到。我是从[这里](http://dosdude1.com/catalina/)找到了 Catalina Installer，下载之后打开，就可以下载 macOS Catalina 了。

![img](/img/post/download-catalina.png)

参考自：
1. [How to download macOS Catalina installer without Mac App Store](https://wccftech.com/how-to/how-to-download-macos-catalina-installer/)

