---
title: Android View事件处理
date: 2016-09-04 17:33:43
tags: [事件处理]
categories: Android
---

## View 模型 & 事件模型

### View 模型
如图所示，是我们日常开发布局所接触的 View 模型：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/Android-View模型.png" alt="Android-View模型"/></div>

简单来说，就是 View 和 ViewGroup 的嵌套组合。ViewGroup 是一种特殊的 View，特殊之处就在于它可以容纳并按照一定的规则排列View，此时的 View 被称为子 View，常见的 View 有 Button、ImageView 等，常见的 ViewGroup 有 LinearLayout、Relativelayout等，Android 开发者应该非常熟悉。<!--more-->

### 事件模型
本文要讨论的是这样一组 View 模型对事件的处理过程。什么是事件？我们触摸屏幕，从我们的手指开始接触屏幕到手指离开屏幕，手指与屏幕之间的交互都被抽象成连续的事件，用于描述一个触摸过程。事件主要分为三类：

1. ACTION_DOWN：Down事件，也就是我们落下手指的事件，它是一个事件流的开始；
2. ACTION_MOVE：Move事件，我们手指按在屏幕上，不会一直不动，比如我们玩切水果游戏，手指会飞快的划过页面，这样就产生了一系列的Move事件，通过Move事件，可以构造出手指移动的路径；
3. ACTION_UP：Up事件，也就是手指拿起的事件；

这三类事件串起来可以描述我们绝大部分的触屏操作：ACTION_DOWN, ACTION_MOVE,ACTION_MOVE...ACTION_MOVE, ACTION_UP。那么这样一组事件流是如何分发到 View 模型中去处理的呢？下面详细分析。

## View的事件处理
### 事件分发——`dispatchTouchEvent`
View 的事件处理是从`dispatchTouchEvent()`方法开始的，View中这个方法比较长，我们抽重点看：

```Java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;

    ...
    final int actionMasked = event.getActionMasked();
    ...

    // 1 过滤
    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        // 2
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        
        // 3
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ...

    return result;
}
```
如上，核心代码就这一段。在事件处理前，首先要做一下安全过滤，`onFilterTouchEventForSecurity()`方法的定义如下:

```Java
/**
 * Filter the touch event to apply security policies.
 *
 * @param event The motion event to be filtered.
 * @return True if the event should be dispatched, false if the event should be dropped.
 *
 * @see #getFilterTouchesWhenObscured
 */
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    //noinspection RedundantIfStatement
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
            && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // Window is obscured, drop this touch.
        return false;
    }
    return true;
}
```
如注释所说，这个过滤是为了一些安全策略，有兴趣的读者可以去查看一下`FILTER_TOUCHES_WHEN_OBSCURED`和`FLAG_WINDOW_IS_OBSCURED`两个变量的注释，尤其是后者的，可以大致了解一下这里的安全指的是什么。这里我们可以当它是返回的 true 来继续往下分析 2 处。分析 2 之前，我们先看几个方法:

```Java
public void setOnTouchListener(OnTouchListener l) {
	getListenerInfo().mOnTouchListener = l;
}
    
ListenerInfo getListenerInfo() {
	if (mListenerInfo != null) {
		return mListenerInfo;
	}
	mListenerInfo = new ListenerInfo();
	return mListenerInfo;
}
```
这两个方法指明了`mListenerInfo`中`mOnTouchListener`的来源，也就是我们平时开发中通过调用`setOnTouchListener()`为View设置的`OnTouchListener`监听，因此 2 处的判断的含义是如果 View 满足以下三个条件：

1. 设置了`OnTouchListener`监听；
2. View 的状态是 enable 的；
3. 调用`OnTouchListener`的`onTouch`方法返回的是true；

那么`dispatchTouchEvent()`方法就会返回true，否则`result`为 false，会继续执行 3 处，也就是执行自己的`onTouchEvent()`方法。以下是处理流程图：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/View事件处理流程图.png" width="600" alt="View事件处理流程图"/></div>

