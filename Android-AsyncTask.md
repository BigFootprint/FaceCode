---
title: AsyncTask 源码分析
date: 2015-12-02 20:32:36
tags: [源码]
categories: Android
---

## 前言
【环境】源码分析基于 6.0。  
【知识】请读者自备FutureTask相关技能。

## AsyncTask的介绍和使用
如官网所述:
>AsyncTask enables proper and easy use of the UI thread. This class allows to perform background operations and publish results on the UI thread without having to manipulate threads and/or handlers.

简而言之，AsyncTask是为了在后台线程和UI线程之间做任务调度而开发的一个类，使得这种常见的操作更加容易简练，而不用去操作线程本身以及Handler。详细介绍见官方文档: [AsyncTask](http://developer.android.com/intl/zh-cn/reference/android/os/AsyncTask.html)。<!--more-->

源码(类注释)中给出的🌰如下:

```java
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long>{
	//在后台线程中执行
	protected Long doInBackground(URL... urls) {
		int count = urls.length;
		long totalSize = 0;
		for (int i = 0; i < count; i++) {
			totalSize += Downloader.downloadFile(urls[i]);
 			publishProgress((int) ((i / (float) count) * 100));
			// Escape early if cancel() is called
			if (isCancelled()) break;
		}
		return totalSize;
	}

	//在UI线程中执行
	protected void onProgressUpdate(Integer... progress) {
		setProgressPercent(progress[0]);
	}

	//在UI线程中执行
	protected void onPostExecute(Long result) {
		showDialog("Downloaded " + result + " bytes");
	}
}
```
执行使用如下代码:

```java
new DownloadFilesTask().execute(url1, url2, url3);
```

使用过程非常方便，任务被分解成耗时的后端任务以及UI线程的回调两部分，开发者不需要手动进行任务的调度，只需要按照约定，将操作分解到对应的函数中，就可以完成任务，非常方便。下面我们就来探究一下，AsyncTask是如何实现这个功能的。

## 源码剖析
如上示例所示，AsyncTask有几个重要的"生命周期"方法:

1. onPreExecute()
2. doInBackground()
3. onProgressUpdate()
4. onPostExecute()
5. onCancelled()

关于它们具体的作用和调用时间，在源码的类注释以及官方API文档里面都有详细的解释，此处不赘述。(懒得写...其实看名字就能猜出大概)

如🌰所示，我们看一下整个过程是如何发生的。

### 实例化
首先是AsyncTask的创建过程，其代码如下:

```java
public AsyncTask() {
	mWorker = new WorkerRunnable<Params, Result>() {
		public Result call() throws Exception {
			mTaskInvoked.set(true);

			Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
			//noinspection unchecked
			Result result = doInBackground(mParams);
			Binder.flushPendingCommands();
			return postResult(result);
		}
	};

	mFuture = new FutureTask<Result>(mWorker) {
		@Override
		protected void done() {
			try {
				postResultIfNotInvoked(get());
			} catch (InterruptedException e) {
				android.util.Log.w(LOG_TAG, e);
			} catch (ExecutionException e) {
				throw new RuntimeException("An error occurred while executing doInBackground()",
					e.getCause());
			} catch (CancellationException e) {
				postResultIfNotInvoked(null);
			}
		}
	};
}
```
创建过程只是实例化了两个变量: mWorker & mFuture。mWorker是一个WorkerRunnable对象，这个类的定义如下:

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
}
```
继承了Callable对象，添加了Params实例，至于Params和Result的类型，在AsyncTask类签名中可以看到:

```java
public abstract class AsyncTask<Params, Progress, Result>
```
是泛型，由使用者自己决定。因此__WorkerRunnable其实是Callable的增强版，可以传递另外一个泛型参数进去__。

mFuture是一个FutureTask对象，接受mWorker对象作为参数。要弄清楚这两个对象的作用，先撇下这两个对象的声明细节，继续往后看。

### 执行开始
使用示例中执行task调用的是execute方法，这个方法会调用另外一个方法:

```java
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
	if (mStatus != Status.PENDING) {
		switch (mStatus) {
			case RUNNING:
				throw new IllegalStateException("Cannot execute task:"	+ " the task is already running.");
			case FINISHED: throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
		}
	}

	mStatus = Status.RUNNING;//设置任务状态
	onPreExecute();
	mWorker.mParams = params;
	exec.execute(mFuture);
	return this;
}
```
这里首先判断任务是否是未执行的状态，如果不是，则根据当前状态抛出异常，这段代码表明：__一个AsyncTask只能执行一次(每次运行都得重新创建)。__
>__PENDING__是创建好但没有执行的状态，FINISHED是终结态，可以通过`AsyncTask.getStatus()`方法来获知状态。

onPreExecute()方法在一开始就会调用，执行在execute()方法的调用线程上。传递进来的参数会被赋值给mWorker的mParams属性，在线城池中执行任务。

### 执行过程
执行过程的代码就在mWorker的声明中:

```java
public Result call() throws Exception {
	mTaskInvoked.set(true);
	Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
	//noinspection unchecked
	Result result = doInBackground(mParams);
	Binder.flushPendingCommands();
	return postResult(result);
}
```
mTaskInvoked是一个AtomicBoolean类，从名称看，是表示该任务是否被执行过，具体作用后面详述。

将线程的优先级设置为后台线程优先级，调用第二个重要的生命周期方法doInBackground执行任务。最后调用postResult方法返回结果，postResult方法如下:

```java
private Result postResult(Result result) {
	@SuppressWarnings("unchecked")
	Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
	message.sendToTarget();
	return result;
}
```
这里从一个getHandler()方法中获取一个Message，同时将一个AsyncTaskResult对象放置到Meesage中，将消息分发到主线程中。下面看一下这段代码中的两个Key Point。

#### AsyncTaskResult
顾名思义，是为AsyncTask执行结果声明的类:

```java
private static class AsyncTaskResult<Data> {
	final AsyncTask mTask;
	final Data[] mData;
	
