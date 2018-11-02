---
title: Android开发笔记
date: 2016-02-22 16:59:14
tags: [合集]
categories: Android
---

**【1】空格问题**  
如何在字符串中添加正确的空格，问题源自于如何添加一个中文空格。
方案: [StackOverFlow](http://stackoverflow.com/questions/1587056/android-string-concatenate-how-to-keep-the-spaces-at-the-end-and-or-beginnin)

**【2】No Debuggable Applications**  
在Android Studio的菜单栏中选择 Tools -> Android -> Enable ADB Integration。

PS:需要关闭 DDMS。<!--more-->

关于原因，[StackoverFlow上这个问题](http://stackoverflow.com/questions/9242148/what-is-the-purpose-of-tools-android-enable-adb-service)下面有回答:
>The ADB (Andorid Debug Bridge) is a service that IDEA uses for debugging Android code on emulator or USB device. This service can be used by only one application in the same time. DDMS tool uses ADB too, so you need to disable ADB-IDEA connection if you want to use DDMS tool without closing IDEA. You won't be able to debug Android applications in IDEA if this service is disabled, but note that if you try to launch debugging IDEA will notify you that ADB service is disabled and offer to enable it again. So there shouldn't be any problems after disabling this service. You just need to do it before launching DDMS.

**【3】生命周期问题**  

1. Activity的`onCreate()`会调用`mFragments.dispatchCreate()`;
2. Activity的`onStart()`会调用`mFragments.dispatchActivityCreated()`以及`mFragments.dispatchStart()`（前者保证只调用一次）;
3. 内存不足导致 Activity 被回收后，恢复的时候生命周期如下：`onCreate` -> `onStart` -> `onRestoreInstanceState` -> `onActivityResult` -> `onResume`；
4. `onActivityResult`是在`onResume`之前被调用的，这一点无论是不是数据恢复，都是一样的，因此在数据回收的情况下，如果要在`onActivityResult`中使用`onResume`方法实例化的对象，则务必要小心 —— StackOverFlow 上有一个问题 [onActivityResult() & onResume()](http://stackoverflow.com/questions/6468319/onactivityresult-onresume) ，关注一下其中的高票答案;

第三点保证`onActivityResult`调用的时候数据都恢复成应有的样子了。

#### _持续更新···_