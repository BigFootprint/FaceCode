---
title: Android App代码优化
date: 2015-10-28 14:57:58
tags: [Android, 代码优化]
categories: Android
---

介绍几个可用于提高 Android 平台代码质量的工具。

## NO.1 StrictMode
[官网介绍](http://developer.android.com/intl/zh-cn/reference/android/os/StrictMode.html)在此。  
推荐阅读文章：[StrictMode 详解](http://android-performance.com/android/2014/04/24/android-strict-mode.html) & [Android性能调优利器StrictMode](http://droidyue.com/blog/2015/09/26/android-tuning-tool-strictmode/)

__StrictMode 有两大类策略：监控线程和 VM。__ 线程方面，主要用于检测一些在主线程意外进行的耗时网络操作和磁盘读写操作，同时也提供方法用于检测开发者需要监控的耗时代码块。VM 方面主攻内存泄露：包括 Activity、SQLite 等。<!--more-->示例代码如下：

```java
protected void onCreate(Bundle savedInstanceState) {
	if (DEVELOPER_MODE) {
		StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                        .detectAll()
                        .penaltyLog()
                        .build());
		StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                        .detectLeakedSqlLiteObjects()
                        .detectLeakedClosableObjects()
                        .penaltyLog()
                        .penaltyDeath()
                        .build());
	}
	super.onCreate(savedInstanceState);
}
```
标记比较慢的代码，可以使用如下方法：

```java
StrictMode.noteSlowCall("slowCallInCustomThread");
```
被`noteSlowCall`标记的方法，如果执行过慢则会触发 StrictMode 报警，这个方法只有用在主线程调用的方法中时，才会有效果（实践证明）。

有时候有些特殊 case 可以绕过，不需要检测，比如主线程是可以进行一些快速简短的磁盘读写操作的，此时可以如下 Coding:

```java
StrictMode.ThreadPolicy old = StrictMode.getThreadPolicy();
StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder(old)
    .permitDiskWrites()
    .build());
// 方法调用
doCorrectStuffThatWritesToDisk();
StrictMode.setThreadPolicy(old);
```

PS：在开发者选项里面，也可以直接启用严格模式，如果主线程中有执行时间长的操作，屏幕则会闪烁。

## NO.2 Findbugs
[官网介绍](http://findbugs.sourceforge.net/users.html)在此。  
它会给代码做一个全面的扫描，根据规则去判断代码中的一些不规范和潜在问题，比如多线程问题。

它的结果可以以 xml 和 html 展示出来。

## NO.3 PMD
[官网介绍](http://pmd.sourceforge.net/pmd-5.1.1/)在此。  
和 Findbugs 差不多，也是代码静态扫描工具，并且两者有一些重复的功能。PMD 的定制性比较强，而且有部分功能是专门用于检测 Android 项目（虽然很弱）：

1. CallSuperFirst: 应该在方法第一行调用 super 方法；
2. CallSuperLast: 应该在方法最后一行调用 super 方法；
3. DoNotHardCodeSDCard: 不应该硬编码 SDCard 的路径；

同样，它的结果也可以以 xml 和 html 展示。

## NO.4 CheckStyle
[官网介绍](http://checkstyle.sourceforge.net/)在此。  
检查代码风格的工具。CheckStyle 插件使用需要一个配置文件，定义具体的代码风格，可以参考 StackOverFlow上的[这篇帖子](http://stackoverflow.com/questions/9339804/where-can-i-find-checkstyle-config-for-android-coding-style)，里面有诸如[Picasso](https://github.com/square/picasso/blob/master/checkstyle.xml)、[Google](http://checkstyle.sourceforge.net/google_style.html)等定义的 CheckStyle 可以参考。

Findbugs 和 PMD 在 gradle 中的配置都非常方便，可以参考[GitHub上这个配置](https://gist.github.com/rciovati/8461832)。也可以直接使用[这个插件](https://github.com/noveogroup/android-check)，它集成了 checkstyle、findbugs 和 pmd 三个工具。