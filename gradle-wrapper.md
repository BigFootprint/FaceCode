---
title: Gradle系列十：Warpper
date: 2016-06-18 10:45:21
tags: [Gradle]
categories: Android
---

大部分的软件都需要提前安装才能使用，如果安装很简单那么还可以接受，但是有时候安装需要考虑版本问题、配置环境，是比较复杂的。构建软件的工具版本有时候还是随着软件的迭代进行升级的，这样安装构建工具更加麻烦。

Wrapper 是 Gradle 提供来解决这个问题的。<!--more-->

### 添加Wrapper
在上面讲`gradle tasks`命令的时候，有一组 Task 被归类为`Build Setup tasks`，它们是用来初始化一个构建项目的，我们新建一个空文件夹，在该文件夹下面执行`gradle init`命令，再查看这个文件夹的内容，会发现内容如下：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/gradle项目初始状态.png"  alt="gradle项目初始状态"/></div>

如图，这就是一个标准的 Gradle 构建项目的配置。

另外还有一个命令是`gradle wrapper`，在我们最开始的例子中，我们只新建了一个`build.gradle`文件，执行该命令之后，也能得到如上的文件结构(不过没有`settings.gradle`文件)。这个命令实际可以指定需要配置的 Wrapper 的版本，如下:`gradle wrapper --gradle-version 2.0`即可。

在这些文件中，有四个文件就是为 Wrapper 功能生成的:

1. gradlew (Unix Shell script)
2. gradlew.bat (Windows batch file)
3. gradle/wrapper/gradle-wrapper.jar (Wrapper JAR)
4. gradle/wrapper/gradle-wrapper.properties (Wrapper properties)

### Wrapper使用
如果项目配置了 Gradle Wrapper (存在上面的目录结构)，就可以在根目录下面执行下面两个命令:

* __`./gradlew <task>`__ （在类Unix机器上，比如Linux或者Mac）
* __`gradlew <task>`__ （在Windows机器上）

因为每一个 Wrapper 都指定了一个特定版本的 Gradle，所以当我们第一次执行以上命令的时候，它就会去下载对应版本 Gradle，然后用这个版本去构建项目——下载的 Gradle 存放在`$USER_HOME/.gradle/wrapper/dists`。
>Wrapper提供了绑定 Gradle 版本到项目以及下载对应版本的 Gradle 的能力，这样只需要把 Wrapper 相关文件和项目一起提交到版本控制工具中，就可以保证以后构建这个项目的时候，始终使用的是同一个版本的构建工具。

如果使用 Wrapper 构建，那么本机上的任何构建版本都会被忽略。

### 配置
在`gradle-wrapper.properties`文件中，有一个配置项如下:

```Java
distributionUrl=https\://services.gradle.org/distributions/gradle-2.3-bin.zip
```
这个 Url 指定了 Gradle 的下载地址，可以修改这个地址下载不同版本的 Gradle。