### 事件处理——`onTouchEvent`
下面我们来重点看一下`onTouchEvent`方法:

```Java
public boolean onTouchEvent(MotionEvent event) {
    // 【注意】这里调用的是getX()和getY()，不加任何的参数！
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    // 1 如果 View 是 DISABLED
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        // 如果是 UP 事件，并且当前 View 处于 PRESSED 状态
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            // 取消 PRESSED 状态
            setPressed(false);
        }
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
    }

    // 2 如果设置了 mTouchDelegate，先调用它的 onTouchEvent 方法
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    // 3 如果 View 可点击
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // 4
                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    // 5 延迟一段时间再去Check
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else { 
                    // 6 直接设置 PRESSED 状态，并 Check LongClick
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(0);
                }
                break;
            case MotionEvent.ACTION_MOVE:
                drawableHotspotChanged(x, y);

                // 7 判断是否移出了 View 的边界
                if (!pointInView(x, y, mTouchSlop)) {
                    // 移出去了就移除 Tap 任务
                    removeTapCallback();
                    // 如果 Tap 任务已经执行
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // 移除 LongPress任务
                        removeLongPressCallback();
                        // 取消 PRESSED 状态
                        setPressed(false);
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                // 8 如果抬起的时候，View处于 PRESSED 状态或者处于 PRESSED 确认期
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    // 获取焦点
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    // 如果处于 PRESSED 确认期，此时确认 PRESSED
                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                    }

                    // 9 如果此时没有进入长按 (另外一个变量不需要care)
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        // 不要再去 Check 长按事件了
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        // 10 判断是否处于焦点状态, 不处于则执行 PerformClick 任务
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                    // 取消 PRESSED 状态
                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }

                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;
            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                break;            
            }
        return true;
    }
    return false;
}
```
代码中几个 case 的处理调换了以下顺序，以便于分析。代码总共做了 10 处注释，基本可以理清楚这段代码的脉络。下面对其中几处做一下补充。

#### getX()/getY()与getX(int)/getY(int)
在方法执行的一开始我就标注了一个__【注意】__，这是一个很细节的地方，这两组方法的区别读者可以自己去查 API 文档，注意到它们的不同就可以解释下面的现象：

__有一个 View 是Clickable的，如果 View 上有两根手指，假设 A 是先按下的，B 是后按下的，那么如果 B 移出 View 边界，A 抬起依然可以触发 Click 事件，但是如果 A 先移出边界，B 后抬起，则不能触发 Click 事件。__(从实践结果来看，虽然 A 和 B 触发的都是 Move 事件，但是实际上 B 的 Move 事件的坐标并不是绝对坐标，而就是 A 的坐标，只有当 A 抬起之后，B 触发的才会恢复自己的绝对坐标。)

#### VIEW 的 CLICKABLE
1 和 3 处合起来可以得出一个结论：一个 View 只要是 `CLICKABLE || LONG_CLICKABLE || CONTEXT_CLICKABLE` 的，它就能消费事件，与是否 DISABLE 无关，只不过 DISABLE 的时候不会做出响应。

View 在没有任何设置的情况下，默认 CLICKABLE 是 false 的，但是可以通过 xml 中设置 `android:clickable="true"`或者调用`setClickable(boolean)`来更改，或者当我们调用`setOnClickListener()`的时候，这个标志位也会被默认设置：

```Java
public void setOnClickListener(@Nullable OnClickListener l) {
	if (!isClickable()) {
		setClickable(true);
	}
	getListenerInfo().mOnClickListener = l;
}
```

#### 手势延迟判断
4 处在处理事件的时候，首先要判断自身是不是处于一个可以滚动的容器中，这个方法其实是调用的 ViewGroup 的`shouldDelayChildPressedState()`来判断的，查看了几个常见的 ViewGroup，其中比如 LinearLayout、FrameLayout 等，这个方法返回的就是 false，而像 ScrollView、ListView 返回的就是 true，ViewGroup 中默认返回的就是true。

