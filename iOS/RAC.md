---
title: 【译】ReactiveCocoa 入门教程（上半部分）
tags: [iOS, RAC, Reactive]
categories: iOS
date: 2017-01-16 15:57:44
---


翻译自：[ReactiveCocoa Tutorial – The Definitive Introduction: Part 1/2](https://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)

> 在网上看了很多的文章，大部分都是例子堆砌或者理论堆砌，只有这篇文章，虽然没有理论描述，但是通过两个 App 的改造和开发过程，详细的展示了 RAC 的优点以及使用场景，娓娓道来，理解起来比较容易。

作为一个 iOS 开发者，你写的每行代码几乎都是用于响应某个时间：一个按钮点击事件、一个网络回复的消息、一个属性变化事件（KVO）或者是因为位置变化而收到的系统通知，这些都是很好的例子。然而，这些事件的通知都被包装秤不同的方式进行：Action、delegate、KVO 以及 callback 等等。[ReactiveCocoa ](https://github.com/ReactiveCocoa/ReactiveCocoa)为事件的响应定义了一套标准的接口，这样处理/过滤/组合起事件来会变得更加简单。

听起来是不是有点费解？还是有点好奇，有点激动？那就继续往下读吧:-D。

ReactiveCocoa 组合了一些编码风格：

* __[Functional Programming](http://en.wikipedia.org/wiki/Functional_programming)__ 使用了其中的高阶函数，也就是以函数作为函数的参数
* __[Reactive Programming](http://en.wikipedia.org/wiki/Reactive_programming)__ 关注数据流以及变化的通知<!-- More -->

> 【译者注】实际上远不止这些，要理解好 RAC，可以多看看函数式编程和响应式编程这两种编码思想，这里作者简化了理论描述。

因为融合了两种编码风格的思想，因此 ReactiveCocoa 又被描述为__函数响应式编程（Functional Reactive Programming，简称 FRP）__框架。

好了，相关理论到此为止，虽然编程范式是一个非常有吸引力的主题，但是接下去本教程主要关注实践部分，会以一个贯穿始终的例子为核心进行讲述。

## The Reactive Playground

在这个教程中，我们会从一个简单的应用程序 —— ReactivePlayground 入手来了解响应式编程。开始之前可以先下载这个[项目](https://koenig-media.raywenderlich.com/uploads/2014/01/ReactivePlayground-Starter.zip)并编译运行以确保一切就绪。

ReactivePlayground 非常简单，只会在屏幕上展示一个登陆页面，提供正确的身份信息：用户名和密码就可以看到一张可爱的小猫咪图片：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/ReactivePlaygroundStarter.jpg" width="320" alt="ReactivePlayground应用示例图"/></div>

看呐，多么可爱！

现在是时候花点时间来浏览一下项目的代码了，它非常简单，花不了多长时间。

打开 __RWViewController.m__ 扫一眼，你需要花多久时间才能看出来 __Sign In__ 按钮被激活的条件？展现/隐藏 __signInFailure__ 标签的规则又是什么？在这个相对简单的例子中，你可能需要花一两分钟的时间来回答这些问题，在一个更为复杂的应用中，这样的分析无疑会花费很长的时间。

> 【译者注】主要原因是逻辑四下分散，要回答这些问题需要来回切换代码。

如果开发者使用 ReactiveCocoa，应用的逻辑就会变得清晰很多，So，让我们开始吧！

> 【译者注】这里省略一段如何添加 ReactiveCocoa 框架的章节不作翻译，了解 CocoaPods 的童鞋并不难。建议读者直接去看 Github Readme指导。

## Time To Play

正如前面介绍的，ReactiveCocoa 为处理应用中发生的各种不同的事件流提供了一套标准的接口，在 ReactiveCocoa 框架中，这些事件被称为__信号(signal)__，由类 RACSignal 表示。

> 【译者注】后面 ReactiveCocoa 简称 RAC。

打开应用的初始 ViewController —— RWViewController，添加以下语句：

```objective-c
#import <ReactiveCocoa/ReactiveCocoa.h>
```

就可以导入 RAC 的头文件。你现在要做的事情不是去替换任何的代码，而是先玩玩它，在`viewDidLoad`方法的末尾添加如下代码：

```objective-c
[self.usernameTextField.rac_textSignal subscribeNext:^(id x) {
  NSLog(@"%@", x);
}];
```

编译并运行程序，在用户名输入框中输入一些文字，并注意观察控制台的输出，会有类似下面的 Log 打印出来：

```xml
2013-12-24 14:48:50.359 RWReactivePlayground[9193:a0b] i
2013-12-24 14:48:50.436 RWReactivePlayground[9193:a0b] is
2013-12-24 14:48:50.541 RWReactivePlayground[9193:a0b] is 
2013-12-24 14:48:50.695 RWReactivePlayground[9193:a0b] is t
2013-12-24 14:48:50.831 RWReactivePlayground[9193:a0b] is th
2013-12-24 14:48:50.878 RWReactivePlayground[9193:a0b] is thi
2013-12-24 14:48:50.901 RWReactivePlayground[9193:a0b] is this
2013-12-24 14:48:51.009 RWReactivePlayground[9193:a0b] is this 
2013-12-24 14:48:51.142 RWReactivePlayground[9193:a0b] is this m
2013-12-24 14:48:51.236 RWReactivePlayground[9193:a0b] is this ma
2013-12-24 14:48:51.335 RWReactivePlayground[9193:a0b] is this mag
2013-12-24 14:48:51.439 RWReactivePlayground[9193:a0b] is this magi
2013-12-24 14:48:51.535 RWReactivePlayground[9193:a0b] is this magic
2013-12-24 14:48:51.774 RWReactivePlayground[9193:a0b] is this magic?
```

你可以看到，每次输入框中的文字变化，block 中的代码就会被执行，然而我们并没有使用 Target-Action、Delegate 等机制，只有信号和 block，多么令人兴奋啊！

RAC 信号（由 RACSignal 类表示）会向它的订阅者发送事件流，__事件分为三种类型：next、error 和 completed__。在信号发送 error 或者 complete 事件结束自己之前，它可以发送任意数量的 next 事件。在本教程中我们主要关注 next 事件，其余两种事件会在第二部分详细描述。

> 【译者注】经过前面的介绍，如果读者对 [ReactiveX](http://reactivex.io/) 熟悉的话，应该已经明白这个框架到底要做什么了，不过融合了响应式编程之后，RAC 更为强大。

RACSignal 类有一系列方法用于注册监听不同的事件类型。每一个方法会要求传入一个或者多个 block 参数，用于在事件发生时执行。在上面的例子中，你已经看到使用 `subscribeNext`方法来为每一个 next 事件添加 block 用于执行的过程了。

> 【译者注】RAC 将事件源抽象成 RACSignal，比如一个 Button，一个属性；将原先的事件抽象成事件，不过事件没有在 RAC 中有明确表示（起码前面还没提到）；一个完整的事件流（比如 Button 的点击事件）由若干个 next 事件和一个 error/complete 类型事件组成。

RAC 框架使用 Category 为标准的 UIKit Control 类添加了很多的信号，以便于开发者可以订阅这些事件，这就是前面例子中`rac_textSignal`属性的由来。

接下来我们要让 RAC 为我们做点儿实际的工作。

RAC 定义了很多的操作用于处理事件流。举个例子，假设你只对超过字符长度超过 3 的用户名感兴趣，你可以使用 `filter`操作符，更新前面的例子代码如下：

```objective-c
[[self.usernameTextField.rac_textSignal
  filter:^BOOL(id value) {
    NSString *text = value;
    return text.length > 3;
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  }];
```

如果你编译运行并在输入框中输入一些字符，就会发现只有在用户名长度大于3的情况下才会有 Log 输出：

```xml
2013-12-26 08:17:51.335 RWReactivePlayground[9654:a0b] is t
2013-12-26 08:17:51.478 RWReactivePlayground[9654:a0b] is th
2013-12-26 08:17:51.526 RWReactivePlayground[9654:a0b] is thi
2013-12-26 08:17:51.548 RWReactivePlayground[9654:a0b] is this
2013-12-26 08:17:51.676 RWReactivePlayground[9654:a0b] is this 
2013-12-26 08:17:51.798 RWReactivePlayground[9654:a0b] is this m
2013-12-26 08:17:51.926 RWReactivePlayground[9654:a0b] is this ma
2013-12-26 08:17:51.987 RWReactivePlayground[9654:a0b] is this mag
2013-12-26 08:17:52.141 RWReactivePlayground[9654:a0b] is this magi
2013-12-26 08:17:52.229 RWReactivePlayground[9654:a0b] is this magic
2013-12-26 08:17:52.486 RWReactivePlayground[9654:a0b] is this magic?
```

你在这里创建的是一个非常简单的__管道__——这个概念对于响应式编程来说至关重要：将 App 的功能表达成数据流形式。下面的图可以帮你理清楚这个流：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/FilterPipeline.png" alt="ReactivePlayground应用示例图"/></div>

在上面的图示中，你可以看到`rac_textSignal`就是事件源，数据流（其中包含事件）流经`fliter`，只有输入字符串长度大于 3 的事件才会穿过去，管道的最后一步是`subscribeNext:`函数——也就是打印事件值的地方。

这里值得注意的是`filter`操作的输出也是一个 RACSingal，你可以使用以下的代码方式来分离管道的处理步骤：

```objective-c
RACSignal *usernameSourceSignal = 
    self.usernameTextField.rac_textSignal;
 
RACSignal *filteredUsername = [usernameSourceSignal  
  filter:^BOOL(id value) {
    NSString *text = value;
    return text.length > 3;
  }];
 
[filteredUsername subscribeNext:^(id x) {
  NSLog(@"%@", x);
}];
```

因为 RACSignal 上的每一个步骤都会返回一个 RACSignal，所以整体形成了一个链式接口，这个特性使得开发者不用为每一个中间操作生成一个局部变量。

> __注意：__ReactiveCocoa 重度依赖 block。如果你是 block 新手，你应该尝试读一下苹果的 [Blocks Programming Topics ](https://developer.apple.com/library/ios/documentation/cocoa/Conceptual/Blocks/Articles/00_Introduction.html)。如果你很熟悉 block，但是觉得这样的语法难以理解，不容易记住，那 [f*****gblocksyntax.com](http://fuckingblocksyntax.com/) 网站对你或许很有用！（部分字符用 * 隐藏以保护纯洁的童鞋，但是链接是有效的。）

## 小小的转换

如果你把你的代码更新成多个 RACSignal 组件对象，现在是时候将它还原成链式调用了：

```objective-c
[[self.usernameTextField.rac_textSignal
  filter:^BOOL(id value) {
    NSString *text = value; // implicit cast
    return text.length > 3;
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  }];
```

这里有个显式地从`id`类型转换为 `NSString`的过程，这行代码看上去不怎么优雅。幸运的是，由于发送给这个 Block 的数据始终都会是`NSString`类型，因此开发者可以自行更改参数类型：

```objective-c
[[self.usernameTextField.rac_textSignal
  filter:^BOOL(NSString *text) {
    return text.length > 3;
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  }];
```

编译并运行，确认结果如预期。

## 什么是事件（Event）？

迄今为止本教程已经描述了不同的事件类型，但是我们还没有去详述这些事件的结构，有趣的是事件可以包含任何东西！

为了弄清楚这点，我们将在管道中再添加一个操作，更新代码如下：

```objective-c
[[[self.usernameTextField.rac_textSignal
  map:^id(NSString *text) {
    return @(text.length);
  }]
  filter:^BOOL(NSNumber *length) {
    return [length integerValue] > 3;
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  }];
```

编译运行这段代码就会发现控制台输出的不再是输入框中的内容，而是它的长度：

```xml
2013-12-26 12:06:54.566 RWReactivePlayground[10079:a0b] 4
2013-12-26 12:06:54.725 RWReactivePlayground[10079:a0b] 5
2013-12-26 12:06:54.853 RWReactivePlayground[10079:a0b] 6
2013-12-26 12:06:55.061 RWReactivePlayground[10079:a0b] 7
2013-12-26 12:06:55.197 RWReactivePlayground[10079:a0b] 8
2013-12-26 12:06:55.300 RWReactivePlayground[10079:a0b] 9
2013-12-26 12:06:55.462 RWReactivePlayground[10079:a0b] 10
2013-12-26 12:06:55.558 RWReactivePlayground[10079:a0b] 11
2013-12-26 12:06:55.646 RWReactivePlayground[10079:a0b] 12
```

新加的 map 操作使用提供的 block 来变换事件的数据。对于每一个它接收到的 next 事件，RAC 都会运行提供的 block，将返回值作为一个新的 next 事件放出。在上面的代码中，map 操作以 NSString 为输入，转换成内容长度后返回。下面的图更形象的展示了这个过程：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/FilterAndMapPipeline.png" alt="RAC 数据流图"/></div>

如读者缩减，map 操作后的所有步骤接收到的都是一个 NSNumber 实例。你可以使用 map 操作将传入的数据转换成任意你喜欢的东西，只要它是一个对象即可。

> __注意：__在上面的例子中，`text.length`属性返回的是 NSUInteger，是一个原始类型，为了将它用作事件的数据内容，它必须被装箱。庆幸的是，在 OC 里面使用字面量语法可以很优雅的进行转换。

现在是时候用前面学到的概念来更新一下 ReactivePlayground 应用了，你可以将前面添加的嗲吗全部移除掉。

## Creating Valid State Signals

第一件要做的事情就是创建一些信号来反映输入的用户名和密码是否是有效的 —— 在__RMViewController.m__的`viewDidLoad`方法最后添加如下代码：

```objective-c
RACSignal *validUsernameSignal =
  [self.usernameTextField.rac_textSignal
    map:^id(NSString *text) {
      return @([self isValidUsername:text]);
    }];
 
RACSignal *validPasswordSignal =
  [self.passwordTextField.rac_textSignal
    map:^id(NSString *text) {
      return @([self isValidPassword:text]);
    }];
```

以上代码通过 map 操作将输入转换成 boolean 输出，同时装箱为 NSNumber 类型。下一步就是进一步转换这两个信号，以便于能为输入框提供合适的背景颜色，一般来说就是订阅信号并使用输出数据来更新背景颜色即可，其中一种实现如下：

```objective-c
[[validPasswordSignal
  map:^id(NSNumber *passwordValid) {
    return [passwordValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
  }]
  subscribeNext:^(UIColor *color) {
    self.passwordTextField.backgroundColor = color;
  }];
```

这段代码看上去非常落后 & 不优雅。

> 为了实现一个简单的功能，拆分出了很多的操作，阅读起来很累赘。

RAC 为这种需求提供了宏，可以优雅的实现它：

```objective-c
RAC(self.passwordTextField, backgroundColor) =
  [validPasswordSignal
    map:^id(NSNumber *passwordValid) {
      return [passwordValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
    }];
 
RAC(self.usernameTextField, backgroundColor) =
  [validUsernameSignal
    map:^id(NSNumber *passwordValid) {
     return [passwordValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
    }];
```

__`RAC`__宏允许将信号的输出赋值给对象的属性。它需要两个参数：第一个是属性设置的对象，第二个是属性名。每次信号释放一个事件，事件的值都会被赋值给属性。

这个实现非常优雅，你觉得呢？

在编译运行前，还需要找到`upadteUIState`方法，移除下面两行代码来清楚非响应式代码：

```objective-c
self.usernameTextField.backgroundColor = self.usernameIsValid ? [UIColor clearColor] : [UIColor yellowColor];
self.passwordTextField.backgroundColor = self.passwordIsValid ? [UIColor clearColor] : [UIColor yellowColor];
```

编译运行代码，你会发现输入框在输入非法的时候高亮了，输入合法的时候高亮消失。

下面这幅图展现了当前的逻辑：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/TextFieldValidPipeline.png" alt="RAC 数据流图"/></div>

你是不是很好奇为什么这里要创建两个的`validPasswordSignal`和`validUsernameSignal`信号变量，这与我们之前提到的链式管道相违背呀？保持耐心，后面会越来越清晰。

## Combining signals

现在 App 里面的`Sign In`按钮只有在用户名和密码输入框都有合法输入的情况下才会可点击，又到了使用响应式编程风格的时候了！

现有的代码已经通过信号来发送一个布尔值反映用户名和密码是不是有效的了。你的任务就是将这两个信号组合起来来确定是否激活登陆按钮。

在`viewDidLoad`方法的最后，添加以下代码：

```objective-c
RACSignal *signUpActiveSignal =
  [RACSignal combineLatest:@[validUsernameSignal, validPasswordSignal]
                    reduce:^id(NSNumber *usernameValid, NSNumber *passwordValid) {
                      return @([usernameValid boolValue] && [passwordValid boolValue]);
                    }];
```

以上代码使用__`combineLatest:reduce:` __方法来组合由`validUsernameSignal`和`validPasswordSignal`信号发出的最新值，并生成一个新的信号。每次这两个信号中的任何一个发出新的值，都会导致`reduce` block 执行。

> __注意：__RACSignal 组合方法可以组合任意数量的信号，reduce block 的参数则对应着每一个源信号的输出值。RAC 在底层做了很多的事情，查看它的实现是很有价值的。

既然已经生成了一个合适的信号，可以把下面的代码添加到`viewDidLoad`方法的末尾，它会去更新按钮的 enabled 属性：

```objective-c
[signUpActiveSignal subscribeNext:^(NSNumber *signupActive) {
   self.signInButton.enabled = [signupActive boolValue];
 }];

```

同样的，移除下面的非响应实现代码：

```objective-c
@property (nonatomic) BOOL passwordIsValid;
@property (nonatomic) BOOL usernameIsValid;
```

以及`viewDidLoad`方法中的实现：

```objective-c
// handle text changes for both text fields
[self.usernameTextField addTarget:self
                           action:@selector(usernameTextFieldChanged)
                 forControlEvents:UIControlEventEditingChanged];
[self.passwordTextField addTarget:self 
                           action:@selector(passwordTextFieldChanged)
                 forControlEvents:UIControlEventEditingChanged];
```

然后移除`updateUIState`、`usernameTextFieldChanged`和`passwordTextFieldChanged`方法，移除调用它们的代码。然后编译运行，__Sign In__ 按钮就会在用户名密码合法的情况下被激活，逻辑展示如下图：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/CombinePipeline.png" alt="RAC 数据流图"/></div>

上图展现了一些重要的概念，理解这些概念有助于你使用 RAC 实现一些强大的功能：

* __分割__  信号可以有多个订阅者，可以成为多个管道的信号源。在上图中，反映用户名密码是否合法的 boolean 信号就被两个管道接收用于不同的目的。
* __组合__  多个信号可以被组合生成新的信号，在上面的例子中，两个 boolean 信号就被组合了起来。组合起来的信号可以生成包含任何类型值的新信号。

使用 RAC 的结果就是现在应用里面再也没有反映两个输入框输入内容是否合法的变量。__这将是你采用响应式编程风格的一个关键原因——不再需要使用实例变量来跟踪可变状态。__

> 【译者注】这或许是一个原因，但是这个例子中并不明显，因为即使不用 RAC，也可以不用变量保存状态就实现上面的功能。

## Reactive Sign-in

示例应用现在使用响应式管道来管理输入框和按钮的状态。然而按钮的点击处理使用的还是 action，所以下一步就是将这块也改造成响应式的！

__Sign In__ 按钮上的 __Touch Up Inside__ 事件会触发`signInButtonTouched`事件。我们首先移除掉这个关系。

> 【译者注】原文中这个 Action 的是通过 Storyboard 添加的，这里移除的过程就不作翻译了。

前面已经展示了如何利用 RAC 框架给标准的 UIKit Control 添加属性和方法了，到目前为止我们已经使用过`rac_textSignal` —— 在文本变化的时候触发事件。为了能够处理事件，我们需要使用另外一个添加进去的方法：`rac_signalForControlEvents`。

回到`RWViewController.m`，将下面的代码添加到`viewDidLoad`方法末尾：

```objective-c
[[self.signInButton
   rac_signalForControlEvents:UIControlEventTouchUpInside]
   subscribeNext:^(id x) {
     NSLog(@"button clicked");
   }];
```

以上代码从按钮的`UIControlEventTouchUpInside`事件中创造出了一个信号，并添加了一个订阅上去：每次在事件发生的时候，都打印一次日志。

编译并运行以确认日志实际会打印。记住按钮只有在用户名和密码都合法的情况下才可点击，所以请在两个输入框中输入合法的内容，点击之后就可以看到控制台的输出如下：

```xml
2013-12-28 08:05:10.816 RWReactivePlayground[18203:a0b] button clicked
2013-12-28 08:05:11.675 RWReactivePlayground[18203:a0b] button clicked
2013-12-28 08:05:12.605 RWReactivePlayground[18203:a0b] button clicked
2013-12-28 08:05:12.766 RWReactivePlayground[18203:a0b] button clicked
2013-12-28 08:05:12.917 RWReactivePlayground[18203:a0b] button clicked
```

既然这个按钮有一个为 Touch 事件而生的信号，那么下一步就是将信号绑定到登陆处理上。打开`RMDummySignInService.h`看一下接口：

```objective-c
typedef void (^RWSignInResponse)(BOOL);
 
@interface RWDummySignInService : NSObject
 
- (void)signInWithUsername:(NSString *)username
                  password:(NSString *)password 
                  complete:(RWSignInResponse)completeBlock;
 
@end
```

这个服务以用户名、密码以及一个 completion block 作为参数，这个 block 在登陆成功或者失败只会被调用，我们可以直接在前面打印 log 的地方替换上对这个接口的调用。

> __注意：__为了实现简单，这个服务是 Mock 的，那样就不用依赖其余的外部 API 了。
>
> 现在我们遇到了一个真正的问题，我们如何去利用没有用信号形式表达过的 API？

## Creating Signals

实际上使用信号来适配一个异步接口是非常简单的。首先，从`RWViewController.m`中移除`signInButtonToouched`方法，后面会用响应式的代码替代它。在`RWViewController.m`中添加如下方法：

```objective-c
-(RACSignal *)signInSignal {
  return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [self.signInService
     signInWithUsername:self.usernameTextField.text
     password:self.passwordTextField.text
     complete:^(BOOL success) {
       [subscriber sendNext:@(success)];
       [subscriber sendCompleted];
     }];
    return nil;
  }];
}
```

以上方法调用了`RACSignal`类的`createSignal:`方法创建了一个以当前输入的用户名和密码登陆的信号。创建信号只需要一个 block 作为参数，当有订阅者订阅这个信号之后，block 里面的代码就会执行。block 本身也只需要一个参数：`subscriber`，它实现了`RACSubscriber`协议， 这个协议有一些方法用于发送事件：你可以发送任意数量的 next 类型事件，最后再发送一个 error/complete 类型的事件作为结束。在上面的例子中，它发送了一个 next 事件来反映登陆是否成功，然后发送一个 complete 类型事件。

block 的返回值类型是一个`RACDisposable`对象，它允许使用者在一段订阅关系需要取消或者删除的时候进行一些清理工作。这个信号没有任何的清理需求，因此返回 nil。

如你所见，将一个异步接口包裹在信号中是多么的简单！

现在我们来利用一下这个新的信号，更新之前添加到`viewDidLoad`方法中的代码：

```objective-c
[[[self.signInButton
   rac_signalForControlEvents:UIControlEventTouchUpInside]
   map:^id(id x) {
     return [self signInSignal];
   }]
   subscribeNext:^(id x) {
     NSLog(@"Sign in result: %@", x);
   }];
```

上面的代码使用了前面提到的`map`方法将按钮点击信号转换为登陆信号，订阅者只是简单的打印结果。

现在编译运行代码，然后点击 __Sign In__ 按钮，就可以看到控制台输出如下 Log：

```xml
2014-01-08 21:00:25.919 RWReactivePlayground[33818:a0b] Sign in result:
                                   <RACDynamicSignal: 0xa068a00> name: +createSignal:
```

`subscribeNext:` block接收到的是一个信号，即登陆信号本身，而不是登陆信号的结果。下面的图展示了前面代码的逻辑：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/SignalOfSignals.png" alt="RAC 数据流图"/></div>

当点击按钮的时候`rac_signalForControlEvents`会触发一个 next 事件（事件数据就是 UIButton），map 则把这个 next 事件转化成了一个登陆信号，也就是说后续的处理步骤中将接收到一个 RACSignal，这就是我们在`subscribeNext:`方法中观察的对象。

订阅登陆信号本身当然不是我们期望的，解决办法称为__信号的信号（signal of signals）__，换句话说也就是外部信号包含着一个内部信号。如果你确实有需要，是可以通过调用外部信号的`subscribeNext:`方法来订阅内部信号的。然而这样会造成嵌套混乱，但庆幸的是这是一个很普遍的问题，RAC 已经为此准备好了解决方案。

## Signal of Signals

问题的解决方案非常直接，只需要将`map`函数换成`flattenMap`：

```objective-c
[[[self.signInButton
   rac_signalForControlEvents:UIControlEventTouchUpInside]
   flattenMap:^id(id x) {
     return [self signInSignal];
   }]
   subscribeNext:^(id x) {
     NSLog(@"Sign in result: %@", x);
   }];
```

这样同样会把按钮的点击事件转换为登陆信号，但是也会把内部信号的事件透传到外部信号，从而转化为外部信号量的事件，这就是__`flatten`(扁平化)__。

编译并运行代码，控制台的输出就会变成如下样子：

```xml
2013-12-28 18:20:08.156 RWReactivePlayground[22993:a0b] Sign in result: 0
2013-12-28 18:25:50.927 RWReactivePlayground[22993:a0b] Sign in result: 1
```

现在整个管道就像我们期望的那样运行，最后一步就是把登陆成功/失败后的逻辑添加到`subscribeNext`的 block 中去：

```objective-c
[[[self.signInButton
  rac_signalForControlEvents:UIControlEventTouchUpInside]
  flattenMap:^id(id x) {
    return [self signInSignal];
  }]
  subscribeNext:^(NSNumber *signedIn) {
    BOOL success = [signedIn boolValue];
    self.signInFailureText.hidden = success;
    if (success) {
      [self performSegueWithIdentifier:@"signInSuccess" sender:self];
    }
  }];
```

这个 block 现在根据登陆信号量发送的事件更新`signFailureText`的可见性，并进行相应的导航。

编译并运行代码，我们就可以看到猫咪啦：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/ReactivePlaygroundStarter.jpg" width="320" alt="ReactivePlayground应用示例图"/></div>

不知道你有没有注意到当前的应用有一个很小的用户体验问题：当在登陆验证的过程中，是否应该禁掉 __Sign In__ 按钮的点击？这可以防止用户重复点击登陆，另外，如果登陆失败，用户再次登陆的时候应该隐藏掉错误信息。

但是这个逻辑怎么添加到当前的管道中去呢？更改按钮的可点击状态并不是一个转换、过滤或者前面提到的任何一个操作概念。它被认为是一个 _side effect_ ，或者更准确一点说，是一个 next 事件发生时要做的一点额外工作，但是又不会改变事件的本身。

## Adding side-effect

替换前面的代码如下：

```objective-c
[[[[self.signInButton
   rac_signalForControlEvents:UIControlEventTouchUpInside]
   doNext:^(id x) {
     self.signInButton.enabled = NO;
     self.signInFailureText.hidden = YES;
   }]
   flattenMap:^id(id x) {
     return [self signInSignal];
   }]
   subscribeNext:^(NSNumber *signedIn) {
     self.signInButton.enabled = YES;
     BOOL success = [signedIn boolValue];
     self.signInFailureText.hidden = success;
     if (success) {
       [self performSegueWithIdentifier:@"signInSuccess" sender:self];
     }
   }];
```

这里展示了如何在管道中 Button 的点击事件发生之后添加一个`doNext:`步骤。`doNext:`步骤不反悔任何的值，因为它只是一个 side-effect，它不会改变事件。

> 【译者注】doNext 所做的事情这里就不解释了。

所以整个流程又变成下图所示：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/SideEffects.png" alt="RAC 数据流图"/></div>

编译并运行代码，确认 __Sign In__ 按钮的可点击状态如预期那样变化。

到这里，所有的工作都完成了，现在整个 App 完全变成响应式的了。Nice！

如果前面有什么不明白的地方，可以下载最终的代码：__[final project](https://koenig-media.raywenderlich.com/uploads/2014/01/ReactivePlayground-Final.zip)__，或者直接从 [GitHub](https://github.com/ColinEberhardt/RWReactivePlayground) 拉取，每一个 commit 都对应着前面的一个步骤。

> __注意：__当某些异步任务进行时禁掉 button 的点击是一个普遍功能，RAC 提供了 RACCommand 来解决这个问题，有兴趣可以研究一下。

## 结论

希望这篇教程为你在自己的应用中使用 RAC 打下了一个良好的基础。熟悉这些概念需要一定的联系，但是和学习语言或者编程一样，一旦你适应了，一切就变得很简单。RAC 的核心就是信号，即事件流，还有什么比这更简单的呢？

使用 RAC 让我觉得有趣的一点是解决同一个问题有无数种解决方案，你可以着用这个示例应用进行更多的探索，让信号和管道以别的方式进行分割和组合。

使用 RAC 的目的是让代码更加整洁，更易理解，记住这个非常有必要。个人观点是将应用的逻辑使用链式语法表达为清晰的管道确实让理解变得更为简单。

在本教程的第二部分，你会学习到诸如错误处理、如何管理不同的线程等高级主题，玩的愉快！