	AsyncTaskResult(AsyncTask task, Data... data) {
		mTask = task;
		mData = data;
	}
}
```
类很简单，持有Task本身和mData——这是我们的任务执行结果。

#### getHandler()
getHandler()返回的是一个InternalHandler实例，类声明如下:

```java
private static class InternalHandler extends Handler {
	public InternalHandler() {
		super(Looper.getMainLooper());
	}

	@SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
	@Override
	public void handleMessage(Message msg) {
		AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
		switch (msg.what) {
			case MESSAGE_POST_RESULT:
				// There is only one result
				result.mTask.finish(result.mData[0]);
				break;
			case MESSAGE_POST_PROGRESS:
				result.mTask.onProgressUpdate(result.mData);
			break;
		}
	}
}
```
首先，这是一个主线程的Handler——保证相关的函数被调度到UI线程执行；其次，处理两种消息：MESSAGE_POST_RESULT和MESSAGE_POST_PROGRESS，即线程执行进度更新和执行完成的回调。result.mTask就是我们当前的AsyncTask。读者可能疑惑MESSAGE_POST_PROGRESS消息是什么时候发出的，看publishProgress()就会明白，这个方法是开发者自己调用的，不是必经的生命周期，因此这里不作分析。

### 执行结束
执行结束之后的代码在一开始也看到了，就是mFuture的声明，执行代码如下:

```java
protected void done() {
	try {
		postResultIfNotInvoked(get());
	} catch (InterruptedException e) {
		android.util.Log.w(LOG_TAG, e);
	} catch (ExecutionException e) {
		throw new RuntimeException("An error occurred while executing doInBackground()",
		e.getCause());
	} catch (CancellationException e) {
		postResultIfNotInvoked(null);
	}
}
```
只调用了一个方法:postResultIfNotInvoked(get())。get()方法是FutureTask的方法，会阻塞直至任务执行完成。postResultIfNotInvoked方法的实现是这样的:

```java
private void postResultIfNotInvoked(Result result) {
	final boolean wasTaskInvoked = mTaskInvoked.get();
	if (!wasTaskInvoked) {
		postResult(result);
	}
}
```
这里又出现了mTaskInvoked，配合方法名称postResultIfNotInvoked可以知道，这段代码是在任务没有被执行的时候才会被调用。前面看到call()方法的一开始就会把mTaskInvoked设置为true，这里有些奇怪:

1. 按道理一个任务执行起来之后，首先就会执行mTaskInvoked的set方法，那么这里postResult肯定不会被执行。但如果任务没有被Invoked，又怎么会执行到postResultIfNotInvoked方法呢？postResult以及postResultIfNotInvoked都是private的，只可能走这个流程调用；
2. 在call()方法的最后，已经调用postResult了，为什么这里还要再次判断并分发结果？

一步步来：要 __!wasTaskInvoked__ 为true，则call方法不能被执行，否则mTaskInvoked肯定会被置为true，那么只有一种办法，就是call方法刚执行的时候就抛出了异常，而异常在FutureTask中是很常见的，至少有两种，就是InterruptedException和CancellationException。下面我们来看第5个生命周期方法，看完就知道如何解释这个判断了。

### 取消执行
AsyncTask是可以取消的，调用方法cancel()就好:

```java
public final boolean cancel(boolean mayInterruptIfRunning) {
	mCancelled.set(true);
	return mFuture.cancel(mayInterruptIfRunning);
}

