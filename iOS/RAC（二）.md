---
title: 【译】ReactiveCocoa 入门教程（下半部分）
tags: [iOS, Reactive, RAC]
categories: iOS
date: 2017-01-17 07:06:37
---

翻译自：[ReactiveCocoa Tutorial – The Definitive Introduction: Part 2/2](https://www.raywenderlich.com/62796/reactivecocoa-tutorial-pt2)

RAC 是 iOS 上一个函数响应式编程框架。通过本教程的上半部分学习，你应该知道如何将应用中的标准 Action 以及事件处理逻辑替换成可以发送事件的信号，也学会了如何转换、分割、组合这些信号。在下半部分教程中，我们将学习一些 RAC 的高级主题，包括：

* 另外两种事件类型：error 和 completed
* Throttling
* Threading
* Continuations
* 更多其他的...

Let's go!

## Twitter Instant

在下半部分教程中，你要开发的是一个叫做 Twitter Instant（来自 Google Instant 概念）的应用：一个 Twitter 即时搜索应用。<!-- More -->

[初始项目](https://koenig-media.raywenderlich.com/uploads/2014/01/TwitterInstant-Starter2.zip)中包含了一些基础的 UI 界面以及一些必须的代码。在教程的上班部分中，需要使用 CocoaPods 来获取 RAC 框架并集成进项目，这个初始项目已经包含必须的 Podfile，所以可以直接打开命令行执行命令：

```commonlisp
pod install
```

> 【译者注】这里编译打开初始项目的过程略去不译。

编译运行后可以看到如下界面：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/TwitterInstantStarter.png" alt="ReactivePlayground应用示例图"/></div>

你可以花一点时间熟悉一下应用代码。这是一个非常简单的基于 UISplitViewController 的应用，左边是 __RWSearchFormViewController__，添加了少量的 UIControls，包括一个搜索框；右边是 __RWSearchResultsViewController__，继承于 __UITableViewController__。

打开 __RWSearchResultsViewController.h__ 文件，你会看到`viewDidLoad`方法将右边展示搜索结果的 ViewController 赋值给了私有属性 __resultsViewController__，应用的大部分逻辑都在 __RWSearchResultsViewController__ 中，这个属性会将搜索结果传递给 __RWSearchResultsViewController__ 。

## Validating the Search Text

我们要做的第一件事情就是验证搜索输入框的内容确保输入的内容长度大于 2，这对于上半部分的学习是一个很好的复习。在__RWSearchFormViewController.m__ 的`viewDidLoad`方法之后添加如下方法：

```objective-c
- (BOOL)isValidSearchText:(NSString *)text {
  return text.length > 2;
}
```

然后在`viewDidLoad`方法的最后添加如下代码：

```objective-c
[[self.searchText.rac_textSignal
  map:^id(NSString *text) {
    return [self isValidSearchText:text] ?
      [UIColor whiteColor] : [UIColor yellowColor];
  }]
  subscribeNext:^(UIColor *color) {
    self.searchText.backgroundColor = color;
  }];
```

来想想这段代码在做什么？

* 获取输入框的内容信号
* 根据内容是否合法将这个信号对应的背景颜色值
* 将背景颜色值通过`subscribeNext:`的 block 设置为搜索输入框的背景；

编译运行代码，就可以看到当输入过短的时候，输入框的颜色是黄色的：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/ValidatedTextField.png" alt="ReactivePlayground应用示例图"/></div>

以上逻辑用图片表达为：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/TextValidationPipeline.png" alt="ReactivePlayground应用示例图"/></div>

`rac_textSignal`信号在每次输入框内容变化的时候释放携带当前搜索输入框输入内容的 next 事件，`map`则将输入内容转换为颜色，最终由`subscribeNext:`方法将颜色值设置为输入框的背景颜色。

当然，看过上半部分教程之后你肯定还记得这些，如果不记得了，最好回过头去好好看看上半部分教程。

在添加 Twitter 搜索逻辑之前，我们先讨论一些有意思的主题。

## Formatting of Pipelines

当我们想要格式化 RAC 代码的时候，普遍的约定是每一个操作都新起一行，然后每一步都垂直对齐，下面就是之前示例的一个格式化展示：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/PipelineFormatting.png" alt="ReactivePlayground应用示例图"/></div>

这样这个管道是由哪些步骤组成的就一目了然。同时也要注意，每个 block 中的代码都要尽量少，超过一定行数之后最好使用一个私有的方法去表达。

> 【译者注】如果 block 内容过长，会导致 RAC 代码的可阅读性大大降低。

不幸的是 Xcode 并不支持这种风格的格式化，所以得开发者自己维护这样的缩进。

## Memory Management

回过头来看看我们添加到 __TwitterInstant__ 应用中的代码，你是否会好奇你刚刚创建的管道是如何被维持在内存中的呢？它既没有被赋值给一个本地变量，也没有赋值给一个属性，是不是它的引用计数不会被增加，一定会被回收掉？

实际上 RAC 的设计目标之一就是允许创建匿名管道这样的编码方式，在前面我们写过的所有 RAC 代码中，这个特性显得自然直观。

为了支持这个模型，RAC 维护这自己的信号全局集合。如果一个信号有一个或者多个订阅者，则信号是激活状态，如果没有订阅者，则信号会被回收掉。这就引起了另外一个问题：我们如何取消订阅一个信号？在信号发送 error/completed 事件之后，订阅关系会被自动移除掉（后面会马上说到），而手动的取消订阅则可以通过`RACDisposable`来进行。

订阅`RACSignal`的时候都会返回一个`RACDisposable`实例，这个实例就允许开发者手动解除订阅关系，下面是一个例子：

```objective-c
RACSignal *backgroundColorSignal =
  [self.searchText.rac_textSignal
    map:^id(NSString *text) {
      return [self isValidSearchText:text] ?
        [UIColor whiteColor] : [UIColor yellowColor];
    }];
 
RACDisposable *subscription =
  [backgroundColorSignal
    subscribeNext:^(UIColor *color) {
      self.searchText.backgroundColor = color;
    }];
 
// at some point in the future ...
[subscription dispose];
```

你会发现你很少去做这件事情，但是应该要知道这种操作的存在。

> __注意：__由此推出，如果你创建了一个管道但是不去订阅它，那这个管道就永远不会执行，包括`doNext:`这样的 side-effect block。

## Avoiding Retain Cycles

RAC 虽然为此做了很多幕后工作，让开发者不必担心有关信号的内存管理，但是仍然有一种情况需要开发者注意。我们来看看前面添加的代码：

```objective-c
[[self.searchText.rac_textSignal
  map:^id(NSString *text) {
    return [self isValidSearchText:text] ?
      [UIColor whiteColor] : [UIColor yellowColor];
  }]
  subscribeNext:^(UIColor *color) {
    self.searchText.backgroundColor = color;
  }];
```

`subscribeNext:` block 在内部使用了 self 来引用外部的输入框，block 持有对 self 的强引用，因此如果 self 对 block 也有强引用，那么就会形成一个循环引用。这个是否会引起问题取决于 self 对象本身，如果 self 贯穿应用的生命周期，就如上面的例子所示，那么这个问题就没那么严重，但是在更为复杂的情况下，恐怕就不是这样的了。

为了避免潜在的循环引用，苹果官方的文档 [Working With Blocks](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html#//apple_ref/doc/uid/TP40011210-CH8-SW16) 建议在 block 中引用 self 的弱引用，如下：

```objective-c
__weak RWSearchFormViewController *bself = self; // Capture the weak reference
 
[[self.searchText.rac_textSignal
  map:^id(NSString *text) {
    return [self isValidSearchText:text] ?
      [UIColor whiteColor] : [UIColor yellowColor];
  }]
  subscribeNext:^(UIColor *color) {
    bself.searchText.backgroundColor = color;
  }];
```

bself 是指向 self 的使用 __weak 修饰的变量，也就是指向 self 的弱引用，而在`subscribeNext:` block 块中，我们把 self 替换成 bself，这样就打破了循环引用。不过这看上去并不那么优雅。

RAC 框架在这里玩了一点小把戏，我们可以导入以下头文件：

```objective-c
#import "RACEXTScope.h"
```

然后用下面的代码实现等同的效果：

```objective-c
@weakify(self)
[[self.searchText.rac_textSignal
  map:^id(NSString *text) {
    return [self isValidSearchText:text] ?
      [UIColor whiteColor] : [UIColor yellowColor];
  }]
  subscribeNext:^(UIColor *color) {
    @strongify(self)
    self.searchText.backgroundColor = color;
  }];
```

`@weakify`和`@strongify`语句是定义在 [Extended Objective-C ](https://github.com/jspahrsummers/libextobjc)中的宏，它们也被包含在了 RAC 框架中，`@weakify`宏允许你创建弱引用变量（如果有需要，还可以传入多个变量作为参数），`@strongify`宏则允许你根据之前传入`@weakify`的参数创建强引用。

> __注意：__如果你对这两个宏的具体实现感兴趣，可以在 Xcode 中选择 __Product->Perform Action->Preprocess “RWSearchForViewController”.__ 这会预处理一遍当前的 view controller，展开所有的宏，开发者将可以看到最终的输出。

最后需要注意的是：在 block 内部使用实例变量也会造成 block 持有对 self 的强引用，你可以打开编译器的告警开关，在代码出现这种问题的时候警告你，在项目的 build settings 中搜索 retain 就可以看到这个选项：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/AvoidRetainSelf.png" alt="ReactivePlayground应用示例图"/></div>

好了，你应该已经掌握这个要点了，恭喜！现在你可以继续学习剩下的有趣部分了：为你的应用添加一些真正的功能。

> 眼尖的读者肯定注意到这里实际不必使用`subscribeNext:`方法，使用 RAC 替代即可，如果你发现了这点，请改掉它，并奖励给自己一个亮闪闪的✨。

## Requesting Access to Twitter

接下来你将要使用 __Social Framework__ 让 TwitterInstant 应用搜索 Tweets，使用 __Accounts Framework__ 来授权访问 Twitter。为了更好的了解 __Social Framework__，可以查看专门为此准备的章节：[iOS 6 by Tutorials](https://www.raywenderlich.com/?page_id=19968)。

在你添加这段代码之前，你需要将 Twitter 认证信息添加到模拟器或者运行应用的设备上。打开__设置__应用然后选择__Twitter__菜单选项，添加你的身份信息：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/TwitterCredentials.png" alt="ReactivePlayground应用示例图"/></div>

初始项目已经包含所需要的 Framework，所以你只需要导入 header 就好。在`RWSearchFormViewController.m`文件中，添加以下语句：

```objective-c
#import <Accounts/Accounts.h>
#import <Social/Social.h>
```

在`import`语句下面添加如下枚举类型和常量值：

```objective-c
typedef NS_ENUM(NSInteger, RWTwitterInstantError) {
    RWTwitterInstantErrorAccessDenied,
    RWTwitterInstantErrorNoTwitterAccounts,
    RWTwitterInstantErrorInvalidResponse
};
 
static NSString * const RWTwitterInstantDomain = @"TwitterInstant";
```

接下去你会用这些枚举类型来定义一些错误。接着添加如下属性：

```objective-c
@property (strong, nonatomic) ACAccountStore *accountStore;
@property (strong, nonatomic) ACAccountType *twitterAccountType;
```

`ACAccountStore`类可以帮助链接各种社交媒体账号，`ACAccountType`类代表一种特定类型的账号。

接着在`viewDidLoad`方法末尾添加如下代码：

```objective-c
self.accountStore = [[ACAccountStore alloc] init];
self.twitterAccountType = [self.accountStore 
  accountTypeWithAccountTypeIdentifier:ACAccountTypeIdentifierTwitter];
```

这段代码创建了 account store 以及 Twitter 账号标记。

当一个 App 想要访问一个社交媒体账号，用户会看到一个弹窗，这是一个异步操作，因此将它包裹在信号中，是它变成响应式是很好的选择。我们在代码中添加如下方法：

```objective-c
- (RACSignal *)requestAccessToTwitterSignal {
 
  // 1 - define an error
  NSError *accessError = [NSError errorWithDomain:RWTwitterInstantDomain
                                             code:RWTwitterInstantErrorAccessDenied
                                         userInfo:nil];
 
  // 2 - create the signal
  @weakify(self)
  return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    // 3 - request access to twitter
    @strongify(self)
    [self.accountStore
       requestAccessToAccountsWithType:self.twitterAccountType
         options:nil
      completion:^(BOOL granted, NSError *error) {
          // 4 - handle the response
          if (!granted) {
            [subscriber sendError:accessError];
          } else {
            [subscriber sendNext:nil];
            [subscriber sendCompleted];
          }
        }];
    return nil;
  }];
}
```

这个方法做了以下事情：

1. 定义了一个 error，如果用户拒绝 App 访问账号，这个错误会被返回；
2. 和上半部分教程一样，方法`createSignal`返回了`RACSignal`类的一个实例；
3. 访问 Twitter 的请求是通过 account store 发出的，用户会看到一个弹窗来请求用户授予权限；
4. 当用户授予 / 拒绝访问权限之后，信号就会被发送。如果授予了权限，next 事件之后会发送 completed 事件，如果请求被拒绝，error 事件会被发送；

如果你还记得上半部分教程，其中提到过信号可以发送三种类型的事件：next、completed 和 error。在一个信号的生命周期中，它可以不发送任何事件，发送一个或者多个 next 事件并以一个 completed / error 事件结束。

最后，为了使用这个信号，将以下代码添加到`viewDidLoad`方法后面：

```objective-c
[[self requestAccessToTwitterSignal]
  subscribeNext:^(id x) {
    NSLog(@"Access granted");
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

编译运行代码，将会看到以下弹窗：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/RequestAccessToTwitter.png" alt="ReactivePlayground应用示例图"/></div>

如果你点击 __OK__，控制台会输出：Access granted；点击 __Don't Allow__，控制台会输出：An error occurred:xxx。

Accounts Framework 会记住你的选择，因此如果需要测试两种情况，需要重置 __iOS Simulator -> Reset Contents and Settings__ 菜单选项，这会有点麻烦，因为你必须再输入一遍你的 Twitter 账号。

## Chaining Signals

一旦用户授予 Twitter 账号权限，应用就需要持续的监听搜索输入框的变化，及时搜索 Twitter。在 `viewDidLoad`方法最后添加如下代码：

```objective-c
[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

`then`方法会在收到 completed 事件之后，立刻订阅由`then`返回的信号，用于高效的从订阅一个信号转移到订阅另外一个。

> __注意：__前面已经声明了一个@weakify(self)，这里不用重复声明。

error 事件会穿过`then`方法，因此后面的`subscribeNext:error:`  block 依然能够接收到前面权限请求所发出的 error 事件。

编译运行代码，然后授予权限。在输入搜索内容的同时，可以看到控制台的如下输出：

```xml
2014-01-04 08:16:11.444 TwitterInstant[39118:a0b] m
2014-01-04 08:16:12.276 TwitterInstant[39118:a0b] ma
2014-01-04 08:16:12.413 TwitterInstant[39118:a0b] mag
2014-01-04 08:16:12.548 TwitterInstant[39118:a0b] magi
2014-01-04 08:16:12.628 TwitterInstant[39118:a0b] magic
2014-01-04 08:16:13.172 TwitterInstant[39118:a0b] magic!
```

接着，可以为管道添加过滤器，过滤非法的搜索关键字，在这里我们定义搜索字符数少于3个即为非法：

```objective-c
[[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  filter:^BOOL(NSString *text) {
    @strongify(self)
    return [self isValidSearchText:text];
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

下图展示了当前的管道逻辑：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/PipelineWithThen.png" alt="ReactivePlayground应用示例图"/></div>

应用程序管道由`requestAccessToTwitterSignal`开始，然后切换到`rac_textSignal`信号，同时，next 类型事件经过`filter`方法过滤，最终到达订阅 block，你可以看到第一步触发的 error 事件还是会被最后的`subscribeNext:error:` block 消费。

> 【译者注】error 事件的这种特性使得管道可以在一个统一的地方处理错误。

既然已经有了一个信号用于通知搜索内容的变化，那接下来是时候用它来搜索 Twitter 了！你觉得有趣吗？我觉得应该是觉得有趣的，因为你已经真正了解一些东西了。

## Searching Twitter

__Social Framework__ 是获取 Twitter 搜索 API 的一种渠道。然而 __Social Framework__ 并不是响应式的，我们要做的就是用信号去包裹这些 API。你现在应该去掌握这样做的窍门了！

在`RWSearchFormViewController.m`内部，添加如下的方法：

```objective-c
- (SLRequest *)requestforTwitterSearchWithText:(NSString *)text {
  NSURL *url = [NSURL URLWithString:@"https://api.twitter.com/1.1/search/tweets.json"];
  NSDictionary *params = @{@"q" : text};
 
  SLRequest *request =  [SLRequest requestForServiceType:SLServiceTypeTwitter
                                           requestMethod:SLRequestMethodGET
                                                     URL:url
                                              parameters:params];
  return request;
}
```

这里使用 [v1.1 REST API](https://dev.twitter.com/docs/api/1.1) 创建了一个搜索 Twitter 的请求，以上代码使用 __q__ 关键字来搜索包含输入内容的 Twitter，你可以从  [Twitter API docs](https://dev.twitter.com/docs/api/1.1/get/search/tweets) 中了解更多关于搜索 API 的细节。

下面一步就是基于这个请求创建一个信号，添加如下方法：

```objective-c
- (RACSignal *)signalForSearchWithText:(NSString *)text {
 
  // 1 - define the errors
  NSError *noAccountsError = [NSError errorWithDomain:RWTwitterInstantDomain
                                                 code:RWTwitterInstantErrorNoTwitterAccounts
                                             userInfo:nil];
 
  NSError *invalidResponseError = [NSError errorWithDomain:RWTwitterInstantDomain
                                                      code:RWTwitterInstantErrorInvalidResponse
                                                  userInfo:nil];
 
  // 2 - create the signal block
  @weakify(self)
  return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    @strongify(self);
 
    // 3 - create the request
    SLRequest *request = [self requestforTwitterSearchWithText:text];
 
    // 4 - supply a twitter account
    NSArray *twitterAccounts = [self.accountStore
      accountsWithAccountType:self.twitterAccountType];
    if (twitterAccounts.count == 0) {
      [subscriber sendError:noAccountsError];
    } else {
      [request setAccount:[twitterAccounts lastObject]];
 
      // 5 - perform the request
      [request performRequestWithHandler: ^(NSData *responseData,
                                          NSHTTPURLResponse *urlResponse, NSError *error) {
        if (urlResponse.statusCode == 200) {
 
          // 6 - on success, parse the response
          NSDictionary *timelineData =
             [NSJSONSerialization JSONObjectWithData:responseData
                                             options:NSJSONReadingAllowFragments
                                               error:nil];
          [subscriber sendNext:timelineData];
          [subscriber sendCompleted];
        }
        else {
          // 7 - send an error on failure
          [subscriber sendError:invalidResponseError];
        }
      }];
    }
 
    return nil;
  }];
}
```

这段代码执行的功能步骤如下：

1. 首先定义了一些不同的 error，一个用于表示用户没有添加任何的 Twitter 账号，一个用于表示查询出错；
2. 创建信号；
3. 利用前面添加的方法创建一个请求；
4. 查询 account store 的第一个账号，如果没有任何账号可查询，报错；
5. 执行请求；
6. 如果请求正确返回（Http Status Code 为 200），解析返回的 JSON 数据，发送 next 事件和 completed 事件；
7. 如果请求失败，发送 error 事件；

现在我们来使用这个新创建的信号。在本教程的上半部分，你学会了使用`flattenMap`方法来"扁平化"信号，现在又到了这个方法施展身手的时候，在`viewDidLoad`方法的结尾，添加`flattenMap`方法如下：

```objective-c
[[[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  filter:^BOOL(NSString *text) {
    @strongify(self)
    return [self isValidSearchText:text];
  }]
  flattenMap:^RACStream *(NSString *text) {
    @strongify(self)
    return [self signalForSearchWithText:text];
  }]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

编译运行代码，在搜索框中输入搜索内容，一旦字符长度大于等于 3 个，你就可以在控制台看到 Twitter 的搜索返回结果：

```xml
2014-01-05 07:42:27.697 TwitterInstant[40308:5403] {
    "search_metadata" =     {
        "completed_in" = "0.019";
        count = 15;
        "max_id" = 419735546840117248;
        "max_id_str" = 419735546840117248;
        "next_results" = "?max_id=419734921599787007&q=asd&include_entities=1";
        query = asd;
        "refresh_url" = "?since_id=419735546840117248&q=asd&include_entities=1";
        "since_id" = 0;
        "since_id_str" = 0;
    };
    statuses =     (
                {
            contributors = "<null>";
            coordinates = "<null>";
            "created_at" = "Sun Jan 05 07:42:07 +0000 2014";
            entities =             {
                hashtags = ...
```

## Threading

我想你一定非常迫切的想要在 UI 上展示搜搜得到的数据，但是在此之前你还有一件事情要做，想知道要做什么，我们得做点小实验。

在`subscribeNext:error:`的 block 中添加断点，如下图所示：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/BreakpointLocation.png" alt="ReactivePlayground应用示例图"/></div>

重新运行程序，如果有必要重新输入 Twitter 的认证信息，输入一些搜索内容，然后就运行到断电处了，类似下面：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/BreakpointResult.png" alt="ReactivePlayground应用示例图"/></div>

我们观察到这段代码并没有运行在主线程（Thread 1）上，而我们首先应当记住的就是只能在主线程更新 UI，因此如果你要展示数据，就必须切换线程。

这里引出了 RAC 框架的一个关键点：以上的代码运行在事件发送的线程上。可以尝试在管道的其余步骤上议案家断点：你会发现它们运行在不止一条线程上。

所以你该如何更新 UI 呢？经典做法就是使用 Operation Queue（参见教程 [How To Use NSOperations and NSOperationQueues](https://www.raywenderlich.com/?p=19788)），但是 RAC 本身提供了更为简单的方法 —— `deliverOn:`，更新代码如下：

```objective-c
[[[[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  filter:^BOOL(NSString *text) {
    @strongify(self)
    return [self isValidSearchText:text];
  }]
  flattenMap:^RACStream *(NSString *text) {
    @strongify(self)
    return [self signalForSearchWithText:text];
  }]
  deliverOn:[RACScheduler mainThreadScheduler]]
  subscribeNext:^(id x) {
    NSLog(@"%@", x);
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

现在再运行程序，就可以看到断点处是在主线程上执行了：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/BreakpointNowOnUIThread.png" alt="ReactivePlayground应用示例图"/></div>

纳尼？调度到另外一个线程处理事件流就这么简单？不可思议！现在你可以放心的刷新你的 UI 了！

> __注意：__读者可以去详细了解一下 RACScheduler 类。

现在是时候展示 tweets 了。

## Updating  the UI

打开`RWSearchResultsViewController.h`文件你会发现这里已经有一个`displayTweets:`方法了，它会让右边的 View Controller 去显示相应的 tweet 数组。实现非常简单，只是填充一个 UITableView 的数据源。该方法期望的参数是一个`RWTweet`数组，这个类也已经在项目中提供好了。

现在到达`subscibeNext:error:`这一步的数据是一个 NSDictionary，所以你要怎么解析它的内容呢？

如果你去看 [Twitter API documentation](https://dev.twitter.com/docs/api/1.1/get/search/tweets)，你会看到一个结果响应示例：它就展示了 NSDictionary 的结构，所以你可以发现它有一个 key 叫做 statuses，其中包含着 tweet 数组，数组的内容也是一个个 NSDictionary 实例。而 `RWTweet`类已经包含解析这个 NSDictionary 的过程了，因此你要做的就是写一个循环，将返回消息里面的 statuses 数组转换为 RWTweet 数组。

但是你甚至连这个都不用做，我们还有更好的解决方案。本教程是有关 RAC 以及 函数式编程的，通过函数式 API 将数据从一种类型转化为另外一种类型会更加优雅，我们将使用 [LinqToObjectiveC](https://github.com/ColinEberhardt/LinqToObjectiveC) 来完成这个任务。

> 【译者注】具体怎么引入这个库这里就略过不译了。

在`RWSearchFormViewController.m`文件中添加如下文件引入：

```objective-c
#import "RWTweet.h"
#import "NSArray+LinqExtensions.h"
```

`NSArray+LinqExtensions.h`头文件来自 __LinqToObjectiveC__，它往 NSArray 中添加了一些方法让你可以变换、排序、分析、过滤数组中的数据，并且也是链式的！现在我们就来使用这个 API，更换`viewDidLoad`方法最后的管道代码：

```objective-c
[[[[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  filter:^BOOL(NSString *text) {
    @strongify(self)
    return [self isValidSearchText:text];
  }]
  flattenMap:^RACStream *(NSString *text) {
    @strongify(self)
    return [self signalForSearchWithText:text];
  }]
  deliverOn:[RACScheduler mainThreadScheduler]]
  subscribeNext:^(NSDictionary *jsonSearchResult) {
    NSArray *statuses = jsonSearchResult[@"statuses"];
    NSArray *tweets = [statuses linq_select:^id(id tweet) {
      return [RWTweet tweetWithStatus:tweet];
    }];
    [self.resultsViewController displayTweets:tweets];
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

如上面的代码所示，我们优雅地将原始的 NSDictionary 数据转换成了 RWTweet 对象。一旦转变完成，我们就可以在 result view controller 上展示对应的数据了。

编译运行代码，搜索之后就可以看到类似的 UI：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/FinallyWeSeeTweets.png" alt="ReactivePlayground应用示例图"/></div>

> __注意：__ RAC 和 LinqToObjectiveC 有着相似的思想。

## Asynchronous Loading of Images

你或许已经发现每一调 tweet 的左侧有一块很大的空白，这块地方实际是用于展示 Twitter 的用户头像的。RWTweet 类中实际已经有一个`profileImageUrl`属性用于记录获取头像的 URL 了，为了能够让 UITableView 流畅的滚动，我们需要保证头像获取的代码不在主线程上执行，这可以使用 GCD 或者 NSOperationQueue 来达到目的，但是为什么不使用 RAC 呢？

打开 __RWSearchResultsViewController.m__ 文件，在最后添加如下代码：

```objective-c
-(RACSignal *)signalForLoadingImage:(NSString *)imageUrl {
 
  RACScheduler *scheduler = [RACScheduler
                         schedulerWithPriority:RACSchedulerPriorityBackground];
 
  return [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:imageUrl]];
    UIImage *image = [UIImage imageWithData:data];
    [subscriber sendNext:image];
    [subscriber sendCompleted];
    return nil;
  }] subscribeOn:scheduler];
 
}
```

对这种模式，你应该非常熟悉了。以上代码获取到了一个后台调度器用于调度信号执行的线程，让它不要在主线程上执行，接着它创建了一个信号用于下载图像数据并创建 UIImage，最后是调用`subscribeOn:`方法，用于保证信号是执行在指定的调度器上的。

现在在同一个文件中，更新`tableView:cellForRowAtIndex:`方法，在`return`语句之前添加：

```objective-c
cell.twitterAvatarView.image = nil;
 
[[[self signalForLoadingImage:tweet.profileImageUrl]
  deliverOn:[RACScheduler mainThreadScheduler]]
  subscribeNext:^(UIImage *image) {
   cell.twitterAvatarView.image = image;
  }];
```

以上代码首先重置了 cell 的图片，因为 cell 会被重用，很有可能携带者过期的数据，然后它创建了对应的信号来获取数据，`deliverOn:`管道步骤用于将`subscriberNext:`的 block 调度到主线程进行。

编译运行代码，可以看到如下的展示：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/AvatarsAtAlast.png" alt="ReactivePlayground应用示例图"/></div>

## Throttling

你或许已经发现每次输入一个新字符，应用都会理解发起一个 Twitter 搜索。如果你是一个快速输入者（或者只是长按着删除键），这会导致应用在短时间内发起大量请求，这并不是合理的：首先，这会对 Twitter 的搜索 API 造成压力，并且大部分的搜索结果都被丢弃了；其次，频繁的更新界面对于用户来说也是很烦人的一件事儿。

一个比较好的方案是在输入框的内容一段时间不变化之后再去发起搜索，比如，500ms。如你所想，RAC 又为此提供了简单的难以置信的解决方案！

打开`RWSearchFormViewController.m`，更新前面的管道代码如下：

```objective-c
[[[[[[[self requestAccessToTwitterSignal]
  then:^RACSignal *{
    @strongify(self)
    return self.searchText.rac_textSignal;
  }]
  filter:^BOOL(NSString *text) {
    @strongify(self)
    return [self isValidSearchText:text];
  }]
  throttle:0.5]
  flattenMap:^RACStream *(NSString *text) {
    @strongify(self)
    return [self signalForSearchWithText:text];
  }]
  deliverOn:[RACScheduler mainThreadScheduler]]
  subscribeNext:^(NSDictionary *jsonSearchResult) {
    NSArray *statuses = jsonSearchResult[@"statuses"];
    NSArray *tweets = [statuses linq_select:^id(id tweet) {
      return [RWTweet tweetWithStatus:tweet];
    }];
    [self.resultsViewController displayTweets:tweets];
  } error:^(NSError *error) {
    NSLog(@"An error occurred: %@", error);
  }];
```

`throttle`操作在收到一个 next 事件之后，如果一定时间内还没有收到下一个 next 事件，才会发送当前这个 next 事件。这真的非常简单。

编译运行代码，我们就可以看到效果，体验好多了是不是？

然后...到这里我们就完成了 Tweet Instant 应用的开发，回顾一下然后尽情地享受吧！同样的，如果有哪款地方有所遗漏或者不明白的，可以下载最终的代码：[final project](https://koenig-media.raywenderlich.com/uploads/2014/01/TwitterInstant-Final.zip)（记得运行`pod install`命令），或者直接从 [GitHub](https://github.com/ColinEberhardt/RWTwitterInstant) clone，每一个 commit 都对应着上面的一个步骤。

## Wrap Up

在离开之前，我们有必要看一下最终整个应用的管道：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/CompletePipeline.png" alt="ReactivePlayground应用示例图"/></div>

这个流程非常的复杂，但是我们使用一根响应式管道就表达清楚了，看起来非常漂亮，不是吗？你能想象不用 RAC 这个应用会变得多么复杂么，它的流程又会变得多么难以理清？你现在再也不用经历这些了！

现在你应该知道 RAC 是多么的令人惊讶了！

最后一点，RAC 使得 MVVM（更好的分离应用逻辑和视图逻辑）的应用更为简单了，如果有人希望能看到如何使用 RAC 来实现 MVVM，请在评论里让我知道，我很希望听到你们的想法以及经历！

## 译者总结

这两篇文章虽然没有讲太多的理论，也没有对某个函数做深入的探讨，但是确实改变了我对响应式编程的看法。

如果你写过 Android，为 Button 添加监听的代码一般如下：

```java
button.setOnClickListener(new View.OnClickListener() {
	@Override
	public void onClick(View v) {
		// 点击事件发生
		// BLABLA...
	}
});
```

这种方法称为 __Callback__。如果你写过 iOS 代码，那么为 UIButton 添加监听事件的代码如下：

```objective-c
[button addTarget:self action:@selector(buttonClicked:) forControlEvents:UIControlEventTouchUpInside];
```

当然还要为此声明并实现一个`buttonClicked:`私有方法用于处理点击事件，这种方法称为 __Target-Action__。

这两种机制虽然叫法不一样，但我认为两者本质上都是一样的（且不讨论方法调用和消息发送），响应式编程也是使用的同样的机制 —— 就是 __观察者模式__。不觉得`subscribe`的过程就是`addListener`的过程么？

> PS：这里忽略所谓的观察者模式和监听回调模式的区别，个人认为做这种区分毫无意义。

这种典型的做法已经可以满足大部分的需求了，但是又存在什么问题呢？

- 相比较于 Android，iOS 的这种方式我是比较反感的，每次写这行代码我都得先去声明一个方法，查看代码的时候得跳转到方法体中查看方法逻辑然后再回来继续往下查看，Android 将绑定过程和绑定的内容放在一块儿，阅读起来更加方便；
- 即使 Android 能够为 “为 View 添加监听事件” 这样的事情提供比较优雅的方案，但是在 Android 上仍然存在一些恶心的长流程，当你想要去了解这个流程的时候，需要到处跳转代码查看；

总结一下：__非响应式代码的逻辑分散，不易阅读__。这就是响应式编码所要解决的问题！__响应式编码的价值在于：__

1）通过定义 next、error、complete 三种类型的事件规范了事件的表达；

2）建立事件流概念；

3）在事件流基础上建立了一系列的函数用于操作事件和事件流；

响应式编码会定义一个事件源，或者称之为 Signal；事件除了类型之外，还会携带任意类型的数据。当事件被触发之后（比如按钮被点击），事件源就会通过发送事件来告诉订阅者，这一点和前面 Andorid 的示例并无区别，甚至实现起来代码形式都差不多。但是关键在于：__响应式编码定义的操作是可以将一个信号源转换为另外一种信号源，换言之，它可以发送完全不同的事件出来，也可以将事件的数据处理成另外一种格式的数据再进行传送。__回顾一张[【译】ReactiveCocoa 入门教程（上半部分）](http://timebridge.space/2017/01/16/RAC/)中的图：

<div align="center"><img src="https://koenig-media.raywenderlich.com/uploads/2014/01/CombinePipeline.png" alt="RAC 数据流图"/></div>

如图，ran_textSignal 就是事件源，它发送的事件携带的数据是 NSString 类型的，经过`map`操作之后会变成 BOOL 类型，再经过一次`map`操作之后就会变成 UIColor 类型。从图中还可以看出：username 和 password 这两个信号源发出的时事件经过`combineLatest:reduce`操作之后会被合并为一个携带 BOOL 类型数据的事件。这种__事件/事件流的变换操作正是响应式编程的核心能力__！通过这种能力是可以比较优雅的解决掉前面提到的问题的（之所以说“比较优雅”，是因为要理解响应式编程定义的各种操作并不是一件容易的事情）。

以上，是我理解的响应式编程的精髓。