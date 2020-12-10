---
layout:     post
title:      Android Bluetooth Communication
subtitle:   
date:       2020-12-5
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Android
---

Android 平台提供了蓝牙 API 的支持，此支持能让设备以蓝牙无线方式与其他蓝牙设备通信。应用框架提供通过 Android Bluetooth API 访问蓝牙功能的权限。\
本文以蓝牙和 Android 手机的通信过程为例，讲解 Android 平台的蓝牙通信过程和调用方法。

### 目标效果

我们有两台 Android 手机 A 和 B 通过蓝牙连接，A 向 B 发送一条消息 "hello"，B 向 A 回应消息 "world"。

### 获取 BluetoothAdapter

所有的蓝牙 Activity 都需要 BluetoothAdapter。通过调用静态方法 getDefaultAdapter() 来获取 BluetoothAdapter，此方法会返回一个 BluetoothAdapter 对象，表示设备自身的蓝牙适配器（蓝牙无线装置）。整个系统只有一个蓝牙适配器，应用可使用此对象与之进行交互。如果 getDefaultAdapter() 返回 null，则表示设备不支持蓝牙。

```java
btAdapter = BluetoothAdapter.getDefaultAdapter();
if(btAdapter == null) {
    Log.e(TAG, "Search: Device doesn't support Bluetooth");
}
```

### 启用蓝牙

下一步需要确保已启用蓝牙。调用 isEnabled() 方法以检查当前是否已启用蓝牙。如果此方法返回 false，则表示蓝牙处于停用状态，此时我们要调用 startActivityForResult() 请求启用，传入一个 **ACTION_REQUEST_ENABLE** Intent 操作。此调用会发出通过系统设置启用蓝牙的请求。

```java
if (!btAdapter.isEnabled()) {
    Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
    startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
}
```

### 查找设备

利用 BluetoothAdapter，我们可以通过查询配对设备的列表来查找远程蓝牙设备，所以这里我们要事先将两台设备已配对。配对与连接之间存在区别：
+ 配对是指两台设备知晓彼此的存在，具有可用于身份验证的共享链路密钥，并且能够与彼此建立加密连接。
+ 连接是指设备当前共享一个 RFCOMM 通道，并且能够向彼此传输数据。当前的 Android Bluetooth API 要求规定，只有先对设备进行配对，然后才能建立 RFCOMM 连接。

```java
Set<BluetoothDevice> pairedDevices = bluetoothAdapter.getBondedDevices();
for (BluetoothDevice device : pairedDevices) {
    String deviceName = device.getName();
    String deviceAddress = device.getAddress();
    // get name and address
}
```

### 连接设备

两台设备之间要创建连接，必须同时实现服务器端和客户端机制，因为其中一台设备必须开放服务器套接字来接收数据，而另一台设备必须使用服务器设备的 MAC 地址发起连接。服务器设备和客户端设备均会以不同方法获得所需的 BluetoothSocket。接受连接后，服务器会收到套接字信息。在打开与服务器相连的 RFCOMM 通道时，客户端会提供套接字信息。

当服务器和客户端在同一 RFCOMM 通道上分别拥有已连接的 BluetoothSocket 时，即可将二者视为彼此连接。这种情况下，每台设备都能获得输入和输出流式传输，并开始传输数据。

```java
if (btAdapter.isDiscovering()) {
    btAdapter.cancelDiscovery();
}
try {
    BluetoothDevice btDevice = btAdapter.getRemoteDevice(targetDeviceAddr);
    BluetoothSocket btSocket = btDevice.createRfcommSocketToServiceRecord(UUID.fromString(BT_UUID));
    btSocket.connect();
    btOs = btSocket.getOutputStream();
    Log.e(TAG, "Connect Success");
} catch (IOException e) {
    e.printStackTrace();
}
```

注意，上面代码中在正式请求连接之前，先调用了 cancelDiscovery() 来取消设备发现，这一点要谨记。然后我们通过设备的 MAC 地址选择设备，并建立 RFCOMM 通道以连接设备。

### 发送数据

如上所说，当建立连接后，每台设备都能获得输入输出流，并开始传输数据。作为客户端，我们可以通过如下方式向服务端设备发送数据。

```java
try {
    if (btOs != null) {
        btOs.write(message.getBytes("GBK"));
    }
    Log.e(TAG, "Send" + message);
    showToast("发送数据: " + message);
} catch (IOException e) {
    e.printStackTrace();
}
```

### 接收数据

同样地，作为服务端设备，我们应 new 一个线程来监听要接收的数据。

```java
class AcceptThread extends Thread {

    private final BluetoothServerSocket btServerSocket;

    public AcceptThread() {
        BluetoothServerSocket tmp = null;
        try {
            tmp = btAdapter.listenUsingRfcommWithServiceRecord("TEST", UUID.fromString(BT_UUID));
        } catch (IOException e) {
            System.out.println("Socket's listen() method failed " + e);
        }
        btServerSocket = tmp;
    }

    public void run() {
        try {
            BluetoothSocket btSocket = btServerSocket.accept();
            btIs = btSocket.getInputStream();
            btOs = btSocket.getOutputStream();
            while (true) {
                synchronized (this) {
                    byte[] byteArray = new byte[btIs.available()];
                    if (byteArray.length > 0) {
                        btIs.read(byteArray, 0, byteArray.length);
                        String content = new String(byteArray, "GBK");
                        Message msg = new Message();
                        msg.obj = content;
                        handler.sendMessage(msg);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 效果展示

![img](/img/post/btDemo.png)

通过上面的设置过程，我们就完成了 Android 设备的蓝牙通信过程调用。完整的代码示例请参阅 [这里](https://github.com/Hoozhang/AndroidBluetoothCommunication)。


参考自：
1. [路径规划之 A* 算法](https://paul.pub/a-star-algorithm/)
