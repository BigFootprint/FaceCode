---
title: AIDL Demo
date: 2016-06-14 11:33:59
tags: [Android, AIDL]
categories: Android
---

## 简介
__AIDL__ ，全称 Android Interface Definition Language，是用于定义 IPC 的客户端和服务端之间的交互接口的。

在 Android 上，进程之间是不能直接通信的，必须通过操作系统对数据进行转化传输，才能打破进程边界，完成进行通信。这个过程如果手动完成会比较枯燥复杂， Android 上的 IPC 实际依赖的是底层 Binder 框架，而 __AIDL__ 是 Android 在 Java 层对 Binder 机制的封装。<!--more-->

官网介绍 __AIDL__ 的地址是 [Android Interface Definition Language (AIDL)](https://developer.android.com/guide/components/aidl.html)。本文主要是实践一个例子，具体详细的内容读者可以直接看官网，本文不做介绍，主要包括:

1. __AIDL__ 支持的数据类型（下面例子中也能看到）；
2. 什么时候应当使用 __AIDL__（还有Messenger等IPC方式）；

## 实践

#### AIDL 文件的创建
首先在项目`A`下面右击一个 module ，选择"New" -> "Folder" -> "AIDL Folder"，就可以在 module 下面新建一个 aidl 文件夹，具体如下:

<div align="center"><img src="https://imgchr.com/i/ituQ0A" width="250" alt="AIDL 文件夹"/></div>

然后我们在这个文件夹下面新建一个包，比如"com.footprint.littleshell"，然后选择"New" -> "AIDL" -> "AIDL File"，输入文件名，就可以新建一个 aidl 文件出来了，在文件中输入如下内容:

```java

import com.footprint.littleshell.MyRect;

interface IMyAidlInterface {
    int basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

    MyRect newRect();
}
```
这两个接口以及`MyRect`都是仿照官网上的例子写的，并且`basicTypes`方法在文件创建的时候就默认被添加了，表明 __AIDL__ 支持的几种基本类型，这里没有删掉，直接拿来使用，而`newRect`的返回值是一个复杂对象。

Android IPC 或者说 __ADIL__ 传输复杂对象是需要进行特殊处理的，这一点在[官网](https://developer.android.com/guide/components/aidl.html#PassingObjects)上有特别描述，主要分为以下几步:

1. 复杂对象类必须实现`Parcelable`接口(这里就不细说了)；
2. 创建`.aidl`文件来声明该类；

我们一步步照做，首先创建`MyRect`类，它的代码如下:

```java
package com.footprint.littleshell;

import android.os.Parcel;
import android.os.Parcelable;

public class MyRect implements Parcelable {
    public int left;
    public int top;
    public int right;
    public int bottom;

    public static final Parcelable.Creator<MyRect> CREATOR = new
            Parcelable.Creator<MyRect>() {
                public MyRect createFromParcel(Parcel in) {
                    return new MyRect(in);
                }

                public MyRect[] newArray(int size) {
                    return new MyRect[size];
                }
            };

    public MyRect() {
    }

    private MyRect(Parcel in) {
        readFromParcel(in);
    }

    public void readFromParcel(Parcel in) {
        left = in.readInt();
        top = in.readInt();
        right = in.readInt();
        bottom = in.readInt();
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(left);
        dest.writeInt(top);
        dest.writeInt(right);
        dest.writeInt(bottom);
    }
}
```
按要求实现了相关的方法，这保证这个类可以在 Android 系统上被序列化反序列化，从而能够在进程间传递。

第二步就是创建 "aidl" 文件，具体位置前面的截图可以看到，文件名为 "MyRect.aidl"，内容为:

```java
// MyRect.aidl
package com.footprint.littleshell;

parcelable MyRect;
```
这样一来我们就可以编译项目了，编译完成之后，可以在下图位置找到生成的接口文件:

<div align="center"><img src="https://imgchr.com/i/itulTI" width="250" alt="AIDL生成文件位置"/></div>

文件的具体内容就详细去看了，后面会有文章详细剖析它的内容，本文主要讲述如何去用。

#### IPC 调用
既然是 IPC，就必然有 Client 和 Server 。先来写一个 Service:

```java
public class RemoteService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
    }

    @Override
    public IBinder onBind(Intent intent) {
        // Return the interface
        return mBinder;
    }

	 // 返回Stub的实例
    private final IMyAidlInterface.Stub mBinder = new IMyAidlInterface.Stub() {

        @Override
        public int basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
            return 1024;
        }

        @Override
        public MyRect newRect() throws RemoteException {
            MyRect myRect = new MyRect();
            myRect.bottom = 1;
            myRect.top = 2;
            myRect.left = 3;
            myRect.right = 4;
            return myRect;
        }
    };
}
```
别忘了在 "AndroidManifest.xml" 文件中进行注册:

```xml
<service
	android:name=".RemoteService"
	android:exported="true">
</service>
```
写完就可以 run 到你的手机上了。接下来就来写 Client。

新建一个项目，按照前面创建文件的步骤，创建一遍同样的 __AIDL__ 文件，或者直接 Copy 也可以。

新建`AIDLActivity`:

```java
package com.footprint.androidaircraftcarrier.lab;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import com.footprint.androidaircraftcarrier.R;
import com.footprint.littleshell.IMyAidlInterface;


public class AIDLActivity extends AppCompatActivity {
    Button ipcButton;

    IMyAidlInterface mService;
    ServiceConnection serviceConnection;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_aidl);

        serviceConnection = new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                mService = IMyAidlInterface.Stub.asInterface(service);
                try {
                    Toast.makeText(AIDLActivity.this, mService.newRect().right + "", Toast.LENGTH_LONG).show();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {

            }
        };

        ipcButton = (Button)findViewById(R.id.button_service);
        ipcButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent();
                intent.setComponent(new ComponentName("com.footprint.littleshell", "com.footprint.littleshell.RemoteService"));
                bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
            }
        });
    }
}
```
当我们点击 Button 之后，该 App 就会尝试调用`RemoteService`，而`RemoteService`存在于另外一个 App 中，运行该 App，点击按钮，就可以看到如下结果:

<div align="center"><img src="https://imgchr.com/i/itu3kt" width="250" alt="IPC调用"/></div>

它确实拿到了远程 Service 创建的`MyRect`实例对象并获取了其中的属性。这样就完成了 IPC 调用。