---
title: Android卡顿监控方案以及局限性
date: 2016-12-08 10:40:56
tags: [Framework]
categories: Android
---

## 方案介绍

最早在 QQ 空间团队的微信公众号上看到一篇文章：[Android卡慢监控组件简介](http://mp.weixin.qq.com/s/40br55yHjABd5RALf5BO0g)，介绍了一种监控Android应用卡顿的思路。原理非常简单：在`Looper.loop()`方法内，有一段这样的代码：

```java
// This must be in a local variable, in case a UI event sets the logger
if (logging != null) {
	logging.println(">>>>> Dispatching to " + msg.target + " " +
		msg.callback + ": " + msg.what);
}

msg.target.dispatchMessage(msg);

if (logging != null) {
	logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
}
```

这段代码就是消息循环机制中处理消息的核心代码，详细可以参考：[Handler源码分析](http://muzileecoding.com/2015/11/23/Android-Handler/)。<!-- more -->在处理消息前后，都调用了logging对象打印消息日志，并且Looper中的消息对象是可以设置的，因此思路就是我们自己开发一个logging对象，通过监控这两段日志打印来统计`dispatchMessage`方法的执行时间。主线程上如果能保证每一个消息的处理时间在16ms以内，就可以保证不卡顿。

> 实际上应该设置的阈值应当远大于16ms，因为诸如Activity创建这样的动作，时间会比较长，而且这种情况下只要不出现黑屏之类的问题，用户不会觉得卡顿。

后来发现已经有相关的实现库了，这就是[BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor)。它的核心代码如下：

```java
@Override
public void println(String x) {
	if (!mPrintingStarted) {
		mStartTimestamp = System.currentTimeMillis();
		mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
		mPrintingStarted = true;
		startDump();
	} else {
		final long endTime = System.currentTimeMillis();
		mPrintingStarted = false;
		if (isBlock(endTime)) {
			notifyBlockEvent(endTime);
		}
		stopDump();
	}
}
```

通过监控两次打印，判断事件是否超出阈值，超出则回调 Block 事件，并打印相关的 CPU 信息、栈信息以辅助解决问题。

以上方案可以说有一点投机取巧，它有局限性的。

## Logger 变化怎么办？

如果某天 Android 系统不再调用这段 Logger，或者在别的地方也打印 Logger，很容易看到 BlockCanary 的方案就会失效。除此之外，读者可能会忽略前面展示的 Looper 的那段代码的一个小注释：

```java
// This must be in a local variable, in case a UI event sets the logger
```

UI 时间可能也会设置这个 logger，因此依赖 Logger 并不靠谱， [SafeLooper](https://github.com/mmin18/SafeLooper) 提供了另外一种思路。

SafeLooper 通过在主线程队列中塞入一个可以托管主线程后续消息的阻塞消息，可以在应用内部执行：

`h.dispatchMessage(msg);`

这段代码，这样就可以完全不依赖 Logger 来统计时间。

## 如果卡顿时间非常非常长怎么办？

BlockCanary 是需要依赖两次 Logger 的调用才能完成监控日志的输出的，那如果 `dispatchMessage`方法永远不返回，比如黑屏了怎么监控呢？

这需要另外开启一个线程，在固定时间之后完成监控，简单实现如下：

```java
@Override
public void println(String x) {
    if (!mPrintingStarted) {
        mStartTimestamp = System.currentTimeMillis();
        mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
        mPrintingStarted = true;
        startDump();
        monitorHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                long endTime = System.currentTimeMillis();
                notifyBlockEvent(endTime);
            }
        }, mBlockThresholdMillis * 3);
    } else {
        monitorHandler.removeCallbacksAndMessages(null);
        final long endTime = System.currentTimeMillis();
        mPrintingStarted = false;
        if (isBlock(endTime)) {
            notifyBlockEvent(endTime);
        }
        stopDump();
    }
}
```

这样即使完全卡死，也可以获取监控信息。

## Application如何监控

> 这个问题和前面一个问题可能最初不在这个方案的设计者的目标之内，但如果能进一步扩充，整个方案会更加健全。

在实际实验过程中，我们发现，Application 的`onCreate()`方法是无法监控的，不论是在`onCreate()`的第一行，还是在 Application 的构造函数中调用都做不到。为什么呢？打印一下这两个方法的调用堆栈信息：

【构造函数】

```xml
12-07 20:51:41.547 I/System.out: *******************************************
12-07 20:51:41.547 I/System.out: stacktrace len:16
12-07 20:51:41.547 I/System.out: ----  the 0 element  ----
12-07 20:51:41.547 I/System.out: toString: dalvik.system.VMStack.getThreadStackTrace(Native Method)
12-07 20:51:41.547 I/System.out: ClassName: dalvik.system.VMStack
12-07 20:51:41.547 I/System.out: FileName: VMStack.java
12-07 20:51:41.547 I/System.out: LineNumber: -2
12-07 20:51:41.547 I/System.out: MethodName: getThreadStackTrace
12-07 20:51:41.547 I/System.out: ----  the 1 element  ----
12-07 20:51:41.547 I/System.out: toString: java.lang.Thread.getStackTrace(Thread.java:580)
12-07 20:51:41.547 I/System.out: ClassName: java.lang.Thread
12-07 20:51:41.547 I/System.out: FileName: Thread.java
12-07 20:51:41.547 I/System.out: LineNumber: 580
12-07 20:51:41.547 I/System.out: MethodName: getStackTrace
12-07 20:51:41.547 I/System.out: ----  the 2 element  ----
12-07 20:51:41.547 I/System.out: toString: com.example.blockcanary.DemoApplication.<init>(DemoApplication.java:31)
12-07 20:51:41.547 I/System.out: ClassName: com.example.blockcanary.DemoApplication
12-07 20:51:41.547 I/System.out: FileName: DemoApplication.java
12-07 20:51:41.547 I/System.out: LineNumber: 31
12-07 20:51:41.547 I/System.out: MethodName: <init>
12-07 20:51:41.547 I/System.out: ----  the 3 element  ----
12-07 20:51:41.547 I/System.out: toString: java.lang.Class.newInstance(Native Method)
12-07 20:51:41.547 I/System.out: ClassName: java.lang.Class
12-07 20:51:41.547 I/System.out: FileName: Class.java
12-07 20:51:41.547 I/System.out: LineNumber: -2
12-07 20:51:41.547 I/System.out: MethodName: newInstance
12-07 20:51:41.547 I/System.out: ----  the 4 element  ----
12-07 20:51:41.547 I/System.out: toString: android.app.Instrumentation.newApplication(Instrumentation.java:996)
12-07 20:51:41.547 I/System.out: ClassName: android.app.Instrumentation
12-07 20:51:41.547 I/System.out: FileName: Instrumentation.java
12-07 20:51:41.547 I/System.out: LineNumber: 996
12-07 20:51:41.547 I/System.out: MethodName: newApplication
12-07 20:51:41.547 I/System.out: ----  the 5 element  ----
12-07 20:51:41.547 I/System.out: toString: android.app.Instrumentation.newApplication(Instrumentation.java:981)
12-07 20:51:41.547 I/System.out: ClassName: android.app.Instrumentation
12-07 20:51:41.547 I/System.out: FileName: Instrumentation.java
12-07 20:51:41.547 I/System.out: LineNumber: 981
12-07 20:51:41.547 I/System.out: MethodName: newApplication
12-07 20:51:41.547 I/System.out: ----  the 6 element  ----
12-07 20:51:41.547 I/System.out: toString: android.app.LoadedApk.makeApplication(LoadedApk.java:573)
12-07 20:51:41.547 I/System.out: ClassName: android.app.LoadedApk
12-07 20:51:41.547 I/System.out: FileName: LoadedApk.java
12-07 20:51:41.547 I/System.out: LineNumber: 573
12-07 20:51:41.547 I/System.out: MethodName: makeApplication
12-07 20:51:41.548 I/System.out: ----  the 7 element  ----
12-07 20:51:41.548 I/System.out: toString: android.app.ActivityThread.handleBindApplication(ActivityThread.java:4680)
12-07 20:51:41.548 I/System.out: ClassName: android.app.ActivityThread
12-07 20:51:41.548 I/System.out: FileName: ActivityThread.java
12-07 20:51:41.548 I/System.out: LineNumber: 4680
12-07 20:51:41.548 I/System.out: MethodName: handleBindApplication
12-07 20:51:41.548 I/System.out: ----  the 8 element  ----
12-07 20:51:41.548 I/System.out: toString: android.app.ActivityThread.-wrap1(ActivityThread.java)
12-07 20:51:41.548 I/System.out: ClassName: android.app.ActivityThread
12-07 20:51:41.548 I/System.out: FileName: ActivityThread.java
12-07 20:51:41.548 I/System.out: LineNumber: -1
12-07 20:51:41.548 I/System.out: MethodName: -wrap1
12-07 20:51:41.548 I/System.out: ----  the 9 element  ----
12-07 20:51:41.548 I/System.out: toString: android.app.ActivityThread$H.handleMessage(ActivityThread.java:1405)
12-07 20:51:41.548 I/System.out: ClassName: android.app.ActivityThread$H
12-07 20:51:41.548 I/System.out: FileName: ActivityThread.java
12-07 20:51:41.548 I/System.out: LineNumber: 1405
12-07 20:51:41.548 I/System.out: MethodName: handleMessage
12-07 20:51:41.548 I/System.out: ----  the 10 element  ----
12-07 20:51:41.548 I/System.out: toString: android.os.Handler.dispatchMessage(Handler.java:102)
12-07 20:51:41.548 I/System.out: ClassName: android.os.Handler
12-07 20:51:41.548 I/System.out: FileName: Handler.java
12-07 20:51:41.548 I/System.out: LineNumber: 102
12-07 20:51:41.548 I/System.out: MethodName: dispatchMessage
12-07 20:51:41.548 I/System.out: ----  the 11 element  ----
12-07 20:51:41.548 I/System.out: toString: android.os.Looper.loop(Looper.java:148)
12-07 20:51:41.548 I/System.out: ClassName: android.os.Looper
12-07 20:51:41.548 I/System.out: FileName: Looper.java
12-07 20:51:41.548 I/System.out: LineNumber: 148
12-07 20:51:41.548 I/System.out: MethodName: loop
12-07 20:51:41.548 I/System.out: ----  the 12 element  ----
12-07 20:51:41.548 I/System.out: toString: android.app.ActivityThread.main(ActivityThread.java:5417)
12-07 20:51:41.548 I/System.out: ClassName: android.app.ActivityThread
12-07 20:51:41.548 I/System.out: FileName: ActivityThread.java
12-07 20:51:41.548 I/System.out: LineNumber: 5417
12-07 20:51:41.548 I/System.out: MethodName: main
12-07 20:51:41.548 I/System.out: ----  the 13 element  ----
12-07 20:51:41.548 I/System.out: toString: java.lang.reflect.Method.invoke(Native Method)
12-07 20:51:41.548 I/System.out: ClassName: java.lang.reflect.Method
12-07 20:51:41.548 I/System.out: FileName: Method.java
12-07 20:51:41.548 I/System.out: LineNumber: -2
12-07 20:51:41.548 I/System.out: MethodName: invoke
12-07 20:51:41.548 I/System.out: ----  the 14 element  ----
12-07 20:51:41.548 I/System.out: toString: com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)
12-07 20:51:41.549 I/System.out: ClassName: com.android.internal.os.ZygoteInit$MethodAndArgsCaller
12-07 20:51:41.549 I/System.out: FileName: ZygoteInit.java
12-07 20:51:41.549 I/System.out: LineNumber: 726
12-07 20:51:41.549 I/System.out: MethodName: run
12-07 20:51:41.549 I/System.out: ----  the 15 element  ----
12-07 20:51:41.549 I/System.out: toString: com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)
12-07 20:51:41.549 I/System.out: ClassName: com.android.internal.os.ZygoteInit
12-07 20:51:41.549 I/System.out: FileName: ZygoteInit.java
12-07 20:51:41.549 I/System.out: LineNumber: 616
12-07 20:51:41.549 I/System.out: MethodName: main
```

【`onCreate()`方法】

```XML
12-07 20:36:45.733 I/System.out: *******************************************
12-07 20:36:45.733 I/System.out: stacktrace len:13
12-07 20:36:45.733 I/System.out: ----  the 0 element  ----
12-07 20:36:45.733 I/System.out: toString: dalvik.system.VMStack.getThreadStackTrace(Native Method)
12-07 20:36:45.733 I/System.out: ClassName: dalvik.system.VMStack
12-07 20:36:45.733 I/System.out: FileName: VMStack.java
12-07 20:36:45.734 I/System.out: LineNumber: -2
12-07 20:36:45.734 I/System.out: MethodName: getThreadStackTrace
12-07 20:36:45.734 I/System.out: ----  the 1 element  ----
12-07 20:36:45.734 I/System.out: toString: java.lang.Thread.getStackTrace(Thread.java:580)
12-07 20:36:45.734 I/System.out: ClassName: java.lang.Thread
12-07 20:36:45.734 I/System.out: FileName: Thread.java
12-07 20:36:45.734 I/System.out: LineNumber: 580
12-07 20:36:45.734 I/System.out: MethodName: getStackTrace
12-07 20:36:45.734 I/System.out: ----  the 2 element  ----
12-07 20:36:45.734 I/System.out: toString: com.example.blockcanary.DemoApplication.onCreate(DemoApplication.java:43)
12-07 20:36:45.734 I/System.out: ClassName: com.example.blockcanary.DemoApplication
12-07 20:36:45.734 I/System.out: FileName: DemoApplication.java
12-07 20:36:45.734 I/System.out: LineNumber: 43
12-07 20:36:45.734 I/System.out: MethodName: onCreate
12-07 20:36:45.734 I/System.out: ----  the 3 element  ----
12-07 20:36:45.734 I/System.out: toString: android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1013)
12-07 20:36:45.734 I/System.out: ClassName: android.app.Instrumentation
12-07 20:36:45.734 I/System.out: FileName: Instrumentation.java
12-07 20:36:45.734 I/System.out: LineNumber: 1013
12-07 20:36:45.734 I/System.out: MethodName: callApplicationOnCreate
12-07 20:36:45.734 I/System.out: ----  the 4 element  ----
12-07 20:36:45.734 I/System.out: toString: android.app.ActivityThread.handleBindApplication(ActivityThread.java:4707)
12-07 20:36:45.734 I/System.out: ClassName: android.app.ActivityThread
12-07 20:36:45.734 I/System.out: FileName: ActivityThread.java
12-07 20:36:45.734 I/System.out: LineNumber: 4707
12-07 20:36:45.734 I/System.out: MethodName: handleBindApplication
12-07 20:36:45.734 I/System.out: ----  the 5 element  ----
12-07 20:36:45.734 I/System.out: toString: android.app.ActivityThread.-wrap1(ActivityThread.java)
12-07 20:36:45.734 I/System.out: ClassName: android.app.ActivityThread
12-07 20:36:45.734 I/System.out: FileName: ActivityThread.java
12-07 20:36:45.734 I/System.out: LineNumber: -1
12-07 20:36:45.734 I/System.out: MethodName: -wrap1
12-07 20:36:45.734 I/System.out: ----  the 6 element  ----
12-07 20:36:45.734 I/System.out: toString: android.app.ActivityThread$H.handleMessage(ActivityThread.java:1405)
12-07 20:36:45.734 I/System.out: ClassName: android.app.ActivityThread$H
12-07 20:36:45.734 I/System.out: FileName: ActivityThread.java
12-07 20:36:45.734 I/System.out: LineNumber: 1405
12-07 20:36:45.734 I/System.out: MethodName: handleMessage
12-07 20:36:45.734 I/System.out: ----  the 7 element  ----
12-07 20:36:45.734 I/System.out: toString: android.os.Handler.dispatchMessage(Handler.java:102)
12-07 20:36:45.734 I/System.out: ClassName: android.os.Handler
12-07 20:36:45.734 I/System.out: FileName: Handler.java
12-07 20:36:45.734 I/System.out: LineNumber: 102
12-07 20:36:45.734 I/System.out: MethodName: dispatchMessage
12-07 20:36:45.734 I/System.out: ----  the 8 element  ----
12-07 20:36:45.734 I/System.out: toString: android.os.Looper.loop(Looper.java:148)
12-07 20:36:45.734 I/System.out: ClassName: android.os.Looper
12-07 20:36:45.734 I/System.out: FileName: Looper.java
12-07 20:36:45.734 I/System.out: LineNumber: 148
12-07 20:36:45.734 I/System.out: MethodName: loop
12-07 20:36:45.734 I/System.out: ----  the 9 element  ----
12-07 20:36:45.734 I/System.out: toString: android.app.ActivityThread.main(ActivityThread.java:5417)
12-07 20:36:45.734 I/System.out: ClassName: android.app.ActivityThread
12-07 20:36:45.734 I/System.out: FileName: ActivityThread.java
12-07 20:36:45.734 I/System.out: LineNumber: 5417
12-07 20:36:45.734 I/System.out: MethodName: main
12-07 20:36:45.734 I/System.out: ----  the 10 element  ----
12-07 20:36:45.734 I/System.out: toString: java.lang.reflect.Method.invoke(Native Method)
12-07 20:36:45.734 I/System.out: ClassName: java.lang.reflect.Method
12-07 20:36:45.734 I/System.out: FileName: Method.java
12-07 20:36:45.734 I/System.out: LineNumber: -2
12-07 20:36:45.734 I/System.out: MethodName: invoke
12-07 20:36:45.734 I/System.out: ----  the 11 element  ----
12-07 20:36:45.734 I/System.out: toString: com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)
12-07 20:36:45.734 I/System.out: ClassName: com.android.internal.os.ZygoteInit$MethodAndArgsCaller
12-07 20:36:45.734 I/System.out: FileName: ZygoteInit.java
12-07 20:36:45.734 I/System.out: LineNumber: 726
12-07 20:36:45.734 I/System.out: MethodName: run
12-07 20:36:45.734 I/System.out: ----  the 12 element  ----
12-07 20:36:45.734 I/System.out: toString: com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)
12-07 20:36:45.734 I/System.out: ClassName: com.android.internal.os.ZygoteInit
12-07 20:36:45.734 I/System.out: FileName: ZygoteInit.java
12-07 20:36:45.734 I/System.out: LineNumber: 616
12-07 20:36:45.734 I/System.out: MethodName: main
```

简单分析可以看到：

1）这两个函数的调用都是由`ActivityThread.main()`函数在5417行发起调用的；

2）最终通过消息分发（注意`handleMessage()`函数的调用）交付给`ActivityThread.handleBindApplication()`执行，在这个方法第 4680 行会调用`LoadedApk.makeApplication()`方法创建 Application 对象，在 4707 行会调用`Instrumentation.callApplicationOnCreate()`方法调用 Application 的`onCreate()`方法；

`ActivityThread.handleBindApplication()`方法的局部实现如下：

```java
try {
    // If the app is being launched for full backup or restore, bring it up in
    // a restricted environment with the base application class.
    // 创建 Application
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;

    // don't bring up providers in restricted mode; they may depend on the
    // app's custom Application class
    if (!data.restrictedBackupMode) {
        List<ProviderInfo> providers = data.providers;
        if (providers != null) {
            installContentProviders(app, providers);
            // For process that contains content providers, we want to
            // ensure that the JIT is enabled "at some point".
            mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
        }
    }

    // Do this after providers, since instrumentation tests generally start their
    // test thread at this point, and we don't want that racing.
    try {
        mInstrumentation.onCreate(data.instrumentationArgs);
    }
    catch (Exception e) {
        throw new RuntimeException(
            "Exception thrown in onCreate() of "
                + data.instrumentationName + ": " + e.toString(), e);
    }

    try {
        // 调用 onCreate 方法
        mInstrumentation.callApplicationOnCreate(app);
    } catch (Exception e) {
        if (!mInstrumentation.onException(app, e)) {
            throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
        }
    }
} finally {
    StrictMode.setThreadPolicy(savedPolicy);
}
```

简而言之：Application 的创建和 `onCreate()`是在一个消息中执行完成的。再看Application 的`onCreate()`方法的注释：

```java
/**
 * Called when the application is starting, before any activity, service,
 * or receiver objects (excluding content providers) have been created.
 * Implementations should be as quick as possible (for example using 
 * lazy initialization of state) since the time spent in this function
 * directly impacts the performance of starting the first activity,
 * service, or receiver in a process.
 * If you override this method, be sure to call super.onCreate().
 */
@CallSuper
public void onCreate() {
}
```

也就是说，这个方法会先于四大组件运行。综上可以得出结论：__在 Application 的构造函数中，开发者可以获得App 发出的第一个主线程消息并执行相关代码__。

根据前面的卡顿原理分析，这种情况下是无法监控到这个消息本身的，也就是说通过 Logger 的打印不能监控到 Application 的`onCreate()`方法。

解决办法也不难，按照 BlockCanary 的实现，我只需要拿到 LooperMonitor 对象，手动插入 Logger 代码即可：

```java
@Override
public void onCreate() {
    super.onCreate();
    monitor.println("Application Create Start");
        
    //Application 初始化代码
    //...
        
    monitor.println("Application Create End");
}
```

这样也能完成`onCreate()`方法的监控。

> 但这种方案实际在 SafeLooper 中比较难操作，要实现和 SafeLooper 的结合需要对项目做比较大的重构。

## 总结

这套方案利用系统暴露的接口和实现，巧妙地实现了监控卡顿的目的。

前面提到，方案的 Block 阈值是比较难设置的，比较理想的状况是把 Application 和四大组件的生命周期方法单独抽取出来做统计，这部分对 Block 的要求远远要比在页面上滚动一个列表页要来的低。

再进一步的，可以构建一个客户端卡顿监控系统，将 BlockCanary 融合 SafeLooper 之后作为监控系统消息的一种手段纳入其中，并提供接口可以让开发者手动监控某段代码的执行时间，从而构建一个比较全面的客户端代码运行监控系统。