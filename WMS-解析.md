---
title: WindowManagerService 解析
date: 2017-10-29 18:48:48
tags: Framework
categories: Android
---

在另外一篇文章 [Android View 绘制过程](http://timebridge.space/2016/09/21/Android-view-display/) 中已经提到一个关于 Window 的概念了—— PhoneWindow，只不过这篇着重是从 PhoneWindow 往 View 方向讲，关于 PhoneWindow 本身关注的很少，这一篇就探究一下 Window 以及 WindowManagerService。

在 Android 里面，Window 是一个非常重要但其实平时开发又不常用的概念。大部分看到它时可能是我们要修改 Dialog 的样式的时候：

```java
mDialog = new Dialog(getActivity(), R.style.IsDelDialog);//自定义的样式，没有贴出代码来
mDialog.setContentView(view);
mDialog.show();
Window dialogWindow = mDialog.getWindow();
WindowManager m = getActivity().getWindowManager();
Display d = m.getDefaultDisplay(); // 获取屏幕宽、高度
WindowManager.LayoutParams p = dialogWindow.getAttributes(); // 获取对话框当前的参数值
        p.height = (int) (d.getHeight() * 0.8); // 高度设置为屏幕的0.6，根据实际情况调整
p.width = (int) (d.getWidth() * 0.8); // 宽度设置为屏幕的0.65，根据实际情况调整
dialogWindow.setAttributes(p);
```

那么 Window 本质上到底是什么？为什么 Activity 的底层以及 Dialog 都与它相关？

## Window 概览

Window 的代码定义如下：

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {
	...
}
```

从以上代码可以得出以下几个结论：

1）Window 负责一些标准的展示以及事件处理，最终的用法是添加到 WindowManager；

2）它是一个抽象类，并且只有一个实现类，就是 PhoneWindow；

关于 PhoneWindow 在做些啥，[Android View 绘制过程](http://timebridge.space/2016/09/21/Android-view-display/) 已经有讲述，我们去看看 WindowManager：

```java
/**
 * The interface that apps use to talk to the window manager.
 * <p>
 * Use <code>Context.getSystemService(Context.WINDOW_SERVICE)</code> to get one of these.
 * </p><p>
 * Each window manager instance is bound to a particular {@link Display}.
 * To obtain a {@link WindowManager} for a different display, use
 * {@link Context#createDisplayContext} to obtain a {@link Context} for that
 * display, then use <code>Context.getSystemService(Context.WINDOW_SERVICE)</code>
 * to get the WindowManager.
 * </p><p>
 * The simplest way to show a window on another display is to create a
 * {@link Presentation}.  The presentation will automatically obtain a
 * {@link WindowManager} and {@link Context} for that display.
 * </p>
 *
 * @see android.content.Context#getSystemService
 * @see android.content.Context#WINDOW_SERVICE
 */
public interface WindowManager extends ViewManager {
	...
}
```

WindowManager 可以通过 `getSystemService(Context.WINDOW_SERVICE)` 来获取，每一个 WindowMagaer 都被绑定到一个特性的 `Display` 实例。

> 这里涉及到 Context 的解析，待补充。

`WindowManager`接口继承于`ViewManager`接口，这个接口只有三个简单的方法：

```java
public void addView(View view, ViewGroup.LayoutParams params);
public void updateViewLayout(View view, ViewGroup.LayoutParams params);
public void removeView(View view);
```

这三个方法代表了对 View 的基本操作，查看这个接口的实现，读者会发现除了 WindowManager，还有一个非常重要的实现类：ViewGroup。这似乎暗示了 WindowManager 和 ViewGroup 之间是有一些联系的。

## Dialog 与 Window

为了搞清楚 Window 相关概念的关系，我们选择从 Dialog 入手，看看 Window 是如何展示一个 Dialog 的。Dialog 的用法一般如下：

```java
AlertDialog ad = new AlertDialog.Builder(MainActivity.this).create();  
ad.setTitle("标题1");  
ad.setIcon(R.drawable.ic_launcher);  
ad.setMessage("我是消息内容");  
ad.setButton("确定", new DialogInterface.OnClickListener() { 
	@Override  
    public void onClick(DialogInterface dialog, int which) { 
    }  
});  
ad.setButton2("取消", new DialogInterface.OnClickListener() {
    @Override  
    public void onClick(DialogInterface dialog, int which) {    
    }  
});  
ad.show(); 
```

最关键的方法就是 `show()`：

```java
public void show() {
    if (mShowing) {
        if (mDecor != null) {
            if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
            }
            mDecor.setVisibility(View.VISIBLE);
        }
        return;
    }

    mCanceled = false;
    
    if (!mCreated) {
        dispatchOnCreate(null);
    }

    onStart();
    mDecor = mWindow.getDecorView(); // mDecor 的来源
    ...

    try {
        mWindowManager.addView(mDecor, l);
        mShowing = true;
    
        sendShowMessage();
    } finally {
    }
}
```

最终的展示是通过`mWindowManager.addView(mDecor, l);`来完成的，这一句到底做了啥呢？在这里我们可以看到一些熟悉的面孔：mWindow，mWindowManager，mDecor，首先我们来看看它们的来源。mDecor 是从 mWindow 中拿出来的，mWindow 呢？

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    if (createContextThemeWrapper) {
        if (themeResId == 0) {
            final TypedValue outValue = new TypedValue();
            context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
            themeResId = outValue.resourceId;
        }
        mContext = new ContextThemeWrapper(context, themeResId);
    } else {
        mContext = context;
    }

    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE); // mWindow 

    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);

    mListenersHandler = new ListenersHandler(this);
}
```

