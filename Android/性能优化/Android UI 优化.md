---
title: UI优化
date: 2016-05-10 20:03:42
tags: [性能优化, 工具]
categories: Android
---

## Overdraw Debugger
过度绘制的诊断可以直接使用手机的开发人员功能支持：

1. 打开设置——>开发人员选项；
2. 找到"Debug GPU Overdraw"；
3. 点击该项，选择"Show ovedraw areas"；

这个时候你的屏幕就变得五颜六色了：

1. 真实色彩：没有过度绘制；
2. 蓝色：过度绘制一次；
3. 绿色：过度绘制两次；
4. 粉色：过度绘制三次；
5. 红色：过度绘制四次以及以上；

有些过度绘制是不可避免的，我们的目标就是尽量让我们的App展现真实色彩或者蓝色。<!--more-->

## Rendering Profiler
Profile GPU Rendering使得开发者可以方便地知道系统是否正在以16ms每帧的速度绘制当前UI，并且可以查看是哪一个绘制的步骤超时。使用方式如下:

1. 打开设置——>开发人员选项；
2. 找到"Profile GPU Rendering"；
3. 点击打开弹窗，选择"On screen as bars"；

你的屏幕上底部就会出现彩色的线条:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/gettingstarted_image002.png" height="320" alt="Screen when Profile GPU Rendering is on"/></div>

详细的含义如下:
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/gettingstarted_image003.png" height="160" alt="Screen when Profile GPU Rendering is on"/></div>
绿色的横线代表16ms/帧的上限（所有的竖线都应该在这条线之下，超过这条线则可能意味着动画的卡顿），而每条竖线则包含着蓝色，紫色(只有4.0以及以上版本才有)，红色以及橙色，它们代表着一次绘制的几个过程

1. 蓝线代表用于创建/更新View的display list的时间，如果一根竖线的这部分很高，可能就意味着有很多的自定义View需要绘制，或者在`onDraw()`方法里面执行了太多的任务；
2. 紫色(只有4.0以及以上版本才有)代表将资源传送到render线程所花费时间；
3. 红色代表Android的2D render发送命令到OpenGL来绘制或者重绘display list所花费的时间，越多的display list意味着更高的红线；
4. 橙色代表CPU等待GPU完成任务的时间，如果这段线很长，那就意味着App在GPU上做了太多的工作；

>虽然这个工具叫做`Profile GPU Rendering`，但实际上监控的任务都在CPU上面，渲染是通过提交命令给GPU来实现的，GPU异步去渲染屏幕。在某些情况下，GPU有太多的任务需要去做，就需要CPU等待一段时间再去提交命令，当这种情况发生的时候，就可以看到很长的Process(橙色)和Execute(红色)线，命令的提交会阻塞，直到GPU命令队列里面有足够的空间容纳新的命令。

优化通用手法：降低ViewTree层级。

## Hierarchy Viewer
这个工具可以可视化App的View结构，并且标注每一个View的渲染速度。我们可以使用这个工具：

1. 降低层级，减少过度绘制；
2. 找到潜在的渲染性能瓶颈；

使用前提条件:

1. 必须使用真机来获得准确数据；
2. 在机器上设置一个`ANDROID_HVPROTO`变量（[配置文档](http://developer.android.com/intl/zh-cn/tools/performance/hierarchy-viewer/setup.html)）；

具体的过程参见[官网](http://developer.android.com/intl/zh-cn/tools/performance/hierarchy-viewer/profiling.html)。

每一个View都可以通过Profile产生三个小点，分别代表Draw、Layout、Execute三个过程。三个点都可以由三种颜色，绿色表示至少比其余一半的View快，黄色表示不是最慢的那一半View，红色表示是最慢的那一半View。

#### 解析
因为Hierarchy Viewer衡量的是相对性能，因此整个Tree中总是有红点，因此红色并不代表很慢。下面讲述一些处理红点View的办法:

1. 找到那些处于叶子节点或者拥有很少子View的ViewGroup，这类View很可能存在问题，使用Systrace或者TraceView可以获得更多信息；
2. 如果一个ViewGroup带有很多的子View，并且显示红色，可以看一下子View的性能如何；
3. 一个显示黄色甚至红色的View在设备上运行并不一定慢；
4. 根View有一个红色的measure阶段，红色的layout阶段以及黄色的绘制阶段是很正常的；
5. 如果一个叶子结点在一个有20个以上的View的Tree中显示红色，这很可能就意味着发生了问题，可以好好看一下`onDraw()`方法；
