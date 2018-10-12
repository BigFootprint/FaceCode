---
title: iOS 网络学习（一）—— URLConnection
tags: [iOS, 网络]
categories: iOS
date: 2017-02-15 14:16:28
---


> 关于 Http 相关知识，推荐书籍：《HTTP : The Definitive Guide》。

## 简介

[`NSURLConnection`](https://developer.apple.com/reference/foundation/nsurlconnection) 用于向服务端发起网络请求，它暴露的接口非常少，只能控制发起和取消请求，大部分的配置是在 `NSURLRequest` 上进行。我们平时所说的 NSURLConnection 更多的是指以 `NSURLConnection` 类为核心的一套网络请求解决方案，它包括：

1. [NSURL](https://developer.apple.com/reference/foundation/nsurl)：请求地址；
2. [NSURLRequest](https://developer.apple.com/reference/foundation/nsurlrequest)：封装一个请求，保存发给服务器的全部数据，包括一个NSURL对象，请求方法、请求头、请求体等；
3. [NSMutableURLRequest](https://developer.apple.com/reference/foundation/NSMutableURLRequest)：NSURLRequest的子类；
4. [NSURLConnection](https://developer.apple.com/reference/foundation/nsurlconnection)：负责发送请求，建立客户端和服务器的连接。发送NSURLRequest的数据给服务器，并收集来自服务器的响应数据；

<!-- More -->NSURLConnection 的学习比较简单，核心包括两块：

1. 同步 / 异步请求；
2. 使用 Block / Delegate 接收返回数据；

下面围绕这两个点进行学习，并补充一些常见的设置和相关知识。

> 这里推荐一个网站，用于快速实验网络功能：http://httpbin.org/。这个网站列举了很多开放的接口，开发者可以通过调用这些接口来测试自己的网络请求，非常方便。比如请求接口 http://httpbin.org/ip，返回的数据就是：
>
> ```json
> {
>   "origin": "103.39.140.10"
> }
> ```

## 同步 / 异步请求

NSURLConnection 同时支持同步和异步请求的发送，但实际使用中，我们大部分情况下都是使用异步请求，因此下面的例子先从异步请求开始。

```objective-c
NSURL *url = [NSURL URLWithString:@"协议://主机地址/路径?参数&参数"];
NSURLRequest *request = [NSURLRequest requestWithURL:url cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:15.0];
// 告诉服务器数据为json类型
[request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"]; 
// 设置请求体(json类型)
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:@{@"userid":@"123456"} options:NSJSONWritingPrettyPrinted error:nil];
request.HTTPBody = jsonData; 
[NSURLConnection sendAsynchronousRequest:request queue:[[NSOperationQueue alloc] init] completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
    // 有的时候，服务器访问正常，但是会没有数据！
    // 以下的 if 是比较标准的错误 处理代码！
    if (connectionError != nil || data == nil) {
        //给用户的提示信息
        NSLog(@"网络不给力");
        return;
    }
}];
```

`NSURL` 用于指向一个资源地址，在这里我们指向的是服务器端的资源。关于代码中给出的格式，如有不懂可以直接参考[开发文档](https://developer.apple.com/reference/foundation/nsurl) 或者翻阅本文开头推荐的书籍。它的常见用法就是通过一个表达 url 的 `NSString` 生成 `NSURL` 对象，比如：

```objective-c
NSURL *url = [NSURL URLWithString:@"http://httpbin.org/ip"];
```

接着，可以使用 `NSURL` 生成基本的 `NSURLRequest` 对象，`NSURL` 定义的是资源的地址，`NSURLRequest` 顾名思义，定义的就是一个完整的网络请求。`NSURLRequest` 的实例化基本分为两种：

```objective-c
+ requestWithURL:
+ requestWithURL:cachePolicy:timeoutInterval:
```

一种是只需要提供 `NSURL` 即可，其余的配置全部默认。另外一种是可以配置缓存策略以及超时时间。但是无论哪种方式，我们可以发现能配置的项实在太少，至少我们有一些常见的需求这样是无法满足的，比如更改请求方法为 POST 请求，添加请求 Header 等。因此很自然地就出现了 `NSMutableURLRequest` 类，该类提供了较为丰富的配置选项。就像例子中看到的一样，还可以设置消息体。

这里我们重点提一下缓存策略，即`cachePolicy`参数。这个参数是一个枚举类型——[`NSURLRequestCachePolicy`](https://developer.apple.com/reference/foundation/NSURLRequestCachePolicy)，它的取值是根据 Http 定义的缓存协议来的，具体可以查询本文开头推荐的书籍，或者翻看 [RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13)，因为这个点可以专门开一篇文章进行描述，这里就不展开了。Http 缓存机制可以满足以下的逻辑路径：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/http_caching.png" height="420" alt="One Piece"/></div>

该枚举有以下值可以选择：

1. `NSURLRequestUseProtocolCachePolicy` 表示使用协议定义的标准缓存逻辑，它是请求默认使用的缓存协议。
2. `[NSURLRequestReloadIgnoringLocalCacheData]` 表示忽略本地缓存，从源地址加载资源。如果使用的是 Http/Https 协议并请求的是资源的部分数据，务必使用该选项；
3. `NSURLRequestReloadIgnoringLocalAndRemoteCacheData` 表示不仅忽略本地缓存，也要忽略任何中间节点的缓存，必须从源地址加载数据；
4. `NSURLRequestReloadRevalidatingCacheData` 表示在使用缓存之前必须由服务端验证缓存是否过期，如果没有过期则使用缓存，否则从服务端拉取最新数据（对应到上图中，走的是 __Issue a HEAD request__ 路径）；

请求定义好之后，就可以通过`NSURLConnection`来发送请求了。发送请求有两个方法：

```objective-c
+ sendSynchronousRequest:returningResponse:error:  // 发送同步请求
+ sendAsynchronousRequest:queue:completionHandler:  // 发送异步请求
```

读者如果这个时候去查看[开发文档](https://developer.apple.com/reference/foundation/nsurlconnection?language=objc)，会发现这两个方法已经被废弃了，这是因为 Apple 发布了更为优秀的请求工具 `NSURLSession`，不再推荐使用` NSURLConnection`，然而因为某些库或者遗留代码还在使用 `NSURLConnection`，所以我们在这里学习一下。

例子中使用的是异步方法，异步方法需要设置回调来进行监听数据的接收处理，iOS 一般使用 Block 或者 Delegate 来实现回调，`NSURLConnection`两种都支持，例子中使用的是 Block。两者的区别下面再说。

使用同步方法的例子如下：

```objective-c
// 发送同步请求, data 就是返回的数据
NSError *error = nil;
NSData *data = [NSURLConnection sendSynchronousRequest:request returningResponse:nil error:&error];
```

如果返回值为 nil，则表示连接创建失败，或者数据加载失败，简而言之，请求失败。使用同步请求会阻塞当前调用线程，所以不建议在 UI 线程调用。

## 使用 Block / Delegate 接收返回数据

上一节的例子中已经展示如何使用 Block 接收数据，Block 作为回调的缺陷是只能在请求结束后对返回数据进行操作，如果遇到像 [SDWebImage](http://timebridge.space/2017/02/10/SDWebImage-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/) 库那样的需求，需要在数据加载开始和加载中进行处理，Block 就不能胜任，此时需要使用 Delegate。使用 Delegate 需要实现协议 `NSURLConnectionDataDelegate`，这个协议继承于`NSURLConnectionDelegate`协议并添加了七个新的接口，涉及到数据加载不同阶段、重定向、缓存等方面，其中四个比较常用的方法如下：

```objective-c
#pragma mark- NSURLConnectionDataDelegate代理方法

//当接收到服务器的响应（连通了服务器）时会调用
-(void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response

//当接收到服务器的数据时会调用（可能会被调用多次，每次只传递部分数据）
-(void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data

//当服务器的数据加载完毕时就会调用
-(void)connectionDidFinishLoading:(NSURLConnection *)connection

//请求错误（失败）的时候调用（请求超时\断网\没有网\，一般指客户端错误）
-(void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error
```

实现之后，代理的设置和请求发送用法如下：

```objective-c
NSURLConnection *conn = [NSURLConnection connectionWithRequest:request delegate:self];
[conn start];
```

下面是一个使用 Delegate 下载文件的例子，以供读者参考：

```objective-c
-(void)sendAFNetworingRequest:(id)button{
    NSURL *url = [NSURL URLWithString:@"https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1487163449810&di=c69d81b7530c246d5d3ab7c157f5463c&imgtype=0&src=http%3A%2F%2Fimg2.78dm.net%2Fforum%2F201409%2F14%2F144254ed19ejda61z5aw5n.jpg"];
    NSURLRequest *request = [NSURLRequest requestWithURL:url cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:15.0];
    NSURLConnection *conn = [NSURLConnection connectionWithRequest:request delegate:self];
    [conn start];
}

-(NSString *)getDownloadFilePath {
    NSString *path = NSTemporaryDirectory();
    return [path stringByAppendingString:@"onepiece.jpeg"];
}

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    NSLog(@"下载开始");
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    NSFileHandle *fileHandle = [NSFileHandle fileHandleForWritingAtPath:self.downloadFilePath];
    fileSize += (unsigned long)data.length;
    NSLog(@"数据下载中，新接收数据量：%lu", (unsigned long)data.length);
    if (fileHandle) {
        [fileHandle seekToEndOfFile];
        [fileHandle writeData:data];
        [fileHandle closeFile];
    } else {
        [data writeToFile:self.downloadFilePath atomically:YES];
    }
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection {
    NSLog(@"下载完成，总数据量：%lu", fileSize);
}
```

下载的资源是百度上的一张图片，大小显示为 158 KB，下面是控制台输出：

```xml
2017-02-15 18:14:47.632 MyP[47339:12735852] 下载开始
2017-02-15 18:14:47.633 MyP[47339:12735852] 数据下载中，新接收数据量：18413
2017-02-15 18:14:47.944 MyP[47339:12735852] 数据下载中，新接收数据量：1336
2017-02-15 18:14:47.944 MyP[47339:12735852] 数据下载中，新接收数据量：6856
2017-02-15 18:14:48.246 MyP[47339:12735852] 数据下载中，新接收数据量：9624
2017-02-15 18:14:48.246 MyP[47339:12735852] 数据下载中，新接收数据量：36864
2017-02-15 18:14:48.400 MyP[47339:12735852] 数据下载中，新接收数据量：1328
2017-02-15 18:14:48.401 MyP[47339:12735852] 数据下载中，新接收数据量：13624
2017-02-15 18:14:48.453 MyP[47339:12735852] 数据下载中，新接收数据量：1336
2017-02-15 18:14:48.459 MyP[47339:12735852] 数据下载中，新接收数据量：96
2017-02-15 18:14:48.463 MyP[47339:12735852] 数据下载中，新接收数据量：23144
2017-02-15 18:14:48.473 MyP[47339:12735852] 数据下载中，新接收数据量：1336
2017-02-15 18:14:48.474 MyP[47339:12735852] 数据下载中，新接收数据量：96
2017-02-15 18:14:48.476 MyP[47339:12735852] 数据下载中，新接收数据量：2664
2017-02-15 18:14:48.790 MyP[47339:12735852] 数据下载中，新接收数据量：1432
2017-02-15 18:14:49.355 MyP[47339:12735852] 数据下载中，新接收数据量：23144
2017-02-15 18:14:49.589 MyP[47339:12735852] 数据下载中，新接收数据量：1432
2017-02-15 18:14:49.609 MyP[47339:12735852] 数据下载中，新接收数据量：1328
2017-02-15 18:14:49.609 MyP[47339:12735852] 数据下载中，新接收数据量：13528
2017-02-15 18:14:49.610 MyP[47339:12735852] 下载完成，总数据量：157581
```

从这个例子还可以看出使用 Delegate 的另外一个好处：因为可以持续处理数据而不用等到数据全部加载完成后再处理，因此可以避免将加载的资源全部保存在内存中，可以将数据持续同步到磁盘上，这是 Block 无法做到的。

## 总结

以上就是关于 NSURLConnction 的一些基本知识。

在 2013 年苹果全球开发者大会（WWDC 2013）上 Apple 发布了新的网络请求类 NSURLSession，从 iOS9.0 开始，NSURLConnection 发送请求的两个方法以及初始化网络连接的方法都被置为过期，Apple 推荐使用 NSURLSession 类来替代 NSURLConnection，这意味着 NSURLConnection 即将推出历史舞台。

下一篇我们将学习 NSURLSession。



【参考文章】

1）[iOS开发网络篇—NSURLConnection基本使用](http://www.cnblogs.com/wendingding/p/3813572.html)

2）[iOS网络1——NSURLConnection使用详解](http://www.cnblogs.com/mddblog/p/5134783.html)