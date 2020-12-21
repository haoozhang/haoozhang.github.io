---
layout:     post
title:      Visual Studio 2017 配置 OpenGL 开发环境
subtitle:   
date:       2020-12-21
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenGL
---

### 下载 GLUT 包

在[这里](http://www.opengl.org/resources/libraries/glut/glutdlls37beta.zip)下载 OpenGL 的文件包，压缩包中一共有如下5个文件，要分别配置到3个目录下。

![img](/img/post/glut1.png)

### glut.h 配置目录

Visual Studio 自带的 GL 有2个头文件：GL.h，GLU.h，我们可以通过在代码中尝试 #include 头文件来查看它们所在的目录。
我们下载的 glut.h 也放置在这个目录下。

![img](/img/post/glut2.png)

### glut.lib, glut32.lib 配置目录

![img](/img/post/glut3.png)

### glut.dll, glut32.dll 配置目录

![img](/img/post/glut4.png)

### Visual Studio 工程中设置静态链接库

打开 **调试-project属性-链接器-输入**，在 **附加依赖项** 中设置静态链接库。

![img](/img/post/glut5.png)

### 测试

用如下代码测试配置是否成功。

```c++
#include <iostream>
#include <gl\glut.h>
#include <gl\GL.h>
#include <gl\GLU.h>

void myDisplay() {
    // 清除帧缓存
    glClear(GL_COLOR_BUFFER_BIT); 
    glRectf(-0.5f, -0.5f, 0.5f, 0.5);
    glFlush();
}

int main(int argc, char * argv[]) {
    // 初始化 GLUT
    glutInit(&argc, argv); 
    // 单缓冲|color buffer
    glutInitDisplayMode(GLUT_SINGLE | GLUT_RGBA); 
    // 窗口设置
    glutInitWindowPosition(100, 100);
    glutInitWindowSize(400, 400);
    glutCreateWindow("第一个OpenGL程序"); 
    // 回调函数，这个函数被 GLUT 内部循环不断的调用
    glutDisplayFunc(&myDisplay); 
    // 开始循环，并且监听回调函数
    glutMainLoop(); 

    return 0;
}
```

### OpenGL 项目实例

最后分享一个利用 OpenGL 基本操作实现的图形处理小项目，包括绘制基本图形、平移、旋转、缩放、填充、裁剪等功能。代码见[这里](https://github.com/Hoozhang/CG-Program)，欢迎 Star!

参考自：
1. [VS2017 配置 OpenGL 环境](https://www.jianshu.com/p/f6b7d768283f)
