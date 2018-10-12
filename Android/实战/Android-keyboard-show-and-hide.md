---
title: Android 的键盘收起展开
date: 2016-02-28 17:00:21
categories: Android
---

## 介绍和使用
键盘的展开和收起主要使用到类 [InputMethodManager](http://developer.android.com/reference/android/view/inputmethod/InputMethodManager.html) 。使用方法如下:

```java
//隐藏键盘
public void hideKeyboard(Context context, View view) {
        InputMethodManager inputMethodManager = (InputMethodManager) context.getSystemService(Activity.INPUT_METHOD_SERVICE);
        inputMethodManager.hideSoftInputFromWindow(view.getWindowToken(), InputMethodManager.HIDE_NOT_ALWAYS);
    }

//显示键盘
public void showKeyboard(Context context, View view) {
        InputMethodManager inputMethodManager = (InputMethodManager) context.getSystemService(Activity.INPUT_METHOD_SERVICE);
        inputMethodManager.showSoftInput(view, InputMethodManager.SHOW_IMPLICIT);
    }
```
<!--more-->
## 参数问题
在调用hideSoftInputFromWindow()和showSoftInput()方法的时候，都要求传入第二个参数，这个参数在官网上的解释并不清楚，比如hideSoftInputFromWindow()方法上关于这个参数的解释如下:
>int: Provides additional operating flags. Currently may be 0 or have the HIDE_IMPLICIT_ONLY bit set.

从官网上看，这个参数总共有以下四个常量，加上0:

在展开和收起方法调用的时候，都要求传入第二个参数。大致有以下四种：

1. HIDE_IMPLICIT_ONLY：indicate that the soft input window should only be hidden if it was not explicitly shown by the user.
2. HIDE_NOT_ALWAYS：to indicate that the soft input window should normally be hidden, unless it was originally shown with SHOW_FORCED
3. SHOW_FORCED：indicate that the user has forced the input method open (such as by long-pressing menu) so it should not be closed until they explicitly do so.
4. SHOW_IMPLICIT：indicate that this is an implicit request to show the input window, not as the result of a direct request by the user.

1和2是用于收起键盘的，3和4是用于展开键盘的。

### 实验结果
1. 如果用户是点击输入框弹出的键盘，则调用1）是无法隐藏的，系统此时判断使用户有意呼唤键盘的！调用2）则可以收起键盘；
2. 如果用户是使用3）作为参数显示键盘的，则1）和2）都是无法隐藏键盘的，此时需要用参数0，强制隐藏一切键盘；
3. 如果用户是使用4）作为参数显示键盘的，则1）和2）都是可以隐藏键盘的。

总结下来就是: __Forced > Explicit > Implicit__。

对于隐藏键盘，其中1）是属于implicit，2）是属于explicit，0则是属于force；对于显示键盘，则4）是属于implicit，输入框呼出属于explicit，3）则是属于force；

__只有隐藏的级别>=展开的级别，才能将键盘隐藏掉。__