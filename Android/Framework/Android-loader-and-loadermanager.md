---
title: Loader & LoaderManager源码分析
date: 2015-12-17 16:24:32
tags: [源码, Loader]
categories: Android
---

## 前提
【环境】代码分析基于 supportv4包。

## Loader的介绍和使用
加载器: 用于在Activity和Fragment中异步加载数据，它有以下特性:

1. __异步数据管理__ 除了可以异步加载数据，还能在数据源更新的时候通知上层;
2. __生命周期管理__ 和Activity、Fragment的生命周期绑定，在Activity的配置变化的时候仍然能够继续在后台运行;
3. __缓存数据__ 如果有异步数据无法分发，它可以缓存数据直到有接收者出现，比如在Activity重建期间;
4. __防止内存泄露__ 不会影响Context对象的回收，因为它只依赖于Application Context;

详细介绍和用法可以看[官网文档](http://developer.android.com/intl/zh-cn/guide/components/loaders.html)。<!--more-->

>开发官网已经逐步国际化了，现在能看中文版啦，唯一要求就是翻·个·墙。

## 源码剖析
我们从使用代码切入，调用一个Loader只需要如下代码即可:

```java
//参数的具体类型见下面的方法
getLoaderManager().initLoader(0, null, this);
```
我们来看一下initLoader（位于LoaderManagerImpl中）:

```java
public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
	if (mCreatingLoader) {//表示正在创建中
		throw new IllegalStateException("Called while creating a loader");
	}
	
	LoaderInfo info = mLoaders.get(id);
        
	if (DEBUG) Log.v(TAG, "initLoader in " + this + ": args=" + args);
	if (info == null) {
		// Loader doesn't already exist; create.
		info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
		if (DEBUG) Log.v(TAG, "  Created new loader " + info);
	} else {
		if (DEBUG) Log.v(TAG, "  Re-using existing loader " + info);
		info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
	}
   //这个点要额外注意: 这里就是从已经存在的Loader中获取数据，比如Activity配置变化的时候——实现了前面说的第三点特性，数据缓存。
	if (info.mHaveData && mStarted) {
		// If the loader has already generated its data, report it now.
		info.callOnLoadFinished(info.mLoader, info.mData);
	}
        
	return (Loader<D>)info.mLoader;
}
```

我们先看一下mLoaders变量:

```java
final SparseArrayCompat<LoaderInfo> mLoaders = new SparseArrayCompat<LoaderInfo>();
```
它是一个从id到LoaderInfo的映射，从代码可以看出，如果没有对应的LoaderInfo，就createAndInstallLoader，否则，更新一下callback。如果LoadInfo记录这个Loader已经启动运行并且保存了之前的数据，那么直接调用回调函数，调用的是LoadInfo的callOnLoadFinished方法。最后返回创建出来的Loader。很明显重点在于createAndInstallLoader方法:

```java
private LoaderInfo createAndInstallLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callback) {
	try {
		mCreatingLoader = true;//1. 设置标志
		LoaderInfo info = createLoader(id, args, callback);//2. 创建
		installLoader(info);//3. install
		return info;
	} finally {
		mCreatingLoader = false;
	}
}
```
这里设置了一个标志位mCreatingLoader，表示正在创建Loader，整个过程分为create和install两步。

### Loader的create过程
```java
private LoaderInfo createLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callback) {
	LoaderInfo info = new LoaderInfo(id, args, callback);
	Loader<Object> loader = callback.onCreateLoader(id, args);
	info.mLoader = loader;
	return info;
}
```
创建过程很简单，创建一个LoaderInfo，从回调的onCreateLoader中获取对应的Loader，再将Loader设置到LoaderInfo中，这样LoaderInfo将保存着有关Loader的所有信息：id，args以及Loader本身。

### Loader的install过程
```java
void installLoader(LoaderInfo info) {
	mLoaders.put(info.mId, info);
	if (mStarted) {
		// The activity will start all existing loaders in it's onStart(),
		// so only start them here if we're past that point of the activitiy's
		// life cycle
		info.start();
	}
}
```
首先将LoaderInfo保存到mLoaders中，如果已经start过了，就调用LoaderInfo的start方法。注意方法内部这段注释：__Activity在onStart()方法中会启动所有的loaders，所以只有在onStart()之后还没有启动的Loader才会在这里启动__。

从官网例子中可以看到，我们只需要调用initLoader方法即可，并不需要手动启动加载过程，再加上上面的注释，我们可以猜测: Loader的数据加载是由Activity/Fragment的生命周期方法启动的，下面就来分析一下。

### Loader的启动过程
在supportv4的FragmentActivity中，查看其onStart方法:

```java
@Override
protected void onStart() {
	super.onStart();

	//...
	//最后三行代码
	mFragments.doLoaderStart();

	mFragments.dispatchStart();
	mFragments.reportLoaderStart();
}
```
mFragments是FragmentManager类型，这里就调用了它的doLoaderStart()方法——这里表明：__为了Loader在Activity和Fragment中都能使用，同时逻辑一致，Loader是通过FragmentManager进行管理的，Activity只是引用了FragmentManager的实例。__

>【特别注意】这里还调用了一个和Loader相关的方法:reportLoaderStart。这个方法后面讲retain的时候会用到，暂时不用关注。

我们来看一下FragmentManager的doLoaderStart()方法:

```java
void doLoaderStart() {
	if (mLoadersStarted) {//只能Start一次
		return;
	}

	mLoadersStarted = true;//标记开始

	if (mLoaderManager != null) {
		mLoaderManager.doStart();
	} else if (!mCheckedForLoaderManager) {
		mLoaderManager = getLoaderManager("(root)", mLoadersStarted, false);
		// the returned loader manager may be a new one, so we have to start it
		if ((mLoaderManager != null) && (!mLoaderManager.mStarted)) {
			mLoaderManager.doStart();
		}
	}
	mCheckedForLoaderManager = true;
}
```
这里会初始化LoaderManager，并调用它的doStart()方法:

```java
void doStart() {
	if (DEBUG) Log.v(TAG, "Starting in " + this);
	if (mStarted) {
		RuntimeException e = new RuntimeException("here");
		e.fillInStackTrace();
		Log.w(TAG, "Called doStart when already started: " + this, e);
		return;
	}
	
	//start之后就会设置标志位，还记得前面LoaderManager.installLoader方法对这个参数的使用么？
	mStarted = true;

	// Call out to sub classes so they can start their loaders
	// Let the existing loaders know that we want to be notified when a load is complete
	for (int i = mLoaders.size()-1; i >= 0; i--) {
		mLoaders.valueAt(i).start();
	}
}
```
这里又看到了mStarted标志位，很明显，一个LoaderManager启动两次是会打印异常信息的(不会crash)。然后遍历mLoaders中的所有的LoaderInfo，调用start()方法启动。

### 又见start()方法
前面的分析中，不论Loader是在Activity的onStrat()方法之前还是之后调用的，最终启动都是调用的Loader的start()方法。下面我们来看LoaderInfo的start()方法。

```java
void start() {
	//A
	if (mRetaining && mRetainingStarted) {
		// Our owner is started, but we were being retained from a
		// previous instance in the started state...  so there is really
		// nothing to do here, since the loaders are still started.
		mStarted = true;
		return;
	}

	//B
	if (mStarted) {
		// If loader already started, don't restart.
		return;
	}

	mStarted = true;
	
	if (DEBUG) Log.v(TAG, "  Starting: " + this);
	//Loader为Null则尝试去创建Loader
	if (mLoader == null && mCallbacks != null) {
		mLoader = mCallbacks.onCreateLoader(mId, mArgs);
	}
	if (mLoader != null) {
		if (mLoader.getClass().isMemberClass() && !Modifier.isStatic(mLoader.getClass().getModifiers())) {
			throw new IllegalArgumentException("Object returned from onCreateLoader must not be a non-static inner member class: "
                            + mLoader);
		}
		if (!mListenerRegistered) {
			mLoader.registerListener(mId, this);
			mLoader.registerOnLoadCanceledListener(this);
			mListenerRegistered = true;
		}
		mLoader.startLoading();
	}
}
```
这里在进入正常流程之前会做两个判断：

__A处__  如果一个LoaderInfo已经被retain过，且retain之前已经start了，则标记一下mStarted为true，直接返回(__关于retain的含义，见后面的解释，这里不需要纠结，知道有这个条件判断就好__)；  
__B处__  如果Loader已经启动了，不需要打断或者做别的处理，直接返回；  

__如果Loader是成员类，并且不是static的，则会抛出异常（这里会直接抛异常，比handler更加严厉）。__之后在Loader上注册监听，这两个监听是OnLoadCompleteListener和OnLoadCanceledListener(注: LoaderInfo类实现了这两个接口)，分别监听加载完成和加载取消两个事件。最后调用Loader的startLoading方法开始加载。

### Loader的load过程
Loader的startLoading方法如下:

```java
public final void startLoading() {
	mStarted = true;
	mReset = false;
	mAbandoned = false;
	onStartLoading();
}

/**
 * Subclasses must implement this to take care of loading their data,
 * as per {@link #startLoading()}.  This is not called by clients directly,
 * but as a result of a call to {@link #startLoading()}.
 */
protected void onStartLoading() {
}
```
如上代码，设置flag之后，startLoading就调用了onStartLoading方法，根据注释，子类必须实现这个方法来加载数据。Loader本身是一个接近接口的具体类，实际的代码量很少，读者可以去看它的源码，与前面提到的异步过程没有关系，这是一个很原始的类，实际使用的时候，比较常用的类是__AsyncTaskLoader__，它的源码分析见另外一篇文章。

OK，到这里，Loader的初始化和启动加载过程已经讲完了。我们只需要在onStartLoading()中加载数据即可。先不要好奇数据怎么抛到调用层，前面说了，因为Loader很原始，方法之间的关系没有建立，这块需要去AsyncTaskLoader中体现。

## Loader的停止过程
有了上述创建启动过程，分析停止过程的切入点就比较好找了。根据经验，在Activity的onStart()方法有创建方法，则在对应的onStop()方法中应该就停止方法。onStop()方法比较绕，但是追踪一下不难发现最终会调用到这个方法:

```java
void onReallyStop() {
	mFragments.doLoaderStop(mRetaining);
	mFragments.dispatchReallyStop();
}
```
这里就调用了FragmentController的doLoaderStop方法:

```java
void doLoaderStop(boolean retain) {
	mRetainLoaders = retain;
	if (mLoaderManager == null) {
		return;
	}
	if (!mLoadersStarted) {
		return;
	}
	mLoadersStarted = false;
	if (retain) {
		mLoaderManager.doRetain();
	} else {
		mLoaderManager.doStop();
	}
}
```

这里在LoaderManager不为null且已经启动的情况下，根据retain判断决定调用LoaderManager的方法。

### Loader的retain过程
retain是从外部传出来的，Activity正常stop的时候，这个值为false，但是下面情况是true的：

```java
public final Object onRetainNonConfigurationInstance() {
	if (mStopped) {
		doReallyStop(true);
	}
	//·····
}
```
因此比如转屏的时候，我们只是做retain，而不是stop。LoaderInfo的retain方法如下:

```java
void retain() {
	if (DEBUG) Log.v(TAG, "  Retaining: " + this);
	mRetaining = true;
	mRetainingStarted = mStarted;
	mStarted = false;
	mCallbacks = null;
}
```
这边只是这是一些标志位。并且将__mCallbacks设置为Null__(防止内存泄露)。这边涉及到两个flag——mRetainingStarted和mStarted。前面start()的时候用到了这两个参数:

```java
if (mRetaining && mRetainingStarted) {
	mStarted = true;
	return;
}
```
详细的retain情况后面补充介绍，了解主流程的话到这里就基本可以了。

### Loader的stop过程
LoaderManager的doStop方法是这样的:

```java
void doStop() {
	if (DEBUG) Log.v(TAG, "Stopping in " + this);
	if (!mStarted) {
		RuntimeException e = new RuntimeException("here");
		e.fillInStackTrace();
		Log.w(TAG, "Called doStop when not started: " + this, e);
		return;
	}

	for (int i = mLoaders.size()-1; i >= 0; i--) {
		mLoaders.valueAt(i).stop();
	}
	mStarted = false;
}
```
没有start的时候，是不允许执行stop的，会打印异常信息(同样不会引起crash)。否则遍历所有的LoaderInfo，调用stop()方法，也就是执行下面的方法:

```java
void stop() {
	if (DEBUG) Log.v(TAG, "  Stopping: " + this);
	mStarted = false;
	if (!mRetaining) {
		if (mLoader != null && mListenerRegistered) {
			// Let the loader know we're done with it
			mListenerRegistered = false;
			mLoader.unregisterListener(this);
			mLoader.unregisterOnLoadCanceledListener(this);
			mLoader.stopLoading();
		}
	}
}
```
在没有retain的情况下，主要是取消监听并且调用Loader的stopLoading方法，注意这边调用stopLoading的条件是很苛刻的(PS: 这边这个写法可能有些问题，注册动作和stop不应该有这么强的关系)。Loader里面的stop方法很简单:

```java
public void stopLoading() {
	mStarted = false;
	onStopLoading();
}

/**
 * Subclasses must implement this to take care of stopping their loader,
 * as per {@link #stopLoading()}.  This is not called by clients directly,
 * but as a result of a call to {@link #stopLoading()}.
 * This will always be called from the process's main thread.
 */
protected void onStopLoading() {
}
```
和startLoading()很像，同样子类需要去覆盖实现onStopLoading完成加载。

## 补充
### Loader的retain过程
前面说start的时候，留了一个和retain有关的方法:reportLoaderStart，我们看一下这个方法在做什么。这个方法最终会调用FragmentHostCallback的reportLoaderStart方法:

```java
void reportLoaderStart() {
	if (mAllLoaderManagers != null) {
		final int N = mAllLoaderManagers.size();
		LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];
		for (int i=N-1; i>=0; i--) {
			loaders[i] = (LoaderManagerImpl) mAllLoaderManagers.valueAt(i);
		}
		for (int i=0; i<N; i++) {
			LoaderManagerImpl lm = loaders[i];
			lm.finishRetain();
			lm.doReportStart();
		}
	}
}
```
这里涉及到mAllLoaderManagers变量，它的声明是这样的:

```java
/* The loader managers for individual fragments [i.e. Fragment#getLoaderManager()] */
private SimpleArrayMap<String, LoaderManager> mAllLoaderManagers;
```
是一个从String到LoaderManager的映射，另外从变量名称可以看出，这里应该集合了属于Activity的所有的LoaderManager。搜索它被使用的地方可以看到:

```java
LoaderManagerImpl getLoaderManager(String who, boolean started, boolean create) {
	if (mAllLoaderManagers == null) {
		mAllLoaderManagers = new SimpleArrayMap<String, LoaderManager>();
	}
	LoaderManagerImpl lm = (LoaderManagerImpl) mAllLoaderManagers.get(who);
	if (lm == null) {
		if (create) {
			lm = new LoaderManagerImpl(who, this, started);
			mAllLoaderManagers.put(who, lm);
		}
	} else {
		lm.updateHostController(this);
	}
	return lm;
}
```
这里是以一种putIfAbsent的方式在创建LoaderManagerImpl，这个方法是包内可见的，我在FragmentActivity和Fragment中getSupportLoaderManager方法和getLoaderManager方法，发现最终都是调用到这个方法获取属于Activity/Fragment的LoaderManager。读者细心的话，应该在doLoaderStart中就注意到它了:

```java
mLoaderManager = getLoaderManager("(root)", mLoadersStarted, false);
```
而在Fragment中，这个方法调用的地方就更多了，比如onStart()中：

```java
public void onStart() {
	mCalled = true;

	if (!mLoadersStarted) {
		mLoadersStarted = true;
		if (!mCheckedForLoaderManager) {
			mCheckedForLoaderManager = true;
			mLoaderManager = mHost.getLoaderManager(mWho, mLoadersStarted, false);
		}
		if (mLoaderManager != null) {
			mLoaderManager.doStart();
		}
	}
}
```
而mWho的设置是这样的，其实可以认为是Fragment的一个Tag:

```java
final void setIndex(int index, Fragment parent) {
	mIndex = index;
	if (parent != null) {
		mWho = parent.mWho + ":" + mIndex;
	} else {
		mWho = "android:fragment:" + mIndex;
	}
}
```
再往上就不追溯了，总之每一个Fragment自身都会有一个独有的mWho，而这个参数又被用来标识一个LoaderManager。因此这里不难猜测出Loader和Fragment、Activity的关系:

__在一个Activity中，可以有N个Fragment，Activity和每一个Fragment都有自己的LoaderManager，其中Activity是以(root)标记的(为什么root是Activity的？可以从FragmentActivity的onStart方法看到，是先调用doLoaderStart，不管onStart之前是否调用了getLoaderManager，这里都会建立一个root为标记的LoaderManager，这个LoaderManager就是属于Activity的)，Fragment则是以自己的mWho属性标记的，它们都被记录在FragmentHostCallback的mAllLoaderManagers属性中(这个属性被FragmentHostCallback引用，FragmentHostCallback属性被FragmentController引用，FragmentController又被FragmentActivity引用)。__

试想一下一个Fragment被移除的过程？所有在它内部被创建的Loader都会被销毁掉，而不会影响到其余的Loader。

上面解释了一下所有的LoaderManager都被mAllLoaderManagers变量使用一个String索引着。我们回到前面reportLoaderStart的代码，再贴一下:

```java
void reportLoaderStart() {
	if (mAllLoaderManagers != null) {
		final int N = mAllLoaderManagers.size();
		LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];
		for (int i=N-1; i>=0; i--) {
			loaders[i] = (LoaderManagerImpl) mAllLoaderManagers.valueAt(i);
		}
		for (int i=0; i<N; i++) {
			LoaderManagerImpl lm = loaders[i];
			lm.finishRetain();
			lm.doReportStart();
		}
	}
}
```
这里遍历所有的LoaderManager的实例，调用他们的finishRetain和doReportStart方法。LoaderManager再将这些分发到响应的Loader中去。LoaderInfo中两个方法的实现如下：

```java
void finishRetain() {
	if (mRetaining) {
		if (DEBUG) Log.v(TAG, "  Finished Retaining: " + this);
		mRetaining = false;
		if (mStarted != mRetainingStarted) {
			if (!mStarted) {
				// This loader was retained in a started state, but
				// at the end of retaining everything our owner is
				// no longer started...  so make it stop.
				stop();
			}
		}
	}

	if (mStarted && mHaveData && !mReportNextStart) {
		// This loader has retained its data, either completely across
		// a configuration change or just whatever the last data set
		// was after being restarted from a stop, and now at the point of
 		// finishing the retain we find we remain started, have
		// our data, and the owner has a new callback...  so
		// let's deliver the data now.
		callOnLoadFinished(mLoader, mData);
	}
}

void reportStart() {
	if (mStarted) {
		if (mReportNextStart) {
			mReportNextStart = false;
			if (mHaveData) {
				callOnLoadFinished(mLoader, mData);
			}
		}
	}
}
```
finishRetain的时候，如果之前retain了，则停止掉(mRetaining置为true)。接下来的判断比较难以理解。首先明确一下mStarted和mRetainingStarted的含义: mStarted标记的是当前Loader是否start的状态，而mRetainingStarted标记的是Loader在retain之前的start状态。所以这个判断的含义是，如果当前的start状态和retain之前的start状态不一致，且当前的状态是未启动的，换句话说，之前是启动的，retain恢复之后又不启动了，那么可以认为这个Loader不再被需要了，因此直接stop掉——读者应当注意FragmentActivity.onStart()方法里面doLoaderStart()和reportLoaderStart()方法的调用顺序，是先全部启动Loader，再report的。

之后，如果Loader已经开始了，数据也有了，而且callback已经被更新了(环境重建)，那么我们就可以调用callOnLoadFinished分发数据了。reportStart类似。

>补充：从代码看，当Fragment的performDestroyView方法被调用，即Fragment被销毁的时候，所有属于它的LoaderInfo的mReportNextStart都会被标为true，reportStart()之后都会被标为false，当然初始化的时候也是false。

callOnLoadFinished代码如下:

```java
void callOnLoadFinished(Loader<Object> loader, Object data) {
	if (mCallbacks != null) {
		String lastBecause = null;
		if (mHost != null) {
			lastBecause = mHost.mFragmentManager.mNoTransactionsBecause;
			mHost.mFragmentManager.mNoTransactionsBecause = "onLoadFinished";
		}
		try {
			if (DEBUG) Log.v(TAG, "  onLoadFinished in " + loader + ": " + loader.dataToString(data));
			//分发数据
 			mCallbacks.onLoadFinished(loader, data);
		} finally {
			if (mHost != null) {
				mHost.mFragmentManager.mNoTransactionsBecause = lastBecause;
			}
		}
		mDeliveredData = true;
	}
}
```

如果mCallbacks不为null，那么调用它的onLoadFinished分发数据。

### Loader的Restart过程
Loader的restart过程是通过`LoaderManager.restartLoader()`方法实现的。方法注释中说，重新创建一个ID指向的Loader，如果这个ID已经有绑定的Loader，那么这个Loader会被cancel、stop或者destroy，然后使用新的arguments创建一个Loader。这就是它与`LoaderManager.initLoader()`方法的区别——__不存在重用__。

因此，如果在组件的生命周期之间使用缓存数据(即数据不会变化)，则应该使用`LoaderManager.initLoader()`，如果可能变化，则应该使用`LoaderManager.restartLoader()`方法。

Restart过程从`LoaderManager.restartLoader()`方法切入即可，基于前面的解释，看懂这个方法并不困难，其中可以看到一些重要的方法是如何调用的，比如：LoaderCallbacks的`onLoaderReset()`方法，Loader的`onAbandon()`方法。请读者自行解析。

## 使用注意点
1. 可以通过`Loader.forceLoad()`方法强制重新加载数据，数据加载完成会调用`onLoadFinished()`回调，也可以调用`Loader.cancelLoad()`取消回调，将不会受到任何回调;
2. 在Activity、Fragment销毁的时候或者手动调用了`LoaderManager.destroyLoader(id)`方法的时候，Loader会被销毁，回调方法`onLoaderReset(Loader)`会被调用，表示数据失效，需要重置;
3. 前面说了Loader可以监测数据的变化，但有时候数据变化过快，导致UI持续刷新，可以通过如下方法降低刷新频率: `setUpdateThrottle(long delayMs)`;

Loader数据加载相关方法执行流程如下:
![Loader数据加载](http://7xktd8.com1.z0.glb.clouddn.com/Loader数据加载.png)

## 总结
以上解析了Loader是如何初始化并启动的，整个数据的加载和停止又是如何绑定到Activity和Fragment的生命周期的。

然而Loader本身的用处并不大，因为Loader本身没有任何的异步调度，直接使用Loader进行开发显得太过原始，开发中一般会使用__AsyncTaskLoader__。AsyncTaskLoader结合了[AsyncTask](http://www.muzileecoding.com/androidsource/Android-AsyncTask.html)和Loader的机制，将数据加载方到了异步任务中(源码里是通过LoadTask实现的)，并重新声明了一些回调函数，有力的支持了Activity和Fragment的异步加载数据任务。
> __AsyncTaskLoader__并不依赖于平台的AsyncTask，因为平台的AsyncTask并行或者串行执行任务是随着平台版本变化的。
>
> __AsyncTaskLoader__会尽可能的减少任务的数量，如果我们调用`forceLoader()`方法，前面的相同任务会被取消掉，是不会有任何回调的。

对于数据库查询，Loader还有一个子类叫做：CursorLoader，读者有兴趣也可以去看一下。

### 自定义Loader
如总结所说，Loader本身的功能很弱，虽然有AsyncTaskLoader和CursorLoader等具体类支持，但是开发者仍可能有需求需要自定义一个Loader，这时候参考实现如下特性:

1. Loader的生命周期;
2. 后台数据加载;
3. 数据管理;
4. 发送缓存结果;

Loader的生命周期方法轮转如下:

![Loader状态](http://7xktd8.com1.z0.glb.clouddn.com/Loader状态.png)
这其中涉及到一些重要方法的实现: 
```java
void startLoading() -> void onStartLoading() void stopLoading() -> void onStopLoading() 
void reset() -> void onReset()void abandon() -> void onAbandon()
```
以及`forceLoad ()`方法的实现，这些方法在Loader类中都有详细的注释，具体实现可以参考注释以及AsyncTaskLoader的实现。

### 数据管理
监控数据变化的方式: 

1. 使用Java的Observable和Observer;
2. 广播告知;
3. FileObserver——监控一个路径上的文件变化;

>PS: 自定义的时候可以仔细研究一下AsyncTaskLoader的实现。
