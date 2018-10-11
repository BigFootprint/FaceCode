---
title: Android单元测试(一)——测试分类和基础
date: 2015-11-17 17:21:50
tags: [测试]
categories: Android
---

翻译总结一篇讲述Android单元测试的文章，比官网的讲述更具条理。同时描述一些在实践中遇到的坑。

__官网文档入口__  [Best Practices for Testing](http://developer.android.com/intl/zh-cn/training/testing.html)  
__英文原文地址__  [Developing Android unit and instrumentation tests](http://www.vogella.com/tutorials/AndroidTesting/article.html)

>【建议】读者最好先大致看一下官网文档，理清楚相关概念，然后再比对英文原文和本文的翻译，边阅读边实践。

## 为什么Android应用需要测试？
### 为什么测试Android应用尤其重要？
android应用在一种资源（内存，CPU，电量等）有限的情况下运行，并且运行情况依赖于外部因素，比如是否联网，使用情况等，因此测试优化android应用非常重要，对应用进行适当的测试有助于提高应用质量，增强可维护性。<!--more-->

### Android测试策略
在所有不同配置的设备上测试一个应用是不大可能的，一般来说只能在典型的设备上进行测试——测试的时候应该选取一个尽可能低配置的设备和尽可能高配置的设备进行测试，所谓配置，是诸如屏幕分辨率，像素密度等。

### Android的测试支持
2015年用于Android应用的测试工具和框架得到了极大的提升。Android的整个测试支持系统被更新到基于JUnit4，单元测试代码既可以运行在JVM上，也可以运行在Android设备上。同时，Google引入了新的UI测试框架——Espresso，使得测试Android应用的UI变得更加容易。

## 前置要求
下面的描述基于如下假设：

1. 读者已经知道如何创建一个Android应用，读者可以阅读[Android Tutorial](http://www.vogella.com/tutorials/Android/article.html)获取更详细的消息;
2. 读者了解Android的编译系统——Gradle，读者可以阅读[Building Android applications with Gradle Tutorial](http://www.vogella.com/tutorials/AndroidBuild/article.html)进行基本了解;
3. 对JUnit测试框架有一定的了解，这个网上资料很多，推荐一篇[Java单元测试(Junit+Mock+代码覆盖率)](http://thihy.iteye.com/blog/1771826)

## Android自动化测试
### 测试什么？
一般来说，我们测试的重点是应用的业务逻辑，以下的测试比例是比较合理的：

1. 70%~80%的单元测试用来保证底层代码的稳定性；
2. 20%~30%的功能测试用来保证应用的确可以正常运行；
3. 少量的功能测试用于测试你的应用和别的应用的交互——如果有的话；

### 测试前置条件
在Android测试中为所有的测试声明一个 __testPreconditions()__ 方法是一个很好的做法，如果这个方法执行失败了，你可以立即知道其他测试所依赖的前置条件不满足（换言之：通过这个方法保证为后续的测试提供一个稳定的测试环境）。

### 测试单个App或者多个App
测试另外一个重要考量标准就是你是单独测试你自己的应用还是测试你的应用和别的应用的集成（即两者的交互）。如果你只是测试你自己的应用，那么你可以使用一些需要了解应用内部信息的测试框架（比如viewId）——这涉及到测试框架的选择。

## Android普通单元和Instrumentation单元测试
### Android单元测试种类
Android单元测试是基于JUnit的，可以分为两类：

1. 本地单元测试——测试运行在JVM上；
2. Instrumentation单元测试——测试需要运行在Android系统上；

测试的时候应该尽可能使用本地单元测试，因为本地单元测试相较而言更快，Instrumentation单元测试需要部署应用并且在Android设备上运行测试。

### 本地单元测试
Android Gradle支持在JVM上运行Android单元测试，为了实现这个目标，Gradle创建了一个特殊版本的android.jar（也成为Android mockable jar）来提供单元测试需要的各种属性、方法、类。调用这个jar包中的任何函数都会导致异常。

因此，如果你的类没有调用任何的Android API或者只是有很简单的依赖，你可以毫无限制的使用JUnit测试框架（或者别的任何Java单元测试框架）。单元测试代码中任何对Android的依赖都应该被替换掉，可以使用诸如Mockito这样的Mock框架。

在JVM上运行测试case的好处就是速度——比起在Android机器上运行要快很多很多。

### 使用Instrumented test测试使用了Android API的类
如果你需要测试使用了Android API的代码，你需要在Android设备上运行测试代码（因为Android工具上mock出来的android.jar并不执行真正的Android代码，而只是简单的抛出异常），不幸的是，这使得测试的执行漫长了许多。

## Android的项目结构和测试文件夹的创建
### Android的测试项目结构
比较推荐的方式是按照约定组织source代码和测试代码（Gradle有自己的约定），你的应用项目结构应该按照下面的文件夹结构进行组织:

* app/src/main/java - 放置项目的源码
* app/src/test/java - 放置运行在JVM上的单元测试代码
* app/src/androidTest/java - 放置需要运行在Android设备上的测试代码

如果你按照这个约定来，那么Android的编译系统（Gradle）会自动将对应的测试代码运行在JVM和Android设备上。

Gradle上可以配置这几个文件夹，读者在不熟悉Gradle的情况下并且没有特殊需求的情况下，不建议修改这个约定。

## 编写本地单元测试并且运行
先上图，让读者有个大致的了解:

![Android本地单元测试项目结构](http://7xktd8.com1.z0.glb.clouddn.com/Android本地单元测试.png)

### 简介
在Android上我们使用术语单元测试来表示那些运行在本地JVM上的测试代码。单元测试应当被用来验证一个Activity的状态以及它与其余组件的交互，前提是在独立的环境下（与系统其余部分没有联系），它一般用来测试代码的一小部分，比如一个方法，一个类或者一个组件，且不依赖系统或者网络资源等外部环境。举个栗子，假设在Activity上有一个button是用来启动另外一个Activity的，单元测试应该被用来测试启动Activity对应的intent是否正确，而不是那个Activity是否被启动。

如前面所述，单元测试的执行基于一个被修改的android.jar包，所有的final修饰符都被移除掉了，这使得使用mock库成为现实——如果你需要依赖Android平台，你就使用mock的框架来代替那些调用。
>实践中发现这一点极大的限制了本地单测的应用。我们有一个业务使用了SparseArray，导致逻辑完全不可侧，因为Mock出来的对象完全不具备SparseArray功能。

### 依赖
开发者官网上建议在build.gradle中添加如下依赖:

```xml
dependencies {
    // Unit testing dependencies
    testCompile 'junit:junit:4.12'
    // Set this dependency if you want to use Mockito
    testCompile 'org.mockito:mockito-core:1.10.19'
    // Set this dependency if you want to use Hamcrest matching
    testCompile 'org.hamcrest:hamcrest-library:1.1'
}
```
每个包的作用注释中都已经标出。注意！！！这里千万不要写成compile，然后去File——>Project Structure——>选中Module——>Dependencies里面，再将这三个库的Scope改为Test Compile，实践中，这样操作这里会变成androidTestCompile。总之，请确保这里是testCompile。

### 代码位置
我的Android Studio版本是1.3.2，新建的Android项目下面自带androidTest目录，但是并没有test目录。按照前面所讲述的目录结构约定，我们将项目视图切换到Project下面，在src目录下面新建一个test目录。读者注意上图中的1，2，3三个部分，另外读者应该点击5，调出"Build Variants"视图，然后将Test Artifact设置为Unit Test。

### 代码编写
这里给出一个实例，读者可以直接Copy测试一把:

```java
public class FirstTest {
    @Test
    public void test_First(){
        String test = "aa";
        assertEquals(test, "aa");
    }

    @Test
    public void test_Second(){
        String test = "aa";
        assertEquals(test, "aa");
    }

    @Test
    public void test_Third(){
        assertEquals(NumberUtils.calculateSum(100), 100);
    }
}
```
写完类之后，请打开Run——>Edit Configurations视图，然后确保整个视图是如下显示的:

![Android单元测试配置](http://7xktd8.com1.z0.glb.clouddn.com/Android单元测试配置.png)

之所以要注意这里，是因为我第一次配置的时候，不知道因为什么原因，写了一个单元测试case，却配置成了Instrument测试case，导致每次运行case的时候都会要求启动Android设备，然后报下面的错误:

```
Running tests
Test running startedTest running failed: 
Instrumentation run failed due to 'java.lang.RuntimeException'
Empty test suite.
```

这个问题困扰了我很久。因此读者一定要注意。

### 运行
最后是运行，我们只需要右击你需要运行的测试Class，选择"Run FirstTest"即可运行，然后在图中7的部分可以看到测试执行的结果。6圈出的按钮可以点击，查看执行的全部case或者执行失败的case。

测试报告在`app/build/reports/tests/debug/`下面。

### 注意
在写本地单测的时候，会遇到`android.jar`某个方法没有被Mock的情况，此时可以通过如下配置:

```xml
android {
  // ...
  testOptions { 
    unitTests.returnDefaultValues = true
  }
} 
```
让Gradle系统为该方法返回默认值。

## Instrumentation——Android底层测试API
### 简介
Android测试API提供了一些深入Android组件和应用本身声明周期的钩子函数。这些钩子函数称为Instrumentation API——它们允许你控制生命周期和用户交互事件。

在正常情况下，Android应用只能对真正的生命周期和用户交互事件做出反应，比如：如果Android创建了一个Activity，那么onCreate()方法就会被调用，或者用户点击了一个按钮，一个按键，那么相应的监听方法就会被调用。但是通过Instrumentation，你可以在测试代码中控制这些事件。

只有基于Instrumentation的测试代码才能在测试环境下向你的应用发送事件。举个栗子：你可以在测试中调用getActivity()方法来唤起一个Activity并获得一个Activity的实例，然后你可以调用finish()方法结束它，然后再调用getActivity()，这样你就可以测试一个Activity能否正确恢复状态。

### Android系统如何执行测试
InstrumentationTestRunner是Android单元测试的基础执行器（Runner，这个概念来自JUnit，前面推荐的文章中有提及），这个测试执行器会加载所有的测试方法，通过Instrumentation API来和Android系统交互。

如果你为Android应用启动了一个测试，Android系统会立刻终止被测试应用的进程，然后启动一个新的实例。它不会启动应用，这是测试方法的职责——测试方法控制应用组件的整个生命周期。

测试执行器在初始化界面的过程中，也会调用Application和Activity的onCreate()方法。（可以认为Instrumentation API提供了一个功能让测试代码扮演系统的角色）。

### Instrumentation框架的使用
有了运行在JVM上的Android单元测试和类似Espresso这样流行的UI测试框架，开发者很少需要直接调用Instrumentation API。

## Instrumented unit testing
### 在Android上使用Instrumented unit testing
Instrumented unit testing是运行在Android真实设备或者模拟器上的测试代码，而不是JVM上。这些测试代码可以获取真实设备的资源，以便于测试那些不能被mock框架简单mock出来的功能模块。

[Mockito框架](https://github.com/mockito/mockito)可以被用来模拟部分的Android系统环境（读者可以查看它的release note，1.9.5版本即支持这个功能），这也是google官方推荐的mock工具。

### Instrumentation测试代码的位置
如前面所述，Instrumentation测试代码应该放置在app/src/androidTest/java目录下面。

### 在Gradle中配置依赖
和本地单元测试一样，使用Instrumentation测试也必须添加一些依赖，同时还必须添加默认的Android Instrumentation测试执行器配置:

```java
defaultConfig {
       ..... more stuff
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }

dependencies {
    // Unit testing dependencies
    androidTestCompile 'junit:junit:4.12'
    // Set this dependency if you want to use the Hamcrest matcher library
    androidTestCompile 'org.hamcrest:hamcrest-library:1.3'
    // more stuff, e.g., Mockito
} 
```

### 相关的类
先来看Context测试类的重要基础类——AndroidTestCase。这个类最终的功能就是提供了getContext()功能，在实际测试中，它的返回值是可以当做测试App对象的Context使用的。这个类的子类如下:

1. ApplicationTestCase;
2. ProviderTestCase;
3. ServiceTestCase
4. CustomTabsIntentCase;
5. LoaderTestCase;

很容易发现，四大组件有两个组件在这里有对应的测试Case类，那么最重要的Activity呢？实际上确实有ActivityTestCase类，但这个类不属于AndroidTestCase继承树，它的父类是InstrumentationTestCase，直接子类是:

1. ActivityInstrumentationTestCase
2. ActivityInstrumentationTestCase2
3. ActivityUnitTestCase

其中第一个类已经被废弃，现在使用的都是ActivityInstrumentationTestCase2这个类。后面两个类就是[官网教程](http://developer.android.com/intl/zh-cn/training/activity-testing/preparing-activity-testing.html)的讲述重点。

以上所有的类都是junit.framework.TestCase的子类，继承关系如下：

![Android单元测试配置](http://7xktd8.com1.z0.glb.clouddn.com/单测类继承关系.png)

#### ActivityUnitTestCase和ActivityInstrumentationTestCase2的区别
两者都能进行简单的UI测试，比如UI元素的布局、显示内容和点击动作，比如：

```java
@MediumTest
public void testClickMeButton_clickButtonAndExpectInfoText() {
    String expectedInfoText = mClickFunActivity.getString(R.string.info_text);
    TouchUtils.clickView(this, mClickMeButton);
    assertTrue(View.VISIBLE == mInfoTextView.getVisibility());
    assertEquals(expectedInfoText, mInfoTextView.getText());
}
```

但是两者的侧重点不一样。

ActivityUnitTestCase创建的Activity会尽量少的和系统有联系，所有的依赖都可以通过Mock或者别的方式注入进去，你的测试对象Activity会在真实的系统上运行，并且不会和别的Activity产生交互。以下方法都不应该被调用，大部分都会抛出异常:

1. createPendingResult(int, Intent, int)
2. startActivityIfNeeded(Intent, int)
3. startActivityFromChild(Activity, Intent, int)
4. startNextMatchingActivity(Intent)
5. getCallingActivity()
6. getCallingPackage()
7. createPendingResult(int, Intent, int)
8. getTaskId()
9. isTaskRoot()
10. moveTaskToBack(boolean)

这些方法都是和环境进行交互的，需要完整的上下文。而

1. startActivity
2. startActivityForResult

这两个调用是没有效果的，可以使用getStartedActivityIntent()和getStartedActivityRequest()来获取调用参数。

1. finish
2. finishActivity
3. finishFromChild

这些调用也不会有任何的效果，同样有方法isFinishCalled()和getFinishedActivityRequest()来获取调用参数。

通过以上方式，一个ActivityUnitTestCase可以测试一些和其余组件的“Mock交互”。

但如果你需要进行功能测试，则建议使用ActivityInstrumentationTestCase2。ActivityInstrumentationTestCase2也是建立在真实的系统基础上的，调用的是InstrumentationTestCase.launchActivity()方法，你可以直接操纵Activity。单元测试一般不太适合用来测试复杂的UI交互动作，而ActivityInstrumentationTestCase2则非常适合，比如调用键盘向EditText中输入文字，真实的发起一个Activity并检测数据的传输。具体可以见官方案例：[Creating Functional Tests](http://developer.android.com/intl/zh-cn/training/activity-testing/activity-functional-testing.html)。大名鼎鼎的robotium就是基于这个类来实现的。

#### 工具类
1. MoreAsserts类包含更多强大的断言方法，如assertContainsRegex(String, String)，可以作正则表达式的匹配。
2. ViewAsserts类包含关于Android View的有用断言方法，如assertHasScreenCoordinates(View, View, int, int)，可以测试View在可视区域的特定X、Y位置。这些Assert简化了UI中几何图形和对齐方式的测试。

### 运行Instrumentation测试
执行命令 __gradlew build connectedCheck__  就可以了。

除了命令行的方式，还有一种就是直接从Android Studio执行。回到最上面那张图，在4部分，即Test Artifact里面，还有一个选择是"Android Instrumentation Tests"，切换到这个模式，然后选中需要执行的类，选择Run即可。

## 总结
这篇文章主要目的是为了让读者初步了解Android单元测试的分类以及基本的测试知识。

从实际使用效果来看，本地普通单元测试和Instrumentation单元测试互相补足，基本满足了与四大组件无关的测试，但是仍然有很多的情况不能Cover:

1. 本地普通单元测试只能Cover与平台无关的代码；
2. Instrumentation测试虽然提供了Context，但是测试结果很多依赖于实际的运行环境，并且执行结果是反映在UI上的，因此效果有限；

举几个🌰:

```java
//与实际编译环境相关
public static int getSdkVersion() {
	try {
		return Build.VERSION.class.getField("SDK_INT").getInt(null);
	} catch (Exception e) {
		return 3;
	}
}

//与App本身相关
public static String getAppName(Context context) {
	return getAppName(context, null);
}

//与设备相关
public static boolean hasFeature(Context context, String feature) {
	return context.getPackageManager().hasSystemFeature(feature);
}

//结果反映在UI上
public static void hideSoftKeyboard(@NonNull View view) {
	InputMethodManager imm = (InputMethodManager) view.getContext().getSystemService(Context.INPUT_METHOD_SERVICE);
	imm.hideSoftInputFromWindow(view.getWindowToken(), 0);
}
```
以上这些例子在客户端很难完成UT，因此它们是不应该使用UT的，必要性也不是很大。

综上，UT在客户端的使用范围和效果是很有限的。客户端开发人员应当熟知这几类测试手段（另外一种是UI自动化测试），在必要的时候根据需求，比如必不可免的复杂逻辑处，使用相应的测试，一定程度上保证客户端的稳定性。

后期待补充的相关知识：

1. Mockito的使用；
2. Android Testing Support Library提供的一些新功能；
3. Android上对Activity、Service、ContentProvider、Application等组件的测试支持；
4. UI自动化测试框架；