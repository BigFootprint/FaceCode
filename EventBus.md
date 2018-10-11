---
title: EventBus杂谈
date: 2015-10-20 19:08:31
tags: [源码]
categories: Android
---

关于EventBus源码的解析，已经有很多文章，比如[EventBus 源码解析](http://a.codekk.com/detail/Android/Trinea/EventBus%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)。此处就不赘述了，我的代码阅读标注在[这里](https://github.com/BigFootprint/AndroidSourceReader/tree/master/EventBus)。

下面随便聊一下。

Android开发一直在说MVC模式，C这块在客户端App上已经不是很明显，M和V两块单独分离是比较简单的。M层本身也可以比较容易的进行架构组织，但是整体在拆的时候会出现两个问题：<!--more-->

1. V对M的变化监听实现比较复杂。在Android中，要么使用回调(需要声明接口，并添加监听)，要么使用广播(需要声明广播监听器，最重要的是传输的数据只能通过Intent，伴随的是声明一堆static final变量)，两种实现成本都比较高，重复工作很多；
2. V本身的拆分比较困难。许多复杂的页面，不可能将所有的View操作都写在同一个类中，因此会独立出一些Custom View(这个一般比较好处理)和Fragment，而Fragment之间的通讯是一件非常麻烦的事情：需要通过Activity中转(得判断Activity是否为空)或者广播进行。实际上，Activity之间的通讯也存在相似的问题。

这些问题的出现在呼唤一个东西的出现——一种更高效的代码间通讯方式。这就是使用EventBus的最佳理由。

EventBus可以使得在代码上看上去毫无关系(只有需要传送的信息对象)的两个对象之间能够完成信息传递，并且代码简短高效。

EventBus提供的一些高级特性：比如Priority(可以用于事件拦截)、Sticky(想了一下，如果A页面跳转到B页面，A页面同时又在等待B页面的信息，此时应该使用sticky事件，因为A页面可能会因为内存不足而被杀掉，使用Sticky则可以在A页面恢复的时候再度获取到事件)，可以为一些特殊场景提供便利。

然而EventBus并不是毫无缺点的，__最不好的一点__是:  
EventBus在事件类型以及监听者的实现上考虑到了继承，但即便不考虑，也会有一个比较麻烦的地方——太多的EventBus会导致程序逻辑不清晰。举个栗子，原先Activity和Fragment之间通讯，可能是Fragment直接调用getActivity()后强转类型，然后调用某个方法，现在可能直接post一个事件，两端代码相比较，明显前者更轻易能看出Fragment在和谁、什么方法通信，再者，如果某个onEvent方法是出现在Activity的父类中，整体代码看上去就更加隐晦，可读性变差。

因此使用时注意适量，并适当添加注释。