---
layout:     post
title:      OpenCV for Android 修改相机分辨率
subtitle:   
date:       2020-12-8
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenCV
    - Android
---

上篇文章中我们学习了如何利用 OpenCV for Android 实现相机的打开和预览。在此基础上，如果我们要修改帧画面的分辨率，该如何实现呢？

### 新建 MyJavaCameraView 类

上篇文章中我们在布局文件中使用了 OpenCV 的 JavaCameraView 视觉组件。这里我们要修改分辨率，所以需要重新修改这个类的属性。具体地，我们新建一个 MyJavaCameraView 类继承该类，并提供一个设置分辨率的方法。

```java
public class MyJavaCameraView extends JavaCameraView {
    private static final String TAG = "MyJavaCameraView";

    public MyJavaCameraView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public void setAbsolution(int width , int height) {
        Log.i(TAG, "setAbsolution: " + width + " " + height);
        disconnectCamera();
        mMaxWidth = width;
        mMaxHeight = height;
        connectCamera(getWidth(), getHeight());
    }
}
```

### 修改布局文件

相应地，我们修改原来的布局文件使用新建的这个类，com.zhao 是我这个工程的 package name 可以忽略。

```java
<com.zhao.MyJavaCameraView
    android:id="@+id/camera_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

### 设置分辨率

最后，我们在 Activity 中将原来定义的 CameraBridgeViewBase 的类型的变量 mCVCamera 修改为 MyJavaCameraView 类型。然后就可以在 onRusume 方法中调用 mCVCamera.setAbsolution() 方法设置帧画面的分辨率了。

参考自：
1. [OpenCV for Android打开相机](https://blog.csdn.net/linshuhe1/article/details/51202799)
