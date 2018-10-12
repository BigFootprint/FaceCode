---
title: Android View绘制过程
date: 2016-09-21 22:01:43
tags: [视图]
categories: Android
---

>__A View occupies a rectangular area on the screen and is responsible for drawing and event handling。__

## View 绘制模型
撇开 Android 系统不谈，我们想象一下自己要画一幅画会首先做哪些事情？首先得构图，我们得规划一下要画哪些东西，画在什么地方以及画多大，然后我们就能下笔成画了：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/viewdraw-mnlsh.jpg" alt="蒙娜丽莎的微笑"/></div>

<!--more-->那么 Android 系统是不是也这么机智呢？确实如此。Android 的绘制过程由`ViewRootImpl.performTraversals()`方法触发，这个方法有800多行，它调用了三个方法：

```Java
performMeasure()
performLayout()
performDraw()
```
这三个方法分别代表着测量尺寸、放置位置、进行绘制三个步骤。

在详细看这三个方法之前，我们需要先了解两件事情: 

1. Activity 内的 View 组织方式；
2. `setCotentView()`是如何工作的；

### Activity 内的 View 组织方式
如下图: 
<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/View层级.png" alt="View层级"/></div>

PhoneWindow 是 Window 的子类，根据注释，Window的作用是:

```Java
 /* Abstract base class for a top-level window look and behavior policy.  An
  * instance of this class should be used as the top-level view added to the
  * window manager. It provides standard UI policies such as a background, title
  * area, default key processing, etc.
  */
```
它负责组织一组 View，并实现它们的绘制显示方式以及对应的行为，来为 window 的最终显示提供支持。

__图中绿色部分和 View 无关，它们是负责包装 View 显示和行为的，而从 DecorView 开始，就是真正显示在手机屏幕上的 View 的根节点。蓝色部分就是我们通常使用`setContentView()`方法进行布局的地方。__

### `setCotentView()`是如何工作的
我们从`Activity.attach()`这个方法开始分析:

```Java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ...
}
```
这里在 Activity 中`new`了一个`PhoneWindow`对象。而当我们`setContentView()`的时候，实际调用的是：

```Java
public Window getWindow() {
	return mWindow;
}

public void setContentView(@LayoutRes int layoutResID) {
	getWindow().setContentView(layoutResID);
	initWindowDecorActionBar();
}
```
也就是说，这个 View/ID 最终被设置到了 mWindow 中去了，也就是 PhoneWindow，它的`setContentView()`方法是这样的：

```Java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } 
    ...
    
    mContentParent.addView(view, params);
    ...
}

private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(); //方法1
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); //方法2
        ...
    }
}

// 方法1实现
protected DecorView generateDecor() {
	return new DecorView(getContext(), -1);
}

// 方法2实现
protected ViewGroup generateLayout(DecorView decor) {
    ...
    int layoutResource;
    int features = getLocalFeatures();
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
    } else
    ...

    mDecor.startChanging();

    View in = mLayoutInflater.inflate(layoutResource, null);
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in;

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ...
    return contentParent;
}
```
到这里，我们往 decor 里面添加了一个 View，然后从 decor 中获取到了 contentParent, 办法是`findViewById(ID_ANDROID_CONTENT)`，那么这个 id 所代表的 View 是什么呢？这个要看到 layoutResource 所代表的资源，给这个变量赋值的时候会做很多的判断，值也有很多的可能性，其中一个值是：

```Java
layoutResource = R.layout.screen_simple;
```
而`R.layout.screen_simple`代表的资源如下:

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
是不是非常眼熟？我们经常会为 Activity 配置一些属性，比如无 ActionBar ，那就对应到不同的 Activity 模板，而这里的 FrameLayout 就是我们找到的 contentParent，我们通过`setContentView()`设置的 View 最终就是被添加到这个 View 中去了。

OK，下面我们来细细分析 View 的绘制三步骤。

## measure 过程
测量尺寸是使用方法`performMeasure()`:

```Java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
	Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
	try {
		mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	} finally {
		Trace.traceEnd(Trace.TRACE_TAG_VIEW);
	}
}
```
这里就看到这个方法其实是调用了 View 的`measure`方法：

```Java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        ...
    }
    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```
这里首先判断有没有设置`PFLAG_FORCE_LAYOUT`标志位，或者父 View 的尺寸和上次的有没有变化，如果有，则继续判断是否是`PFLAG_FORCE_LAYOUT`引起的，如果是，则忽略缓存的内容，调用`onMeasure()`进行测量，在方法的最后，会更新缓存在 mMeasureCache 中的内容。而`onMeasure()`方法的默认实现如下：

