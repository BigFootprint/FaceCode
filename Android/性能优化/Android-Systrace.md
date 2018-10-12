---
title: Systrace
date: 2016-05-10 23:44:01
tags: [性能优化, 工具]
categories: Android
---

当我们开发应用，我们会希望应用使用起来非常顺畅，刷新页面的时候始终可以达到60帧每秒的速度。但是如果因为某些原因导致丢帧，我们要做的第一步事情就是搞清楚系统到底在做什么。

>Systrace允许获取Android系统的信息，称为__trace__。它会展现时间以及CPU时间片被用到了哪里，在给定的时间内，每个进程和线程在做什么事情，它还会高亮有问题的trace信息，并给出修复意见。<!--more-->

## Overview
![An example Systrace, showing 5 seconds of scrolling an app when it is not performing well.](http://7xktd8.com1.z0.glb.clouddn.com/systrace_overview.png)

上图展示了一个渲染起来并不顺畅的App所被捕捉到的trace信息，展示的信息组有Kernel、SurfaceFlinger（Android的compositor进程），然后是App进程（可以看到进程名），每一个App进程都包含所有它所含有的线程的trace信号。

关于如何产生一个trace文件，可以直接查看[官网](http://developer.android.com/intl/zh-cn/tools/performance/systrace/index.html)。这里有个[示例文件](http://7xktd8.com1.z0.glb.clouddn.com/trace.html)，可以下载。

## Analyzing a Trace
![Trace文件视图](http://7xktd8.com1.z0.glb.clouddn.com/trace视图.png)

当产生trace文件之后，就可以使用浏览器打开html文件了。
>实际使用chrome直接打开trace文件的时候，遇到如下错误: `Could not find an importer for the provided eventData.`。解决方案是:
>
>1. 在chrome中打开链接`chrome://tracing`;
>2. 在打开的页面的左上角有一个Load按钮(见图中`A`处)，点击选择trace文件，打开即可；

>另外，看图的时候记住一句话： A well-behaved application executes many small operations quickly and with a regular rhythm, with individual operations completing within few milliseconds, depending on the device and the processes being performed。

每一个渲染帧的App都会展现一行frame圆圈（见`C`处）——通常是绿色的，如果圆圈是黄色或者红色的，那就表明这一帧超过了16.6ms的帧渲染时间限制。放大视图就可以看清楚哪些影响App顺滑的帧了（见`D`处）。
>[官网](http://developer.android.com/intl/zh-cn/tools/help/systrace.html)上列出了关于该工具的一些命令行和快捷键。

点击某一个frame圆圈，这个圆圈就会被高亮，从而聚焦到绘制这一帧的时候所做的工作上。在运行着5.0以及更高版本系统的设备上，工作呗分为UI线程和Reader线程，在之前的版本上，创建帧的所有工作都是在UI线程上执行的。

在这一帧上点击不同的部分（即`D`处不同的色块），可以查看它们耗时多少。选中frame圆圈或者某一个部分，在`E`处都可以看到相关的有用信息，即Alert。

### Investigating Alerts
Systrace会对trace中的信息做一些自动分析，然后将存在的性能问题作为alert展示出来，从而为解决问题提供指导。

当选中某个存在问题的帧之后，alert就可能在`E`处展现：如果是UI线程符合太重，就可以使用[TraceView](http://www.muzileecoding.com/androidoptimize/Android-traceview.html)来诊断具体是什么导致的问题。

在整个面板的右侧，还有一个Alert Tab，你可以通过它查看trace文件信息中的所有的Alert以及它的数量。如下:
![帧问题诊断](http://7xktd8.com1.z0.glb.clouddn.com/帧问题诊断.png)
下面的红框展现了我所选当前问题帧暴露出来的三个Alert，而右侧红框则展现了当前trace文件中的所有alert的统计信息。
>这个非常有用，当操作比较集中时，可以很容易发现问题，配合TraceView可以比倨傲容易找出问题所在。官网建议把统计出来的问题当做Bug修掉。

## Tracing Application Code
framework定义的trace信号并不能全面展现app所做的所有事情，所以你可能需要添加你自己的信号，在Android4.3以及更高的版本上，开发者可以使用`Trace`类来给代码添加信号，这个技术可以帮助你观察任何时候App所做的工作。Trace开始和结束本身也会造成一定的影响。

下面是一个示例:

```java
public void ProcessPeople() {
    Trace.beginSection("ProcessPeople");
    try {
        Trace.beginSection("Processing Jane");
        try {
            // code for Jane task...
        } finally {
            Trace.endSection(); // ends "Processing Jane"
        }

        Trace.beginSection("Processing John");
        try {
            // code for John task...
        } finally {
            Trace.endSection(); // ends "Processing John"
        }
    } finally {
        Trace.endSection(); // ends "ProcessPeople"
    }
}
```
>要注意: `beginSection()`和`endSection()`要成对出现。而且两个方法要在同一个线程出现。

当使用App级别的trace，就必须要在用户UI或者命令行中使用`-a`/`--app=`指定包名。

当profile app的时候，最好打来app级别的trace功能，这样一些框架将通过trace提供一些非常重要的信息，比如`RecyclerView`。