那么为什么需要做这个判断呢？因为在可滚动的容器中，DOWN 事件发生的时候是无法判断用户的用意到底是点击还是滚动的，因此这里只回去设置一个 `PFLAG_PREPRESSED` 状态，并且启动了一个 `CheckForTap` 任务，延迟 `ViewConfiguration.getTapTimeout()` 再执行；否则就直接设置 Pressed 状态，并且添加一个 `CheckForLongPress` 任务。 

#### 手势识别任务
上一小节提到了两个任务：`CheckForTap` 和 `CheckForLongPress`。下面细看看这两个任务具体做什么。

##### CheckForTap
```Java
private final class CheckForTap implements Runnable {
	public float x;
	public float y;

	@Override
	public void run() {
		mPrivateFlags &= ~PFLAG_PREPRESSED;
		setPressed(true, x, y);
		checkForLongClick(ViewConfiguration.getTapTimeout());
	}
}
```
这个任务一旦执行，就说明用户触摸的一个 View 的时间已经到了可以确认为 Tap 的时长了，所以首先要做的就是取消 `PFLAG_PREPRESSED`状态，并设置 PRESSED 状态，至此整个 View 确实进入到了 PRESSED 状态。然后它会调用`checkForLongClick()`方法：

```Java
private void checkForLongClick(int delayOffset) {
	if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
		mHasPerformedLongPress = false;

		if (mPendingCheckForLongPress == null) {
			mPendingCheckForLongPress = new CheckForLongPress();
		}
		
		mPendingCheckForLongPress.rememberWindowAttachCount();
		postDelayed(mPendingCheckForLongPress,
				ViewConfiguration.getLongPressTimeout() - delayOffset);
	}
}
```
这个方法的功能一目了然：如果 View 是`LONG_CLICKABLE`的，就发起一个`CheckForLongPress`任务。要注意延迟时间：如果是`CheckForTap`发起的，就需要减去之前等待确认 Tap 动作的时间，如果是 DOWN 事件直接触发的，那就需要等待`ViewConfiguration.getLongPressTimeout()`时长。

##### CheckForLongPress

```Java
private final class CheckForLongPress implements Runnable {
	private int mOriginalWindowAttachCount;

	@Override
	public void run() {
		if (isPressed() && (mParent != null)
				&& mOriginalWindowAttachCount == mWindowAttachCount) {
			if (performLongClick()) {
				mHasPerformedLongPress = true; // 置位
			}
		}
	}

	public void rememberWindowAttachCount() {
		mOriginalWindowAttachCount = mWindowAttachCount;
	}
}
```
这个方法判断如果当前 View 处于 Pressed 状态，那么就执行`performLongClick()`方法，这个方法会回调设置的`OnLongClickListener`监听，具体可以查看`performLongClick()`方法，这个方法如果返回 true，则会设置标志位。一般来说，如果我们设置了`OnLongClickListener`，这里就该返回 true。

#### 边界问题
7 处主要处理的是边界问题，如果 View 移出了边界，那么首先要移除`CheckForTap`任务。根据前面的分析，只有当 View 处于可滚动容器的时候，才会添加这个任务。移出之后查看 View 是否还处于 PFLAG_PRESSED 状态，如果存在，说明 View 是以下两种情况之一：

1. View 的 Down 事件走的是④-⑤路线，并且`CheckForTap`任务已经执行了，这个时候任务队列中很可能有`CheckForLongPress`任务；
2. View 的 Down 事件走的是④-⑥路线，队列中肯定有一个`CheckForLongPress`任务；

因此接着要做的事情就是移除`CheckForLongPress`任务，并且取消 Pressed 状态。