```Java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
			getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
这里计算宽高的两组方法是对应的，看一组即可，我们看看宽度的计算相关的方法实现：

```Java
protected int getSuggestedMinimumWidth() {
	return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
    
public static int getDefaultSize(int size, int measureSpec) {
	int result = size;
	int specMode = MeasureSpec.getMode(measureSpec);
	int specSize = MeasureSpec.getSize(measureSpec);

	switch (specMode) {
		case MeasureSpec.UNSPECIFIED:
			result = size;
			break;
		case MeasureSpec.AT_MOST:
		case MeasureSpec.EXACTLY:
			result = specSize;
			break;
	}
	return result;
}
```
首先看`getSuggestedMinimumWidth()`的计算，它得到的是 View 本身和它的背景允许的最小宽度的较大值。而至于`getDefaultSize()`方法，则是根据 sepcMode，决定是否采用前面的较大值，那么这几种 specMode 分别代表什么意思呢？[官网](https://developer.android.com/guide/topics/ui/how-android-draws.html)有详细的描述：
>A MeasureSpec can be in one of three modes:
>
>__UNSPECIFIED__: This is used by a parent to determine the desired dimension of a child View. For example, a LinearLayout may call measure() on its child with the height set to UNSPECIFIED and a width of EXACTLY 240 to find out how tall the child View wants to be given a width of 240 pixels.
>
>__EXACTLY:__ This is used by the parent to impose an exact size on the child. The child must use this size, and guarantee that all of its descendants will fit within this size.
>
>__AT MOST__: This is used by the parent to impose a maximum size on the child. The child must guarantee that it and all of its descendants will fit within this size.

简单来说，__UNSPECIFIED__ 是父 View 用来查看子 View 自己想要的尺寸的一种 Mode，它并没有详细说明尺寸的限制，View 根据自身的条件自己去判断即可。__EXACTLY__ 是父 View 用于对子 View 强制施加一个尺寸用的，子 View 必须保证使用这个尺寸，且所有的子 View 都能够适配于这个尺寸，__AT MOST__则是父 View 告诉子 View 一个最大可以使用的尺寸，子 View 必须保证不超过这个尺寸。
>EXACTLY 对应于match_parent和具体的数值这两种模式，而 AT_MOST 对应于 wrap_content。

因此父 View 在告诉子 View 测量尺寸的时候，不仅会告诉它自己的尺寸是多少，还会告诉它自己对这个尺寸的一个要求，是要严格遵守，还是自行考量。在测量完尺寸之后，需要调用`setMeasuredDimension()`方法进行设置:

```Java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```
设置的时候其实就是将值赋值给 mMeasuredWidth 和 mMeasuredHeight，而这两个值正是`getMeasuredWidth()`和`getMeasuredHeight()`这两个方法的返回值。因此只有在测量完毕之后，这两个方法才能返回正确的有效值。

以上是 View 的 measure 过程，下面看一下 ViewGroup 的。ViewGroup 的测量与三个方法相关：`measureChildren()`、`measureChild()`和`measureChildWithMargins()`。一个个来看:

```Java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```
这个方法很简单：循环遍历所有的子 View ，只要可见性不是 GONE 的，就调用`measureChild()`:

```Java
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
这个会计算传递给子 View `measure()` 的宽高，调用的方法是`getChildMeasureSpec()`:

```Java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
这个方法比较长，但是细看并不难，可以看到我们平时在添加子 View 的时候，使用的 LayoutParams 对布局的影响。`measureChildWithMargins()`方法就不分析了，它只是把 margin 也从 View 的空间中扣除了。
>由此可见，一个 View 的 MeasureSpec 由父布局 MeasureSpec 和自身的 LayoutParams 共同产生。

虽然 ViewGroup 有这样一组与测量有关的方法，但是 framework 层面好像并没有去使用，像 FrameLayout、LinearLayout 等都是自己重载了`onMeasure()`，并在这个方法中按照自己的逻辑完成了子 View 的测量。

以上，就是 View 的测量过程。

## layout 过程
View 的 layout 过程起始于 ViewRootImpl 的 `performLayout`方法: 

```Java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
    int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;

    final View host = mView;
    try {
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        ...
    } finally {
        ...
    }
    mInLayout = false;
}
```
这个方法里面调用了一个很重要的方法，即 View 的`layout`方法: 

```Java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```
这里首先通过`setOpticalFrame()`或者`setFrame()`方法判断边界有没有变化，这两个方法最终都调用到了`setFrame()`方法:

```Java
/**
 * Assign a size and position to this view.
 *
 * This is called from layout.
 *
 * @param left Left position, relative to parent
 * @param top Top position, relative to parent
 * @param right Right position, relative to parent
 * @param bottom Bottom position, relative to parent
 * @return true if the new size and position are different than the
 *         previous ones
 * {@hide}
 */
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    // 如果与之前的值不一致，就说明 View 的边界变化了
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;

        // Remember our drawn bit
        int drawn = mPrivateFlags & PFLAG_DRAWN;

        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        // sizeChanged 表示大小有没有变化
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // Invalidate our old position
        invalidate(sizeChanged);

        // 重新赋值
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

        mPrivateFlags |= PFLAG_HAS_BOUNDS;

        // 回调
        if (sizeChanged) {
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }

        if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
            // If we are visible, force the DRAWN bit to on so that
            // this invalidate will go through (at least to our parent).
            // This is because someone may have invalidated this view
            // before this call to setFrame came in, thereby clearing
            // the DRAWN bit.
            mPrivateFlags |= PFLAG_DRAWN;
            invalidate(sizeChanged);
            // parent display list may need to be recreated based on a change in the bounds
            // of any child
            invalidateParentCaches();
        }

        // Reset drawn bit to original value (invalidate turns it off)
        mPrivateFlags |= drawn;

        mBackgroundSizeChanged = true;
        if (mForegroundInfo != null) {
            mForegroundInfo.mBoundsChanged = true;
        }

        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
    return changed;
}
```
重点已经标注在代码里面了，判断的依据就是和原有的旧值相比较。如果变化了或者是设置了`PFLAG_LAYOUT_REQUIRED`标记，就调用`onLayout()`方法。因此 View 的`onLayout()`方法虽然会被调用，但是只要它的上下左右的位置不变，则`changed`参数就为 false。我们来看一下`onLayout()`方法：

```Java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```
这是一个空方法，而在 ViewGroup 里面，方法则是:

```Java
protected abstract void onLayout(boolean changed,
		int l, int t, int r, int b);
