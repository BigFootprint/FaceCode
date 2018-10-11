---
title: XCode 8.2.1 ClangFormat 插件安装
date: 2017-03-23 19:36:59
tags: [工具]
categories: iOS
---

之前开发 Android ，Android Studio 有比较强大的 Format 工具，习惯性的写完代码顺手 Format 一下。然 XCode 本身并不支持 Format，需要手动调整，这让我痛苦无比而且觉得浪费时间，因此搜到了 [ClangFormat-Xcode](https://github.com/travisjeffery/ClangFormat-Xcode)。

本文主要记录安装过程，具体的使用教程可以参考：[超详细的Xcode代码格式化教程，可自定义样式](http://ios.jobbole.com/89104/)。

官网上提供了两种安装方式，一种是通过 Alcatraz 包管理工具，一种是 clone 代码下来到 XCode 中运行一下。这里使用第二种方法：<!-- More -->

1）clone 项目，导入到 XCode 中运行；

2）之后可以在文件夹 `~/Library/Application Support/Developer/Shared/Xcode/Plug-ins/` 下面发现一个文件 __ClangFormat.xcplugin__。这个文件夹下面存放的都是 XCode 上安装的插件，删除插件只要把这个目录下对应的插件文件删除即可；

3）在命令行执行命令：`sudo gem install update_xcode_plugins`；

4）在命令行执行命令：`update_xcode_plugins`，会看到如下的输出：

```java
2017-03-23 19:49:41.254 mdfind[15638:271887] Metadata.framework [Error]: couldn't get the client port
Found:
- Xcode (8.2.1) [E0A62D1F-3C18-4D74-BFE5-A4167D643966]: /Applications/Xcode.app

Plugins:
- ClangFormat (1.0)

Updating...

Finished! 🎉

It seems that you have Xcode 8+ installed!
Some plugins might not work on recent versions of Xcode because of library validation.
See https://github.com/alcatraz/Alcatraz/issues/475

Run `update_xcode_plugins --unsign` to fix this.
```

这里会列出你安装的插件，并且提示你有些插件不能使用，建议执行命令：`update_xcode_plugins --unsign` 进行修复；

5）在命令行执行命令：`update_xcode_plugins --unsign`；

6）完全退出 XCode，重新打开，会有一个弹窗出现，警告你有 "Unexpected code bundles"，选择 __Load Bundles__。

之后在 Edit 菜单选项中就可以看到 Clang Format 选项了，安装完成！跟着前面推荐的文章配置吧。