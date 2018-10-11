---
title: Android的数据恢复(二)
date: 2015-10-17 10:58:26
tags: [源码]
categories: Android
---

## 背景
在Android上，会有以下原因导致需要保存和恢复当前页面的数据：  

1. 一个Activity退到后台：比如跳转到另外一个Activity，或者按Home键退到后端;
2. 系统配置变化，如：屏幕旋转(在用户意识里只是屏幕旋转，实际上Activity是被杀掉后重建);

第一种情况只有在内存不足，后台Activity确实被系统回收掉的时候才会进行数据恢复，如果Activity正常存在于内存中，则不需要数据恢复，但一定会做数据保存。第二种情况是因为在用户眼里只是界面旋转了一下，而实际上Activity是被销毁重建的，这个时候做数据恢复是合理而且应当的。<!--more-->

## 数据如何保存
一般来说，我们通过三个方法进行Activity的数据恢复与保存，分别如下： 
```java
protected void onSaveInstanceState(Bundle outState)  
protected void onCreate(Bundle savedInstanceState)  
protected void onRestoreInstanceState(Bundle savedInstanceState)
```
我们通过第一个方法的outState参数保存数据，后面两个方法的savedInstanceState参数获取保存的数据。onRestoreInstanceState方法在onStart后面onResume前面执行。

