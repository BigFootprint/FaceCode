---
title: Instant Run 简介
date: 2016-05-03 19:07:35
tags: [工具]
categories: Android
---

翻译自: [Instant Run](http://tools.android.com/tech-docs/instant-run)

>Instant Run并不支持[Jack](https://source.android.com/source/jack.html)，Jack需要Android N提供的对Java 8的语言特性的支持。关于Java 8和Jack，可以看[这里](http://developer.android.com/intl/zh-cn/preview/j8-jack.html)。

Instant Run在Android Studio 2.0正式进入生产环境，主要目的是为了减少通过`Run`和`Debug`部署App的时间。Instant Run不会在项目有更改之后重新打包APK，而是会把代码和资源的变化直接Push到机器上，因此变化几乎是瞬时反应在设备上的。<!--more-->

当配置好项目并运行一次之后，`Run`的箭头Icon左边就会出现一个黄色的闪电Icon（`Debug`也是一样，需要先跑一次），这就表示Instant Run已经准备好为你更新目标应用了——某些时候它甚至不需要重启你的Activity来反映变化，更改的效果会立刻显示出来。

>【注意】Instant Run会暂时性地关闭了JaCoCo和Proguard。但是因为Instant Run只对debug构建有效，因此这种行为不会影响你的release构建。

Instant Run有三种方式可以将更新后的项目Push到设备上：Hot Swap，Warm Swap和Cold Swap，它会自动选择一种方式进行更新。下面的表格描述了项目发生变化之后Instant Run会如何Push更新:

| 代码变化           | Instant Run行为                            |
| -------------- | ---------------------------------------- |
| 实例方法或者静态方法代码变化 | Hot Swap: 最快的Push变化的方法，几乎可以立即看到变化，应用程序会继续运行，下次调用这个方法的时候，新的方法就会被调用 |
| 更改或者移除资源       | Warm Swap: 这种方式依然很快，但是在Push变化的时候要求Activity重新启动，App本身不会重启 |
|代码结构变更，比如: 1)增加、删除或者改动注解、实例变量、静态变量、静态方法签名、实例方法签名；2）改变继承的父类；3）改变实现的接口；4）变更静态构造器；5）使用动态Id重新构建Layout|Code Swap(API Level 21以及以上才支持): Instant Run会将变化的结构代码Push到目标设备上，并重启整个App； API Level 20以及以下会重新构建APK。
|1）更改Manifest文件；2）更改Manifest文件引用的资源；3）更改widget的UI元素(需要执行`Clean and Rerun`)|对于前两者，Android Studio会自动重新构建，这是因为诸如icon、名字、intent-filter是在APK安装的时候就确定了。如果构建过程默认更改了Manifest文件，那么你就不会获得任何Instant Run带来的好处，因此不建议在构建中自动更改任何的Manifest文件内容。|

Android Studio默认会在Hot Swap之后保持App运行并重启当前的Activity，这个设置可以更改:

1. 打开Settings或者Prefereneces面板；
2. 打开Build, Execution, Deployment > Instant Run；
3. 取消选中`Restart activity on code changes`；

当这个设置被取消之后，开发者可以在菜单栏的Run菜单中选择`Restart Activity`来手动重启当前的Activity。

#### 使用`Rerun`
当诸如`onCreate()`方法发生变化后，你可能会需要重启App来反应变化，这个时候可以点击`Rerun`命令(Icon是一个右下角一个方块，左边一个向上弯曲的箭头)来停止App运行、执行增量构建并重启App。

如果需要进行一个Clean构建，可以选择`Run > Clean and Rerun 'app'`命令进行，或者按住`Shift`键的同时按下`Rerun`命令。这个动作会停止当前运行的App，执行一个全新的构建，然后将新的APK部署到目标设备上去。

## 为项目配置Instant Run
当使用2.0.0或者更高版本的Android Gradle插件时，Android Studio自动应用Instant Run。为了让已有项目支持Gradle插件的最新版本，可以做以下配置:

1. 打开Settings或者Preferences面板；
2. 跳转到Build, Execution, Deployment > Instant Run，点击`Update Projec`，如果没有出现这个选项，那就说明这个项目已经支持最新的插件了；

## 需要知道的事情
Instant Run是设计来加速大多数情况下的构建部署过程的，下面是一些需要注意的地方。

#### Legacy Multidex with Android 5.0 or higher
如果项目是Multidex的，即build.gradle里面配置了`multiDexEnabled true`和`minSdkVersion 20`或者更低版本，且目标设备运行的是Android 5.0(API Level 21)或者更高版本，那么部署一个Clean构建的效率可能会下降。当初始化部署完成之后，增量构建就会显著加快。

这里通过引用涉及到另外一处[文档](http://developer.android.com/intl/zh-cn/tools/building/multidex.html#mdex-pre-l)。引用的文档描述的很清楚:
>如果你项目是Multidex的，同时minSdkVersion配置的是20或者更低，并且目标设备运行的是Android 4.4(API Level 20)或者更低版本，Android Studio是不支持Instant Run的。

解决办法是新建一个Product Flavor，将`minSdkVerion`提高到20以上。

#### 超出64K方法限制
Instant Run会在每一个Debug APK中增加额外的方法，数量如下:
>approximately 140 plus three times the number of classes and their local dependencies。
>
>恕我英语渣...

因此可能会造成一些APK超出方法数限制，解决办法可以参考[configuring apps for Multidex](http://developers.android.com/tools/building/multidex.html#mdex-gradle)。

#### 部署到多设备
因为Instant Run在选择更新方式的时候使用了不同的技术，这依赖于目标设备的API Level，因为这个原因，在一次部署到多个设备的时候，Android Studio会暂时关闭掉Instant Run。

#### Instrumentation测试
当运行Instrumentation测试的时候，Android Studio会关闭这个特性，不会插入额外的代码。

当分析一个App的时候，Instant Run会造成一定的性能影响，这些影响会干扰性能分析工具提供的信息，另外它也会使得stack trace更加复杂，因此这个时候需要关闭掉Instant Run。

## 关闭Instant Run
1. 打开Settings或者Preferences面板；
2. 跳转到Build, Execution, Deployment > Instant Run；
3. 取消勾选`Enable Instant Run`；


-----
>__今天是2016-05-03，上面描述的内容可以作为参考，实际情况已经有所变化。__