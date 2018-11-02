---
title: Android6.0权限问题
date: 2015-10-28 11:29:03
tags: [Android, 权限]
categories: Android
---

建议阅读[这篇文章](http://inthecheesefactory.com/blog/things-you-need-to-know-about-android-m-permission-developer-edition/en)和[官方文档](http://developer.android.com/intl/zh-cn/training/permissions/requesting.html#perm-check)。

简单来说，Android 要向 iOS 靠拢，实行运行时权限控制啦！在很久以前，我们只需要在 AndoridManifest.xml 文件中声明程序运行所需要的权限，用户在安装的时候一次同意就可以了。

在 Android M 中，权限被分为两类：一类是普通权限，不会涉及到用户隐私，一类是高危权限，在使用权限的时候需要用户同意。当然用户也能在设置中取消应用的某个权限。
<!--more-->
因此 App 需要考虑到用户拒绝权限的逻辑情况，需要对此做出适配。至于如何适配，如何申请权限，两篇文章都讲得很清楚。

这里主要讲一下其中一个重要的方法：shouldShowRequestPermissionRationale()。根据官网描述，这个方法其实是用于判断是否需要向用户提供一个 explanation 的。那么这个方法依据什么返回呢？

1. 如果 app 之前要求获取某个权限，但是被拒绝了，方法会返回 true（这种情况下，Android 认为用户不理解应用获取该权限的目的，因此需要解释）；
2. 如果用户在拒绝的时候选择了 "Don't ask again"（不再询问），方法会返回 false（用户理解目的，并且断然拒绝）；
3. 如果设备本身就禁止这个应用获取该权限，方法会返回 false（无关用户理解与否，用户无法对权限的许可作出任何事情）；

因此实际上 Android 的整个权限控制分两个逻辑：1）判断权限是否被授予；2）判断获取权限是否需要提供 explanation。因此其官网的获取权限的示例代码如下：
```java
// Here, thisActivity is the current activity
if (ContextCompat.checkSelfPermission(thisActivity,
                Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {

        // Show an expanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {
        // No explanation needed, we can request the permission.
        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}
```
注意：内部的`if...else...`两个逻辑点都应该去请求权限！

那么如果用户在拒绝的时候选了"Don't ask again"（不再询问），再调用`ActivityCompat.requestPermissions`获取权限会怎么样呢？会以`PERMISSION_DENIED`返回值直接进入回调。

然而实践下来，很多 ROM 的权限申请都存在坑，部分手机在权限检测以及申请返回上都会给出错误值。
