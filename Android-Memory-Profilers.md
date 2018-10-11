---
title: Memory Profilers
date: 2016-05-09 15:32:29
tags: [性能优化, 工具]
categories: Android
---

关于内存，Android提供了三个工具：

1. Memory Monitor: 可以找出是否有异常的GC操作导致性能问题； 
2. Heap Viewer: 可以找出被意外分配或者内存泄露的对象；
3. Allocation Tracker: 找出存在内存问题的代码；

现在Android Stuido的Android Monitor面板已经集成了这三个工具，比起使用Android Device Monitor来更加方便快捷。如下图:<!--more-->

![Android Monitor](http://7xktd8.com1.z0.glb.clouddn.com/Android-Monitor.png)

>Android Monitor不仅可以查看内存分配，还可以查看CPU、GPU、Network活动。

## Memory Monitor
这是一个很简单的工具，就在Android Studio的Android Monitor里面，也即上图中的`D`处。在`A`处选中设备，在`B`处选中进程，就可以在`D`处查看应用的内存使用情况。更详细的教程可以看[官网文档](http://developer.android.com/intl/zh-cn/tools/performance/memory-monitor/index.html)。

在[Android性能优化典范（一）](http://www.csdn.net/article/2015-01-20/2823621-android-performance-patterns/2)第九段中专门提到了一种内存现象，是可以从Memory Monitor中很容易看出来的——内存抖动：短时间发生了多次内存的涨跌。

## Heap Viewer
Heap Viewer展现了某一时刻内存分配的对象的快照，可以用于发现哪些对象发生了内存泄露。

在[官网](http://developer.android.com/intl/zh-cn/tools/performance/heap-viewer/index.html#WhatYouNeed)上是展示了该工具的使用步骤，简述如下：

1. 首先安装MAT([下载地址](http://www.eclipse.org/mat/downloads.php))；
2. 在Android Studio中打开Android Device Monitor或者DDMS；
3. 选择需要测试的进程；
4. 在进程列表上方点击Update Heap按钮；
5. 在右边的面板中选择Heap Tab；
6. 点击Cause GC；
7. 点击进程列表上方的Dump HPROF按钮，保存HPROF文件；
8. 运行命令进行文件转换：`./hprof-conv path/file.hprof exitPath/heap-converted.hprof`（hprof-conv命令在sdk的platform-tools目录下面）
9. 使用MAT打开这个文件；

>翻译自: [How to analyze memory using android studio-StackOverFlow](http://stackoverflow.com/questions/24547555/how-to-analyze-memory-using-android-studio)

但是Android Studio 1.2.x已经将该功能集合到Android Monitor面板中了：`C`处有三个按钮，第一个按钮就可以触发GC操作，第二个按钮就是Dump Java Heap。生成的文件是上面第7步生成的文件，但是转换不需要进行第8步，如图:

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/hprof.png" height="160" alt="HPROF"/></div>

只需要点击Captures边栏，选中生成的原始hprof文件，右击选择"Export to standard .hprof"文件即可生成标准的hprof文件，然后在用MAT打开文件即可。

具体的分析过程可以参见:

1. [MemoryAnalyzer](http://wiki.eclipse.org/MemoryAnalyzer)
2. [MAT使用入门](http://www.jianshu.com/p/d8e247b1e7b2)
3. [Markus Kohler博客](http://kohlerm.blogspot.jp/)
4. [Memory Analysis for Android Applications](http://android-developers.blogspot.jp/2011/03/memory-analysis-for-android.html)
5. [10 Tips for using the Eclipse Memory Analyzer](http://eclipsesource.com/blogs/2013/01/21/10-tips-for-using-the-eclipse-memory-analyzer/)

使用这个工具找寻内存泄露问题是没有一个准则的，需要对自己的程序比较了解（参见[Google I/O视频](https://www.youtube.com/watch?v=_CruQY55HOk)）。从实际开发经验来说，遵循一定的开发规则，在使用异步线程、静态变量、内部类等技术的时候多加注意是更好的办法。

## Allocation Tracker
一大利器，可以做的事情如下：

1. 展示在什么时候什么地方分配了什么样子的对象，对象的大小，分配的线程以及栈信息；
2. 通过重复进行分配、释放动作帮助分析内存抖动；
3. 可以和Heap Viewer结合分析内存泄露；

但是是需要时间来学习分析统计数据和报表的。

它的使用同样是既可以在Android Monitor中进行，也可以在Android Device Monitor中进行的。前者只需要两次点击`C`处的第三个按钮即可，后者则稍微麻烦一些。具体可参考:

1. [官网](http://developer.android.com/intl/zh-cn/tools/performance/allocation-tracker/index.html)
2. [使用Allocation tracker跟踪Android应用内存分配](http://blog.csdn.net/p106786860/article/details/9248693)

在Android Device Monitor中查看数据，个人感觉比较清晰：

![Android Monitor](http://7xktd8.com1.z0.glb.clouddn.com/Allocaition-TrackerB.png)
可以看到中间红框展现了所有分配的对象，对象的分配顺序、大小、类型、所属线程，在哪个类的哪个方法中被分配，都非常清楚。

选中一行之后，就可以在下面看到详细的堆栈信息。而关于Android Monitor的图标解析，可以查看[Android性能专项测试之Allocation Tracker(Android Studio)](http://blog.csdn.net/itfootball/article/details/48750849)。

## 内存使用分析
具体可以参考[官网文章](http://developer.android.com/intl/zh-cn/tools/debugging/debugging-memory.html)，这里还有[翻译](http://android.jobbole.com/80926/)。

总结一下，主要分析手段有：

1. 解析GC日志信息：主要关注`<Heap_stats>`，如果值一直增大并且不会减小下来，那么就可能有内存泄露了；
2. 查看堆的更新：Heap视图显示了堆内存使用的基本状况，每次垃圾回收后会更新，可以看分配的内存是否在增长；
3. 跟踪内存分配：查看对象的分配情况；
4. 查看总体内存分配：通过命令`adb shell dumpsys meminfo <package_name|pid> [-d]`可以查看进程中活动的根视图的数量和当前驻留在进程中的Context和Activity对象的数量；
5. 获取堆转储：查看持久引用，费静态内部类以及不必要的对象缓存；