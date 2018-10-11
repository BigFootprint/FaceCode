---
title: ReactiveX 初探
date: 2016-07-30 22:00:17
tags: [Reactive]
categories: ReactiveX
---

ReactiveX，[GitHub](https://github.com/ReactiveX/RxJava) 上的的定义如下：
>A library for composing asynchronous and event-based programs by using observable sequences.
>
>It extends the observer pattern to support sequences of data/events and __adds operators__ that allow you to compose sequences together declaratively __while abstracting away concerns about things like low-level threading, synchronization, thread-safety and concurrent data structures__.

它通过扩展观察者模式来支持基于数据/事件流的开发模式，允许声明一组操作来处理数据/事件，同时能够避免开发者花精力去关注一些底层的概念，比如线程、同步、线程安全以及并发数据结构。

下面来详细讨论一下它的特点。<!--more-->

## Rx的理念
Rx 是基于观察者模式的，这个模式在日常应用开发中非常常见，这里不多说。Rx 在观察者模式的基础之上，明确提出四个概念：

1. 被观察者(Observable)；
2. 观察者(Subscriber/Observer)；
3. 事件(Event)；
4. 观察关系(Subscription)；

被观察者代表的是事件的发出者，观察者是这个事件的处理者，事件可以理解为一个消息或者一块数据（这里可以想想EventBus），观察关系则是用来描述 A 订阅 B 发出的 E 事件这样一个关系，这个关系有时候需要被明确的管理，比如解除订阅，判断是否还在订阅。这四个概念共同组成了Rx的核心，括号里面的类名是RxJava对这四个概念的抽象。

除了抽象这四个概念以外，最前面那段定义还有一个关键的地方：__add operators__，也就是说可以添加操作。Rx不但提供了将事件从 Observale 发送到 Subscriber 的能力，还允许对事件添加一些操作，因此对于 Rx 来说，程序逻辑实际是这样的：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Rx理念.png" alt="Rx理念"/></div>

如图，使用 Rx 构建一段逻辑时，思路就是这样一个过程。我们要做的就是定义好图中的各个模块，以便于它们能够串起来工作。

## 为什么要使用Rx
这里建议读者先去看一下[扔物线](https://github.com/rengwuxian)童鞋的文章[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)。这篇文章中有几个经典的例子，说明了使用 Rx 的场景。

个人认为 Rx 的使用是有条件的：

1. 一个较长的逻辑链 & 存在嵌套；
2. 一个较长的逻辑链 & 线程间切换；

较长的逻辑链是指从已有的信息到最终需要输出的信息之间是有一个比较长、比较复杂的处理过程的，比如说扔物线所举的`imageCollectorView`例子，我们需要需要经过以下几个步骤：

1. 遍历所有的文件夹；
2. 遍历文件夹下面所有的文件；
3. 过滤出".png"结尾的文件；
4. 从文件中解析出Bitmap；
5. 把Bitmap添加到`imageCollectorView`中去；

这个链很长，并且嵌套着多层循环，如果直接以普通的`for`循环方式进行处理，看上去会比较复杂，但是修改成 Rx 的形式之后，就变得极为整齐，以 Lambda 形式查看代码更是享受。不但如此，这个例子还涉及到线程间切换：

1. 在解析出 Bitmap 之前，整个过程都应该发生在后台线程中；
2. 第5步则需要调度到主线程中进行；

一般来说在线程间进行切换的代码都非常丑陋，尤其是需要来回切换的时候。Rx 简化了这种切换所需要的形式，只需要简单的调用一个 API 就可以搞定。

为什么我一定要在较长的逻辑链后面添加几个条件呢？因为我觉得 Rx 对代码的可读性是有所降低的，如果仅仅是一些平坦的逻辑，使用普通的 Java 代码会是一个更好的选择，一层嵌套不需要，简单的从后台线程切换到主线程，使用 AsyncTask 可能会是更好的选择。而当使用普通的 Java 代码实现出来的东西已经让你抓狂，我觉得可以去尝试使用 Rx。

平时进行业务开发的时候一般不太会遇到这么复杂的情况，因为我们会用到很多的库，比如Volley，这些库通过封装后台线程操作，已经抹除了很大一部分的复杂性，因此业务层使用 Rx 的可能性好像不大。但是像 Retrofit 这样的底层库，涉及到复杂的流程操作的，使用 Rx 会是一个很好的选择，再也不用手动 new 那么多的 Handler 了。

## 总结
事实上，Rx 的亮点就在于线程的调度以及丰富的 Operator，以便于开发者通过简单的链式调用就可以很优雅的实现一个功能。

粗略的学习会有粗略的论断，前面给出的条件很可能非常片面，等深入学习各类操作符以及 Rx 的其余特性之后，再来补充修正。

## 其余资料
先占个坑，后面慢慢补上。

1. [ReactiveX 网站](http://reactivex.io/intro.html)
2. [GitHub ReactiveX Group](https://github.com/ReactiveX)

ReactiveX 网站上面有对各类操作的带图释义。

[文档中文翻译](https://www.gitbook.com/book/mcxiaoke/rxdocs/details)