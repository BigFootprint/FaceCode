---
title: Android的数据恢复(二)
date: 2015-10-17 10:58:26
tags: [源码]
categories: Android
---

## 背景
最近同事在开发中使用到FragmentStatePagerAdapter，在数据恢复的时候出现了非常奇怪的现象，因此研究了一下Fragment的数据恢复机制。

*本文主要解决以下问题*

1. Fragment如何进行数据恢复和保存？
2. Fragment到底是啥？
3. Fragment的使用注意事项

## Fragment如何进行数据恢复和保存
和Actiivty一样，Fragment也有自己的生命周期，并且与Activity的生命周期紧密绑定在一起，一般一个页面要做数据保存和恢复都会涉及到Fragment，我们在[Android的数据恢复（一）](http://muzileecoding.com/blog/2015/08/01/androidde-shu-ju-hui-fu/)中解释了Activity是如何保存状态的，那么Fragment是否一样呢？<!--more-->

[Android的数据恢复（一）](http://muzileecoding.com/blog/2015/08/01/androidde-shu-ju-hui-fu/)中说到: onSaveInstanceState方法是整个Activity保存数据的出发点，它的实现如下:

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
这段代码的3-6行就涉及到Fragment的数据保存。我们看到mFragment通过saveAllState方法返回一个Parcelable对象，并最终存入outState中去了，mFragments是什么呢？其声明类型为：FragmentManagerImpl。顾名思义，是FragmentManager的实现类型。
>【Note】读者请打开Android Studio，查看FragmentManager的源码，版本是support-v4。

由于FragmentManager的代码非常长，而且相互之间联系比较紧密，因此将标注过的代码传到了服务器：[FragmentManager.java](http://7xktd8.com1.z0.glb.clouddn.com/FragmentManager.java)、 [Fragment.java](http://7xktd8.com1.z0.glb.clouddn.com/Fragment.java)。
读者可以结合源码以及标注进行理解。建议就从FragmentManager.saveAllState()函数入手。
>【Note】代码很多非常长，有部分类也没有贴出来，但这两个类已经可以让读者知道Fragment的保存过程了。

这里重点阐述几件事情:

1.  一个Activity对应着一个FragmentManager，每一个被添加的Fragment都会被记录在mActive中，显示移除的时候会将该位置记录为null，mAdded则记录着当前添加到容器里面的Fragment(mActive记录这所有被add的Fragment，mAdded则记录着所有被add和attach的Fragment，attach后的Fragment会被FragmentManager管理，但不会在ViewTree中);
2.  保存状态的时候，状态保存是会遍历mActive里面的Fragment，同时也会以下标形式保存mAdded里面的元素，所有操作记录的BackStackRecord也会被保存下来：目的是可以恢复所有Active的Fragment的状态以及之前所做的Transaction动作。数据都被保存在FragmentManagerState中(此处分析的时候发现一个问题，从代码看，mActive的元素是mAdded元素的子集，当一个fragment detach、attach、detach一遍之后，保存mAdded下标就会crash);
3.  恢复的时候则按照保存的数据逐步恢复，很重要的一个动作是在Activity的onCreate方法中，这个方法有如下调用：  
```java
if (savedInstanceState != null) {
		Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
		mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
		                    ? mLastNonConfigurationInstances.fragments : null);
	}
mFragments.dispatchCreate();
```
第3行会restore保存的Fragment的状态，使变量赋值。第6行则会通知所有的Fragment进行onCreate()。注意restoreAllState这个函数第二个参数的判断，恢复Fragment发生在两种情况下：1)Config变化，最经典是屏幕旋转；2)saveInstanceState发生的时候。如果是前者就可以从mLastNonConfigurationInstances中拿出保存的fragments对象，否则就是null；
4.  Fragment恢复之后并不会替代你新建的Fragment，即：如果你在container里面加入了FragmentA，之后这个FragmentA实例被恢复重建，而你在Activity的onCreate方法中没有做任何检测再次新建FragmentA的实例并添加进入container，则会导致container有两个FragmentA的实例;

## Fragment到底是啥
要解答这个问题，只需要看FragmentManager中moveToState方法的下列代码片段：
```java
f.mView = f.performCreateView(f.getLayoutInflater(f.mSavedFragmentState), container, f.mSavedFragmentState);
if (f.mView != null) {
	f.mInnerView = f.mView;
	f.mView = NoSaveStateFrameLayout.wrap(f.mView);
	if (container != null) {
		Animation anim = loadAnimation(f, transit, true, transitionStyle);
		if (anim != null) {
			f.mView.startAnimation(anim);
		}
		container.addView(f.mView);
	}
	if (f.mHidden) 
		f.mView.setVisibility(View.GONE);
	f.onViewCreated(f.mView, f.mSavedFragmentState);
} else {
	f.mInnerView = null;
}
```
第1行就是回调Fragment的onCreateView方法，所以f.mView就是我们在onCreateView中返回的View，而第10行则将f.mView添加到container中去。第14行回调Fragment的onViewCreated方法。到这里就很清楚：Fragment就是一个包装了生命周期的View，真正显示的机制过程和View一致。读者还可以通过FragmentManager.hideFragment方法验证一下这个结论。

## Fragment的使用注意事项
Fragment只是一个带有生命周期的View。写代码的时候首先需要注意的一点是：  
往Container中添加Fragment的时候，需要去检测该Fragment是否存在，方法是getFragmentById和getFragmentByTag，否则在数据恢复的时候，就会造成多个相同的Fragment被创建。

其次，读者可以注意一下FragmentState这个类（用于保存Fragment的上下文环境以及重要参数，比如mIndex，mFragmentId，mContainerId，mClassName等），这个类里面还有一个非常重要的参数：mArguments。这个就是设置Fragment参数标准做法时所用到的，请见[Developers](http://developer.android.com/reference/android/app/Fragment.html)上的标准写法：
```java
public static DetailsFragment newInstance(int index) {
	DetailsFragment f = new DetailsFragment();
	// Supply index input as an argument.
	Bundle args = new Bundle();
	args.putInt("index", index);
	f.setArguments(args);
	return f;
}
```
为什么要这么写？或者说为什么这么写是标准的？因为通过Fragment.setArguments方法设置的bundle参数在数据保存的时候是会被保存下来的，恢复之后还能获取。而普通的带参构造函数并不能做到这一点。

## 总结
最近开发的时候和同事遇到不少Fragment的相关问题，因此稍微做了一下研究。Fragment里面还有很多东西没有仔细参研，后面有时间会进一步挖掘。下一篇博客会以FragmentStatePagerAdapter为主题，因为在这个上面也遇到很坑的问题。  
以上。