#### 焦点
8 处的处理和焦点相关，如果我们没有移出边界，那么很显然此时肯定处于 PRESSED 或者 PFLAG_PREPRESSED 状态。View 此时要尝试去拿一下焦点。默认来说，Touch Mode 下是无法拿到焦点的，因此 `focusTaken` 肯定为 false。但是如果我们把 View 的 `FOCUSABLE_MASK` 和 `FOCUSABLE_IN_TOUCH_MODE` 标志位都设置了（通过代码和XML都可以），那么 View 就可以在 Touch Mode 下获取焦点，也就是说 `focusTaken` 在第一次 Up 事件的时候，会被置为 true ，而第二次因为 `isFocused()` 返回 true（第一次Up的时候因为执行`requestFocus()`会获取焦点，所以第二次就会返回true）而被置为 false。这个务必注意。

9 除首先判断是否执行了 `CheckForLongPress`，如果没有执行，那么移除任务，判断 View 是否在 Up 处理中获取到了焦点，如果没有，就会执行`PerformClick`任务，这个任务从名字看就很眼熟，它执行的就是下面这个方法：

```Java
public boolean performClick() {
	final boolean result;
	final ListenerInfo li = mListenerInfo;
	if (li != null && li.mOnClickListener != null) {
		playSoundEffect(SoundEffectConstants.CLICK);
		li.mOnClickListener.onClick(this);
		result = true;
	} else {
		result = false;
	}

	sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
	return result;
}
```
也就是我们经常为 View 设置的点击监听。