```
它是一个抽象方法。那么到底 ViewGroup 会如何去使用这些参数呢？可以看一个简单的例子: FrameLayout。

```Java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}

void layoutChildren(int left, int top, int right, int bottom,
                            boolean forceLeftGravity) {
    final int count = getChildCount();

    // 1. 计算 FrameLayout 上下左右的边界位置
    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    // 2. 遍历所有的子 View 
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            // 获取之前子 View 测量出来的尺寸
            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }

            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

            // 3. 根据 ViewGroup 设置的属性，比如 gravity，计算子 View 的上下左右边界位置
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }

            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }

            // 4. 传递给子 View 它的位置
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```
关键的四步已经标注在代码中了，第 4 步之后，就回到我们前面分析的 View 的`setFrame()`方法，View 会记录下 mLeft/mTop/mRight/mBottom 四个值。

以上，就是 View 的 layout 过程。

## draw 过程
经过前面两个过程，我们已经知道了要在什么位置画多大的一个 View ，接下去就是要开始绘制了，起始当然是`performDraw()`方法:

```Java
private void performDraw() {
    ...
    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
    try {
        draw(fullRedrawNeeded);
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```
这里调用了`draw()`方法:

```Java
private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    if (!surface.isValid()) {
        return;
    }
    ···
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
            ...
            mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
        } else {
            // If we get here with a disabled & requested hardware renderer, something went
            // wrong (an invalidate posted right before we destroyed the hardware surface
            // for instance) so we should just bail out. Locking the surface with software
            // rendering at this point would lock it forever and prevent hardware renderer
            // from doing its job when it comes back.
            // Before we request a new frame we must however attempt to reinitiliaze the
            // hardware renderer if it's in requested state. This would happen after an
            // eglTerminate() for instance.
            ...
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
}
```
这里会去判断是否是开启了硬件加速，如果没有，就调用`drawSoftware()`方法:

```Java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {

    // Draw with software renderer.
    final Canvas canvas;
    try {
        final int left = dirty.left;
        final int top = dirty.top;
        final int right = dirty.right;
        final int bottom = dirty.bottom;

        canvas = mSurface.lockCanvas(dirty);

        // The dirty rectangle can be modified by Surface.lockCanvas()
        //noinspection ConstantConditions
        if (left != dirty.left || top != dirty.top || right != dirty.right
                || bottom != dirty.bottom) {
            attachInfo.mIgnoreDirtyState = true;
        }

        // TODO: Do this in native
        canvas.setDensity(mDensity);
    } catch (Surface.OutOfResourcesException e) {
        handleOutOfResourcesException(e);
        return false;
    } catch (IllegalArgumentException e) {
        ...
        return false;
    }

    try {
        // If this bitmap's format includes an alpha channel, we
        // need to clear it before drawing so that the child will
        // properly re-composite its drawing on a transparent
        // background. This automatically respects the clip/dirty region
        // or
        // If we are applying an offset, we need to clear the area
        // where the offset doesn't appear to avoid having garbage
        // left in the blank areas.
        if (!canvas.isOpaque() || yoff != 0 || xoff != 0) {
            canvas.drawColor(0, PorterDuff.Mode.CLEAR);
        }

        dirty.setEmpty();
        mIsAnimating = false;
        mView.mPrivateFlags |= View.PFLAG_DRAWN;

        canvas.translate(-xoff, -yoff);
        if (mTranslator != null) {
            mTranslator.translateCanvas(canvas);
        }
        canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
        attachInfo.mSetIgnoreDirtyState = false;
        // ！！！
        mView.draw(canvas);
        drawAccessibilityFocusedDrawableIfNeeded(canvas);    
    } finally {
        try {
            surface.unlockCanvasAndPost(canvas);
        } catch (IllegalArgumentException e) {
            ...
            return false;
        }
    }
    return true;
}
```
这个方法锁定了 Surface 上的一块 Canvas 区域，然后调用了 View 的`draw()`方法:

```Java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     * 1. Draw the background
     * 2. If necessary, save the canvas' layers to prepare for fading
     * 3. Draw view's content
     * 4. Draw children
     * 5. If necessary, draw the fading edges and restore layers
     * 6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // we're done...
        return;
    }
    ...
}
```
这个方法的注释非常清楚，绘制过程主要分为 6 个步骤:

1. 绘制背景；
2. 判断是否要显示横向或者纵向的fading edge;
3. 绘制自身的内容；
4. 绘制子View；
5. 如果需要绘制fading edge，则在这一步进行绘制；
6. 绘制装饰品，比如滚动条等；

2 和 5会尽可能的省略，因为不是所有的 View 都需要绘制。`draw()`方法会根据有没有fading edge 选择两条不同的逻辑进行绘制，代码片段中展示的是简单的一条，在这个过程中，step 3，也就是绘制自身内容这块，它调用了另外一个重要的方法，即`onDraw()`:

```Java
protected void onDraw(Canvas canvas) {
}
```
在 View 类中，这个方法是一个空方法，因此它不会绘制任何内容，读者可以去看一下 TextView 等子类关于该方法的实现。而 step 4调用的另外一个方法`dispatchDraw()`也是一个空方法，在 ViewGroup 中会有相应的实现。

以上，就是 View 的绘制过程。

## 一些问题
### `ViewRootImpl.performTraversals()`是如何被调用的？
这个问题要到[Activity启动过程](https://www.muzileecoding.com/framework/activity-launch-progress.html)一文中解答。

### `getWidth()/getHeight()`和`getMeasuredWidth()/getMeasuredHeight()`两组方法有什么区别和联系？
两组方法的实现分别如下:

```Java
// 第一组
public final int getWidth() {
	return mRight - mLeft;
}

