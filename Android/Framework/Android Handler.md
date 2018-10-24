---
title: Handler 源码分析
date: 2015-11-23 11:03:34
tags: [源码]
categories: Android
---

## 前言
【环境】源码分析基于 4.4.2_r1。

## Handler 的使用
日常开发中我们经常用到 Handler，Handler 用于向一个线程的消息队列中发送消息并负责处理。我们可以用它实现定时任务调度、将耗时任务放到异步线程执行并在执行完毕后通知主线程更新 UI 等功能。下面是一个小🌰：

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
这是一个使用 Handler 的经典过程：发出的消息会在时间到达的时候进入到 Handler 的 handleMessage 中被处理。下面我们就来探究一下 Message 从被创建到被处理到底经历了什么样的过程。<!--more-->

## 源码剖析
如🌰所示，我们从`new Handler()`切入，然后追踪`sendMessage`方法，查看 Message 对象如何进入 handleMessage 方法，一步步剖析，解析整个流程。

### Handler 片段—— Handler 创建
先看一下 Handler 的创建过程:

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
这里重点是`mLooper = Looper.myLooper();`以及后面三行代码。我们看到，新建一个 Handler 的时候，如果Looper.myLooper 为 null，则会抛出异常；如果不为 null，则获取 Looper 中的 mQueue。其余两个参数不是重点，暂不分析。

> 注意看 FIND_POTENTIAL_LEAKS 这个变量，虽然不是我们分析的重点，但是这里指出了 Handler 非静态内部类的实例会容易导致内存泄漏。

