---
title: Handler 源码分析
date: 2015-11-23 11:03:34
tags: [源码]
categories: Android
---

## 前言
【环境】源码分析基于 4.4.2_r1。

## Handler的使用
日常开发中我们经常用到Handler，Handler用于向一个线程的消息队列中发送消息并负责处理。我们可以用它实现定时任务调度、将耗时任务放到异步线程执行并在执行完毕后通知主线程更新UI等功能。下面是一个小🌰：

```java
//创建Handler
Handler myHandler = new Handler() { 
	public void handleMessage(Message msg) {   
		switch (msg.what) {  
		}   
		super.handleMessage(msg);   
	}   
};

//通过Handler发送消息
Message message = new Message();      
message.what = 1;      
myHandler.sendMessage(message); 
```
这是一个使用Handler的经典过程：发出的消息会在时间到达的时候进入到Handler的handleMessage中被处理。下面我们就来探究一下Message从被创建到被处理到底经历了什么样的过程。<!--more-->

## 源码剖析
如🌰所示，我们从new Handler()切入，然后追踪sendMessage方法，查看Message对象如何进入handleMessage方法，一步步剖析，解析整个流程。

### Handler片段——Handler创建
先看一下Handler的创建过程:

```java
public Handler() {
	this(null, false);
}

public Handler(Callback callback, boolean async) {
	if (FIND_POTENTIAL_LEAKS) {
		final Class<? extends Handler> klass = getClass();
		if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
				(klass.getModifiers() & Modifier.STATIC) == 0) {
			Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
		}
	}

	mLooper = Looper.myLooper();
	if (mLooper == null) {
		throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
	}
	mQueue = mLooper.mQueue;//获取消息队列
	mCallback = callback;
	mAsynchronous = async;
}
```
这里重点是 __mLooper = Looper.myLooper();__ 以及后面三行代码。我们看到，新建一个Handler的时候，如果Looper.myLooper为null，则会抛出异常；如果不为null，则获取Looper中的mQueue。其余两个参数不是重点，暂不分析。

### Looper片段——Looper创建
Android中提供了相关步骤，可以使得一个普通Thread也具备消息循环处理机制（因为主线程默认做了一些操作，因此不好分析，我们从普通线程入手，可以更清楚的看到整个创建过程）。示例代码可以在Looper源码(类注释)中找到:

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

这里涉及到另外一个重要的对象：__Looper__。首先来看一下 __Looper.prepare();__ 方法，源码如下:

```java
/** Initialize the current thread as a looper.
 * This gives you a chance to create handlers that then reference
 * this looper, before actually starting the loop. Be sure to call
 * {@link #loop()} after calling this method, and end it by calling
 * {@link #quit()}.
 */
public static void prepare() {
	prepare(true);
}

private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
	sThreadLocal.set(new Looper(quitAllowed));
}
```
我们关注一下异常，异常消息的含义是：每一个线程只能有一个Looper。而判断的条件是，从sThreadLocal中获取的对象不为null——[ThreadLocal](http://www.muzileecoding.com/java/Java-threadlocal.html)控制每个线程只有一个Looper实例。
如果之前没有创建Looper，则会实例化一个Looper，看一下构造方法:

```java
//注意：这个方法是私有的
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
构造函数分为两步：1）创建了一个MessageQueue；2）获取当前的线程。注意mQueue这个变量，就是上一节中Handler从Looper中获取的消息队列对象。

到这里，我们看到了Handler、Looper和MessageQueue三个对象的创建过程。我们知道以下几个重要的点:

1. new Handler()之前，必须保证Looper.myLooper()方法的返回不为null；
2. Looper在新建的时候会建立对应的MessageQueue，并且prepare()方法会通过ThreadLocal保证每一个线程只会实例化一个Looper。
3. Handler既会持有Looper的引用，又会持有MessageQueue的引用；

但是这几个类之间的关系我们却不是很清楚，下面解释。

### 线程、Handler、Looper的联系
在消息机制里面，Looper只是负责管理消息队列，也就是取出消息进行处理，而Handler则是负责发送消息以及处理消息的，那么Handler和Looper又是如何绑定到一起的呢？从Handler的源码看，关键代码是:  __mLooper = Looper.myLooper();__ ，源码如下:

```java
/**
 * null if the calling thread is not associated with a Looper.
 */
public static Looper myLooper() {
	return sThreadLocal.get();
}
```
这里把注释也Copy过来了，myLooper方法是直接从sThreadLocal中读取的变量，注释说到：myLooper()方法在当前没有Looper绑定到调用线程时会返回null。搜索整个Looper类，会发现只有前面提到的prepare()方法中有sThreadLocal.set()调用。即在为一个线程实例化Handler之前，必须在该线程中调用Looper.prepare()方法：只有这样，才能为一个线程建立对应的Looper——这解释了前面LooperThread代码的构建逻辑。

到这里我们知道，通过调用Looper.prepare()方法一次，就可以将一个Looper对象绑定到方法调用线程上去(即设置到ThreadLocal中)。新建Handler的时候，Handler会直接调用Looper.myLoop()方法获取绑定到线程的Looper并通过这个Looper获取到对应的MessageQueue。

这就是这几个概念之间的关系。

### Looper片段——Looper.loop()
LooperThread构建的最后一步就是调用Loop.loop()方法，它的源码如下:

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
	final Looper me = myLooper();
	if (me == null) {
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
	final MessageQueue queue = me.mQueue;

	// Make sure the identity of this thread is that of the local process,
	// and keep track of what that identity token actually is.
	Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();

	for (;;) {
		Message msg = queue.next(); // might block
		if (msg == null) {
			// No message indicates that the message queue is quitting.
			return;
		}

		// This must be in a local variable, in case a UI event sets the logger
		Printer logging = me.mLogging;
			if (logging != null) {
  				logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
			}

			msg.target.dispatchMessage(msg);

			if (logging != null) {
				logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
			}

			// Make sure that during the course of dispatching the
			// identity of the thread wasn't corrupted.
			final long newIdent = Binder.clearCallingIdentity();
			if (ident != newIdent) {
				Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
			}

			msg.recycle();//回收消息，注意这个位置
	}
}
```
首先，同样要求myLooper不能为null（根据前面的分析，实现调用Looper.prepare（)方法即可）并从这个Looper中拿到相应的消息队列。之后我们进入一个无限循环，这个循环中最重要的事情就是queue.next()方法的调用，这个方法根据注释：可能会引起阻塞。相关的实现可以深入到MessageQueue中，这里涉及到Native代码，具体的理解不影响到原理讲述，暂且不表。