在 Dialog 的构造方法里面，这两个变量被实例化，首先是根据 context 获取 WindowManager，而 mWindow 实际就是 PhoneWindow 的实例。那么 `WindowManager` 又是如何添加 view 的呢，即 `mWindowManager.addView(mDecor, l);` 是如何运行的？

mWindowManager 的实现类是谁呢？它是通过 `context.getSystemService()`获取的，它的注册是在`android.app.SystemServiceRegistry#registerService`中进行的：

```java
registerService(Context.WINDOW_SERVICE, WindowManager.class,
                new CachedServiceFetcher<WindowManager>() {
            @Override
            public WindowManager createService(ContextImpl ctx) {
                return new WindowManagerImpl(ctx.getDisplay());
}});
```

因此这里的实现类是`WindowManagerImpl`，并且在创建的时候是通过 context 获取到了 display。我们来看看类 `WindowManagerImpl` 的 `addView` 方法:

```java
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
     applyDefaultToken(params);
     mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

这个方法被代理到了 `WindowManagerGlobal` 类里面：

```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
    //... 参数检查 & 调整

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
    
    ...
    // do this last because it fires off messages to start doing things
    try {
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        ...
}
```

这里的核心方法被代理到了 `ViewRootImpl#setView()`方法：

```java
/**
 * view 是前面 Window 的 decorView
 */
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    mView = view;
    ...
    int res; /* = WindowManagerImpl.ADD_OKAY; */

    // Schedule the first layout -before- adding to the window
    // manager, to make sure we do the relayout before receiving
    // any other events from the system.
    requestLayout();
    ...

    // 这个方法用于添加展示，注意 mDisplay 就是 activity 的 display
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
    ...                             
    if (res < WindowManagerGlobal.ADD_OKAY) {
        mAttachInfo.mRootView = null;
        mAdded = false;
        mFallbackEventHandler.setView(null);
        unscheduleTraversals();
        setAccessibilityFocus(null, null);
        switch (res) {
            case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
            case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                throw new WindowManager.BadTokenException(
                        "Unable to add window -- token " + attrs.token
                        + " is not valid; is your activity running?");
            case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                throw new WindowManager.BadTokenException(
                        "Unable to add window -- token " + attrs.token
                        + " is not for an application");   
            .....
          }                          
     ......
}
```

再往下很重要的就是 mWindowSession 变量的来源了，它来自`WindowManagerGlobal#getWindowSession()`：

```java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService();
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                                @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to open window session", e);
            }
        }
        return sWindowSession;
    }
}

public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            sWindowManagerService = IWindowManager.Stub.asInterface(
                    ServiceManager.getService("window"));
            try {
                sWindowManagerService = getWindowManagerService();
                ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to get WindowManagerService, cannot set animator scale", e);
            }
        }
        return sWindowManagerService;
    }
}
```

再往下走就是`WindowManagerService` 相关的了。

## WindowManagerService