### Looper 片段—— Looper 创建
Android 中提供了相关步骤，可以使得一个普通 Thread 也具备消息循环处理机制（因为主线程默认做了一些操作，因此不好分析，我们从普通线程入手，可以更清楚的看到整个创建过程）。示例代码可以在 Looper 源码（类注释）中找到：

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
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

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
我们关注一下异常，异常消息的含义是：每一个线程只能有一个 Looper。而判断的条件是，从 sThreadLocal 中获取的对象不为 null——[ThreadLocal](https://github.com/BigFootprint/FaceCode/blob/master/Java/Java%20ThreadLocal.md) 控制每个线程只有一个 Looper 实例。
如果之前没有创建 Looper，则会实例化一个 Looper，看一下构造方法：

```java
//注意：这个方法是私有的
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
构造函数分为两步：1）创建了一个 MessageQueue；2）获取当前的线程。注意 mQueue 这个变量，就是上一节中 Handler 从 Looper 中获取的消息队列对象。

到这里，我们看到了 Handler、Looper 和 MessageQueue 三个对象的创建过程。我们知道以下几个重要的点:

1. `new Handler()`之前，必须保证`Looper.myLooper()`方法的返回不为 null；
2. Looper 在新建的时候会建立对应的 MessageQueue，并且`prepare()`方法会通过`ThreadLocal`保证每一个线程只会实例化一个 Looper。
3. Handler 既会持有 Looper 的引用，又会持有 MessageQueue 的引用；

但是这几个类之间的关系我们却不是很清楚，下面解释。

### 线程、Handler、Looper 的关系
在消息机制里面，Looper 只是负责管理消息队列，也就是取出消息进行处理，而 Handler 则是负责发送消息以及处理消息的，那么 Handler 和 Looper 又是如何绑定到一起的呢？从 Handler 的源码看，关键代码是:  __mLooper = Looper.myLooper();__ ，源码如下:

```java
/**
 * null if the calling thread is not associated with a Looper.
 */
public static Looper myLooper() {
	return sThreadLocal.get();
}
```
这里把注释也 Copy 过来了，myLooper 方法是直接从 sThreadLocal 中读取的变量，注释说到：myLooper() 方法在当前没有 Looper 绑定到调用线程时会返回null。搜索整个 Looper 类，会发现只有前面提到的 prepare() 方法中有 `sThreadLocal.set()` 调用。即在为一个线程实例化 Handler 之前，必须在该线程中调用Looper.prepare() 方法：只有这样，才能为一个线程建立对应的 Looper —— 这解释了前面 LooperThread 代码的构建逻辑。

到这里我们知道，通过调用 Looper.prepare() 方法一次，就可以将一个 Looper 对象绑定到方法调用线程上去(即设置到 ThreadLocal 中)。新建 Handler 的时候，Handler 会直接调用`Looper.myLoop()`方法获取绑定到线程的 Looper 并通过这个 Looper 获取到对应的 MessageQueue。

这就是这几个概念之间的关系。

### Looper 片段——`Looper.loop()`
LooperThread 构建的最后一步就是调用`Loop.loop()`方法，它的源码如下:

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
首先，同样要求 myLooper 不能为 null（根据前面的分析，实现调用 `Looper.prepare()`方法即可）并从这个 Looper 中拿到相应的消息队列。之后我们进入一个无限循环，这个循环中最重要的事情就是`queue.next()`方法的调用，这个方法根据注释：可能会引起阻塞。相关的实现可以深入到 MessageQueue 中，这里涉及到 Native 代码，具体的理解不影响到原理讲述，暂且不表。

这里有一个很重要的判断：如果 Message 拿出来为空，则认为这个 MessageQueue 已经被丢弃了，整个循环会被打破返回，读者可以注意一下。

再之后就进入到 `msg.target.dispatchMessage` 方法。即消息的分发和处理。

> 注意 `msg.target.dispatchMessage `前后的 logging 调用，流行的 BlockCanary 库就是据此原理开发的。

### 关系总结
这里总结一下: 

1. 我们在让一个线程具备消息循环调度能力的时候，首先需要在这个线程中调用 Looper.prepare()，这个方法很重要，它会为一个线程绑定一个 Looper，且只能绑定一个。创建 Looper 的同时，Looper 会创建一个 MessageQueue，这俩是消息循环的基础。之后在线程中创建 Handler 的时候，会从 Looper 中引用MessageQueue 放到 Handler 实例中去。至此 Handler、Looper、MessageQueue 三个类的关系建立。
2. 关系建立之后，调用 Looper.loop() 方法会让 Loop 运行起来：不断从 MessageQueue 中获取 Handler 通过`sendMessage`方法发送的 Message 并 dispatchMessage。至此，整个消息循环的核心部分构建完毕。

### 消息的分发和处理
整个消息循环核心部分已经运行起来，剩下来的就是我们日常最长使用到的消息的发送和处理，接下来看看消息从发送到处理之间的流程。首先是 sendMessage 方法源码:

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
看源码就会发现，同名/功能相似的有好几个方法，但最终都会调用`sendMessageAtTime`方法，首先获取 mQueue：注意这个方法在 Handler 新建的时候就获取了。之后调用了一个 enqueueMessage 方法，源码如下:

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	msg.target = this;
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	return queue.enqueueMessage(msg, uptimeMillis);
}
```
首先，将`msg.target`赋值为本身，之后根据 Handler 新建时候传入的参数（前面忽略了它）设置 msg 属性，之后就调 queue 的 enqueueMessage 向队列中压入消息 —— __完成消息的发送__。

这里很重要的一点是`msg.target = this;`。查看 Message 的源码就可以看到，target 是一个 Handler 变量。而在前面讲述的 `Looper.loop()`方法实现中，取出消息后调用的方法是`msg.target.dispatchMessage(msg);`。嘿！这难道不是调用的 Handler 的`dispatchMessage`方法么？看源码发现果然如此:

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

__【1】__ Message 是可以自带一个 callback 变量的，查看 Meesgae 源码可知这是一个 Runnable 变量，即我们可以使用一个 Message 实现 post 一个 Runable 对象的功能，因为 handleCallback 的实现如下：

```java
private static void handleCallback(Message message) {
	message.callback.run();
}
```
实际上读者可以查看`Handler.postDelayed(Runnable)`方法，内部正是做了这一层转换。

__【2】__如果实例化 Handler 的时候设置了 mCallback 对象（日常开发很少用，但的确存在这种用法），那么所有的小弟都先交给 mCallback 处理，mCallback 是一个 CallBack 对象，这是一个接口:

```java
public interface Callback {
	public boolean handleMessage(Message msg);
}
```
这边可以通过返回 true 来拦截 handleMessage 的执行。

> BlockCanary 也可以通过这种方式进行，不过一旦 msg 的 callback 不为空就无法监控。

__【3】__以上条件都不满足，才轮到`handleMessage`方法执行 —— 关于这个方法，就不多说了，写 Handler 的时候肯定会重载它。

到此，消息的处理分析完毕。

## 剖析总结
到这里，对于整个消息处理发送、循环、发送的机制基本解释清楚。剩下一块比较模糊：MessageQueue 的分析比较少，原因是这块涉及到一些 Native 代码，且对理解整个 Handler 机制的理解影响不大，在这篇文章中暂不分析。

我们用一张图来总结一下几个概念之间的关系:

![Handler消息流转图](http://7xktd8.com1.z0.glb.clouddn.com/Handler原理图.png)

该图重点展示了几个问题：

1. Handler 直接向 MessageQueue 中发送消息；
2. Looper 负责获取消息并作分发（但是具体分发到哪个 Handler 是由 Message 的 target 属性决定的）；
3. __一个线程只会有一个 Looper 和一个 MessageQueue，不论该线程有多少个 Handler，它们都公用这个Looper 和 MessageQueue；__

HandlerThread 是串行化执行任务，因此在里面执行任务是线程安全的（指的是任务之间，而不是和其余线程之间），但是有时候也需要注意将耗时任务和非耗时任务区分在不同的队列里面，提高效率。这个类还很适合执行链式任务，比如编译任务。

## 一些问题
### 主线程和主 Handler
在所有的线程中，主线程是非常特殊的，开发时在主线程中新建 Handler 的实例不需要走上面的流程，直接创建即可，官方文档给出的解释是，主线程本身就已经启动 Looper 了。其实在 Looper 中，是有一个专门的方法做这件事的：

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
注意看注释！！！这个方法就是创建主线程的 Looper 的。注释中说，这个创建是由 Android Environment 执行的，所以开发者__从来不需要__手动调用这个方法。而下面这个方法则是用于获取主 Loope r的：

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

### 消息的 Recycle
Message 内部维护着一个链表，所有被回收的 Message 都挂在这个链表上。关键的方法如下:

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
可以看到消息机制下面其实维护着一个消息链表，这个链表充当着对象池的角色，因此建议大家使用`obtain()`来“创建”消息，这样可以避免创建大量的 Message 对象。补充一点：消息会在被压入队列时置为 FLAG_IN_USE。  

>【注意】在 5.0 以下的时候，gCheckRecycle 开关是关闭的，这意味着回收时不会去检查 Message 是否在使用中。而在 5.0 以上就会检查，此处很坑爹的是：FLAG_IN_USE 标志位的重置是在 obtain 的时候清除重置的，在进入消息队列则会被置位。从前面 `Looper.loop()` 方法可以看到，一个消息会在 dispatchMessage 之后立刻调用 recycle 方法，此时消息的 FLAG_IN_USE 还没有被重置，recycle 必然导致异常抛出——这应该是 SDK 的一个 Bug。

## 补充
1. 以下方法被从 MessageQueue 暴露到 Handler 里面：`hasMeesgae()`，这个方法有好几个重载方法，可以关注一下，用于判断队列中有没有特定的消息;
2. Handler 还有一个这样的方法: `Handler.postAtFrontOfQueue()`，可以让消息插队;
