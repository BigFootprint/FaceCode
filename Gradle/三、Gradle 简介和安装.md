---
title: Gradle系列八：简介&安装
date: 2016-06-12 19:41:53
tags: [Gradle]
categories: Android
---

## 一、简介
和 Maven、Ant 类似，Gradle 是一个优秀的构建工具。它基于一门 JVM 脚本语言——[Groovy](http://www.muzileecoding.com/gradlestudy/groovy.html)，具有以下特点：

1. 支持多种语言，官网说到，LinkedIn 使用 Gradle 完成对60多种语言的编译，包括 Java、Scala、Python 等；
2. 支持 Eclipse、Android Studio、IntelliJ 以及 Jekins 等工具插件；
3. 支持 Maven、Ivy、flat 文件等多种依赖处理，并且支持依赖传递；
4. 对整个构建过程的精细化控制；
5. 通过增量式构建以、构建缓存和并行构建三种方式最大限度的提高了Gradle Daemon 的编译速度；
6. 提供对编译过程的数据分析，便于开发者优化编译步骤和模块，解决编译问题；<!--more-->

>[官网](http://gradle.org/whygradle-build-automation/)上可以看到详情，在[User Guide第二章](https://docs.gradle.org/current/userguide/overview.html)中也有相关的描述。

## 二、环境
如果你的设备已经有完善的 Java 环境(JDK 6以及上)，那么以下简单的几步就可以配置好 Gradle 环境:

1. 从官网下载 ZIP 包: [链接](http://gradle.org/gradle-download/)；
2. 解压 ZIP 包到某个文件夹，地址 A；
3. 在环境变量中添加`GRADLE_HOME`变量，指向 A；
4. 把`GRADLE_HOME/bin`添加到 PATH 环境变量中去；

这几步就足以运行 Gradle 了。在命令行中执行`gradle -v`命令，如果能正确打印出 Gradle 的版本，就 OK 了。

另外要注意下载的 ZIP 包中，除了我们需要的 Gradle Binary ，还有以下内容:

1. User Guide（PDF & HTML）
2. DSL Reference
3. API Documentation(Java Doc and Goovydoc)
4. Sample
5. Binary source

>官网上可以找到非常完整的表述: [Installing Gradle](https://docs.gradle.org/current/userguide/installation.html)。