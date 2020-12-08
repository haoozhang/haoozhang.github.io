---
layout:     post
title:      OpenCV for Android 实现相机预览
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

在之前篇章中，我们已经配置好了 OpenCV for Android 的开发环境。接下来我们学习如何利用 OpenCV for Android 的 API 来实现相机的打开和预览功能。

### 实现 CvCameraViewListener2 接口

我们要想实现在应用中通过 OpenCV for Java API 实现打开相机全屏显示，并获取预览帧画面，MainActivity 需要实现 CvCameraViewListener2 接口。该接口具体实现三个方法，分别是：onCameraViewStarted、onCameraViewStopped和onCameraFrame，关键的图像处理写在 onCameraFrame 函数中：

```java
public class MainActivity extends AppCompatActivity implements CvCameraViewListener2 {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    public void onCameraViewStarted(int width, int height) {
        // Auto-generated method
    }

    @Override
    public void onCameraViewStopped() {
        // Auto-generated method
    }

    @Override
    public Mat onCameraFrame(CameraBridgeViewBase.CvCameraViewFrame frame) {
        // Auto-generated method
        return null;
    }
}
```

### 修改 AndroidManifest.xml 文件

添加相机的相关权限：

```java
<uses-permission android:name="android.permission.CAMERA"/>
<uses-feature android:name="android.hardware.camera" android:required="false"/>
<uses-feature android:name="android.hardware.camera.autofocus" android:required="false"/>
<uses-feature android:name="android.hardware.camera.front" android:required="false"/>
<uses-feature android:name="android.hardware.camera.front.autofocus" android:required="false"/>
```

设置应用界面为全屏显示，在 activity 标签中添加：

```java
android:screenOrientation="landscape"
android:configChanges="keyboardHidden|orientation"
```

### 添加相机显示的组件

打开 res/layout 下面的 activity_main.xml 布局文件，添加一个OpenCV的视觉组件 JavaCameraView：

```java
<org.opencv.android.JavaCameraView
    android:id="@+id/camera_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

并在 MainActivity 中引用 JavaCameraView 组件，声明一个 CameraBridgeViewBase 对象，绑定和添加事件监听：

```java
mCVCamera = (CameraBridgeViewBase) findViewById(R.id.camera_view);
mCVCamera.setCvCameraViewListener(this);
```

### 修改 onCameraFrame 方法

接下来修改 onCameraFrame 回调方法的内容，相机刷新每一帧都会调用一次，输入参数是当前相机视图信息，我们直接获取其中的 RGBA 信息作为 Mat 数据返回给显示组件即可：

```java
    @Override
    public Mat onCameraFrame(CameraBridgeViewBase.CvCameraViewFrame frame) {
        // TODO: image processing for yourself
        return frame.rgba();
    }
}
```

### 异步加载 OpenCV 库

通过以上操作，我们在 OnCreate 函数中已经获取到 mCVCamera 对象，只有调用 mCVCamera.enableView() 之后，预览组件才会显示每一帧的 Mat 图像，但在显示之前我们必须提前加载好 OpenCV 的库文件，所以调用此方法需要进行异步处理：

```java
/**
 * 通过OpenCV管理Android服务，异步初始化OpenCV
 */
BaseLoaderCallback mLoaderCallback = new BaseLoaderCallback(this) {
	@Override
	public void onManagerConnected(int status){
		switch (status) {
	    	case LoaderCallbackInterface.SUCCESS:
	    		Log.i(TAG,"OpenCV loaded successfully");
	    		mCVCamera.enableView();
	    		break;
	    	default:
			    break;
	    }
	}
};
```

只有当 mLoaderCallback 收到 LoaderCallbackInterface.SUCCESS 消息的时候，才会打开预览显示，那么这个消息是从哪里发出来的呢？这就需要我们重写 Activity 的 onRusume 方法了，因为每次当前 Activity 激活都会调用此方法，所以可以在此处检测OpenCV的库文件是否加载完毕：

```java
@Override
public void onResume() {
	super.onResume();
	if (!OpenCVLoader.initDebug()) {
		Log.d(TAG,"OpenCV library not found!");
	} else {
		Log.d(TAG, "OpenCV library found inside package. Using it!");
		mLoaderCallback.onManagerConnected(LoaderCallbackInterface.SUCCESS);
	}
};
```

至此，我们就完成了设备相机打开以及帧画面的预览获取。完整源码见[这里](https://github.com/Hoozhang/Opencv4AndroidDemo)。


参考自：
1. [OpenCV for Android打开相机](https://blog.csdn.net/linshuhe1/article/details/51202799)
