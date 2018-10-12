---
title: 反编译APK
date: 2016-04-21 20:38:37
tags: [工具, 反编译]
categories: Android
---

以下是在Mac环境下进行的操作。

如果只是想看代码，准备以下两个工具即可:

1. __dex2jar__ [下载地址](https://sourceforge.net/projects/dex2jar/)
2. __jd-gui__ [下载地址](http://jd.benow.ca/)

这两个工具前者是为了把Android解压缩出来的`classes.dex`文件解析成jar文件，后者则是查看转化出的jar文件，即源码文件的。<!--more-->

### 具体步骤
1. 将APK文件重命名为.zip文件，并解压。在解压文件夹下面可以看到一个`classes.dex`文件（有的会有多个dex文件）；
2. 将下载下来的dex2jar压缩包解压，里面有一个`dex2jar.sh`脚本文件，执行如下命令：__`sh sh ./dex2jar.sh classes.dex文件地址`__即可生成对应的jar文件。
3. 打开jd-gui应用，打开生成的jar文件，就可以看到反编译出来的代码了；

以上可以反编译出对应的代码，但是资源文件，比如xml文件都不读，如果想要看这部分内容，需要用到另外一个工具:  
__apktool__ [下载地址](http://ibotpeaches.github.io/Apktool/) 
它的作用就是用于将资源文件反解析成接近原始格式的样子的，并且可以在修改后重新build文件；可以一行行debug smali代码。相关资料：

1. [安装教程](http://ibotpeaches.github.io/Apktool/install/);
2. [使用教程](http://ibotpeaches.github.io/Apktool/documentation/)
