---
title: iOS 网络学习（二）—— NSURLSession
date: 2017-02-15 14:16:28
tags: [网络]
categories: iOS
---

## 简介

2013 年 WWDC 上，Apple 发布了 NSURLConnection 的继任者 [NSURLSession](https://developer.apple.com/reference/foundation/urlsession)，支持 iOS7.0+，而 NSURLCOnnection 在 iOS9 被宣布弃用。

## URL 加载系统

在学习 NSURLSession 之前，有必要先了解一下 URL 加载系统（URL Loading System）。

> 这个系统本来应该在 [iOS 网络学习（一）—— URLConnection](http://timebridge.space/2017/02/15/iOS-%E7%BD%91%E7%BB%9C%E5%AD%A6%E4%B9%A0%EF%BC%88%E4%B8%80%EF%BC%89/) 提及，但 NSURLConnection 本身比较简单，而且已经被废弃，NSURLSession 将会是后面网络请求的核心，因此在这里系统了解一下。

URL 加载系统，顾名思义就是根据 URL 从某个地方加载资源，这个“加载”是广义的，既包括从服务端下载资源，也包括上传资源。它支持 http、https、ftp、file、data 资源传输协议以及自定义扩充协议，由一组辅助类以及 Protocol 组成，主要分为五个部分：<!-- More -->

1. 配置管理
2. 缓存管理
3. cookie 存储
4. 认证和证书
5. 自定义协议支持

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/NSURLSession-Family.png" height="320" alt="One Piece"/></div>

实际上我们平时所说的 NSURLSession 指的就是上面一组类，而并不单单是指 NSURLSession 类本身，下面我们通过对 NSURLSession 用法的学习来了解一下这五个部分。

> 图中将 NSURLConnection 也列在其中，它也属于 URL 加载系统的一部分。

## 使用 NSURLSession

NSURLSession 以及相关类为 Http 请求提供了相关接口，为了使用 NSURLSession，需要我们创建一组 session 对象，每一个 session 对象负责执行一组数据传输任务。举个栗子，如果你在开发一个网页浏览器，你可能会为每一个 Tab 或者 Window 创建一个 session，每个 session 会负责一组传输任务，每个任务代表着对一个特定 URL 指向的资源的请求。

和 NSURLConnection 一样，NSURLSession 可以通过 Block 或者 Delegate 来接收处理请求返回的结果数据。Block 被设计为 Delegate 的替换选择，如果一个任务设置了 Block，那么 Delegate 就不会被回调。除此之外，NSURLSession 还提供了取消、暂停、唤醒任务的接口。

### URL Session 相关概念

session 中任务的行为取决于三个因素：1）session 的类型（取决于创建时候的配置）；2）任务的类型；3）任务创建的时候应用是否处于前台。

#### session 类型

NSURLSession 支持三种类型的 session，由创建时候的配置决定：

* __Default Session__ 默认 session 与 Foundation 中其他下载 URL 资源的方法类似，使用基于磁盘的缓存，并将证书存储在 keychain 中；
* __Ephemeral Session__ 这种类型的 session 不会在磁盘上存储任何类型的数据，所有的缓存、证书存储以及其他数据都会被保存在内存中，仅限于当前 session 可用，因此当你的应用关闭这个 session，相关数据会自动被清除；
* __Background Session__ 和 Default Session 类似，但是它是在独立的进程中进行数据传输的，并且还有一些额外的限制，后面会细说。

#### 任务类型

在一个 session 里面，NSURLSession 支持三种类型的任务：数据任务，下载任务和上传任务。

* __数据任务__ 使用 NSData 发送和接受数据，适用于比较信息量较少的服务端请求，数据任务可以将服务端返回的数据分批交给应用，又或者在全部接收完成之后通过 Block 一次性返回；
* __下载任务__ 以文件形式接收数据，并且支持当 App 不运行的时候后台下载；
* __上传任务__ 以文件形式发送数据，并且支持当 App 不运行的时候后台上传；

#### 后台数据传输考量

NSURLSession 支持当应用挂起的时候在后台进行数据传输，后台传输仅在使用 Background Session 配置的时候起作用。由于使用 Background Session 的时候下载任务是在独立的进程中进行的，且重启你的 App 有比较大的代价，因此使用这种模式有一些限制：

// TODO

#### 生命周期和 Delegate 交互

理解这个点取决于你使用 NSURLSession 类来干什么，有可能需要理解 session 生命周期，包括 session 如何与delegate 交互， delegate 接口的调用顺序，服务端重定向资源后会发生什么，当 App 重启一个失败的下载后会发生什么，等等。后面会详述。

#### NSCoping 行为

Session 和任务对象遵从 NSCopying 协议，并有如下实现：

* 拷贝 session 和任务对象，会得到相同的对象；
* 拷贝配置对象，会得到新的对象，可以独立修改；

### 创建并配置 Session

NSURLSession 提供了丰富的配置项：

* 单个 session 特有的存储配置，包括缓存、cookie、证书，以及协议；
* 绑定到单个任务或者 session 的身份认证信息；
* 上传和下载任务，并支持分离数据（文件内容）和元数据（URL 和 设置）；
* 配置到每个 Host 的最大连接数；
* 如果整个资源不能在某个时间内被下载下来，那么每个组成资源都会被处罚超时；
* 支持的 TLS 版本范围设置；
* 自定义代理字典；
* 缓存策略；
* Http 管道行为控制；

因为大部分的配置项都包含在一个单独的配置配置项中，开发者可以重用通用的设置，当一个 session 对象被初始化之后，需要完成：

* 一个控制 session 和 session 内任务行为的配置独享；
* 一个用于处理加载数据和事件的 delegate，进行服务端身份认证，决定一个资源加载是否需要被转化为下载等工作，这是可选的；

如果开发者不提供自定义 delegate， NSURLSession 会使用系统提供的 delegate。但是如果你要进行后台数据传输任务，就必须提供一个自定义的 delegate。

当一个 session 初始化完毕之后，开发者就无法再更改配置对象或者代理了，除非创建一个新的 session。

下面代码展示了如何三种类型的 session：

```objective-c
// Creating session configurations
NSURLSessionConfiguration *defaultConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
NSURLSessionConfiguration *ephemeralConfiguration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
NSURLSessionConfiguration *backgroundConfiguration = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier: @"com.myapp.networking.background"];
 
// Configuring caching behavior for the default session
NSString *cachesDirectory = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES).firstObject;
NSString *cachePath = [cachesDirectory stringByAppendingPathComponent:@"MyCache"];
 
/* Note:
 iOS requires the cache path to be
 a path relative to the ~/Library/Caches directory,
 but OS X expects an absolute path.
 */
#if TARGET_OS_OSX
cachePath = [cachePath stringByStandardizingPath];
#endif
 
NSURLCache *cache = [[NSURLCache alloc] initWithMemoryCapacity:16384 diskCapacity:268435456 diskPath:cachePath];
defaultConfiguration.URLCache = cache;
defaultConfiguration.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;
 
// Creating sessions
id <NSURLSessionDelegate> delegate = [[MySessionDelegate alloc] init];
NSOperationQueue *operationQueue = [NSOperationQueue mainQueue];
 
NSURLSession *defaultSession = [NSURLSession sessionWithConfiguration:defaultConfiguration delegate:delegate operationQueue:operationQueue];
NSURLSession *ephemeralSession = [NSURLSession sessionWithConfiguration:ephemeralConfiguration delegate:delegate delegateQueue:operationQueue];
NSURLSession *backgroundSession = [NSURLSession sessionWithConfiguration:backgroundConfiguration delegate:delegate delegateQueue:operationQueue];
```

除了 Background Session 的配置，开发者可以复用配置对象创建任意的 session（Background session 的配置对象之所以不能复用，是因为如果两个 Background Session 的 identifier 一样，会发生不可预知的行为）。

开发者介意在任何时候修改配置，因为当你创建一个 session 的时候，配置对象就会被深度拷贝，因此修改配置不会影响已经创建的 session。举个栗子，你或许会创建新的 session，并要求只能在链接 wifi 的情况下下载资源：

```objective-c
ephemeralConfiguration.allowsCellularAccess = NO;
NSURLSession *ephemeralSessionWiFiOnly = [NSURLSession sessionWithConfiguration:ephemeralConfiguration delegate:delegate delegateQueue:operationQueue];
```

### 使用系统提供的 Delegate 拉取资源

使用 NSURLSession 最直接的方法就是使用系统提供的 Delegate 来请求资源，开发者只需要提供两段代码来实现这种方式的使用：

* 一段代码创建配置对象以及基于配置对象的 session；
* 在数据全部接收完成后处理数据的 Block；

使用系统提供的 Delegate，每个资源请求只需要一行代码就可以搞定：

```objective-c
NSURLSession *sessionWithoutADelegate = [NSURLSession sessionWithConfiguration:defaultConfiguration];
NSURL *url = [NSURL URLWithString:@"https://www.example.com/"];
 
[[sessionWithoutADelegate dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
    NSLog(@"Got response %@ with error %@.\n", response, error);
    NSLog(@"DATA:\n%@\nEND DATA\n", [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}] resume];
```

> 系统提供的 delegate 只定义了有限的网络行为，如果 App 有特殊的需要，比如自定义的认证或者后台下载，这种方式是不合适的。

### 使用自定义的 Delegate 拉取数据

如果你使用自定义的 Delegate 拉取数据，Delegate 必须实现至少如下两个方法：

* `URLSession:dataTask:didReceiveData:` 会分批吐出服务端返回的数据。
* `URLSession:task:didCompleteWithError:` 会告诉 App 数据全部接收完毕。

这种情况下，如果 App 需要使用完整的数据，就需要自己存储数据。举个栗子，网页浏览器可能需要渲染当前接收到的数据和之前接收到的数据，因此，它可能需要 `appendData:` 来不停的将新收到的数据保存下来。

下面的代码展示了如何创建和启动一个数据任务：

```objective-c
NSURL *url = [NSURL URLWithString: @"https://www.example.com/"];
NSURLSessionDataTask *dataTask = [defaultSession dataTaskWithURL:url];
[dataTask resume];
```

### 下载文件

文件下载其实和拉取数据类似，使用的时候应该要实现以下 deleggate 方法：

* `URLSession:downloadTask:didFinishDownloadingToURL: ` 告知 App 指向下载内容临时存储文件的 URL；

  > 这个方法返回之前，它必须打开文件读取其中的数据或者将文件移动到一个永久的地址，因为当这个方法返回后，临时文件就会被删除。

* `URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite: ` 告知 App 下载进度；

* `URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes:` 告知 App 它成功重启了一个失败的下载；

* `URLSession:task:didCompleteWithError:` 告知 App 下载失败了；

如果你在一个 Background Session 中调度下载任务，即使 App 不运行了，下载任务依然会运行；如果是在 Default / Ephemeral Session 中调度下载任务，应用重启之后必须重新开启下载任务。

当从服务端拉取数据的时候，如果用户选择暂停下载，App 可以通过调用`cancelByProducingResumeData: ` 方法来取消任务，之后，App 可以通过将已经下载的数据传递给` downloadTaskWithResumeData:` 或者 ` downloadTaskWithResumeData:` 方法来创建一个新的下载任务继续下载。

如果数据传输失败，delegate 的 `URLSession:task:didCompleteWithError:` 方法会被调用。

下面的代码展示了下载一个中等大小文件的过程：

```objective-c
NSURL *url = [NSURL URLWithString:@"https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/ObjC_classic/FoundationObjC.pdf"];
NSURLSessionDownloadTask *downloadTask = [backgroundSession downloadTaskWithURL:url];
[downloadTask resume];
```

下面是 delegate 实现：

```objective-c
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    NSLog(@"Session %@ download task %@ wrote an additional %lld bytes (total %lld bytes) out of an expected %lld bytes.\n", session, downloadTask, bytesWritten, totalBytesWritten, totalBytesExpectedToWrite);
}
 
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes
{
    NSLog(@"Session %@ download task %@ resumed at offset %lld bytes out of an expected %lld bytes.\n", session, downloadTask, fileOffset, expectedTotalBytes);
}
 
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    NSLog(@"Session %@ download task %@ finished downloading to URL %@\n", session, downloadTask, location);
 
    // Perform the completion handler for the current session
    self.completionHandlers[session.configuration.identifier]();
 
   // Open the downloaded file for reading
    NSError *readError = nil;
    NSFileHandle *fileHandle = [NSFileHandle fileHandleForReadingFromURL:location error:readError];
    // ...
 
   // Move the file to a new URL
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSURL *cacheDirectory = [[fileManager URLsForDirectory:NSCachesDirectory inDomains:NSUserDomainMask] firstObject];
    NSError *moveError = nil;
    if ([fileManager moveItemAtURL:location toURL:cacheDirectory error:moveError]) {
        // ...
    }
}
```

### 上传消息体内容

App 可以通过三种方式为 HTTP POST 消息提供消息体：NSData、文件或者流。总的来说，你的 App 应该：

* 如果内存中已经存在一个 NSData 对象并且没有理由废弃它（比如内存问题），那么可以直接使用它；
* 如果上传内容是以文件的形式存在于磁盘上的，或者你是在进行后台传输任务，又或者 App 有必要将数据写到磁盘上来释放那部分数据占用的内存空间，那么使用文件即可；
* 如果你是从网络上接收数据，那么使用流；

不管你选择何种方式上传，如果你的 App 自定义了 session delegate，它都应该实现 `URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:` 方法来获取上传进度信息；

另外，如果你的 App 是通过流的形式提供消息体的，它必须提供自定义的 session delegate 并实现 `URLSession:task:needNewBodyStream:` 方法，下面会详细解释。

#### 使用 NSData 上传数据

使用 NSData 进行上传，App 必须通过调用 `uploadTaskWithRequest:fromData:` 或者 `uploadTaskWithRequest:fromData:completionHandler:` 方法来创建上传任务，并在 `fromData:` 参数中传入数据对象。session 对象会根据 NSData 对象计算 `Content-Length` Header 的值，服务器需要的其余参数需要开发者手动提供，比如 `Content-Type`。

下面的代码展示了如何通过 NSData 进行数据上传：

```objective-c
NSURL *textFileURL = [NSURL fileURLWithPath:@"/path/to/file.txt"];
NSData *data = [NSData dataWithContentsOfURL:textFileURL];
 
NSURL *url = [NSURL URLWithString:@"https://www.example.com/"];
NSMutableURLRequest *mutableRequest = [NSMutableURLRequest requestWithURL:url];
mutableRequest.HTTPMethod = @"POST";
[mutableRequest setValue:[NSString stringWithFormat:@"%lld", data.length] forHTTPHeaderField:@"Content-Length"];
[mutableRequest setValue:@"text/plain" forHTTPHeaderField:@"Content-Type"];
 
NSURLSessionUploadTask *uploadTask = [defaultSession uploadTaskWithRequest:mutableRequest fromData:data];
[uploadTask resume];
```

#### 使用文件上传数据

使用文件上传数据，App 必须通过调用 `uploadTaskWithRequest:fromFile:` 或者 `uploadTaskWithRequest:fromFile:completionHandler:` 方法来创建上传任务，并且提供文件的 URL 以供任务读取上传数据。session 对象会根据文件对象计算 `Content-Length` Header 的值，如果 App 没有设置 `Content-Type`  Header，session 也会提供一个。

下面的代码展示了如何通过文件进行数据上传：

```objective-c
NSURL *textFileURL = [NSURL fileURLWithPath:@"/path/to/file.txt"];
 
NSURL *url = [NSURL URLWithString:@"https://www.example.com/"];
NSMutableURLRequest *mutableRequest = [NSMutableURLRequest requestWithURL:url];
mutableRequest.HTTPMethod = @"POST";
 
NSURLSessionUploadTask *uploadTask = [defaultSession uploadTaskWithRequest:mutableRequest fromFile:textFileURL];
[uploadTask resume];
```

#### 使用流上传数据

使用流上传数据，必须通过调用 `uploadTaskWithStreamedRequest:` 方法来创建一个上传任务，App 必须为这个方法提供一个上传数据对象的关联流。App 同时必须提供一些服务器需要的 Header 字段，比如 `Content-Type` 和 `Content-Length`。

另外，session 如果遇到必须重新尝试一个请求的情况，比如如果身份验证失败，就必须重新读取流，但是流不一定能够倒回去重新读取，这个时候 App 就必须负责重新提供一个新的流。因此，App 需要实现 `URLSession:task:needNewBodyStream:` 方法，当这个方法被调用的时候，App 应该尽量提供一个新的流，然后以 新的流为参数调用 completion handler block。

> 这种技术不能和系统提供的 delegate 兼容。

下面的代码展示了如何通过流进行数据上传：

```objective-c
NSURL *textFileURL = [NSURL fileURLWithPath:@"/path/to/file.txt"];
 
NSURL *url = [NSURL URLWithString:@"https://www.example.com/"];
NSMutableURLRequest *mutableRequest = [NSMutableURLRequest requestWithURL:url];
mutableRequest.HTTPMethod = @"POST";
mutableRequest.HTTPBodyStream = [NSInputStream inputStreamWithFileAtPath:textFileURL.path];
[mutableRequest setValue:@"text/plain" forHTTPHeaderField:@"Content-Type"];
[mutableRequest setValue:[NSString stringWithFormat:@"%lld", data.length] forHTTPHeaderField:@"Content-Type"];
 
NSURLSessionUploadTask *uploadTask = [defaultSession uploadTaskWithStreamedRequest:mutableRequest];
[uploadTask resume];
```

## 缓存

URL 加载系统为请求和响应提供了磁盘缓存和内存缓存，这个设计让应用介绍了对网络的依赖，提高了性能。

### 缓存策略

NSURLRequest 通过设置缓存策略来指定如何使用本地缓存，缓存策略的值是由枚举 `NSURLRequestCachePolicy` 指定的，包括：`NSURLRequestUseProtocolCachePolicy`, `NSURLRequestReloadIgnoringCacheData`, `NSURLRequestReturnCacheDataElseLoad`, 和 `NSURLRequestReturnCacheDataDontLoad`。`NSURLRequestUseProtocolCachePolicy` 是默认的缓存策略，它是根据协议规范实现的； `NSURLRequestReloadIgnoringCacheData` 表示完全忽略本地缓存；`NSURLRequestReturnCacheDataElseLoad` 表示只要有缓存，不论是否过期直接使用，没有缓存才从远端加载；`NSURLRequestReturnCacheDataDontLoad` 表示只从缓存加载数据，也就是离线模式。

> Http 的标准缓存机制可见：[RFC 2616, Section 13]([http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13](http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13))。

### 缓存控制

一般来说，请求会基于缓存策略进行缓存。如果开发者需要更加精细的控制，可以通过实现 delegate 方法来决定每个请求的响应如何进行缓存。对于 NSURLSession 的数据任务和上传任务，需要实现 `URLSession:dataTask:willCacheResponse:completionHandler:` 方法，这个方法只有在执行数据任务和上传任务的时候才会被调用。下载任务的缓存只能由设置的缓存策略控制。

delegate 方法通过调用 completion handler 来告知 session 应该如何做缓存，通常有以下三种情况：

* 直接返回提供的响应；
* 修改提供的响应并返回一个新的响应；
* 返回 nil 拒绝缓存；

delegate 方法也可以向 NSCacheURLResponse 对象的 `userInfo` 字典中插入对象，这些对象也会随着响应一起被缓存。

以下的例子就是拒绝在磁盘上缓存 HTTPS 响应，并且在 userInfo 字典中添加了当前日期：

```objective-c
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse * __nullable cachedResponse))completionHandler {
    NSCachedURLResponse *newCachedResponse = proposedResponse;
    NSDictionary *newUserInfo;
    newUserInfo = [NSDictionary dictionaryWithObject:[NSDate date]
                                              forKey:@"Cached Date"];
    if ([proposedResponse.response.URL.scheme isEqualToString:@"https"]) {
#if ALLOW_IN_MEMORY_CACHING
        newCachedResponse = [[NSCachedURLResponse alloc]
                             initWithResponse:proposedResponse.response
                             data:proposedResponse.data
                             userInfo:newUserInfo
                             storagePolicy:NSURLCacheStorageAllowedInMemoryOnly];
#else // !ALLOW_IN_MEMORY_CACHING
        newCachedResponse = nil;
#endif // ALLOW_IN_MEMORY_CACHING
    } else {
        newCachedResponse = [[NSCachedURLResponse alloc]
                             initWithResponse:[proposedResponse response]
                             data:[proposedResponse data]
                             userInfo:newUserInfo
                             storagePolicy:[proposedResponse storagePolicy]];
    }
 
    completionHandler(newCachedResponse);
}
```

## Cookie 存储

由于 Http 协议无状态的天然特性，客户端通常使用 cookie 来维护状态。URL 加载系统为创建和管理 cookie 提供了接口，并将 cookie 作为请求的一部分发送，会在服务器响应返回的时候接收 cookie。

NSHTTPCookie 类封装了一个 cookie 并对很多的常见 cookie 属性提供了读写方法。NSHTTPCookieStorage 类则提供了所有 App 共享的 NSHTTPCookie 对象的接口。

> 注意：iOS 上的应用之间不共享 Cookie。

NSHTTPCookieStorage 允许 App 指定 Cookie 设置策略，可以设置 Cookie 永久可设置，永久不可设置或者只有来自相同域名的请求才可以设置。

## 处理身份认证和 TLS 链认证

如果远程服务返回的状态码要求进行身份认证，又或者认证需要在连接建立的时候进行（比如需要一个 SSL 客户端证书），NSURLSession 会调用一个 delegate 方法：

// TODO

## 协议支持

URL 加载系统允许应用扩展协议来支持数据传输。开发者可以继承 NSURLProtocol 协议实现自己的协议类，通过 NSURLProtocol 的类方法 `registerClass:` 进行注册，当 NSURLSession 对象为一个 NSURLRequest 初始化链接的时候，URL 加载系统会通过 NSURLProtocol 类来倒叙遍历所有的注册协议类，第一个在方法 `canInitWithRequest:` 中返回 `YES` 的协议类被用于处请求。

如果你自定义的协议需要在请求或者响应中添加新的属性，可以通过 Caterogry 在 NSURLRequest、NSMutableURLRequest 和 NSURLResponse 类中为这些属性提供读写方法。

URL 加载系统负责在连接建立的时候创建 NSURLProtocol 实例，在请求完成之后释放实例，因此开发者永远不应该自己去创建 NSURLProtocol 的实例。

## 其他

### 处理重定向和其他请求变化

重定向发生在服务端告诉客户端需要对一个新的  URL 发起请求才能获取资源的时候， NSURLSession 遇到这种情况会通过 delegate 告知 App。为了处理这种情况，delegate 必须实现 `URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:` 方法，在这个方法里面可以获取到引起重定向的 response，并通过 comletionHandler 返回一个新的请求。delegate 可以做以下事情：

- 简单的返回获取的 request 以允许重定向；
- 创建一个新的 request 指向一个不同的 URL，并返回 request；
- 通过返回 nil，拒绝重定向；

另外，delegate 可以同时取消重定向和连接，只需要调用 task 对象的 `cancel` 方法即可。

如果处理请求的 NSURLProtocol 的子类为了标准化请求的格式，更改了 NSRequest 的内容，比如把请求 URL 的地址从 http://www.apple.com 改为 http://www.apple.com/ ，delegate 也会收到 `URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:`  消息，在这种特殊的情况下，response 参数会为 nil，delegate 应该直接返回收到的 request 对象。

下面的代码展现了`URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:` 的实现：

```objective-c
- (void)URLSession:(NSURLSession *)session
        task:(NSURLSessionTask *)task
        willPerformHTTPRedirection:(NSHTTPURLResponse *)redirectResponse
        newRequest:(NSURLRequest *)request
        completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    NSURLRequest *newRequest = request;
    if (redirectResponse) {
        newRequest = nil;
    }
 
    completionHandler(newRequest);
}
```

这段代码允许请求上因为标准化而发生的任何变化，拒绝了所有服务端的重定向要求。

### 处理 iOS 后台活动

如果你使用 NSURLSession，你的 App 会在下载任务完成后自动重新启动，App 的`application:handleEventsForBackgroundURLSession:completionHandler:` delegate 方法负责重新创建合适的 session，提供 completion handler，并在 session 调用 delegate 的 `URLSessionDidFinishEventsForBackgroundURLSession:` 方法的时候调用这个 handler。

下面的代码展示了如何在后台创建下载任务：

```objective-c
SURL *url = [NSURL URLWithString:@"https://www.example.com/"];
 
NSURLSessionDownloadTask *backgroundDownloadTask = [backgroundSession downloadTaskWithURL:url];
[backgroundDownloadTask resume];
```

下面的代码展示了 session 的代理方法实现：

```objective-c
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    AppDelegate *appDelegate = (AppDelegate *)[[[UIApplication sharedApplication] delegate];
    if (appDelegate.backgroundSessionCompletionHandler) {
        CompletionHandler completionHandler = appDelegate.backgroundSessionCompletionHandler;
        appDelegate.backgroundSessionCompletionHandler = nil;
        completionHandler();
    }
 
    NSLog(@"All tasks are finished");
}
```

下面的代码展示了 AppDelegate 的实现：

```objective-c
@interface AppDelegate : UIResponder <UIApplicationDelegate>
@property (strong, nonatomic) UIWindow *window;
@property (copy) CompletionHandler backgroundSessionCompletionHandler;
 
@end
 
@implementation AppDelegate
 
- (void)application:(UIApplication *)application
handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)())completionHandler
{
    self.backgroundSessionCompletionHandler = completionHandler;
}
 
@end
```

【参考资料】

1）[从 NSURLConnection 到 NSURLSession](https://objccn.io/issue-5-4/)

2）[URL Session Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165i)
