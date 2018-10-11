---
title: UIResponder 和 UIControl
tags:
  - iOS
  - 事件
categories: iOS
date: 2017-01-11 18:06:43
---


先来看一幅图：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/view_class_tree.png" width="480" alt="View 类继承树"/></div>

这幅图展示了 iOS View 类的继承关系，可以看到 UIResponder 和 UIControl 之间的关系。为什么要比对这两个类呢？从继承关系中可以看出：

1. UIResponder 的直接子类覆盖了我们平时用到的几大重要类：UIApplication，UIView 和 UIViewController，也就是说所有的 UIView 的子类 —— 我们平时用的 View 组件都是 UIResponder 的子类；
2. UIView 的子类分为两组，一类是直接子类，一类则是 UIControl 的子类；

所以从继承树上看，研究这两个类对于理解 iOS 的 View 结构有很重要的意义。<!-- More -->

## UIResponder

官网描述：

> The `UIResponder` class defines an interface for objects that respond to and handle events. It is the superclass of `UIApplication`, `UIView` and its subclasses (which include `UIWindow`). Instances of these classes are sometimes referred to as responder objects or, simply, responders.

也就是说，这个类为 __响应和处理事件__ 定义了相关接口，通常继承这个类的子类被称为 __响应者__。UIResponder 的 [API](https://developer.apple.com/reference/uikit/uiresponder) 大致分为以下几类：

1. 关于 FirstResponder；
2. touch 事件；
3. press 事件；
4. motion 事件；
5. remote 事件；

touch 事件即触摸事件，press 事件是由物理键触发的，比如手机遥控器，motion 事件简单翻译过来是手势事件，比如摇动设备，remote 事件是远程控制事件，比如耳机。

> 关于事件，可以阅读：[Event Handling Guide for iOS](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009541)。

## UIControl

官网描述：

> The `UIControl` class implements common behavior for visual elements that convey a specific action or intention in response to user interactions. Controls implement elements such as buttons and sliders, which your app might use to facilitate navigation, gather user input, or manipulate content. Controls use the [Target-Action](https://developer.apple.com/library/content/documentation/General/Conceptual/Devpedia-CocoaApp/TargetAction.html#//apple_ref/doc/uid/TP40009071-CH3) mechanism to report user interactions to your app.

UIControl 使用 Target-Action 机制来告知 App 用户的交互，并把这种交互抽象成一些通用的行为，比如`UIControlEventTouchUpInside`。为 UIButton 添加过监听的童鞋应该很容易理解。

>Target-Action 机制可参考文章[UIControl 的基本使用方法和 Target-Action 机制](http://www.cocoachina.com/ios/20160111/14932.html)。
>
>在我看来和 Java 中的回调监听一样，只不过 Java 中并没有 SEL 这样的概念，而 iOS 中需要 Target 和 Action 才能指定执行某个实例的某个方法，两者理念上应该没有什么不同。

简单查看一下 UIControl 的属性和方法，可以看出 UIControl 是在事件的基础上封装出来一些高级概念，以便于开发者更容易的相应用户的交互行为，比如：通过添加 Target 来相应 `UIControlEventTouchUpInside` 动作，而这个动作在 iOS 上的典型代表就是一次 UIButton 的点击过程，拆分成事件就是 TouchBegin -> TouchMove ->…-> TouchMove -> TouchEnd，还有对 selected、highlighted 等属性的定义，都是根据实际使用场景定义一些通用的用户行为模式，然后在 UIControl 为这些模式提供相应入口。

另外一个值得注意的点是下面两组接口：

```objective-c
// UIResponder
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;

// UIControl
- (BOOL)beginTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event;
- (BOOL)continueTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event;
- (void)endTrackingWithTouch:(nullable UITouch *)touch withEvent:(nullable UIEvent *)event; // touch is sometimes nil if cancelTracking calls through to this.
- (void)cancelTrackingWithEvent:(nullable UIEvent *)event;   // event may be nil if cancelled for non-event reasons, e.g. removed from window
```

这两组接口非常相似，但是参数上却又略微的不同：UIResponder 组得到的是一个 UITouch 集合，而 UIControl 得到的却是单个的 UITouch，简单猜测是因为判断用户行为模式并不需要同时监控多个 UITouch 变量，大多数的 UIControlEvents 定义的用户行为模式只需要一个 UITouch 即可判断。

> 这个猜测受 Android 系统影响，可见文章[Android View事件处理](http://timebridge.space/2016/09/04/Android-touchevent-process/)。
>
> 关于这两组 API 的详细情况，可以参考 [史上最详细的iOS之事件的传递和响应机制](http://www.cnblogs.com/machao/p/5471094.html)。

## 两者的区别和联系

区别前面已有论述，总结来说：__UIResponder 类表明一个组件是可以接受事件的，它定义的方法可以获得一个组件最原始的事件信息，包括多点触控等等；UIControl 则通过这些事件抽象出用户的行为，比如点击事件，拖曳事件，并通过 Target-Action 机制让开发者可以方便的监听这些行为的发生。__

而为了研究两者之间的联系，我重载了上面提到的两组方法：

```objective-c
@interface XYButton ()
-(void)printTouchInfo:(UITouch *)touch withTag:(NSString *)tag;
@end

@implementation XYButton

// UIControl
- (BOOL)beginTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:touch withTag:@"UIControl-Begin"];
    return YES;
}

- (BOOL)continueTrackingWithTouch:(UITouch *)touch withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:touch withTag:@"UIControl-Move"];
    return YES;
}

- (void)endTrackingWithTouch:(nullable UITouch *)touch withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:touch withTag:@"UIControl-End"];
}

- (void)cancelTrackingWithEvent:(nullable UIEvent *)event {
    [self printTouchInfo:[[event.allTouches allObjects] objectAtIndex:0] withTag:@"UIControl-End"];
}

// UIResponder
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:[[touches allObjects] objectAtIndex:0] withTag:@"UIResponder-Begin"];
    [super touchesBegan:touches withEvent:event];
}
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:[[touches allObjects] objectAtIndex:0] withTag:@"UIResponder-Move"];
    [super touchesMoved:touches withEvent:event];
}
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:[[touches allObjects] objectAtIndex:0] withTag:@"UIResponder-End"];
    [super touchesEnded:touches withEvent:event];
}
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event {
    [self printTouchInfo:[[touches allObjects] objectAtIndex:0] withTag:@"UIResponder-Cancel"];
    [super touchesCancelled:touches withEvent:event];
}

-(void)printTouchInfo:(UITouch *)touch withTag:(NSString *)tag {
    CGPoint point = [touch locationInView:self];
    NSLog(@"【%@】%f, %f", tag, point.x, point.y);
}
@end
```

当这个 UIButton 显示在界面上的时候，我在 Button 上拖动鼠标（点击位置），输出的 Log 如下：

```xml
2017-01-11 17:55:25.079 MyP[37537:3413743] 【UIResponder-Begin】119.000000, 121.000000
2017-01-11 17:55:25.080 MyP[37537:3413743] 【UIControl-Begin】119.000000, 121.000000
2017-01-11 17:55:25.123 MyP[37537:3413743] 【UIResponder-Move】119.000000, 123.666656
2017-01-11 17:55:25.123 MyP[37537:3413743] 【UIControl-Move】119.000000, 123.666656
2017-01-11 17:55:25.142 MyP[37537:3413743] 【UIResponder-Move】121.000000, 135.333328
2017-01-11 17:55:25.143 MyP[37537:3413743] 【UIControl-Move】121.000000, 135.333328
2017-01-11 17:55:25.175 MyP[37537:3413743] 【UIResponder-Move】136.000000, 221.333328
2017-01-11 17:55:25.176 MyP[37537:3413743] 【UIControl-Move】136.000000, 221.333328
2017-01-11 17:55:25.192 MyP[37537:3413743] 【UIResponder-Move】147.666656, 291.666656
2017-01-11 17:55:25.193 MyP[37537:3413743] 【UIControl-Move】147.666656, 291.666656
2017-01-11 17:55:25.209 MyP[37537:3413743] 【UIResponder-Move】159.666656, 352.666656
2017-01-11 17:55:25.210 MyP[37537:3413743] 【UIControl-Move】159.666656, 352.666656
2017-01-11 17:55:25.226 MyP[37537:3413743] 【UIResponder-Move】164.666656, 392.333328
2017-01-11 17:55:25.226 MyP[37537:3413743] 【UIControl-Move】164.666656, 392.333328
2017-01-11 17:55:25.243 MyP[37537:3413743] 【UIResponder-Move】168.666656, 424.000000
2017-01-11 17:55:25.244 MyP[37537:3413743] 【UIControl-Move】168.666656, 424.000000
2017-01-11 17:55:25.260 MyP[37537:3413743] 【UIResponder-Move】170.666656, 445.666656
2017-01-11 17:55:25.260 MyP[37537:3413743] 【UIControl-Move】170.666656, 445.666656
2017-01-11 17:55:25.277 MyP[37537:3413743] 【UIResponder-Move】170.666656, 455.666656
2017-01-11 17:55:25.277 MyP[37537:3413743] 【UIControl-Move】170.666656, 455.666656
2017-01-11 17:55:25.294 MyP[37537:3413743] 【UIResponder-Move】170.666656, 460.666656
2017-01-11 17:55:25.294 MyP[37537:3413743] 【UIControl-Move】170.666656, 460.666656
2017-01-11 17:55:25.348 MyP[37537:3413743] 【UIResponder-End】170.666656, 460.666656
2017-01-11 17:55:25.348 MyP[37537:3413743] 【UIControl-End】170.666656, 460.666656
2017-01-11 17:55:26.431 MyP[37537:3413743] 【UIResponder-Begin】128.000000, 154.666656
2017-01-11 17:55:26.431 MyP[37537:3413743] 【UIControl-Begin】128.000000, 154.666656
2017-01-11 17:55:26.450 MyP[37537:3413743] 【UIResponder-Move】128.000000, 157.666656
2017-01-11 17:55:26.450 MyP[37537:3413743] 【UIControl-Move】128.000000, 157.666656
2017-01-11 17:55:26.467 MyP[37537:3413743] 【UIResponder-Move】128.000000, 163.333328
2017-01-11 17:55:26.468 MyP[37537:3413743] 【UIControl-Move】128.000000, 163.333328
2017-01-11 17:55:26.484 MyP[37537:3413743] 【UIResponder-Move】129.000000, 174.333328
2017-01-11 17:55:26.484 MyP[37537:3413743] 【UIControl-Move】129.000000, 174.333328
2017-01-11 17:55:26.501 MyP[37537:3413743] 【UIResponder-Move】131.000000, 200.000000
2017-01-11 17:55:26.501 MyP[37537:3413743] 【UIControl-Move】131.000000, 200.000000
2017-01-11 17:55:26.518 MyP[37537:3413743] 【UIResponder-Move】135.000000, 235.666656
2017-01-11 17:55:26.518 MyP[37537:3413743] 【UIControl-Move】135.000000, 235.666656
2017-01-11 17:55:26.534 MyP[37537:3413743] 【UIResponder-Move】139.000000, 277.000000
2017-01-11 17:55:26.535 MyP[37537:3413743] 【UIControl-Move】139.000000, 277.000000
2017-01-11 17:55:26.551 MyP[37537:3413743] 【UIResponder-Move】141.000000, 300.666656
2017-01-11 17:55:26.551 MyP[37537:3413743] 【UIControl-Move】141.000000, 300.666656
2017-01-11 17:55:26.568 MyP[37537:3413743] 【UIResponder-Move】141.000000, 325.333328
2017-01-11 17:55:26.709 MyP[37537:3413743] 【UIResponder-End】141.000000, 346.333328
2017-01-11 17:55:26.710 MyP[37537:3413743] 【UIControl-End】141.000000, 346.333328
```

在单指情况下，可以看到 UIResponder 方法的 UITouch 集合的第一个 UITouch 对象记录的信息和 UIControl 方法的 UITouch 对象记录的信息完全一致，且事件类型也一一对应，基本印证了前面的猜测。

> 多指情况下待验证。

这一点设计和 Android 很不一样，Android 的触摸以及事件监听是 View 提供的特性，所有 View 都有一致的事件监听添加方法，并不像 iOS 这样做了划分，以至像 UILabel 这样的组件无法像 UIButton 一样添加点击事件监听。