---
title: Gradle系列十一：Daemon
date: 2016-06-18 10:47:27
tags: [Gradle]
categories: Android
---

维基百科定义:
> A daemon is a computer program that runs as a background process, rather than being under the direct control of an interactive user.

Gradle 运行在 JVM 上，需要耗费一定的时间运行多个支持库，因此看上去运行起来比较慢，Gradle 使用 Daemon 来解决这个问题：__通过一个长期运行在后台来执行构建的进程，可以缓存一些构建数据在内存中，利用 JVM 在运行代码期间对代码的优化（避免每次运行都做优化）来提高构建速度__。

决定是否使用 Daemon 只需要简单的配置即可，且使用和不使用 Daemon 的时候构建没有任何不同。<!--more-->

### 1. 打开Daemon
__默认情况下是不打开 Daemon 的，但是强烈建议在开发机器上打开这个配置（持续集成服务器上关闭）。__ 有很多方式打开 Daemon，但是最常用的方式是在`«USER_HOME»/.gradle/gradle.properties`中加入下面一行:

```Java
org.gradle.daemon=true
```
>«USER_HOME» 通常的目录是:
>
>1. C:\Users\<username> (Windows Vista & 7+)
>2. /Users/<username> (Mac OS X)
>3. /home/<username> (Linux)
>
>如果找不到对应文件，可以新建一个。

一旦打开 Daemon，不论 Gradle 是什么版本，所有的构建都会得到速度的提升。

除了上面的方法，还可以通过添加`--daemon`或者`--no-daemon`来对单次构建进行配置。

### 2. 关闭Daemon
每一个 Gradle Daemon 进程在3小时无活动的情况下都会自动关闭。但是我们也可以通过执行`gradle --stop`命令显示的关闭它 —— 这个命令会关闭所有的执行该命令所用同一版本的 Gradle 唤起的 Daemon 进程。
>如果安装了 JDK，可以很简单使用`jps`命令查看关闭情况。
>
>有时候会有多个 Gradle Daemon 进程，原因可能是因为 Gradle 找不到空闲的 Daemon 进程和 Compatible 进程（比如 JVM 配置不符合当前构建的要求）—— 关于这块，文档中有详细的描述。
>
>实际在关闭之后还是会看到`GradleDaemon`命名的进程。