这里有一个很重要的判断：如果Message拿出来为空，则认为这个MessageQueue已经被丢弃了，整个循环会被打破返回，读者可以注意一下。

再之后就进入到msg.target.dispatchMessage方法。即消息的分发和处理。

### 关系总结
这里总结一下: 

1. 我们在让一个线程具备消息循环调度能力的时候，首先需要在这个线程中调用Looper.prepare()，这个方法很重要，它会为一个线程绑定一个Looper，且只能绑定一个。创建Looper的同时，Looper会创建一个MessageQueue。之后在线程中创建Handler的时候，会从Looper中引用MessageQueue放到Handler实例中去。至此Handler、Looper、MessageQueue三个类的关系建立。
2. 关系建立之后，调用Looper.loop()方法会让Loop运行起来：不断从MessageQueue中获取handler通过sendMessage发送的Message并dispatchMessage。至此，整个消息循环的核心部分构建完毕。

### 消息的分发和处理
整个消息循环核心部分已经运行起来，剩下来的就是我们日常最长使用到的消息的发送和处理，接下来看看消息从发送到处理之间的流程。首先是sendMessage方法源码:

```java
/**
 * Enqueue a message into the message queue after all pending messages
 * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
 * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
 * You will receive it in {@link #handleMessage}, in the thread attached
 * to this handler.
 * 
 * @param uptimeMillis The absolute time at which the message should be
 *         delivered, using the
 *         {@link android.os.SystemClock#uptimeMillis} time-base.
 *         
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
		if (queue == null) {
			RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
			Log.w("Looper", e.getMessage(), e);
			return false;
		}
	return enqueueMessage(queue, msg, uptimeMillis);
}
```
看源码就会发现，同名&功能相似的有好几个方法，但最终都会调用sendMessageAtTime方法，首先获取mQueue：注意这个方法在Handler新建的时候就获取了。之后调用了一个enqueueMessage方法，源码如下:

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	return queue.enqueueMessage(msg, uptimeMillis);
}
```
首先，将msg.target赋值为本身，之后根据Handler新建时候传入的参数(前面忽略了它)设置msg属性，之后就调queue的enqueueMessage向队列中压入消息——__完成消息的发送__。

这里很重要的一点是__msg.target = this;__。查看Message的源码就可以看到，target是一个Handler变量。而在前面讲述的Looper.loop()方法实现中，取出消息后调用的方法是msg.target.dispatchMessage(msg);。嘿！这难道不是调用的Handler的dispatchMessage方法么？看源码果然发现一枚:

```java
public void dispatchMessage(Message msg) {
	if (msg.callback != null) {
		handleCallback(msg);
	} else {
		if (mCallback != null) {
			if (mCallback.handleMessage(msg)) {
				return;
			}
		}
		handleMessage(msg);
	}
}
```
这里我们可以看到很多我们平时不常用的方法:

【 1 】Message是可以自带一个callback变量的，查看Meesgae源码可知这是一个Runnable变量，即我们可以使用一个Message实现post一个Runable对象的功能，因为handleCallback的实现如下：

```java
private static void handleCallback(Message message) {
	message.callback.run();
}
```
实际上读者可以查看Handler.postDelayed(Runnable)方法，内部正式做了这一层转换。

【 2 】如果实例化Handler的时候设置了mCallback对象（日常开发很少用，但的确存在这种用法），那么所有的小弟都先交给mCallback处理，mCallback是一个CallBack对象，这是一个接口:

```java
public interface Callback {
	public boolean handleMessage(Message msg);
}
```
这边可以通过返回true来拦截handleMessage的执行。

【 3 】以上都绕过了，才轮到handleMessage方法执行——关于这个方法，就不多说了，写Handler的时候肯定会重载它。

到此，消息的处理分析完毕。

## 剖析总结
到这里，对于整个消息处理发送、循环、发送的机制基本解释清楚。剩下一块比较模糊：MessageQueue的分析比较少，原因是这块涉及到一些Native代码，且对理解整个Handler机制的理解影响不大，在这篇文章中暂不分析。

我们用一张图来总结一下几个概念之间的关系:

![Handler消息流转图](http://7xktd8.com1.z0.glb.clouddn.com/Handler原理图.png)

该图重点展示了几个问题：

1. Handler直接向MessageQueue中发送消息；
2. Looper负责获取消息并作分发（但是具体分发到哪个Handler是由Message的target属性决定的）；
3. __一个线程只会有一个Looper和一个MessageQueue，不论该线程有多少个Handler，它们都公用这个Looper和MessageQueue；__

HandlerThread是串行化执行任务，因此在里面执行任务是线程安全的(指的是任务之间，而不是和其余线程之间)，但是有时候也需要注意将耗时任务和非耗时任务区分在不同的队列里面，提高效率。这个类还很适合执行链式任务，比如编译任务。

## 一些问题
### 主线程和主Handler
在所有的线程中，主线程是非常特殊的，开发时在主线程中新建Handler的实例不需要走上面的流程，直接创建即可，官方文档给出的解释是，主线程本身就已经启动Looper了。其实在Looper中，是有一个专门的方法做这件事的：

```java
/**
 * Initialize the current thread as a looper, marking it as an
 * application's main looper. The main looper for your application
 * is created by the Android environment, so you should never need
 * to call this function yourself.  See also: {@link #prepare()}
 */