> __Google 官方不建议设置 `FOCUSABLE_MASK` 和 `FOCUSABLE_IN_TOUCH_MODE` 的理由也在这里，10 处很明显的展示了打开这两个标志位的后果：第一次 Click 的时候，`View.OnClickListener`方法不会被执行__。相关的资料可以看这篇文章：[Touch Mode](http://android-developers.blogspot.jp/2008/12/touch-mode.html)。

## ViewGroup的事件处理
分析完单个 View 的事件处理流程，接下来就是要分析 ViewGroup 的事件处理流程，首先要明白的事情是 ViewGroup 也是一种 View，因此它和 View 一样，也具备消费事件的能力，所不同的是：ViewGroup 可能拥有很多个子 View，它要决定事件分发的策略以便于事件能够分发到合适的子 View 去处理。因此虽然 ViewGroup 和 View 一样都是从 `onTouchEvent`方法开始事件处理，但是行为上是完全不一样的。

### ViewGroup 的`dispatchTouchEvent`
相比较于 View 的`dispatchTouchEvent`方法，ViewGroup 的`dispatchTouchEvent`方法要复杂的多：

```Java
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 1 初始化 ViewGroup 的事件处理状态
        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
        
        // 2 判断是否要拦截事件
        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        ...

        // @@ 查看事件是否需要取消
        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        // 3.0 以上，这里默认置位
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;

        // Down 事件的处理流程
        if (!canceled && !intercepted) {
            ...

            // 3 如果是 Down 事件或者在允许多指触摸的时候
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                // 多指触摸
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    // 4 默认情况下这个列表就是 View 添加的顺序
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    // 5 倒叙遍历所有的子 View，这样最后添加的，在最上面绘制的 View 会首先被分派事件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        ...

                        // 6 判断 View 能否接受 Touch 事件，并且触摸点是否在本 View 内部
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        // 7 查看目标 View 是否存在在 TargetView 链表中，如果在，则说明已经处理过 Down
                        // 事件，不需要再去处理一次（想象一下两根手指同时触摸一个 View），只需要记录一下第二
                        // 次触摸的 PoniterId
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        // 8 分发事件给子 View 或者自己，这个方法会重点分析，它是 ViewGroup 事件处理的核心
                        // 如果返回true，表示事件被成功消费
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 把 TargetView 添加到链表中去，并且指明它是处理哪些 PointerId 的
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 置位
                            alreadyDispatchedToNewTouchTarget = true;
                            // 这个时候一个 Down 事件已经找到目标 View 去消费，后续就不用再继续找了
                            break;
                        }
                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }

                // 9 如果没找到处理对象，就把这个 PointerId 赋值给最早添加的 TargetView，【Why】
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // 10 所有类型事件的处理流程！
        // 这里如果没有找到任何一个目标 View，即这个 ViewGroup 里面的所有
        // View都不消费事件，则调用 dispatchTransformedTouchEvent 处理，这个后面重点说
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                // 针对 Down 事件，前面已经找到 View 处理过了
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    // 11 判断事件是否要取消，以便于分发正确类型的事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    // 如果是 CANCEL 事件，就把 TargetView 从链表中删除掉
                    if (cancelChild) {
                        // 说明是 TargetView 链表中的第一个
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                // 至少循环一次才会赋值
                predecessor = target;
                target = next;
            }
        }

        // 12 针对 Up 事件做的处理 —— 重置状态
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    ...

    return handled;
}
```
这么长的代码，只能在代码中进行注释，大概的流程是比较清楚的：

下面还是分小点一一补充。

#### `cancelAndClearTouchTargets`事件清理
在 1 处，一旦发现事件类型是 Down 类型，首先要做的就是事件清理，因为 Down 事件是一个事件流的开始，ViewGroup 一旦接收到该事件，则意味着应该初始化状态来准备处理一个新的事件流。我们来看看具体是怎么清理的：

```Java
/**
 * Cancels and clears all touch targets.
 */
private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        if (event == null) {
            final long now = SystemClock.uptimeMillis();
            event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            syntheticEvent = true;
        }

        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            resetCancelNextUpFlag(target.child);
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
        }
        clearTouchTargets();

        if (syntheticEvent) {
            event.recycle();
        }
    }
}
```
如果传入的 event 为 Null，那么会 Mock 一个 CANCEL 事件出来，否则会使用原 Event，尽量保留原始信息。然后遍历所有的 TargetView，重置状态并分发事件，这分为两步。下面会详细分析。状态重置之后，会调用`clearTouchTargets()`方法把所有的 TargetView清除掉。这样就完成了初始化的过程，总结一下核心就是两步：

1. 之前正在消费事件的View，全部发送一次 CANCEL 事件；
2. 记录消费事件的View链，清空；

##### `resetCancelNextUpFlag`
这个方法在很多地方都被调用，这里统一解释一下：

```Java
/**
 * Resets the cancel next up flag.
 * Returns true if the flag was previously set.
 */
private static boolean resetCancelNextUpFlag(View view) {
    if ((view.mPrivateFlags & PFLAG_CANCEL_NEXT_UP_EVENT) != 0) {
        view.mPrivateFlags &= ~PFLAG_CANCEL_NEXT_UP_EVENT;
        return true;
    }
    return false;
}
```
其实就是重置一个标志位：`PFLAG_CANCEL_NEXT_UP_EVENT`。这个标志位是什么意思呢？注释是这样说的：Indicates whether the view is temporarily detached。打开可以猜测一下：如果一个 View 正在从 Window 上 detach（一个中间态），那么这个标志位就会被置位。如果调用该方法的时候，这位被置，则返回true。而这里的清理，只是重置而已，对于是否置位并不关心。

##### `dispatchTransformedTouchEvent`
这个方法在 8 处和 10 处都出现过，其它的步骤主要是负责定位事件应该传递给什么 View，而这个函数的作用就是做实际分发的：

```Java
/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // 注释说的极为清楚，CANCEL 事件是一个特殊事件，重点在于事件类型而非内容
    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            // child 如果是Null，就分发事件到父类，也就是调用前面分析的 View 的 dispatchTouchEvent 方法
            handled = super.dispatchTouchEvent(event);
        } else {
            // 否则下发到 child 去处理 CANCEL 事件
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // 校验事件是否合法，如果为 0，说明事件不应该由这个 View 处理
    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }

    // 只要 child 为 null，就会调用父类，也就是 View 的 dispatchTouchEvent 方法
    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                // 为事件做偏移
                event.offsetLocation(offsetX, offsetY);
                // 分发事件
                handled = child.dispatchTouchEvent(event);
                // 恢复事件
                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        // 分化出一个新的事件
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }

        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```
它其实只做以下几件事情：

1. 事件是否是 CANCEL 的，如果是，直接分发即可；
2. 确定分发的事件合法，主要是看 newPointerIdBits 是否为 0，读者要去理解这个方法，需要看一下`MotionEvent.getPointerIdBits()`方法；
3. 重新计算事件的坐标，并分发，分发的时候判断 child 是否为 null，如果为 null，则分发给 ViewGroup 的父类也就是 View 处理，否则分发给子 View 处理；

#### 事件拦截
2 处的判断看上去可能比较难懂，换一个等价的判断就明白了：`actionMasked != MotionEvent.ACTION_DOWN && mFirstTouchTarget == null`。`mFirstTouchTarget`为 null，也就是说没有找到 View 处理事件，Action 不为 Down，也就是说不是事件流的起始，那这个事件完全就是非法的，不应该传递到这里来，因此肯定不会下发，所以`intercepted`就置为 true。

if 语句里面的判断和方法调用，就不做解释了。读者去了解一下`requestDisallowInterceptTouchEvent()`方法和`onInterceptTouchEvent()`就好，说白了就是 ViewGroup 提供的事件拦截以及禁止事件拦截的两个方法。要注意的是这里方法返回不同的结果对事件流处理造成的影响。

这里接个尾巴，在判断完事件拦截后，@@ 处又调用了一次`resetCancelNextUpFlag()`方法，这一次它的返回结果就被记录下来了，用于判断事件是否要取消，也就是说我们可以通过这个方法来判断一个 View 是否能合法的接收事件。

#### 多指触摸
3 处在判断的时候是考虑到了多指触摸的情况的，一个 ViewGroup 里面可能有多个 View，这一个或者多个 View 可以同时被多根手指 Touch，那么在这样的情况下，就需要记录哪个 View 被哪个 Pointer（形象一点可以想象成手指）触摸，这个是通过 TouchTarget 的 pointerIdBits 来记录的，TouchTarget 可以代表着一个触摸对象，它封装了一个 View 以及与触摸相关的信息在内部。

#### View 是否能接受事件
6 处判断一个View是否能接受事件，调用了这样一个方法：

```Java
/**
 * Returns true if a child view can receive pointer events.
 * @hide
 */
private static boolean canViewReceivePointerEvents(View child) {
	return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                || child.getAnimation() != null;
}
```
这个判断有依据的是 View 是否可见，或者 View 是否正在执行动画。

#### 关于`ACTION_XXXX`和`ACTION_POINTER_XXXX`
XXXX 代表着 DOWN 或者 UP（不存在`ACTION_POINTER_MOVE`），这里可以从实验得出如下结论：

1. 第一个按下去的事件类型是`ACTION_DOWN`，其余不论按下多少个手指，事件类型都是`ACTION_POINTER_DOWN`;
2. 不论按下去和抬起来的顺序，最后一个抬起的手指，它触发的事件类型就是`ACTION_UP`，这个与`ACTION_DOWN`并不一定发生在同一根手指上；
3. PointerId 可以唯一标记一个手指触摸事件，第一个触摸的手指 ID 为1, 第二个触摸的手指 ID 为2，第三个触摸的手指 ID 为 3，以此类推...因此可以通过下面的方法很方便的将事件发生时手指的触摸情况反映成一个 Bit 组：

   ```Java
   public final int getPointerIdBits() {
   	int idBits = 0;
   	final int pointerCount = nativeGetPointerCount(mNativePtr);
   	for (int i = 0; i < pointerCount; i++) {
   		idBits |= 1 << nativeGetPointerId(mNativePtr, i);
   	}
   	return idBits;
   }
   ```
   所以如果我有四根手指触摸着屏幕，那么通过调用这个方法就可以得到的`1111`（二进制，后面同含义的数字也是二进制表示），如果我第一个按下的手指抬起，那么计算出来的值就是`1110`，第四根按下的手指抬起，计算出来的值就是`0110`，以此类推...因此在 MOVE 中可以通过这种手段确认是哪几根手指处于按下状态。通过以下方法可以获知每一根手指的坐标位置：

   ```Java
   int pointerCount = event.getPointerCount();
   for (int i = 0; i < pointerCount; i++) {
   	int id = event.getPointerId(i);
   	int x = (int) event.getX(i);
   	int y = (int) event.getY(i);
   }
   ```
>相关的还有一个方法是`getActionIndex()`，这个方法只有在事件类型为`ACTION_POINTER_DOWN`或者`ACTION_POINTER_UP`的时候才有意义，其余情况下返回的都是0。详细可见附表数据。

配合以上几个结论，看 12 处的时候会更加清晰：在`ACTION_UP`的时候，表示所有手指已经离开 View，所以可以重置状态。而 View 中只会处理`ACTION_UP`和`ACTION_DOWN`事件，因此对于 View 来说（这里是狭义的 View，不包括 ViewGroup），只有单指事件。

#### `ACTION_CANCEL`事件发生时机
11 处__指明了`ACTION_CANCEL`事件发生的时机：父 ViewGroup 拦截了子 View 的事件流__。 具体可见[`VierGroup.onInterceptTouchEvent()`](https://developer.android.com/reference/android/view/ViewGroup.html#onInterceptTouchEvent(android.view.MotionEvent))方法的 API 文档。

__其余的点都不是很难理解，第 ⑨ 处留有一个疑问：这里为什么要把 PointerId 记录到第一个添加的 TargetView 中去？。__

## 结论
至此，View 和 ViewGroup 的事件分发处理机制已经解析完毕。

事件被分发到 ViewGroup 之后，会由 ViewGroup 查看哪些子 View 能够接受该事件，判断的初始标准很简单：参考方法`canViewReceivePointerEvents()`和`isTransformedTouchPointInView()`，如果找到一个这样的的子 View，尝试调用`dispatchTransformedTouchEvent()`方法将事件分发给子 View ，如果子 View 处理事件成功（处理过程参考前面的图），则该 View 被加入到 ViewGroup 的 TargetView 链中去，后续的事件会继续分发给它，否则它将不能再接受事件。

如果 ViewGroup 找不到这样一个子 View 进行事件处理，就会尝试调用父类的`dispatchTouchEvent()`方法处理事件，此时 ViewGroup 把自己当做一个 View 来看待。

以上就是 ViewGroup 进行事件分发 和 View 处理事件的主流程。

## 附表
下面是一张三根手指操作 View 的时候，它收到的事件的相关信息打印，打印代码如下：

```Java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	int actionIndex = ev.getActionIndex();
	int pointerCount = ev.getPointerCount();
	for (int index = 0; index < pointerCount; index++) {
		Log.e(TAG, "PointId: " + ev.getPointerId(index) + " # ActionIndex: " + actionIndex + " # ActionType: " + ev.getActionMasked());
		Log.e(TAG, "X" + index + ": " + ev.getX(index) + " # Y" + index + ": " + ev.getY(index) + " # PointId: " + ev.getPointerId(actionIndex));
	}
	Log.e(TAG, "======================");
	return super.dispatchTouchEvent(ev);
}
```

打印内容：

```Java
09-19 10:14:06.045  PointId: 0 # ActionIndex: 0 # ActionType: 0
09-19 10:14:06.045  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:06.045  ======================
09-19 10:14:06.584  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:06.584  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:06.584  ======================
09-19 10:14:06.584  PointId: 0 # ActionIndex: 1 # ActionType: 5
09-19 10:14:06.584  X0: 224.0 # Y0: 1232.0 # PointId: 1
09-19 10:14:06.584  PointId: 1 # ActionIndex: 1 # ActionType: 5
09-19 10:14:06.584  X1: 483.0 # Y1: 973.0 # PointId: 1
09-19 10:14:06.584  ======================
09-19 10:14:06.953  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:06.953  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:06.953  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:06.953  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:06.953  ======================
09-19 10:14:06.954  PointId: 0 # ActionIndex: 2 # ActionType: 5
09-19 10:14:06.954  X0: 224.0 # Y0: 1232.0 # PointId: 2
09-19 10:14:06.954  PointId: 1 # ActionIndex: 2 # ActionType: 5
09-19 10:14:06.954  X1: 483.0 # Y1: 973.0 # PointId: 2
09-19 10:14:06.954  PointId: 2 # ActionIndex: 2 # ActionType: 5
09-19 10:14:06.954  X2: 811.0 # Y2: 1076.0 # PointId: 2
09-19 10:14:06.954  ======================
09-19 10:14:07.264  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.264  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.264  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.264  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:07.264  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.264  X2: 809.0 # Y2: 1076.0 # PointId: 0
09-19 10:14:07.264  ======================
09-19 10:14:07.280  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.280  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.280  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.280  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:07.280  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.280  X2: 807.0 # Y2: 1076.0 # PointId: 0
09-19 10:14:07.280  ======================
09-19 10:14:07.297  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.297  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.297  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.297  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:07.297  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.297  X2: 805.0 # Y2: 1076.0 # PointId: 0
09-19 10:14:07.297  ======================
09-19 10:14:07.314  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.314  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.314  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.314  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:07.314  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.314  X2: 803.0 # Y2: 1076.0 # PointId: 0
09-19 10:14:07.314  ======================
09-19 10:14:07.331  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.331  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.331  PointId: 1 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.331  X1: 483.0 # Y1: 973.0 # PointId: 0
09-19 10:14:07.331  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.331  X2: 802.0 # Y2: 1076.0 # PointId: 0
09-19 10:14:07.331  ======================
09-19 10:14:07.405  PointId: 0 # ActionIndex: 1 # ActionType: 6
09-19 10:14:07.405  X0: 224.0 # Y0: 1232.0 # PointId: 1
09-19 10:14:07.405  PointId: 1 # ActionIndex: 1 # ActionType: 6
09-19 10:14:07.406  X1: 483.0 # Y1: 973.0 # PointId: 1
09-19 10:14:07.406  PointId: 2 # ActionIndex: 1 # ActionType: 6
09-19 10:14:07.406  X2: 802.0 # Y2: 1076.0 # PointId: 1
09-19 10:14:07.406  ======================
09-19 10:14:07.415  PointId: 0 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.415  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.415  PointId: 2 # ActionIndex: 0 # ActionType: 2
09-19 10:14:07.415  X1: 803.0 # Y1: 1076.0 # PointId: 0
09-19 10:14:07.415  ======================
09-19 10:14:07.421  PointId: 0 # ActionIndex: 0 # ActionType: 6
09-19 10:14:07.421  X0: 224.0 # Y0: 1232.0 # PointId: 0
09-19 10:14:07.421  PointId: 2 # ActionIndex: 0 # ActionType: 6
09-19 10:14:07.421  X1: 803.0 # Y1: 1076.0 # PointId: 0
09-19 10:14:07.421  ======================
09-19 10:14:07.421  PointId: 2 # ActionIndex: 0 # ActionType: 1
09-19 10:14:07.421  X0: 803.0 # Y0: 1076.0 # PointId: 2
09-19 10:14:07.421  ======================
```
注意：手指抬起的顺序不一致，也会引起一些数据变化，读者要理解，可以自己复制代码做实验。
> 1 是`ACTION_UP`，2 是`ACTION_DOWN`，5 是`ACTION_POINTER_DOWN`，6 是`ACTION_POINTER_UP`。