public final int getHeight() {
	return mBottom - mTop;
}

// 第二组
public final int getMeasuredWidth() {
	return mMeasuredWidth & MEASURED_SIZE_MASK;
}

public final int getMeasuredHeight() {
	return mMeasuredHeight & MEASURED_SIZE_MASK;
}
```
因此我们只需要看看这些变量什么时候被赋值就好了，根据前面的分析： mRight/mLeft/mBottom/mTop 四个变量是在 layout 之后才被赋值（`setFrame()`方法体内），因此之后在 layout 之后，这组方法才会有效。而 mMeasuredWidth/mMeasuredHeight 是在`onMeasure()`方法中调用`setMeasuredDimension()`设置后才有值的，因此这组方法是在 measure 过程完成之后就有效了。

这两组值可能不一样，也可能一样，具体要看在实际 layout 的过程中有没有遵循 measure 过程中设置好的限制。

### `invalidate()`/`postInvalidate()`/`requestLayout`
这三个方法出现的背景是：一个页面已经绘制好了，如果因为一些原因它的位置、尺寸、外观需要发生变化，比如一个 View 正在执行一个动画，那就需要 View 自己能够主动请求重新计算、展现这些变化。

#### `invalidate()`和`postInvalidate()`
`postInvalidate()`方法的实现如下:

```Java
/**
 * <p>Cause an invalidate to happen on a subsequent cycle through the event loop.
 * Use this to invalidate the View from a non-UI thread.</p>
 *
 * <p>This method can be invoked from outside of the UI thread
 * only when this View is attached to a window.</p>
 *
 * @see #invalidate()
 * @see #postInvalidateDelayed(long)
 */