public final boolean isCancelled() {
	return mCancelled.get();
}
```
mCancelled也是一个AtomicBoolean对象，标识任务是否被取消，我们看一下这个变量有什么影响。

在任务结束的时候，调用的是AsyncTask.finsh()方法(可以查看InternalHandler对MESSAGE_POST_RESULT类型消息的处理):

```java
private void finish(Result result) {
	if (isCancelled()) {
		onCancelled(result);
	} else {
		onPostExecute(result);
	}
	//注意，不论任务是正常完成还是取消，状态都标记为FINISHED
	mStatus = Status.FINISHED;
}
```
这里会做出判断，如果被取消了，是回调生命周期方法onCancelled，否则调用onPostExecute。

到此还没有回答前面的问题，但是取消是很关键的。熟悉FutureTask的人应该知道CancellationException发生的情况：如果有人在AsyncTask运行之前，即call执行之前调用cancel方法，那么只要AsyncTask一运行，就会立刻抛出CancellationException异常，进入done()方法执行。

__postResultIfNotInvoked()方法就是为了防止在这种情况下不回调onCancelled()设置的: 如果正常执行，不发生任何的异常，则会执行call最后的postResult()方法直接分发结果，但如果发生了异常，则会直接进入done()函数，done函数可以保证，如果是因为CancellationException异常发生执行错误，一定会回调onCancelled()方法。并且call()和done()方法虽然都会去调用postResult()方法，但条件上互斥，即结果不会分发两次。__

可以参见这个[【Bug】](https://android.googlesource.com/platform/frameworks/base/+/5ba812b9e9c886425a5736c2ae6fbe0fc94afd8b%5E%21/#F0)

关于Cancel方法的几个影响，可以参见下列表格: 

![AsyncTask-Cancel方法](http://7xktd8.com1.z0.glb.clouddn.com/AsyncTask-Cancel方法.png)
以上，相关方法都分析完毕。

## 线程池
AsyncTask本身其实是调度后台线程执行耗时任务，底层是使用线程池管理后台线程的——这也就意味着，大量的AsyncTask并不会无限制的创建后台线程。在AsyncTask内部，有两种线程池，一个是默认的，一个是串行线程池。

### 默认线程池
默认线程池的声明是这样的:

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE = 1;
private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```
根据CPU核数量设置线程池的运行能力，缓存队列中最多存储128个任务，一般来说，这个数量应该够用了。但是开发者还是要注意管理，任务超过128 + CPU_COUNT * 2 + 1，AsyncTask就会报错，这是线程池的机制。

### 串行线程池
串行线程池是AsyncTask自己实现的:

```java
private static class SerialExecutor implements Executor {
	final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
	Runnable mActive;
	
	public synchronized void execute(final Runnable r) {
		mTasks.offer(new Runnable() {
			public void run() {
				try {
					r.run();
				} finally {
					scheduleNext();//执行下一个
				}
			}
		});
		if (mActive == null) {
			scheduleNext();//执行下一个
		}
	}

	protected synchronized void scheduleNext() {
		if ((mActive = mTasks.poll()) != null) {
			THREAD_POOL_EXECUTOR.execute(mActive);
		}
	}
}
```
它会不停的从队列中拉取Runnable对象，丢入默认线程池中执行，但是每次只会执行一个：execute()的时候会去检测有没有线程正在运行，没有的话会去队列中获取任务，或者在一个任务执行完成之后也会去队列中获取任务继续执行。通过这种方式，实现任务的串行。

