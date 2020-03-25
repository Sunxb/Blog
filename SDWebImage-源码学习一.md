---
title: SDWebImage-源码学习(一)
date: 2018-02-05 12:40:53
tags: [iOS,SDWebImage]
---


##### 通过最常用的方法切入
    sd_setImageWithURL: placeholderImage:
这是我们最常用到的SDWebImage的方法--为imageView添加网络图片,并设置展位图. 
这篇文章就先用这个最近本的方法入手, 梳理SDWebImage在为imageView加载图片时的整个流程.
先看一张图:
![流程图](http://upload-images.jianshu.io/upload_images/1491333-bbe596e8b10dbd6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这是从SDWebImage的github主页上拿到的一张图片, 描述的就是加载图片这一基本流程.
先看一下涉及到的几个类及其在整个流程中的作用(当前最新版本4.2.1):

`UIImageView+WebCache` : 为imageView设置图片,

`UIView+WebCache` : 所有的UIButton、UIImageView都回调用这个分类的方法来完成图片加载的处理。同时通过UIView+WebCacheOperation分类来管理请求的取消和记录工作。所有UIView及其子类的分类都是用这个类的sd_internalSetImageWithURL:来实现图片的加载。,

`SDWebImageManager`: 这是SDWebImage的核心,是一个单例类,拥有SDImageCache和SDWebImageDownloader的属性,分别用于图片缓存和加载处理, 通过这个类为UIView及其子类提供了加载图片的统一接口. 还可以管理正在操作的集合, 加载选项处理等.

`SDImageCache`:核心类,单例,负责图片的缓存工作. 加载图片的时候先要从缓存中取, 缓存中没有在进行downloader这一步

`SDWebImageDownloader`:实现了图片加载的具体处理. 如果存在缓存,直接使用缓存,如果不存在缓存,则创建下载任务. 当然上面的图片有一个点没有别墅出来, 是一个细节, **如果缓存中没有的话, 也不是立即就创建下载任务, 而是先查看正在进行的任务集合里是否已经有对应的任务正在进行了, 如果有了就不需要重复创建.**


##### 下面来看具体的代码
提示: 我看过了最近版的代码, 因为SDWebImage这个框架更新到现在已经很成熟, 功能也很完备, 如果直接从最新版的源码入手, 很吃力, 因为里面有很多功能性的代码不利于我们学习这个框架的思想. 

<!-- more -->

首先, 我看了1.0版本, 根本不需要解释, 只有最核心的这几个文件, 完全印证了基本流程.
![版本1.0](http://upload-images.jianshu.io/upload_images/1491333-23a998987af9ea11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后, 是3.8.2版本,(看了一个视频公开课, 用的这个版本),下面具体看一下代码, 这部分代码在最新的版本里已经重构到UIView+WebCache的sd_internalSetImageWithURL这个方法中的,没关系,重点是逻辑, 我代码中加了基本的注释


    - (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
        //移除UIImageView当前绑定的操作.
        //当TableView的cell包含的UIImageView被重用的时候首先执行这一行代码,保证这个ImageView的下载和缓存组合操作都被取消
        [self sd_cancelCurrentImageLoad];
        
        //利用运行时,把UIImageView的加载图片操作和他自身用关联对象关联起来，方便后面取消等操作。关联的key就是UIImageView对应的类名
        objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
        if (!(options & SDWebImageDelayPlaceholder)) {
            dispatch_main_async_safe(^{
                self.image = placeholder;
            });
        }
        
        if (url) {
            if ([self showActivityIndicatorView]) {
                [self addActivityIndicator];
            }
    
            __weak __typeof(self)wself = self;
            id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            // 这是下载完成之后的回调代码
                [wself removeActivityIndicator];
                if (!wself) return;
                
                // 根据具体设置的策略完成显示
                dispatch_main_sync_safe(^{
                    if (!wself) return;
                    if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                    {
                        completedBlock(image, error, cacheType, url);
                        return;
                    }
                    else if (image) {
                        wself.image = image;
                        [wself setNeedsLayout];
                    } else {
                        if ((options & SDWebImageDelayPlaceholder)) {
                            wself.image = placeholder;
                            [wself setNeedsLayout];
                        }
                    }
                    if (completedBlock && finished) {
                        completedBlock(image, error, cacheType, url);
                    }
                });
            }];
            [self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
        } else {
            dispatch_main_async_safe(^{
                [self removeActivityIndicator];
                if (completedBlock) {
                    NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                    completedBlock(nil, error, SDImageCacheTypeNone, url);
                }
            });
        }
    }
    

下一段代码就是SDWebImageManager里面的loadImageWithURL部分了

    - (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                             options:(SDWebImageOptions)options
                                            progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                           completed:(SDWebImageCompletionWithFinishedBlock)completedBlock {
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
    
    
        //图片加载获取获取过程中绑定一个`SDWebImageCombinedOperation`对象。以方便后续再通过这个对象对url的加载控制。
        __block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
        __weak SDWebImageCombinedOperation *weakOperation = operation;
    
        BOOL isFailedUrl = NO;
        //@synchronized() 多线程同步锁
        @synchronized (self.failedURLs) {
            // 检查当前url是否是失败url
            isFailedUrl = [self.failedURLs containsObject:url];
        }
    
    //SDWebImageRetryFailed 失败后要重试 如果url在失败url集合中并且没有设置重试, 直接处理掉
        if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
            dispatch_main_sync_safe(^{
                NSError *error = [NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil];
                completedBlock(nil, error, SDImageCacheTypeNone, YES, url);
            });
            return operation;
        }
    
    // 当前的操作添加到runningOperations这个集合中, 里面都是正在进行的operation
        @synchronized (self.runningOperations) {
            [self.runningOperations addObject:operation];
        }
    
        NSString *key = [self cacheKeyForURL:url];
    // 根据key查找缓存
        operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
            //如果操作被取消, 则要把操作从runningOperations移除
            if (operation.isCancelled) {
                @synchronized (self.runningOperations) {
                    [self.runningOperations removeObject:operation];
                }
    
                return;
            }
            
            // 没有缓存图片 或者 设置SDWebImageRefreshCached
            if ((!image || options & SDWebImageRefreshCached) && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url])) {
                //有图片 并且 设置SDWebImageRefreshCached 先把图片返回去 然后 重新下载图片刷新缓存
                if (image && options & SDWebImageRefreshCached) {
                    dispatch_main_sync_safe(^{
                        // If image was found in the cache but SDWebImageRefreshCached is provided, notify about the cached image
                        // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
                        completedBlock(image, nil, cacheType, YES, url);
                    });
                }
    
                //根据图片加载的`SDWebImageOptions`类型枚举   设置图片下载的`SDWebImageDownloaderOptions`类型的枚举
                // download if no image or requested to refresh anyway, and download allowed by delegate
                SDWebImageDownloaderOptions downloaderOptions = 0;
                if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
                if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
                if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
                if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
                if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
                if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
                if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
                if (image && options & SDWebImageRefreshCached) {
                    // force progressive off if image already cached but forced refreshing
                    downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                    // ignore image read from NSURLCache if image if cached but force refreshing
                    downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
                }
                
                //下载
                id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) {
                    __strong __typeof(weakOperation) strongOperation = weakOperation;
                    if (!strongOperation || strongOperation.isCancelled) {
                        // Do nothing if the operation was cancelled
                        // See #699 for more details
                        // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                    }
                    else if (error) {
                        dispatch_main_sync_safe(^{
                            if (strongOperation && !strongOperation.isCancelled) {
                                completedBlock(nil, error, SDImageCacheTypeNone, finished, url);
                            }
                        });
    
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
                    else {
                        //如果有重试失败下载的选项。则把url从failedURLS中移除
                        if ((options & SDWebImageRetryFailed)) {
                            @synchronized (self.failedURLs) {
                                [self.failedURLs removeObject:url];
                            }
                        }
                        
                        BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);
    
                        if (options & SDWebImageRefreshCached && image && !downloadedImage) {
                            // Image refresh hit the NSURLCache cache, do not call the completion block
                        }
                        
                        // 下载的可能是gif图? 做transform处理
                        else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
                            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                                UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];
    
                                if (transformedImage && finished) {
                                    BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                    [self.imageCache storeImage:transformedImage recalculateFromImage:imageWasTransformed imageData:(imageWasTransformed ? nil : data) forKey:key toDisk:cacheOnDisk];
                                }
    
                                dispatch_main_sync_safe(^{
                                    if (strongOperation && !strongOperation.isCancelled) {
                                        completedBlock(transformedImage, nil, SDImageCacheTypeNone, finished, url);
                                    }
                                });
                            });
                        }
                        else {
                            if (downloadedImage && finished) {
                                [self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
                            }
    
                            dispatch_main_sync_safe(^{
                                if (strongOperation && !strongOperation.isCancelled) {
                                    completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
                                }
                            });
                        }
                    }
    
                    //结束后移除操作
                    if (finished) {
                        @synchronized (self.runningOperations) {
                            if (strongOperation) {
                                [self.runningOperations removeObject:strongOperation];
                            }
                        }
                    }
                }];
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
            
            //如果有缓存图片直接返回
            else if (image) {
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
            else {
                // 没有图片  也没有下载
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


本篇就先写这么多吧 ~ 








