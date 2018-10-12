---
title: Gradle系列十三：实战
date: 2016-06-18 22:15:08
tags: [Gradle]
categories: Android
---

#### 生成项目报告
可以使用插件 __project-report__ ，具体见[The Project Report Plugin](https://docs.gradle.org/current/userguide/project_reports_plugin.html)。

这里面包含依赖（文本格式、HTML格式）、Task、属性等的报告<!--more-->

#### 指定JDK版本
在“gradle.properties”文件夹下面添加如下语句:

```Java
org.gradle.java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home
```
值是`JAVA_HOME`。

#### 关于"provided"
这篇文章有比较深入详细的解释：[PROVIDED SCOPE IN GRADLE](https://sinking.in/blog/provided-scope-in-gradle/)。实际上编译和打包这两个过程是独立的，这可以通过 SourceSet 的属性来控制：首先定义"provided" configuration，将由它配置的文件加到compileClasspath，这样编译即可通过，但是在runtimeClasspath中我们并不加入 "provided" 配置的文件，这就OK了。

>日了狗，Javan Plugin的`compile`居然不会把依赖打包入最终的jar包，与设想的很不一样。

#### 强制刷新缓存 

命令：`gradle build --refresh-dependencies`。