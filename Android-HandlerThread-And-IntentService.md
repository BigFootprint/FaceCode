---
title: HandlerThread & IntentService源码分析
date: 2016-02-26 19:04:33
tags: [源码]
categories: Android
---

## 前提
【环境】代码分析基于 API-23。

## 介绍和使用
本文将介绍两个类：HandlerThread和IntentService，后者使用到了前者，两个类都短小精悍，放在一起分析很nice。

在学习Service的时候，我们一定会知道IntentService:，官方文档不止一次强调: Service本身是运行在主线程中的，而主线程中是不适合进行耗时任务的，因而官方文档叮嘱我们一定要在Service中另开线程进行耗时任务处理。IntentService正是为这个目的而诞生的一个优雅设计，让程序员不用再管理线程的开启和运行。

两个类具体的使用例子见下面的分析。<!--more-->

## HandlerThread源码剖析
HandlerThread是一个很简单的类，在[Handler一文](http://www.muzileecoding.com/androidsource/Android-Handler.html)中，我们描述了创建一个带Looper(消息循环机制)的线程的做法:

```java
class LooperThread extends Thread {
     public Handler mHandler;

     public void run() {
          Looper.prepare();

          mHandler = new Handler() {
             public void handleMessage(Message msg) {
                 // process incoming messages here
              }
         };
         Looper.loop();
     }
}
```
这是一个模板，而且对于Looper.prepare()，Looper.loop()以及Handler的创建顺序是有相应要求的，HandlerThread是对这个创建过程的一个封装——直接提供一个带有Looper机制的Thread。

HandlerThread的完整源码只有149行，删除注释和License之后，只有71行，如下:

```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }

	 //回调方法
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}

```
HandlerThread是Thread的一个子类，之前的创建过程都被放到了run()方法中，看过[Handler一文](http://www.muzileecoding.com/androidsource/Android-Handler.html)的读者应该对此非常熟悉。run()方法中有几行代码比较奇特:

```java
synchronized (this) {
	mLooper = Looper.myLooper();
	notifyAll();
}
```
这里为什么要加上__synchronized__关键字呢？答案在于getLooper()方法。

getLooper()方法是将HandlerThread的Looper暴露到外面以便于其余的线程建立这个Thread的Handler与之交互。如果其余线程在HandlerThread启动之前(即run()方法执行之前，即Looper.prepare()执行之前)调用getLooper()方法，那么这个时候HandlerThread是没有可用的Looper的，所以在getLooper()方法中，这样的外部线程就会被wait()。

而等到run()方法运行到Looper.prepare()之后，Looper已经完备，此时应该通知所有的wait线程获取Looper——这里是需要做同步的。

解释到此，对于Handler熟悉的读者一定知道HandlerThread是如何使用的，如果不知道，下面的IntentService提供了完美的例子。

>线程，是可以设置优先级的。

## IntentService源码剖析
IntentService也很短小，只有164行，真是的代码则只有70行不到:

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
    
    @WorkerThread
    protected abstract void onHandleIntent(Intent intent);
}
```
大致看上去是这样的: IntentService在内部建立了一个HandlerThread，每一个被发送到Service的Handler都被抛到Handlerthread中执行，回调方法: onHandlerIntent()。下面我们来仔细分析一下。

首先贴一下类的注释:

```java
/**
 * IntentService is a base class for {@link Service}s that handle asynchronous
 * requests (expressed as {@link Intent}s) on demand.  Clients send requests
 * through {@link android.content.Context#startService(Intent)} calls; the
 * service is started as needed, handles each Intent in turn using a worker
 * thread, and stops itself when it runs out of work.
 *
 * <p>This "work queue processor" pattern is commonly used to offload tasks
 * from an application's main thread.  The IntentService class exists to
 * simplify this pattern and take care of the mechanics.  To use it, extend
 * IntentService and implement {@link #onHandleIntent(Intent)}.  IntentService
 * will receive the Intents, launch a worker thread, and stop the service as
 * appropriate.
 *
 * <p>All requests are handled on a single worker thread -- they may take as
 * long as necessary (and will not block the application's main loop), but
 * only one request will be processed at a time.
 *
 * <div class="special reference">
 * <h3>Developer Guides</h3>
 * <p>For a detailed discussion about how to create services, read the
 * <a href="{@docRoot}guide/topics/fundamentals/services.html">Services</a> developer guide.</p>
 * </div>
 *
 * @see android.os.AsyncTask
 */
```
注释里面有几个很重要的点:

1. Client是通过startService(Intent)来向Service发送请求的，请求被包装在Intent里面；
2. 在有任务的时候，Service会自己启动，没有任务的时候自己Stop；
3. 这个类实现了所谓的"work queue processor"模式，类似于主线程处理消息的机制，也就是串行的执行队列任务；
4. 使用IntentService只需要继承该类，并实现`onHandleIntent(Intent intent)`方法即可；

接下来我们关注几个重要的方法:

1. __onStartCommand():__ 这个方法每次在startService()方法调用的时候都会被执行，它可以根据一个boolean值决定返回值（这里可以去查看一下Service该方法返回值的含义，它决定了Service被杀死之后如何复苏），这里我们可以通过setIntentRedelivery()方法控制返回值；
2. __onStart():__ 这个方法在包装消息的时候是把startId包装到了消息里面。这个startId很有用，影响到了前面说的第2点（自动启动、停止）的实现；

关于startId的用处:

```java
However, if your service handles multiple requests to onStartCommand() 
concurrently, then you shouldn't stop the service when you're done 
processing a start request, because you might have since received a new 
start request (stopping at the end of the first request would terminate the 
second one). To avoid this problem, you can use stopSelf(int) to ensure 
that your request to stop the service is always based on the most recent 
start request. That is, when you call stopSelf(int), you pass the ID of the 
start request (the startId delivered to onStartCommand()) to which your 
stop request corresponds. Then if the service received a new start request 
before you were able to call stopSelf(int), then the ID will not match and 
the service will not stop.
```
## 用法
1. The IntentService internally handles the two start types `START_NOT_STICKY` and `START_REDELIVER_INTENT`. The first is default, so the latter needs to be set with `setIntentRedelivery(true)`.
2. There is no need to stop the IntentService with `stopSelf()`, because that is done internally.

>BroadcastReceiver中有一个`BroadcastReceiver.goAsync()`方法，用于维持BroadcastReceiver的生命周期。

## 总结
Android上为了将耗时操作剥离出主线程做了很多的工作，为此也为线程间调度做了很多的努力。

所有的线程消息调度都是基于[Handler机制](http://www.muzileecoding.com/androidsource/Android-Handler.html)，Android还基于Handler做了更多高级的封装，最经典的就是 __AsyncTask__，这些类都有相似的特点：短小精悍，便于开发。同时也是非常不错的学习材料。

IntentService专门用于处理串行化处理任务，一旦复杂性超过这种需求，IntentService就不大适用了，比如并发处理任务。