关于以上两点，开发者官网上都有比较完善的描述，读者可以仔细阅读：[【原文链接】](http://developer.android.com/training/basics/activity-lifecycle/recreating.html)

这一部分稍有经验的开发者都比较熟悉，本文重点讲述的是__View如何去保存恢复数据__。

## View状态如何保存
一个Activity展示给用户的除了数据，就是View，所以当我们做恢复的时候，不仅需要恢复重要数据，也需要尽可能的恢复View的状态。

日常开发中我们一般是通过onSaveInstanceState去保存View的状态(比如EditTex的输入内容)，恢复的时候将数据读出设置到View中去，但这种方式粗暴有用但不优雅，举个🌰：我封装了一个CustomView，其中包含着一个EditText，如果按照前面的方式去做恢复则EditText一定要暴露给外部，否则Activity就不能获取并设置值，这破坏了组件的封装性。

其实Android内部是有一套View状态保存和恢复机制的，不需要通过Activity的生命周期方法去实现。阅读官方文档的读者可以注意一点，文中提到了EditText状态的保存条件：需要有一个唯一的id。这是为什么呢？

下面我们来详细看一下View的数据保存机制，从中找出回答前面问题的答案。

>【Note】建议打开Android Studio，下面的代码片段比较多，直接阅读源码对照阅读比较容易理解。

### Activity的onSaveInstanceState()方法
View说到底是属于Activity的，onSaveInstanceState是整个Activity保存数据的出发点，我们从这里入手:
```java
protected void onSaveInstanceState(Bundle outState) {
	outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
	Parcelable p = mFragments.saveAllState();
	if (p != null) {
		outState.putParcelable(FRAGMENTS_TAG, p);
	}
	getApplication().dispatchActivitySaveInstanceState(this, outState);
}
```
如代码所示，第二行将整个Window的状态保存在一个Bundle中，将值保存到ontState中去，看上去很有可能与View相关，我们继续往下走。

3-6行则是触发Fragment数据保存的（下一篇博文主题）。关于Windnow的解释，可以查看这篇博客：[【Android】Android界面从里至外浅析（一）](http://www.cnblogs.com/lqminn/archive/2013/05/01/3050776.html)

### Window的saveHierarchyState()方法
了解之后，我们定位到PhoneWindow中的saveHierarchyState方法：
```java
public Bundle saveHierarchyState() {
	Bundle outState = new Bundle();
	if (mContentParent == null) {
 		return outState;
 	}
 	SparseArray<Parcelable> states = new SparseArray<Parcelable>();
 	mContentParent.saveHierarchyState(states);
 	outState.putSparseParcelableArray(VIEWS_TAG, states);
 	....
}
```
此处省略了部分代码。我们看到，这个方法触发了mContentParent的saveHierarchyState的方法。mContentParent又是何方神圣？查看其声明发现：它是一个ViewGroup。比对[【Android】Android界面从里至外浅析（一）](http://www.cnblogs.com/lqminn/archive/2013/05/01/3050776.html)中的图，这个mContentParent又在什么位置呢？读者可以阅读一下PhoneWindow里面的setContentView方法，其实这个mContentParent就是图中DecorView下的LinearLayout，即我们setContentView的父容器。

然后我们看到方法声明了一个SparseArray对象，存储的是Parcelable，这个SparseArray对象被传递给了ViewGroup的saveHierarchyState方法，之后这个SparseArray对象又被存储到了outBundle中去了，那么问题就在于saveHierarchyState这个方法到底做了什么？

### ViewGroup的saveHierarchyState()方法
标题虽然是ViewGroup的saveHierarchyState()方法，但ViewGroup本身其实并没有saveHierarchyState方法，这个方法继承于它的父类——View。

```java
public void saveHierarchyState(SparseArray<Parcelable> container) {
	dispatchSaveInstanceState(container);
}

protected void dispatchSaveInstanceState(SparseArray<Parcelable> container){
	if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
		mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
		Parcelable state = onSaveInstanceState();
		if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
			throw new IllegalStateException("Derived class did not call super.onSaveInstanceState()");
		}
		if (state != null) {
			// Log.i("View", "Freezing #" + Integer.toHexString(mID)
			// + ": " + state);
			container.put(mID, state);
		}
	}
}
```
如上所示，在这个方法中，它调用了dispatchSaveInstanceState方法，而这个方法则调用本身的onSaveInstanceState方法获取一个Parcelable对象，通过View的id将这个对象存储到SparseArray里面去了。而ViewGroup里面重写了dispatchSaveInstanceState方法：
```java
protected void dispatchSaveInstanceState(SparseArray<Parcelable> container){
	super.dispatchSaveInstanceState(container);
	final int count = mChildrenCount;
	final View[] children = mChildren;
	for (int i = 0; i < count; i++) {
		View c = children[i];
		if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
			c.dispatchSaveInstanceState(container);
		}
	}
}
```
代码很好理解：它不但保存自己的状态，还遍历所有的子View，调用他们的dispatchSaveInstanceState方法去保存数据。

### View数据保存总结
综上，View本身的数据保存机制是这样的: __Activity会触发根ViewGroup将该命令传达给所有的子View，同时传递一个SparseArray给它们。View调用自己的onSaveInstanceState()将自己的状态以<id, State>方式存储到SparseArray中去，最终SparseArray会被保存到outState中。__

现在解答前面说到的一个问题：id唯一的必要性。平时编码中id仅用于定位View，所以如果能实现这个功能，即使Id重复也没有关系。但是在保存状态的时候，因为Key就是id，如果id重复，那会怎么样呢？保存的时候数据会被覆盖，恢复的时候相同id的View会被以同样的数据恢复，很多人都遇到这样的问题，原因就在于此。

## 例子
这里通过以TextView为🌰看一下实际的情况:

```java
public Parcelable onSaveInstanceState() {
	Parcelable superState = super.onSaveInstanceState();
	// Save state if we are forced to
	boolean save = mFreezesText;
	int start = 0;
	int end = 0;
	if (mText != null) {
		start = getSelectionStart();
		end = getSelectionEnd();
		if (start >= 0 || end >= 0) {
			// Or save state if there is a selection
			save = true;
		}
	}
	if (save) {
		SavedState ss = new SavedState(superState);
		// XXX Should also save the current scroll position!
		ss.selStart = start;
		ss.selEnd = end;
		if (mText instanceof Spanned) {
			Spannable sp = new SpannableStringBuilder(mText);
			if (mEditor != null) {
				removeMisspelledSpans(sp);
				sp.removeSpan(mEditor.mSuggestionRangeSpan);
			}
			ss.text = sp;
		} else {
			ss.text = mText.toString();
		}
		if (isFocused() && start >= 0 && end >= 0) {
			ss.frozenWithFocus = true;
		}
		ss.error = getError();
		return ss;
	}
	return superState;
}
```
如代码所示，TextView就通过这个方法保存了它显示的text，不过开发者可以通过设置mFreezesText控制否要保存。读者有兴趣还可以去看看ScrollView这个方法的实现。

##总结
__数据恢复的过程和数据保存的流程类似，此处略去不表。__

从开发经验来看，Android上回收资源发生的比较频繁，做好状态保存恢复工作对于提升用户体验有很大帮助。文章主要阐述了View状态保存恢复的过程，有助于开发者更好的利用该机制: 

1. 注意id问题——粗暴的方式问题也不大，但是如果粗暴已经引起了设计或者实现问题，可以考虑View的内建机制，出现问题的时候，id的重复也可以作为一个考虑方向;
2. 如果想写一个CustomView并且想保存状态，可以通过重载onSaveInstanceState()方法实现，恢复也有响应的方式，简单优雅且不依赖于使用者;