AsyncTask默认到底是并行还是串行的，其实是不一定的，一下是串行还是并行与版本的关系:

![Asynctask-Execution-Differences](http://7xktd8.com1.z0.glb.clouddn.com/Asynctask-Execution-Differences.png)
最终默认方式采用并行还是串行，是依据targetSdkVersion来的，targetSdkVersion<13的时候默认是串行的(即使运行的平台大于等于13)，targetSdkVersion>=13的时候默认就是串行的。(现在这些问题都不需要担心了)。我们顺便看一下其余几个方法的情况:

![Android-AsyncTask- Execution](http://7xktd8.com1.z0.glb.clouddn.com/Android-AsyncTask- Execution.png)
>PS: `AsyncTask.execute(Runnable)`方法是一个很奇怪的方法，它执行在AsyncTask内部的线程池中，这个时候其实就是把AsyncTask当做线程池在使用。

如果是使用默认的线程池，则AsyncTask全局使用的是同一个线程池，这在某些情况下可能会出现问题——我们可以通过自定义线程池来改善。

## 总结
AsyncTask本质是基于Java的FutureTask和Android的Handler做的一层封装，添加了后台线程和主线程之间的任务调度，精巧实用，下面是一些使用AsyncTask的参考点:

1. 如果我们创建了一个AsyncTask，执行它的时候不需要传递参数，或者只需要重载`AsyncTask.doInBackground()`方法，那么可以考虑别的手段，比如[HandlerThread](http://www.muzileecoding.com/androidsource/Android-HandlerThread-And-IntentService.html)。
2. AsyncTask自身是没有Looper的，所以不能实现消息的传递，理论上可以通过一些方法进行改造，但很明显这样做不值得，这个时候同样可以考虑[HandlerThread](http://www.muzileecoding.com/androidsource/Android-HandlerThread-And-IntentService.html)。
3. AsyncTask和Local Service是一组很好的搭配。因为Service的执行也是在主线程的，如果执行一些耗时任务，就需要开启背景线程，这个时候使用AsyncTask是一个不错的选择，但是最好自己起一个线程池。

大部分时候AsyncTask使用起来都不会有什么问题，几个生命周期方法也非常好懂。但是通过刚刚的分析，__要注意的是onPreExecute()方法的使用__。这个方法并没有通过Handler强制调度到主线程执行，它是可能运行在后台线程上的——取决于AsyncTask的execute方法在什么线程上被执行。但就算这样也不必过于担心，我们绝大部分情况下都是在主线程上使用AsyncTask的，并且，这个方法用到的也不多。

下面是一个试验:

```java
package com.footprint.littleshell;

import android.app.Activity;
import android.os.AsyncTask;
import android.os.Bundle;
import android.os.Looper;
import android.util.Log;

public class TestActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        new Thread(new Runnable() {
            @Override
            public void run() {
                MyTask myTask = new MyTask();
                myTask.execute(0);
            }
        }).start();
    }
}

class MyTask extends AsyncTask<Integer, Integer, String> {
    private static final String TAG = "MyTask";

    @Override
    protected String doInBackground(Integer... params) {
        if (isMainThread())
            Log.e(TAG, "【doInBackground】- Main");
        else Log.e(TAG, "【doInBackground】- Non-Main");

        return "";
    }

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        if (isMainThread())
            Log.e(TAG, "【onPreExecute】- Main");
        else Log.e(TAG, "【onPreExecute】- Non-Main");
    }

    @Override
    protected void onPostExecute(String s) {
        if (isMainThread())
            Log.e(TAG, "【onPostExecute】- Main");
        else Log.e(TAG, "【onPostExecute】- Non-Main");
    }

    private boolean isMainThread() {
        return Looper.myLooper() != null && Looper.myLooper() == Looper.getMainLooper();
    }
}
```

打印结果:

```java
02-29 21:29:41.959     905-2384/com.footprint.littleshell E/MyTask﹕ 【onPreExecute】- Non-Main
02-29 21:29:42.389     905-2385/com.footprint.littleshell E/MyTask﹕ 【doInBackground】- Non-Main
02-29 21:29:42.389     905-905/com.footprint.littleshell E/MyTask﹕ 【onPostExecute】- Main
```
结果验证了之前关于onPreExecute()方法的结论。