public void postInvalidate() {
    postInvalidateDelayed(0);
}

public void postInvalidateDelayed(long delayMilliseconds) {
    // We try only with the AttachInfo because there's no point in invalidating
    // if we are not attached to our window
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}
```
从注释看，这个方法是为了从非-UI 线程来刷新 View，最终调用到的是`ViewRootImpl.dispatchInvalidateDelayed()`方法:

```Java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
	Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
	mHandler.sendMessageDelayed(msg, delayMilliseconds);
}
```
这里产生了一个`MSG_INVALIDATE`消息，它的处理如下:

```Java
final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
        case MSG_INVALIDATE:
            ((View) msg.obj).invalidate();
            break;
        ...
        }
    }
}
```
很清楚，这里是通过主线程的 Handler 将 invalidate 消息调度到至主线程。因此，__`invalidate()`和`postInvalidate()`的区别就是前者是在主线程调用，后者可以从非主线程调度__。

接着我们来看`invalidate()`和`requestLayout`各自所起的作用。

#### `requestLayout`方法
```Java
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    ...
    // 设置两个标志位
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    // 调用父 ViewGroup 的 requestLayout() 方法
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```
它设置了两个标志位，之后调用父 View 的`requestLayout()`方法，根据我们前面分析的 Activity 的 View 结构，这个调用会层层向上，会到哪里呢？最终会调用到 ViewRootImpl 的`requestLayout()`方法:

```Java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```
这里会设置 mLayoutRequested 标志位，并调用`scheduleTraversals()`，这个方法又会调用到我们前面分析的`performTraversals()`方法：

```Java
private void performTraversals() {
    ...
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        ...
        // Ask host how big it wants to be
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    ...
    // 成立
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout || mAttachInfo.mRecomputeGlobalAttributes;
	if (didLayout) {
		performLayout(lp, desiredWindowWidth, desiredWindowHeight);
		...
	}
	...  
}
```
因为前面将 mLayoutRequested 置为 true，因此一定会调用`measureHierarchy()`方法:

```Java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    ...
    boolean goodMeasure = false;
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
        ....
        if (baseSize != 0 && desiredWindowWidth > baseSize) {
            ...
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            ...
            if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                goodMeasure = true;
            } else {
                ...
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    if (DEBUG_DIALOG) Log.v(TAG, "Good!");
                    goodMeasure = true;
                }
            }
        }
    }

    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
    ...
}
```
在这个代码片段中，肯定会调用`performMeasure()`方法：

```Java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
        widthMeasureSpec != mOldWidthMeasureSpec ||
        heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();
        // 设置了 PFLAG_FORCE_LAYOUT，cacheIndex 为 -1
        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        ...
        // 设置标志位：PFLAG_LAYOUT_REQUIRED
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```
前面调用`requestLayout()`方法的时候，我们设置了`PFLAG_FORCE_LAYOUT`标志位，因此这里会执行到`onMeasure()`方法。也就是说会进行测量过程。

我们再回到`performTraversals()`方法，接下去会调用`performLayout()`方法，这个方法前面分析过了:

```Java
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    // 如果设置了 PFLAG_LAYOUT_REQUIRED 标志位，则进行 onLayout
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```
因为在 measure 过程中设置了`PFLAG_LAYOUT_REQUIRED`标志位，所以这里会重新设置位置。而`performDraw()`方法虽然有触发条件，但是如果 View 的尺寸和位置变化了，一定会触发该方法的调用。

#### `invalidate()`方法
这个方法其实有一些重载方法：

```Java
public void invalidate() {
    invalidate(true);
}

