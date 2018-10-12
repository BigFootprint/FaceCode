---
title: SDWebImage 源码解析
date: 2017-02-10 08:18:30
tags: [源码]
categories: iOS
---

> 分析版本：3.7.4

[SDWebImage](https://github.com/rs/SDWebImage) 是 iOS 上使用范围最广的网络图片加载库，在 Github 上有高达近 17K 的 Star 数量。它提供的功能主要如下：

1. 加载普通图片，包括 png、jpeg、gif、WebP等，支持的组件有 UIImageView、UIButton 和 MKAnnotationView；
2. 为 UIImageVew 加载多帧图片；
3. 预加载图片；

使用示例如下：

```objective-c
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://www.domain.com/path/to/image.jpg"]
             placeholderImage:[UIImage imageNamed:@"placeholder.png"]];
```

简单的情况下只需要告诉它图片的 URL 以及占位图片就可以轻松完成网络图片的加载。知其然更要知其所以然，下面从源码角度分析一下 SDWebImage 是如何完成图片加载的。

<!-- More -->

## 加载流程

在详细看代码之前，先来看看图片加载的流程（来自项目的 Wiki）：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/SDWebImageSequenceDiagram.png" alt="SDWebImage 图片加载流程图" alt="One Piece"/></div>

文字描述如下：

1. 发起图片加载请求；
2. 尝试从缓存中读取图片；
3. 缓存读取成功则返回图片，读取失败则从网络下载；
4. 下载成功后添加到缓存并返回图片；

图片加载的流程并不复杂，大部分图片加载库的流程都大同小异，只不过每个库在实现的时候在策略控制以及细节实现上有所差别。在理解大致流程的基础上，下面分析每一步的实现。

## 任务创建与调度

### 任务创建

我们从上面的示例代码切入，即方法`- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder;`。这个方法来自于`UIImageView+WebCache.h`，有很多的重载方法，最终调用到的方法如下：

```objective-c
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
    // 1. 取消任务
    [self sd_cancelCurrentImageLoad];
    // 2. 记录要加载的 URL
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

    // 3. 设置占位图
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            self.image = placeholder;
        });
    }
    
    // 4. 如果 URL 存在，则加载图片，否则报错，直接回调
    if (url) {
        // 5. 显示加载菊花
        // check if activityView is enabled or not
        if ([self showActivityIndicatorView]) {
            [self addActivityIndicator];
        }

        // 6. 创建加载任务
        __weak __typeof(self)wself = self;
        id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            [wself removeActivityIndicator];
            if (!wself) return;
            dispatch_main_sync_safe(^{// 调度到主线程
                if (!wself) return;
                // 如果加载选项设置为不自动设置图片，并且有回调 Block，则调用回调 Block
                if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                {
                    completedBlock(image, error, cacheType, url);
                    return;
                }
                else if (image) {// 否则直接设置图片
                    wself.image = image;
                    [wself setNeedsLayout];
                } else {// 如果加载失败，在此处设置为占位符（关于 SDWebImageDelayPlaceholder 可以查看该枚举的说明）
                    if ((options & SDWebImageDelayPlaceholder)) {
                        wself.image = placeholder;
                        [wself setNeedsLayout];
                    }
                }
                // 回调
                if (completedBlock && finished) {
                    completedBlock(image, error, cacheType, url);
                }
            });
        }];
        // 7. 将该任务记录为 "UIImageViewImageLoad" 对应的任务。一个组件有两种任务，另外一种是 "UIImageViewAnimationImages"
        // 加载的是 Gif 动态图，每种类型的任务同时只能存在一个，该方法调用是会默认检测并取消旧有的任务
        [self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
    } else {
        // URL 非法，直接报错回调
        dispatch_main_async_safe(^{
            [self removeActivityIndicator];
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
            if (completedBlock) {
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
```

这个方法的主要作用是准备加载环境，创建加载任务。相关注释都已经标记在代码上，下面说一些细节。

首先是`dispatch_main_async_safe`和`dispatch_main_sync_safe`两个宏，它们的定义如下：

```objective-c
#define dispatch_main_sync_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_sync(dispatch_get_main_queue(), block);\
    }

#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
```

这两个宏虽然简化了一些写法，但是定义很糟糕：

1. 所谓`safe`，并不能很明显的看出来，唯一加强安全的地方就是`dispatch_main_sync_safe`防止了在主线程上调用`dispatch_sync(dispatch_get_main_queue(), block)`产生死锁，而`async`完全看不出来；
2. `dispatch_main_async_safe`如果执行在主线程上，实际是同步的，并不与名字所反映的一样，使用的时候务必要注意；

其次是任务的取消。在方法的一开始即 1 处就调用了取消任务的方法，图片加载任务创建的最后，也就是 7 处将任务记录为一个变量，后期会通过这个变量操作任务。`sd_setImageLoadOperation`方法的调用链如下：

```objective-c
// 来自 UIImageView+WebCache.h
- (void)sd_cancelCurrentImageLoad {
    [self sd_cancelImageLoadOperationWithKey:@"UIImageViewImageLoad"];
}

// 来自 UIView+WebCacheOperation.h
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    [self sd_cancelImageLoadOperationWithKey:key];
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}

// 来自 UIView+WebCacheOperation.h
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // Cancel in progress downloader from queue
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    id operations = [operationDictionary objectForKey:key];
    if (operations) {
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation in operations) {
                if (operation) {
                    [operation cancel];
                }
            }
        } else if ([operations conformsToProtocol:@protocol(SDWebImageOperation)]){
            [(id<SDWebImageOperation>) operations cancel];
        }
        [operationDictionary removeObjectForKey:key];
    }
}
```

在 UIView 内部，通过 Category 创建了一个 Dictionary，用于记录当前在该 View 上正在执行的所有任务，在注释 7 处我说了这个 Dictionary 中记录的任务有两种 key：

1.  __"UIImageViewImageLoad"__对应的是单个`SDWebImageOperation`，用于下载单张普通图片；
2.  __"UIImageViewAnimationImages"__对应的是`SDWebImageOperation`数组，用于下载一组图片展现帧动画。

`SDWebImageOperation`是一个接口，只有一个`cancel()`方法，取消任务调用它即可。

> 要注意的是，在加载任务执行完成之后，并没有主动从字典中删除对应任务。

最后有一个疑问没有得到解答：下载完成的回调 Block 中为什么要`dispatch_main_sync_safe`方法进行图片最后的显示处理？这里使用`dispatch_main_async_safe`明显可以更快地释放加载线程，而且实现效果一致。

### 加载任务的执行

注释 6 处已经创建了加载任务，我们来仔细看看加载任务到底做了什么：

```objective-c
// 来自 SDWebImageManager.h
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock {
    // 1. 参数检测
    // Invoking this method without a completedBlock is pointless
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // Very common mistake is to send the URL using NSString object instead of NSURL. For some strange reason, XCode won't
    // throw any warning for this type mismatch. Here we failsafe this error by allowing URLs to be passed as NSString.
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

    // 2. 创建真正的 Operation
    __block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    __weak SDWebImageCombinedOperation *weakOperation = operation;

    // 3. 判断这个 URL 之前有没有加载失败过
    BOOL isFailedUrl = NO;
    @synchronized (self.failedURLs) {
        isFailedUrl = [self.failedURLs containsObject:url];
    }

    // 如果 URL 为空，或者这个 URL 加载失败过但是开发者不要求重试失败的 URL，则直接回调
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        dispatch_main_sync_safe(^{
            NSError *error = [NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil];
            completedBlock(nil, error, SDImageCacheTypeNone, YES, url);
        });
        return operation;
    }

    // 4. 记录这个 Operation
    @synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
    NSString *key = [self cacheKeyForURL:url];

    // 5. &&& 为任务创建缓存任务，先从磁盘读取任务，根据读取的结果进行操作，根据后面的分析，这个 Block 的执行是在主线程进行的
    operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
        if (operation.isCancelled) {
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
            return;
        }

        // 6. @@@ 如果图片没有从缓存获取到/开发者要求刷新缓存图片（换句话说，获取到的图片不能用，需要下载），并且代理反馈需要根据 URL 下载图片
        if ((!image || options & SDWebImageRefreshCached) && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url])) {
            // 如果从缓存中获取了图片，但是开发者设置为必须刷新缓存
            if (image && options & SDWebImageRefreshCached) {
                dispatch_main_sync_safe(^{
                    // If image was found in the cache but SDWebImageRefreshCached is provided, notify about the cached image
                    // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
                    // 先回调再重新下载
                    completedBlock(image, nil, cacheType, YES, url);
                });
            }

            // download if no image or requested to refresh anyway, and download allowed by delegate
            SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            // 如果是设置为刷新缓存并且获取到了图片，则取消渐进式加载，并添加忽略缓存响应选项
            if (image && options & SDWebImageRefreshCached) {
                // force progressive off if image already cached but forced refreshing
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // ignore image read from NSURLCache if image if cached but force refreshing
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
            // 7. 创建下载任务
            id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) {
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (!strongOperation || strongOperation.isCancelled) { // 如果任务取消
                    // Do nothing if the operation was cancelled
                    // See #699 for more details
                    // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                }
                else if (error) { // 如果下载失败
                    dispatch_main_sync_safe(^{ 
                        if (strongOperation && !strongOperation.isCancelled) {
                            completedBlock(nil, error, SDImageCacheTypeNone, finished, url);
                        }
                    });
                    // 如果不是因为以下原因下载失败，则记录为失败的 URL
                    if (   error.code != NSURLErrorNotConnectedToInternet
                        && error.code != NSURLErrorCancelled
                        && error.code != NSURLErrorTimedOut
                        && error.code != NSURLErrorInternationalRoamingOff
                        && error.code != NSURLErrorDataNotAllowed
                        && error.code != NSURLErrorCannotFindHost
                        && error.code != NSURLErrorCannotConnectToHost) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs addObject:url];
                        }
                    }
                }
                else { // 如果下载成功
                    if ((options & SDWebImageRetryFailed)) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs removeObject:url];
                        }
                    }
                    
                    // 判断是否要缓存到磁盘上
                    BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);

                    // 如果要求刷新缓存但是下载图片不存在
                    if (options & SDWebImageRefreshCached && image && !downloadedImage) {
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                    }
                    // 如果下载的是多图，进行转换
                    else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
                        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                            UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

                            if (transformedImage && finished) {
                                BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                // 存储图片
                                [self.imageCache storeImage:transformedImage recalculateFromImage:imageWasTransformed imageData:(imageWasTransformed ? nil : data) forKey:key toDisk:cacheOnDisk];
                            }
                            // 回调
                            dispatch_main_sync_safe(^{
                                if (strongOperation && !strongOperation.isCancelled) {
                                    completedBlock(transformedImage, nil, SDImageCacheTypeNone, finished, url);
                                }
                            });
                        });
                    }
                    else { // 如果下载的是普通图片
                        if (downloadedImage && finished) {
                            // 存储图片
                            [self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
                        }

                        // 回调
                        dispatch_main_sync_safe(^{
                            if (strongOperation && !strongOperation.isCancelled) {
                                completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
                            }
                        });
                    }
                }

                if (finished) {
                    @synchronized (self.runningOperations) {
                        if (strongOperation) {
                            [self.runningOperations removeObject:strongOperation];
                        }
                    }
                }
            }];
            
            // 取消回调
            operation.cancelBlock = ^{
                [subOperation cancel];
                
                @synchronized (self.runningOperations) {
                    __strong __typeof(weakOperation) strongOperation = weakOperation;
                    if (strongOperation) {
                        [self.runningOperations removeObject:strongOperation];
                    }
                }
            };
        }
        else if (image) { // 获取缓存成功
            dispatch_main_sync_safe(^{
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (strongOperation && !strongOperation.isCancelled) {
                    completedBlock(image, nil, cacheType, YES, url);
                }
            });
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
        }
        else { // 开发者想 “只从缓存获取图片” 失败
            // Image not in cache and download disallowed by delegate
            dispatch_main_sync_safe(^{
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (strongOperation && !weakOperation.isCancelled) {
                    completedBlock(nil, nil, SDImageCacheTypeNone, YES, url);
                }
            });
            @synchronized (self.runningOperations) {
                [self.runningOperations removeObject:operation];
            }
        }
    }];

    return operation;
}
```

相关流程关键点都标记在代码中，这边有些地方不能解释的很清楚，比如下载任务回调 Block 中的 finished 参数（7 处）到底指什么？这个参数是从最终的下载任务中传递出来的，后面再解释。要注意的是这个方法包含着图片加载的核心流程：__先尝试从缓存获取（5 处），根据获取结果再决定是否要从网络获取（6 处）。__根据缓存获取结果以及开发者设置的选项，分为三种情况：

1. 如果从缓存获取图片失败，或者开发者要求刷新缓存图片，并且根据设置的 delegate 开发者要求重新下载图片（没有设置 delegate 则默认下载），则重新从网络加载图片；
2. 如果从缓存获取图片成功，则直接回调 Block，回传图片；
3. 如果从缓存获取失败，并且根据设置的代理，开发者禁止从网络下载，则直接调用回调 Block；

接下来就来详细看看缓存获取和下载流程。

#### 缓存获取

从缓存获取图片是通过下面方法进行的：

```objective-c
// 来自 SDImageCache.h
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock {
    //1. 参数检测
    if (!doneBlock) {
        return nil;
    }

    if (!key) {
        doneBlock(nil, SDImageCacheTypeNone);
        return nil;
    }

    // 2. 首先从内存换取读取
    // First check the in-memory cache...
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

    // 3. 调度到 IO 线程，从磁盘读取
    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

    return operation;
}
```

SDImageCache 专门负责进行图片缓存，并依赖 AutoPurgeCache 进行内存缓存，它是__单例__的。这里的策略是：

1. 首先从内存缓存中读取，读取到则回调返回；
2. 读取不到调度到异步线程从磁盘读取，读取到了则尝试添加到内存缓存；
3. 调度到主线程执行回调 block；

AutoPurgeCache 继承于 NSCache，添加了内存回收相关功能：

```objective-c
@interface AutoPurgeCache : NSCache
@end

@implementation AutoPurgeCache

- (id)init
{
    self = [super init];
    if (self) {
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(removeAllObjects) name:UIApplicationDidReceiveMemoryWarningNotification object:nil];
    }
    return self;
}

- (void)dealloc
{
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidReceiveMemoryWarningNotification object:nil];

}
```

可以在收到 App 内存警告以及被销毁的时候自动释放所有内存缓存的图片。NSCache 本身提供通过数据数量以及数据体积来限制缓存的功能，这部分功能也通过接口暴露给了外部：

```objective-c
- (void)setMaxMemoryCost:(NSUInteger)maxMemoryCost {
    self.memCache.totalCostLimit = maxMemoryCost;
}

- (void)setMaxMemoryCountLimit:(NSUInteger)maxCountLimit {
    self.memCache.countLimit = maxCountLimit;
}
```

也就是说我们可以在外部根据 App 实际情况设置内存缓存大小。磁盘缓存的控制清理需要手动调用如下方法：

```objective-c
- (void)cleanDisk {
    [self cleanDiskWithCompletionBlock:nil];
}

- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock {
	// 实际清理操作
}
```

根据代码，实际的清理操作流程与时间和缓存文件大小相关，策略如下：

1. 遍历缓存文件夹，先删除过期的文件，期限时长默认设置为一周，即默认删除一周前的文件，时长可设置；
2. 遍历过程中会顺便计算经过 步骤 1 清理之后总的缓存文件大小；
3. 如果开发者设置了缓存文件大小上限，检测剩余缓存文件是否超限，上限值默认为 0，表示未设限制；
4. 如果超限，总体积为 M，则将理想尺寸设置为 M/2，将剩余文件根据修改时间排序，从最旧的文件开始删除，直到体积小于 M/2；

相似的还有一个 clear 操作，它会清除所有缓存。

以上就是 SDWebImage 实现的两级缓存，既保证了图片的加载速度，又保证了不会占用太多的设备资源。

回到缓存获取返回处，即前面注释的代码 5 处（读者可以直接搜 "&&&" 跳转）。从缓存返回之后，有两种情况：1）返回的图片对象可用；2）返回的图片对象不可用。不可用的情况同样分为两种：1）缓存未命中，没有获取到缓存图片；2）开发者要求刷新缓存，从缓存中获取的图片必须经过服务端验证。不可用的情况必须向服务端发送请求，这就是注释 7 处（用户可以直接搜 "@@@" 跳转）的目的。

#### 图片下载

如果从缓存获取的图片不可用，那么就需要从网络进行下载，它调用的是`SDWebImageDownloader`的方法：

```objective-c
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock {
    // SDWebImageDownloaderOperation 真正的任务下载类，继承自 NSOperation
    __block SDWebImageDownloaderOperation *operation;
    __weak __typeof(self)wself = self;

    // 该方法将创建任务下载类实例 & 执行请求
    [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^{
        // 执行请求
        ...
    }];

    return operation;
}
```

这个方法会调用以下方法：

```objective-c
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
    // The URL will be used as the key to the callbacks dictionary so it cannot be nil. If it is nil immediately call the completed block with no image or data.
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }
	// 经典用法
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        // Handle single download of simultaneous download request for the same URL
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
}
```

这段代码比较有趣，首先来了解一下设计场景，SDWebImage 考虑到一个 App 可能同时发起对一张图片的多个请求这种场景，因此当一个图片网络请求执行前，它会先去检测是否已经有一个同样的请求在执行了，如果有，则只是在对应的请求流程上添加回调，而不是直接发起请求。

在`SDWebImageDownloader`内部，维护着一个称为`URLCallbacks`的字典，这个字典的 key 就是请求的 URL，value 是一个数组，每一个数组又包含一个字典，这个字段存储着两个 Block，一个是下载进度回调 Block，一个是下载完成回调 Block，数组里面可能有多个元素，每一个元素代表着一组这样的回调。

如果以该 URL 为 key 的回调组不存在，则说明没有它的下载任务存在，这时候这个方法就会回调`createCallback` Block。这部分代码如下：

```objective-c
// 该方法将创建任务下载类实例 & 执行请求
[self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^{
    NSTimeInterval timeoutInterval = wself.downloadTimeout;
    if (timeoutInterval == 0.0) {
        timeoutInterval = 15.0;
    }

    // 1. 创建请求 & 设置参数
    // In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
    request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
    request.HTTPShouldUsePipelining = YES;
    if (wself.headersFilter) {
        request.allHTTPHeaderFields = wself.headersFilter(url, [wself.HTTPHeaders copy]);
    }
    else {
        request.allHTTPHeaderFields = wself.HTTPHeaders;
    }
    // 2. 初始化任务，添加 Block 回调监听
    operation = [[wself.operationClass alloc] initWithRequest:request
                                                      options:options
                                                     progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                         SDWebImageDownloader *sself = wself;
                                                         if (!sself) return;
                                                         __block NSArray *callbacksForURL;
                                                         // 回调所有的下载进度监听
                                                         dispatch_sync(sself.barrierQueue, ^{
                                                             callbacksForURL = [sself.URLCallbacks[url] copy];
                                                         });
                                                         for (NSDictionary *callbacks in callbacksForURL) {
                                                             dispatch_async(dispatch_get_main_queue(), ^{
                                                                 SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                                                                 if (callback) callback(receivedSize, expectedSize);
                                                             });
                                                             }
                                                     }
                                                    completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                        SDWebImageDownloader *sself = wself;
                                                        if (!sself) return;
                                                        __block NSArray *callbacksForURL;
                                                        // 回调所有的下载完成监听
                                                        dispatch_barrier_sync(sself.barrierQueue, ^{
                                                            callbacksForURL = [sself.URLCallbacks[url] copy];
                                                            if (finished) {
                                                                [sself.URLCallbacks removeObjectForKey:url]; // 要移除回调监听
                                                            }
                                                        });
                                                        for (NSDictionary *callbacks in callbacksForURL) {
                                                            SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                                                            if (callback) callback(image, data, error, finished);
                                                        }
                                                    }
                                                    cancelled:^{
                                                        // 被取消则移除所有的监听
                                                        SDWebImageDownloader *sself = wself;
                                                        if (!sself) return;
                                                        dispatch_barrier_async(sself.barrierQueue, ^{
                                                            [sself.URLCallbacks removeObjectForKey:url];
                                                        });
                                                    }];
    operation.shouldDecompressImages = wself.shouldDecompressImages;
    
    // 设置认证信息(具体可看 Http StatusCode - 401)
    if (wself.urlCredential) {
        operation.credential = wself.urlCredential;
    } else if (wself.username && wself.password) {
        operation.credential = [NSURLCredential credentialWithUser:wself.username password:wself.password persistence:NSURLCredentialPersistenceForSession];
    }
        
    // 设置优先级
    if (options & SDWebImageDownloaderHighPriority) {
        operation.queuePriority = NSOperationQueuePriorityHigh;
    } else if (options & SDWebImageDownloaderLowPriority) {
        operation.queuePriority = NSOperationQueuePriorityLow;
    }

    // 3. 添加并执行任务
    [wself.downloadQueue addOperation:operation];
    // 模仿实现 LIFO 任务
    if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
        // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
        [wself.lastAddedOperation addDependency:operation];
        wself.lastAddedOperation = operation;
    }
}];
```

代码中同样在关键点进行了标注，这里实际只能看到一个发起请求的过程和请求结果的回调，而实际的下载操作在`SDWebImageDownloaderOperation`类中，这个类是使用 URLConnection  发送请求的，下面一个主题将进行分析。

这里解释一下前面提到的一个参数，即加载完成回调 Block 中的 finish 参数：SDWebImage 可以选择采用渐进式图片加载方式，也就是下载多少显示多少，只需要在下载的时候设置参数 `SDWebImageDownloaderProgressiveDownload`即可，finish 参数是用于表示下载是否已经完全结束。

另外再关注一下`downloadQueue`的初始化：

```objective-c
_downloadQueue = [NSOperationQueue new];
_downloadQueue.maxConcurrentOperationCount = 6;
```

最大线程数设置为 6。也就是最多可以同时进行 6 个下载任务。

## 下载 & 解码

SDWebImage 下载的真正实现类是`SDWebImageDownloaderOperation`，这个类继承自__NSOperation__，实现`NSURLConnectionDataDelegate`协议，将网络下载相关操作和下载图片解码操作封装在内部。

> 阅读代码之前最好了解一下如何自定义 NSOperation 和 RunLoop 相关知识。
>
> RunLoop 知识可以参考：[深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/#base)

### 关键方法`start`

`SDWebImageDownloaderOperation`的`start`方法实现大致如下：

```objective-c
- (void)start {
    @synchronized (self) {
        if (self.isCancelled) {
            self.finished = YES;
            [self reset];
            return;
        }

        ....

        self.executing = YES;
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
        self.thread = [NSThread currentThread];
    }

    [self.connection start];

    if (self.connection) {// 1. connection 初始化成功
        if (self.progressBlock) {
            self.progressBlock(0, NSURLResponseUnknownLength);
        }
        // 2. 发送通知
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:self];
        });

        // 3. 启动 RunLoop
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_5_1) {
            // Make sure to run the runloop in our background thread so it can process downloaded data
            // Note: we use a timeout to work around an issue with NSURLConnection cancel under iOS 5
            //       not waking up the runloop, leading to dead threads (see https://github.com/rs/SDWebImage/issues/466)
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 10, false);
        }
        else {
            CFRunLoopRun();
        }

        // 4. 执行取消清理工作
        if (!self.isFinished) {
            [self.connection cancel];
            [self connection:self.connection didFailWithError:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorTimedOut userInfo:@{NSURLErrorFailingURLErrorKey : self.request.URL}]];
        }
    }
    else { // 5. connnection 初始化失败
        if (self.completedBlock) {
            self.completedBlock(nil, nil, [NSError errorWithDomain:NSURLErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Connection can't be initialized"}], YES);
        }
    }

    ...
}
```

比较难理解的是 RunLoop 的启动。在网上查了一些资料，有的人认为这里启动 RunLoop 是为了保证数据加载的流畅性，在滑动的时候不去加载数据。

但我的理解并不是这样，这个下载任务执行是在异步线程里面，与主线程毫无关系，不存在页面流畅不流畅的问题，如果要做到流畅，也应该是在最后调度到主线程的时候控制。再看注释里面有一句 "Make sure to run the runloop in our background thread so it can process downloaded data"，这句话的意思是说必须要启动 RunLoop 以保证它能够处理下载的数据，而下载的的数据是在 delegate 中进行回调处理的。所以我认为：__`SDWebImageDownloaderOperation`实现了`NSURLConnectionDataDelegate`协议并把自己设置为 connection 的 delegate，这里之所以启动 RunLoop，是为了保证`SDWebImageDownloaderOperation`运行在`start`方法中不退出，从而该对象不会被回收，否则一旦`start`方法执行完成，connection 就失去代理无法处理数据了。__

### 代理实现

再来看代理，代理有三个关键时间节点：

1. 初次获得响应，也就是连通服务器的时候；
2. 接收数据的过程；
3. 数据接收完成；

这部分主要是网络相关的内容，在数据接收上没有什么大问题，关键是看数据的处理。下面一一分析。

#### 连接打通

代码如下：

```objective-c
- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    //'304 Not Modified' is an exceptional one
    if (![response respondsToSelector:@selector(statusCode)] || ([((NSHTTPURLResponse *)response) statusCode] < 400 && [((NSHTTPURLResponse *)response) statusCode] != 304)) {
        NSInteger expected = response.expectedContentLength > 0 ? (NSInteger)response.expectedContentLength : 0;
        self.expectedSize = expected;
        if (self.progressBlock) {
            self.progressBlock(0, expected);
        }

        self.imageData = [[NSMutableData alloc] initWithCapacity:expected];
        self.response = response;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadReceiveResponseNotification object:self];
        });
    }
    else {
        NSUInteger code = [((NSHTTPURLResponse *)response) statusCode];
        //This is the case when server returns '304 Not Modified'. It means that remote image is not changed.
        //In case of 304 we need just cancel the operation and return cached image from the cache.
        if (code == 304) {
            [self cancelInternal];
        } else {
            [self.connection cancel];
        }
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:self];
        });

        if (self.completedBlock) {
            self.completedBlock(nil, nil, [NSError errorWithDomain:NSURLErrorDomain code:[((NSHTTPURLResponse *)response) statusCode] userInfo:nil], YES);
        }
        CFRunLoopStop(CFRunLoopGetCurrent());
        [self done];
    }
}
```

如果响应正常，即 statusCode < 400，并且不是 304，那么正常接收数据，否则 cancel 掉连接。关于 304 的实现，其实并没看太明白。在`SDWebImageDownloader.m`中初始化下载 request 的时候，有如下注释：

```objective-c
// In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
request.HTTPShouldUsePipelining = YES;
```

除非开发者手动设置`SDWebImageDownloaderUseNSURLCache`选项，否则不会使用`NSURLRequestUseProtocolCachePolicy`缓存策略，否则会直接从服务端加载，系统是不会帮忙做缓存的，而在代码里面是没有看到手动解析保存请求的 Header 的地方的，因此如果不用系统缓存信息，并使用`NSURLRequestReloadRevalidatingCacheData`缓存策略，那 304 就不可能出现，所以这里 304 到底是什么效果，需要实际调试一把才能弄清楚。

> iOS 缓存可以参考：[NSURLCache](http://nshipster.cn/nsurlcache/)
>
> PS：一般图片变化后 URL 都会随之变化，请求图片要求服务端进行缓存验证的情况并不多见。

#### 接收数据

调用方法如下：

```objective-c
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
	...
}
```

这段代码就不贴出来了，重点在于接收数据的过程中使用 ImageIO 实现渐进式图片加载，关于 ImageIO，可以参考文档 [Image I/O Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/ImageIOGuide/imageio_basics/ikpg_basics.html)。代码注释作者表明代码是来自网站 http://www.cocoaintheshell.com/ 的，但貌似这个网站不存在了。

这段代码有一个小小的细节点需要注意，就是下载下来的图片会通过下面的方法进行拉伸变化：

```objective-c
// SDWebImageCompat.m
inline UIImage *SDScaledImageForKey(NSString *key, UIImage *image) {
    if (!image) {
        return nil;
    }
    
    if ([image.images count] > 0) {
        NSMutableArray *scaledImages = [NSMutableArray array];

        for (UIImage *tempImage in image.images) {
            [scaledImages addObject:SDScaledImageForKey(key, tempImage)];
        }

        return [UIImage animatedImageWithImages:scaledImages duration:image.duration];
    }
    else {
        if ([[UIScreen mainScreen] respondsToSelector:@selector(scale)]) {
            CGFloat scale = [UIScreen mainScreen].scale;
            if (key.length >= 8) {
                NSRange range = [key rangeOfString:@"@2x."];
                if (range.location != NSNotFound) {
                    scale = 2.0;
                }
                
                range = [key rangeOfString:@"@3x."];
                if (range.location != NSNotFound) {
                    scale = 3.0;
                }
            }

            UIImage *scaledImage = [[UIImage alloc] initWithCGImage:image.CGImage scale:scale orientation:image.imageOrientation];
            image = scaledImage;
        }
        return image;
    }
}
```

即会根据屏幕分辨率以及图片 URL 中的特定字符"@2x."和"@3x."来进行图片的比例缩放。

#### 接收完成

调用方法如下：

```objective-c
- (void)connectionDidFinishLoading:(NSURLConnection *)aConnection {
    SDWebImageDownloaderCompletedBlock completionBlock = self.completedBlock;
    @synchronized(self) {
        // 停掉 Loop
        CFRunLoopStop(CFRunLoopGetCurrent());
        self.thread = nil;
        self.connection = nil;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:self];
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadFinishNotification object:self];
        });
    }
    
    // 判断响应是否来自缓存
    if (![[NSURLCache sharedURLCache] cachedResponseForRequest:_request]) {
        responseFromCached = NO;
    }
    
    if (completionBlock) {
        // 如果响应是来自缓存，但是开发者又设置了忽略缓存响应的选项，则回调无图片
        if (self.options & SDWebImageDownloaderIgnoreCachedResponse && responseFromCached) {
            completionBlock(nil, nil, nil, YES);
        } else if (self.imageData) {
            UIImage *image = [UIImage sd_imageWithData:self.imageData];
            NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
            image = [self scaledImageForKey:key image:image];
            
            // 解码图片
            // Do not force decoding animated GIFs
            if (!image.images) {
                if (self.shouldDecompressImages) {
                    image = [UIImage decodedImageWithImage:image];
                }
            }
            if (CGSizeEqualToSize(image.size, CGSizeZero)) {
                completionBlock(nil, nil, [NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}], YES);
            }
            else {
                completionBlock(image, self.imageData, nil, YES);
            }
        } else {
            completionBlock(nil, nil, [NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}], YES);
        }
    }
    self.completionBlock = nil;
    [self done];
}
```

这段代码本身没有什么难懂的地方，但这里面涉及到一个有用的基本知识——__图片类型判断__，记录如下：

```objective-c
+ (NSString *)sd_contentTypeForImageData:(NSData *)data {
    uint8_t c;
    [data getBytes:&c length:1];
    switch (c) {
        case 0xFF:
            return @"image/jpeg";
        case 0x89:
            return @"image/png";
        case 0x47:
            return @"image/gif";
        case 0x49:
        case 0x4D:
            return @"image/tiff";
        case 0x52:
            // R as RIFF for WEBP
            if ([data length] < 12) {
                return nil;
            }

            NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(0, 12)] encoding:NSASCIIStringEncoding];
            if ([testString hasPrefix:@"RIFF"] && [testString hasSuffix:@"WEBP"]) {
                return @"image/webp";
            }

            return nil;
    }
    return nil;
}
```

以前虽然看过一些图片加载库并对比过不同格式的图片的参数，但没有整理过每种图片格式开头的魔法数字，这个函数表达的很清晰，以后如果有相关需求可以拿来直接使用。

> 在 SDWebImage 库中，是需要根据不同的类型使用不同的解码方式生成图片的，读者有兴趣也可以看一下这部分代码。

## 总结

以上内容对 SDWebImage 加载图片的细节做了简单的分析描述。下面通过类图来大致整理一下前面提到的函数和功能实现位置（同样来自项目 Wiki）：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/SDWebImage 类关系图.png" alt="SDWebImage SDWebImage 类关系图" alt="One Piece"/></div>

经过前面的分析，可以系统看一下 SDWebImage 的核心模块：

1. __UIView 模块__ 处于上图的左上角部分，通过 Category 为 UIView、UIImageView 和 UIButton 添加方法，衔接 SDWebImage 的功能；
2. __SDWebImageManager__ 处于上图的中心，它是获取图片流程的制定者，负责调度缓存获取和网络下载两种图片获取方式；
3. __SDImageCache__ 处于上图左下角，负责图片的内存缓存和磁盘缓存；
4. __下载模块__ 处于上图的右边部分，负责图片的下载；

以上四个模块配合前面所说的流程，完成图片的加载、解码以及显示。

最后，SDWebImage 是我学习 iOS 以来分析的第一个流行开源库，因为之前研究过 Andorid 上的一些图片加载库，比如 UIL，Picasso 等，因此觉得分析这样一个库会比较得心应手，也相对有个参考，是一个比较好的学习机会，分析完成之后，对于 GCD，NSOperation，Block，RunLoop 等这样的基础概念有了更多的认识，也算达到了我的目的😁。