public static void prepareMainLooper() {
	prepare(false);
	synchronized (Looper.class) {
		if (sMainLooper != null) {
			throw new IllegalStateException("The main Looper has already been prepared.");
		}
		sMainLooper = myLooper();
	}
}
```
注意看注释！！！这个方法就是创建主线程的Looper的。注释中说，这个创建是由Android Environment执行的，所以开发者__从来不需要__手动调用这个方法。而下面这个方法则是用于获取主Looper的：

```java
/** 
 * Returns the application's main looper, which lives in the main thread of the application.
 */
public static Looper getMainLooper() {
	synchronized (Looper.class) {
		return sMainLooper;
	}
}
```

### 消息的recycle
Message内部维护着一个链表，所有被回收的Message都挂在这个链表上。关键的方法如下:

```java
private static final Object sPoolSync = new Object();
private static Message sPool;
private static int sPoolSize = 0;
private static final int MAX_POOL_SIZE = 50;

/**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
public static Message obtain() {
	synchronized (sPoolSync) {
		if (sPool != null) {
			Message m = sPool;
			sPool = m.next;
			m.next = null;
			m.flags = 0; // clear in-use flag
			sPoolSize--;
			return m;
		}
	}
	return new Message();
}

public void recycle() {
	if (isInUse()) {
		if (gCheckRecycle) {
			throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
		}
		return;
	}
	recycleUnchecked();
}

void recycleUnchecked() {
	// Mark the message as in use while it remains in the recycled object pool.
	// Clear out all other details.
	flags = FLAG_IN_USE;
	what = 0;
	arg1 = 0;
	arg2 = 0;
	obj = null;
	replyTo = null;
	sendingUid = -1;
	when = 0;
	target = null;
	callback = null;
	data = null;

	synchronized (sPoolSync) {
		if (sPoolSize < MAX_POOL_SIZE) {
			next = sPool;
			sPool = this;
			sPoolSize++;
		}
	}
}

public static void updateCheckRecycle(int targetSdkVersion) {
	if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
		gCheckRecycle = false;
	}
}
```
可以看到消息机制下面其实维护着一个消息链表，这个链表充当着对象池的角色，因此建议大家使用obtain()来“创建”消息，这样可以避免创建大量的Message对象。补充一点：消息会在被压入队列时置为FLAG_IN_USE。  

>【注意】在5.0以下的时候，gCheckRecycle开关是关闭的，这意味着回收时不会去检查Message是否在使用中。而在5.0以上就会检查，此处很坑爹的是：FLAG_IN_USE标志位的重置是在obtain的时候清除重置的，在进入消息队列则会被置位。从前面Looper.loop()方法可以看到，一个消息会在dispatchMessage之后立刻调用recycle方法，此时消息的FLAG_IN_USE还没有被重置，recycle必然导致异常抛出——这应该是SDK的一个Bug。

## 补充
1. 以下方法被从MessageQueue暴露到Handler里面:`hasMeesgae()`，这个方法有好几个重载方法，可以关注一下，用于判断队列中有没有特定的消息;
2. Handler还有一个这样的方法: `Handler.postAtFrontOfQueue()`，可以让消息插队;