void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    if (mGhostView != null) {
        mGhostView.invalidate(true);
        return;
    }

    if (skipInvalidate()) {
        return;
    }

    // 重绘执行的条件
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        if (fullInvalidate) {
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~PFLAG_DRAWN;
        }

        // 设置标志位
        mPrivateFlags |= PFLAG_DIRTY;

        // 判断成立
        if (invalidateCache) {
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }

        // Propagate the damage rectangle to the parent view.
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;

        // 调用父 View 的 invalidateChild 方法
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            p.invalidateChild(this, damage);
        }

        // Damage the entire projection receiver, if necessary.
        if (mBackground != null && mBackground.isProjected()) {
            final View receiver = getProjectionReceiver();
            if (receiver != null) {
                receiver.damageInParent();
            }
        }

        // Damage the entire IsolatedZVolume receiving this view's shadow.
        if (isHardwareAccelerated() && getZ() != 0) {
            damageShadowReceiver();
        }
    }
}
```
如注释所说，最终调用了父 ViewParent 的`invalidateChild()`方法，并把当前 View 的范围默认传递给了该方法:

```Java
/**
 * Don't call or override this method. It is used for the implementation of
 * the view hierarchy.
 */
public final void invalidateChild(View child, final Rect dirty) {
    // 把自身赋值给 parent
    ViewParent parent = this;

    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        ...

        final int[] location = attachInfo.mInvalidateChildLocation;
        location[CHILD_LEFT_INDEX] = child.mLeft;
        location[CHILD_TOP_INDEX] = child.mTop;
        ...
        // while 循环
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }

            ...
            parent = parent.invalidateChildInParent(location, dirty);
            if (view != null) {
                // Account for transform on current parent
                Matrix m = view.getMatrix();
                if (!m.isIdentity()) {
                    RectF boundingRect = attachInfo.mTmpTransformRect;
                    boundingRect.set(dirty);
                    m.mapRect(boundingRect);
                    dirty.set((int) (boundingRect.left - 0.5f),
                            (int) (boundingRect.top - 0.5f),
                            (int) (boundingRect.right + 0.5f),
                            (int) (boundingRect.bottom + 0.5f));
                }
            }
        } while (parent != null);
    }
}
```
这里会不停的调用 parent 的`invalidateChildInParent()`方法，这个方法一直调用，会调用到`ViewRootImpl.invalidateRectOnScreen()`方法:

```Java
private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;
    if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
        mAttachInfo.mSetIgnoreDirtyState = true;
        mAttachInfo.mIgnoreDirtyState = true;
    }

    // Add the new dirty rect to the current one
    localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
    // Intersect with the bounds of the window to skip
    // updates that lie outside of the visible region
    final float appScale = mAttachInfo.mApplicationScale;
    final boolean intersected = localDirty.intersect(0, 0,
            (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
    if (!intersected) {
        localDirty.setEmpty();
    }
    // 成立
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        scheduleTraversals();
    }
}
```
因此这里还是去调用了`scheduleTraversals()`方法，不同的是，这次没有设置 mLayoutRequested 为 true，因此不会调用`performLayout()`方法，也不会去调用`measureHierarchy()`方法。
>关于这一点论断，读者可以去看这两个方法的调用处，都和 mLayoutRequested 相关。

因此只会直接去调用`performDraw()`方法。

#### 方法总结
下面这幅图出自 Google+ ：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/android-view-method.webp" height="400" alt="android-view-method"/></div>

它很好的总结了相关过程，指出了`invalidate()`和`requestLayout()`方法的区别。

## 总结
View 的绘制过程一如我们自己作画的过程一样，抽象出来无非是确认 View 的大小，设置 View 的位置，绘制 View 的内容三个步骤，前面的分析能够让读者对绘制的过程有一个粗略的了解。

结合[Android View事件处理](https://www.muzileecoding.com/android/Android-touchevent-process.html)，我们还能得出一个结论：__View 的绘制过程无非是把我们想要的 View 界面绘制在 Surface 的 Canvas 上并最终显示到屏幕上，最终的显示和我们屏幕截图下来的图片本质上是没有区别的。它的奇特之处就在于它能接受事件并作出反应，而这是通过底层的View 对象来实现的。__

然而，贴出来分析的代码是经过删减的，读者如果自己去看方法体，会发现过程复杂的多，就比如我们最开始分析的`ViewRootImpl.performTraversals()`方法，它并不是简简单单的就调用了这三个方法，甚至也不是调用了一次。但这并不妨碍我们去了解这样一个过程，当我们以后遇到问题，至少可以有一个基本的概念去猜测一下问题点。