---
title: Groovy 安装和 Eclipse 插件安装
date: 2016-01-29 21:18:34
tags: [Gradle]
categories: Android
---

## Groovy的安装
Groovy的安装可以参考[官网上的教程](http://www.groovy-lang.org/download.html)，我是在Mac上安装的，所以可以很偷懒的直接执行命令：__brew install groovy__。

在本地安装后需要配置一下环境变量，就和Java一样。在命令行里面执行如下命令：__vi ~/.bash_profile__。单独加入如下这行:

```java
export GROOVY_HOME=/usr/local/opt/groovy/libexec
```
然后退出保存，接着执行 __source ~/.bash_profile__ 即可。在命令行里面输入 __groovy -v__ 命令，有类似如下输出：

```java
Groovy Version: 2.4.3 JVM: 1.6.0_65 Vendor: Apple Inc. OS: Mac OS X
```
就表示安装成功了。<!--more-->

## IDE的插件集成
首先说明两点:  
1）groovy自己是有一个简单GUI组件可以实现编码的，如果怕麻烦，可以直接使用，方法是在命令行里面执行 __groovyConsole__ 命令，你就可以看见下面的UI：

![Groovy自带GUI](http://7xktd8.com1.z0.glb.clouddn.com/GroovyGUI.png)
这个简单的编辑器基本可以满足联系的需要，上半部分是编码的，下半部分是输出，编写完成之后执行Cmd+R即可运行程序。另外字体也可以放大缩小，考虑的还是比较全面的。读者可以研究一下菜单栏，命令不多，可以很快的过一遍；  
2）虽然平时使用Intellij比较多，但这里只讲一下如何在Eclipse中配置插件，原因是我对这个比较熟悉；

看到这里，先不要急着去下载Eclipse，Groovy的安装对Eclipse的版本有要求。插件是一个[Github项目](https://github.com/groovy/groovy-eclipse/wiki)，它的[Wiki](https://github.com/groovy/groovy-eclipse/wiki)里面写了支持的Eclipse和对应的插件下载链接，比如写这篇文章的时候，它最新支持到4.4，而Eclipse已经到4.6了，Wiki里面有如下描述：

![Eclipse和Groovy插件版本对应图](http://7xktd8.com1.z0.glb.clouddn.com/Eclipse和Groovy插件版本对应图.png)
看着这个，去下载一个4.4(Luna)版本的Eclipse并安装完成。打开Eclipse，在菜单栏中依次点击：Help->Install New Software...，就看到如下界面：

![Eclipse安装Groovy插件](http://7xktd8.com1.z0.glb.clouddn.com/Eclipse安装Groovy插件.png)
点击Add，弹出Add Resposity框，随便取一个名字，之后将前面表格中对应版本的插件下载链接填到Location里面，点击OK，点击Next，在拉下来的组件中，只有一个是Required，其余的可以不选，之后一路Accept & Next，直到完成安装。

最后Eclipse会要求重启，重启之后，插件安装完成。