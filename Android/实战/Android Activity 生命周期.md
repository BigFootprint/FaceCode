---
title: Android 的 Activity 生命周期
date: 2016-02-28 15:44:18
categories: Android
---

## 前提
【环境】测试手机为Nexus 5，系统为4.4和6.0。
>PS: 最早测试是在4.4下进行，当时用的还是Eclipse，因此截图都是Eclipse下面的。最新在6.0 + Android Studio下验证，结果完全一致。[原文地址](http://www.cnblogs.com/lqminn/p/3856089.html)

<!--more-->
## 测试代码
```java
package com.example.androidalarm;

import android.app.Activity;
import android.content.Context;
import android.content.res.Configuration;
import android.os.Bundle;
import android.util.AttributeSet;
import android.util.Log;
import android.view.View;
import android.widget.Button;

public class MainActivity extends Activity {
    Button addButton, cancelButton;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.d("BigFootprint", "onCreate");
    }

    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        Log.d("BigFootprint", "onConfigurationChanged");
        super.onConfigurationChanged(newConfig);
    }

    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        Log.d("BigFootprint", "onCreateView");
        return super.onCreateView(name, context, attrs);
    }

    @Override
    protected void onDestroy() {
        Log.d("BigFootprint", "onDestroy");
        super.onDestroy();
    }

    @Override
    protected void onPause() {
        Log.d("BigFootprint", "onPause");
        super.onPause();
    }

    @Override
    protected void onRestart() {
        Log.d("BigFootprint", "onRestart");
        super.onRestart();
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        Log.d("BigFootprint", "onRestoreInstanceState");
        super.onRestoreInstanceState(savedInstanceState);
    }

    @Override
    protected void onResume() {
        Log.d("BigFootprint", "onResume");
        super.onResume();
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        Log.d("BigFootprint", "onSaveInstanceState");
        super.onSaveInstanceState(outState);
    }

    @Override
    protected void onStart() {
        Log.d("BigFootprint", "onStart");
        super.onStart();
    }

    @Override
    protected void onStop() {
        Log.d("BigFootprint", "onStop");
        super.onStop();
    }
}
```

XML配置:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.androidalarm"
    android:versionCode="1"
    android:versionName="1.0" >

    <uses-sdk
        android:minSdkVersion="8"
        android:targetSdkVersion="18" />

    <application
        android:allowBackup="true"
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
        <activity
            android:name="com.example.androidalarm.MainActivity"
            android:label="@string/app_name" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

## 正常情况的生命周期
### 常见的打开退出
运行项目，当项目启动，MainActivity显示出来的时候，Console的打印如下:

![没有配置-启动](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-启动.jpg)
几个主流的生命周期方法和文档描述的一样，但是onCreatView()方法则会多次调用(从实验两次来看，次数不固定)，虽然这个方法不常用，但是用到的时候要注意。

接下去我们按Back键退出:

![没有配置-退出](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-退出.jpg)
生命周期方法调用正常。

### 屏幕暗下亮起
然后我们打开Activity，等着屏幕自己暗掉，这个时候的Log是这样的:

![没有配置-屏幕暗掉](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-屏幕暗掉.jpg)
没有调用onDestroy()方法，但是在onPause()和onStop()之间调用了onSaveInstanceState()方法。

我们按下电源键，亮起屏幕:

![没有配置-屏幕亮起](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-屏幕亮起.jpg)
没有onCreate()方法(因为Activity没有被杀死)，但是调用了onRestart()方法，这也是一个不常用的方法。

### Home键退出
我们按下Home键退出MainActivity:

![没有配置-Home退出](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-Home退出.jpg)
和屏幕暗调的Log完全一致。

再点击应用图标进入:

![没有配置-Home进入](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-Home进入.jpg)

## 横竖屏切换的生命周期
在以上代码不变的情况下，我们将手机切换为横屏(注:因为onCreateView调用次数太多，而且不是重点研究对象，后面的实验去除了它的Log):

![没有配置-竖横](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-竖横.jpg)

这里可以看到，Activity是被杀掉后重建的，并且调用了onSaveInstanceState()和onRestoreInstanceState()两个方法做了数据恢复和保存。

将屏幕从横屏切换为竖屏，Log如下:

![没有配置-横竖](http://7xktd8.com1.z0.glb.clouddn.com/没有配置-横竖.jpg)
和切换为横屏是一样的。

接下来我们为Activity的XML声明添加如下配置:

```xml
android:configChanges="orientation"
```
我们来一次竖屏 -> 横屏 —> 竖屏的切换，其Log如下:

![配置orientation-切换](http://7xktd8.com1.z0.glb.clouddn.com/配置orientation-切换.jpg)

和没有配置是一样的。我们将配置改为:

```xml
android:configChanges="orientation|keyboardHidden"
```
打印的Log没有变化。再将配置改为:

```xml
android:configChanges="orientation|screenSize"
```
这个时候切换打印的Log就如下:

![配置orientation|screenSize-切换](http://7xktd8.com1.z0.glb.clouddn.com/配置orientation|screenSize-切换.jpg)

这样的配置，在横竖屏切换的时候就只会调用onConfigurationChanged()方法，而不会引起Activity重建。

## 总结
从以上实验中可以得到以下结论。

### Android什么时候认为Activity可能会被意外关闭?
这个是以onSaveInstanceState方法被调用作为标志的，很明显有三种情况:

1. 屏幕暗下去；
2. 按下Home键退出Activity；
3. 横竖屏切换；

因为实验的问题，这里给出的三个情景并不完整，其实还有一个很重要的情景：当前Activity启动别的Activity的时候。它们有一个共同的特点: 用户离开了这个Activity，但没有主动关闭它。Android系统这个时候判定这个Activity有意外被关闭的可能性，因此需要保存数据。

### onConfigurationChanged()方法什么时候被调用?
这个方法和Activity的__android:configChanges__配置息息相关，我们先看一下官网的解释。

关于onConfigurationChanged()方法，其 [注释](http://developer.android.com/intl/zh-cn/reference/android/app/Activity.html#onConfigurationChanged(android.content.res.Configuration) 是这样的:
>Called by the system when the device configuration changes while your activity is running. Note that this will only be called if you have selected configurations you would like to handle with the configChanges attribute in your manifest. If any configuration change occurs that is not selected to be reported by that attribute, then instead of reporting it the system will stop and restart the activity (to have it launched with the new configuration).  
>At the time that this function has been called, your Resources object will have been updated to return resource values matching the new configuration.

也就是说：这个方法只有在你的应用程序运行且配置发生变化的时候被调用，但并不是所有的配置变化都会调用这个方法，必须通过__configChanges__属性进行配置，如果一个配置发生了变化，但你没有在__configChanges__属性中进行配置，那么系统会重新创建这个Activity。在这个方法调用的时候，该配置对应的资源文件已经被更新。

关于__android:configChanges__的 [解释](http://developer.android.com/intl/zh-cn/guide/topics/manifest/activity-element.html#config) 是这的:
>Lists configuration changes that the activity will handle itself. When a configuration change occurs at runtime, the activity is shut down and restarted by default, but declaring a configuration with this attribute will prevent the activity from being restarted. Instead, the activity remains running and its onConfigurationChanged() method is called.

翻译一下: 列举了Activty自己会处理的配置变化选项。当程序运行且一个配置发生变化的时候，Activity默认是重新创建的，但是如果在这里声明了这个配置，Activity就不会被重新创建，而是调用它的onConfigurationChanged()方法。

一般来说，我们最常关注的配置变化就是__orientation__。那么为什么我们单独配置__android:configChanges="orientation"__没有达到预期效果呢？官网上关于这个配置有解释:
>The screen orientation has changed — the user has rotated the device.
>>Note: If your application targets API level 13 or higher (as declared by the minSdkVersion and targetSdkVersion attributes), then you should also declare the "screenSize" configuration, because it also changes when a device switches between portrait and landscape orientations.

在API Level 13以及更高的版本上，这里必须加上__screenSize__配置才行。因为切换横竖屏的时候会导致屏幕尺寸变化(宽高不一致)，而如果没有配置这个，那系统会认为你不会去处理__screenSize__配置，自然就会重新创建Activity。

## 最后
官网上有一篇 [API指南](http://developer.android.com/intl/zh-cn/guide/topics/resources/runtime-changes.html#HandlingTheChange) ，指导开发者如何处理运行时配置变更，有几个重要的点整理如下:

1. 配置变化有两种处理办法，一种是直接重启Activity，这种办法成本较高，因为要进行大量的数据保存和恢复，一种是选择自行处理配置变更；
2. 使用Bundle保存数据会很麻烦，要经过序列化反序列化操作且一些对象并不能被序列化反序列化，一些对象例如Bitmap，重新实例化的成本很大，这个时候我们可以通过Fragment来保存数据(当Android系统因配置变更而关闭Activity时，不会销毁您已标记为要保留的Activity的Fragment。您可以将此类片段添加到Activity以保留有状态的对象。)；
3. 自行处理配置变更可能导致备用资源的使用更为困难，因为系统不会为您自动应用这些资源，开发者有责任重置为所有元素设置新的资源(如果需要变化的话)。 只能在您必须避免Activity因配置变更而重启这一万般无奈的情况下，才考虑采用自行处理配置变更这种方法，而且对于大多数应用并不建议使用此方法；

这里再备注两个方法: __onRetainNonConfigurationInstance()__和__getLastNonConfigurationInstance()__，具体的方法解释可以看[文档](http://developer.android.com/intl/zh-cn/reference/android/app/Activity.html)。

这两个方法原本也是用于处理Configuration变化的情况的，和2的使用情况是一致的。在onRetainNonConfigurationInstance ()方法的注释中特别说明了以下几点:
>This function is called purely as an optimization, and you must not rely on it being called(不一定会被调用). When it is called, a number of guarantees will be made to help optimize configuration switching:
>
>1. The function will be called between onStop() and onDestroy().在这两个方法之间调用。
>2. A new instance of the activity will always be immediately created after this one's onDestroy() is called. In particular, no messages will be dispatched during this time (when the returned object does not have an activity to be associated with).注意使用的场景，必须保证在重建的间隙中没有消息传送处理。
>3. The object you return here will always be available from the getLastNonConfigurationInstance() method of the following activity instance as described there.可以通过getLastNonConfigurationInstance()获取数据

但是在API Level 13上，这个放弃已经被废弃了，官网建议通过Fragment的setRetainInstance(boolean)方法来替代这种方案。