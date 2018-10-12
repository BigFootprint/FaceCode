---
title: ADB 介绍
date: 2016-02-29 15:51:12
tags: [Android, ADB]
categories: Android
---

## 介绍
__ADB，全称Android Debug Bridge__，是一个集成了与调试机器(真机/模拟器)通信丰富功能的命令行工具，它是client-server模式的，包括三个部分:

1. __Client__ 运行在你的开发机器上(例如电脑)，开发者可以通过在shell中运行adb命令来调用client，另外像DDMS这样的工具也会创建一个adb client；
2. __Daemon__ 运行在真机或者模拟器上，接受并处理adb命令； 
3. __Server__ 在你的开发机上运行的后台进程，它负责管理Client和Daemon之间的交互；
   <!--more-->
   adb命令工具可以在 __<sdk>/platform-tools__ 下面可以找到。

当一个Client启动的时候，Client会检测有没有adb Server进程在运行，如果没有，它就会启动一个。当Server启动起来后，它就会绑定到本地机器的5037端口监听从adb Client发来的命令。所有的adb Client都使用5037端口与Server通信。

adb Server负责建立与所有的模拟器/真机的连接。它会扫描所有模拟器/真机在[5555, 5585]之间的奇数端口去寻找adb Daemon，一旦找到就会连接到该端口。连接建立后，就可以通过adb命令来操作模拟器/真机。因为Server维护着到这些模拟器/真机的所有的实例，同时也处理所有的adb Client的命令，因此开发者可以从任何一个Client控制任何一台模拟器/真机(Server其实是一个消息转发中心)。

## ADB命令
### 基本格式
adb命令的格式是这样的:

```java
adb [-d|-e|-s <serialNumber>] <command>
```
其中 __[-d|-e|-s <serialNumber>]__ 是用于在多设备状态下选择命令目标设备的，含义分别如下:
>`-d` : Direct an adb command to the only attached USB device.  
>`-e` : Direct an adb command to the only running emulator instance.	 
>`-s <serialNumber>` : Direct an adb command a specific emulator/device instance, referred to by its adb-assigned serial number (such as "emulator-5556").

### 常用命令
即 `<command>` 部分。
#### 1. `devices`
Prints a list of all attached emulator/device instances.打印格式如下 : [serialNumber] [state]。state有三种: 

1. __offline__ 机器没有连接或者不响应adb命令
2. __device__ 机器连接正常
3. __no device__ 没有任何机器连接

使用如下命令即可以指定一个设备运行命令，示例如下：
`adb -s [serialNumber] shell wm size`

#### 2. `logcat [option] [filter-specs]`
Prints log data to the screen.
#### 3. `install <path-to-apk>`
Pushes an Android application (specified as a full path to an .apk file) to an emulator/device.
#### 4. `pull <remote> <local>`
Copies a specified file from an emulator/device instance to your development computer.
#### 5.`push <local> <remote>`
Copies a specified file from your development computer to an emulator/device instance.
#### 6. `start-server`
Checks whether the adb server process is running and starts it, if not.
#### 7. `kill-server`
Terminates the adb server process.
#### 8. `get-serialno`
Prints the adb instance serial number string.
#### 9. `shell`
Starts a remote shell in the target emulator/device instance.
#### 10. `adb shell dumpsys activity top`
打印栈顶Activity的信息（类名、布局信息）。
>PS: 这个命令是`adb shell dumpsys`的子集，`adb shell dumpsys`可以dump整个系统的信息，曾经用来追踪注册到系统的广播，非常有用，但是信息量大的时候速度比较慢，后面加上别的参数可以更有针对性的dump信息。

#### 11. `adb shell pm list instrumentation`
打印所有的intrumentation包。
#### 12. `adb logcat -c`
清空日志。
#### 13. `adb shell wm size`
打印屏幕尺寸。
#### 14. `adb shell wm density`
打印屏幕密度。
#### 15. `adb shell dumpsys window displays`
显示屏幕参数。

### 无线调试
Android还支持无线调试，步骤如下:

1. 连接到同一个Wifi上；
2. 用USB线连接手机和电脑，并通过`adb tcpip 5555`命令设置设备在5555端口上监听TCP/IP连接，之后拔出USB线；
3. 找到手机的IP地址，使用`adb connect <device-ip-address>`命令连接设备；
4. 使用`adb devices`命令查看设备是否被连接；

官网上也有提到有关防火墙的设置，需要支持ADB无线调试才行。我在公司试了一下，是不行的。断开无线连接的命令是`adb disconnect <device-ip-address>`。

### Shell命令
ADB提供了一个Unix Shell，以便于可以在连接的设备上执行一些命令，相关的命令二进制数据存储在设备文件系统下面，地址是 __/system/bin/...__

执行shell命令有两种办法:

```java
//不进入shell，而是直接运行一个shell命令
adb [-d|-e|-s <serialNumber>] shell <shell_command>

//进入shell模式后再执行
adb [-d|-e|-s <serialNumber>] shell
```

### am命令
__am，即activity manager__。我们可以使用这个工具来触发一些系统行为，比如启动Activity，停止进程，发送广播，修改屏幕属性(在Nexus5 + 6.0上尝试有关命令，发现找不到)等。在shell里使用 `am <command>` 即可。常见易用的命令如下:

#### 1. `force-stop <PACKAGE>`
Force stop everything associated with <PACKAGE> (the app's package name).
#### 2. `start [options] <INTENT>`
Start an Activity specified by [<INTENT>](https://developer.android.com/intl/zh-cn/tools/help/shell.html#IntentSpec).可以携带Url等参数打开一个Acitivity，由此可以攻击别人的应用。
#### 3. `broadcast [options] <INTENT>`
Issue a broadcast intent.

### pm命令
__pm，即package manager__。看一些比较易用的命令吧。

#### 1. `adb shell pm clear <PACKAGE>`
Deletes all data associated with a package.
#### 2. `adb shell pm list packages`
列出所有的应用。
#### 3. `enable|disable <PACKAGE_OR_COMPONENT>`
Enable/Disable the given package or component (written as "package/class").
#### 4. `grant/revoke <PACKAGE_NAME> <PERMISSION>`
Grant/Revoke a permission to an app. On devices running Android 6.0 (API level 23) and higher, may be any permission declared in the app manifest. On devices running Android 5.1 (API level 22) and lower, must be an optional permission defined by the app.(可以收回权限，测试的时候非常有用)

__pm__下面还有几个列举设备支持的feature、library、permissoon等特性的命令，直接查阅[文档](https://developer.android.com/intl/zh-cn/tools/help/shell.html#IntentSpec)吧，有很多选项。

### 其他命令
#### 1. `screencap <filename>`
具体执行如下:

```java
adb shell screencap /sdcard/screen.png
```
用于给当前屏幕截图，非常有用，主要是导出图片非常方便，使用pull即可。

#### 2. `screenrecord [options] <filename>`
具体执行如下:

```java
adb shell screenrecord /sdcard/demo.mp4
```
用于录制操作视频。默认是3分钟后停止录制，可以使用Ctrl-C命令终止录制，或者使用`--time--limit`来设置录制时间。导出方法和图片一致。

<br><hr>
### 一篇整理的更加详细的文章：[Awesome ADB](https://github.com/mzlogin/awesome-adb)。