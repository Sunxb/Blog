---
title: SDWebImage-源码学习(二)
date: 2018-02-07 12:28:46
tags: [iOS,SDWebImage]
---

#### 常用的枚举
在使用SDWebImage的时候, 会根据实际的需求对操作进行设置, 这就用到了**SDWebImageManager类中的枚举SDWebImageOptions**
, 首先我们就先来看一下这个枚举, 了解一下每个字段代表什么意思

```objc
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {
    /**
     * 默认情况下, 当一个url下载失败会被加入黑名单, 被加入黑名单的url就不会再被尝试下载了
     * 本标识作用: 禁用黑名单, 也就是失败的url会再次尝试请求
     */
    SDWebImageRetryFailed = 1 << 0,

    /**
     默认情况, 图片在交互的时候就开始下载,
     本标识作用:  降低图片下载的优先级, 让图片在UIScrollView减速的时候才下载
     */
    SDWebImageLowPriority = 1 << 1,

    /**
      禁用磁盘缓存
     */
    SDWebImageCacheMemoryOnly = 1 << 2,

    /**
      默认情况下, 图片是下载完事之后在直接显示出来
      本标识的作用是开启下载进度, 让图片的显示跟pc上的浏览器加载图片一样从上到下一边下载一边展示(差不多就是这个意思啦, 可能描述不太准确)
     */
    SDWebImageProgressiveDownload = 1 << 3,

    /**
      即使图片已经缓存, 也要根据HTTP缓存策略去网上重新加载图片,然后刷新缓存数据
      这个磁盘缓存是用NSURLCache处理而不是SDWebImage,这会导致一些性能下降
      这个选项是用来处理那些相同url但是图片改变了的情况
      如果缓存的图片刷新了, complete block会调用两次, 一次是原来的缓存图片,一次是新的图片
      在你不能保证url对应图片不变的时候使用此选项
     */
    SDWebImageRefreshCached = 1 << 4,

    /**
     在iOS4之后版本中, 如果app进入后台也可以继续下载. app进入后台,通过申请额外时间来完成下载. 如果后台任务超时, 则操作会被取消
     */
    SDWebImageContinueInBackground = 1 << 5,

    /**
     通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES, 处理缓存在NSHTTPCookieStore中的cookies
     */
    SDWebImageHandleCookies = 1 << 6,

    /**
    允许不受信任的SSL证书. 在测试环境下很有用, 生产环境慎用
     */
    SDWebImageAllowInvalidSSLCertificates = 1 << 7,

    /**
      默认情况下, 图片在队列中按顺序下载, 本选项可以提高他们的优先级~ 
     */
    SDWebImageHighPriority = 1 << 8,
    
    /**
    默认情况下, 占位图会在图片开始加载时就先显示出来
    本选项会延迟占位图的显示, 直到图片加载完才加载占位图(那还不是不显示占位图了?)
     */
    SDWebImageDelayPlaceholder = 1 << 9,

    /**
    通常我们不会去调用animated images(估计就是多张图片循环显示或者GIF图片)的transformDownloadedImage的代理方法来处理图片。因为大部分transformation操作会对图片做无用处理。
     这个选项表示无论如何都要对图片做transform处理
     */
    SDWebImageTransformAnimatedImage = 1 << 10,
    
    /**
    默认情况下, 图片在下载完成后就会添加到imageView上. 但是有一些情况下, 我们希望在添加到imageView前对图片做一些操作,  所以可以在你想要手动设置下载的图片时使用此选项
     */
    SDWebImageAvoidAutoSetImage = 1 << 11
};
```

>拓展 : NS_ENUM和NS_OPTIONS

<!-- more -->

之前我们定义枚举一般都是用NS_ENUM(enum太老 ~ 就不提了), 它和NS_OPTIONS有什么区别呢 ? 
先看一下定义时的规范
```objc
//**要注意值的区间不要超过所使用类型的最大容纳范围。**
typedef NS_OPTIONS(NSUInteger, UISwipeGestureRecognizerDirection) {
    UISwipeGestureRecognizerDirectionNone = 0,  //值为0
    UISwipeGestureRecognizerDirectionRight = 1 << 0,  //值为2的0次方
    UISwipeGestureRecognizerDirectionLeft = 1 << 1,  //值为2的1次方
    UISwipeGestureRecognizerDirectionUp = 1 << 2,  //值为2的2次方
    UISwipeGestureRecognizerDirectionDown = 1 << 3  //值为2的3次方
};
```
```objc
typedef NS_ENUM(NSInteger, NSWritingDirection) {
    NSWritingDirectionNatural = 0,  //值为0    
    NSWritingDirectionLeftToRight,  //值为1
    NSWritingDirectionRightToLeft  //值为2       
};
```
NS_ENUM定义通用枚举，NS_OPTIONS定义位移枚举,  位移枚举即是在你需要的地方可以同时存在多个枚举值如这样：

```objc
UISwipeGestureRecognizer *swipeGR = [[UISwipeGestureRecognizer alloc] init];
  swipeGR.direction = UISwipeGestureRecognizerDirectionDown | UISwipeGestureRecognizerDirectionLeft | UISwipeGestureRecognizerDirectionRight;
```
而NS_ENUM定义的枚举不能几个枚举项同时存在，只能选择其中一项，像这样：
```objc
NSMutableParagraphStyle *paragraph = [[NSMutableParagraphStyle alloc] init];
paragraph.baseWritingDirection = NSWritingDirectionNatural;
```

#### SDImageCache 缓存部分

缓存这部分的代码, 虽然看起来不少, 但是主要的流程还是比较清晰的, 代码也不难懂, 下面是主要的两个方法

```objc
/**
 把一张图片存入缓存的具体实现

 @param image 缓存的图片对象
 @param imageData 缓存的图片数据
 @param key 缓存对应的key
 @param toDisk 是否缓存到磁盘
 @param completionBlock 缓存完成回调
 */
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    //缓存到内存
    if (self.config.shouldCacheImagesInMemory) {
        //计算缓存数据的大小
        NSUInteger cost = SDCacheCostForImage(image);
        //加入缓存
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    if (toDisk) {
        /*
         在一个线性队列中做磁盘缓存操作。
         */
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;
            if (!data && image) {
                //获取图片的类型GIF/PNG等
                SDImageFormat imageFormatFromData = [NSData sd_imageFormatForImageData:data];
                //根据指定的SDImageFormat。把图片转换为对应的data数据
                data = [image sd_imageDataAsFormat:imageFormatFromData];
            }
            //把处理好了的数据存入磁盘
            [self storeImageDataToDisk:data forKey:key];
            if (completionBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {
        if (completionBlock) {
            completionBlock();
        }
    }
}

/**
 把图片资源存入磁盘

 @param imageData 图片数据
 @param key key
 */
- (void)storeImageDataToDisk:(nullable NSData *)imageData forKey:(nullable NSString *)key {
    if (!imageData || !key) {
        return;
    }
    
    [self checkIfQueueIsIOQueue];
    //缓存目录是否已经初始化
    if (![_fileManager fileExistsAtPath:_diskCachePath]) {
        [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    
    // get cache Path for image key
    //获取key对应的完整缓存路径
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    // transform to NSUrl
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    //把数据存入路径
    [_fileManager createFileAtPath:cachePathForKey contents:imageData attributes:nil];
    
    // disable iCloud backup
    if (self.config.shouldDisableiCloud) {
        //给文件添加到运行存储到iCloud属性
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
```

其中还涉及到一些图片类型和转化的问题, 具体可以看代码来了解

#### 运行时
从源码中可以看到大量的使用运行时特性的地方, 如果加载图片时候的菊花loading, 就是通过运行时加上的

```objc
#pragma mark - Activity indicator
#pragma mark -
#if SD_UIKIT
- (UIActivityIndicatorView *)activityIndicator {
    return (UIActivityIndicatorView *)objc_getAssociatedObject(self, &TAG_ACTIVITY_INDICATOR);
}

- (void)setActivityIndicator:(UIActivityIndicatorView *)activityIndicator {
    objc_setAssociatedObject(self, &TAG_ACTIVITY_INDICATOR, activityIndicator, OBJC_ASSOCIATION_RETAIN);
}
#endif

- (void)sd_setShowActivityIndicatorView:(BOOL)show {
    objc_setAssociatedObject(self, &TAG_ACTIVITY_SHOW, @(show), OBJC_ASSOCIATION_RETAIN);
}

- (BOOL)sd_showActivityIndicatorView {
    return [objc_getAssociatedObject(self, &TAG_ACTIVITY_SHOW) boolValue];
}

#if SD_UIKIT
- (void)sd_setIndicatorStyle:(UIActivityIndicatorViewStyle)style{
    objc_setAssociatedObject(self, &TAG_ACTIVITY_STYLE, [NSNumber numberWithInt:style], OBJC_ASSOCIATION_RETAIN);
}

- (int)sd_getIndicatorStyle{
    return [objc_getAssociatedObject(self, &TAG_ACTIVITY_STYLE) intValue];
}
```
这几个方法不仅仅包括使用runtime在类中动态添加变量, 还包括为动态添加的变量赋值,取值, 同时呢我还注意到了`OBJC_ASSOCIATION_RETAIN`这个宏, 然后就查了一下

```objc
/* Associative References */

/**
 * Policies related to associative references.
 * These are options to objc_setAssociatedObject()
 */
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

`objc_AssociationPolicy`是一个枚举类型的数据结构定义了`OBJC_ASSOCIATION_ASSIGN`、`OBJC_ASSOCIATION_RETAIN_NONATOMIC`、`OBJC_ASSOCIATION_COPY_NONATOMIC`、`OBJC_ASSOCIATION_RETAIN`和`OBJC_ASSOCIATION_COPY`这样五个关联对象特性，每个特性的描述如下：

- OBJC_ASSOCIATION_ASSIGN, 给关联对象指定弱引用,相当于@property(assign)或@property(unsafe_unretained)

- OBJC_ASSOCIATION_RETAIN_NONATOMIC, 给关联对象指定非原子的强引用, 相当于@property(nonatomic, strong)或@property(nonatomic, retain)

- OBJC_ASSOCIATION_COPY_NONATOMIC, 给关联对象指定非原子的copy特性, 相当于@property(nonatomic, copy)

- OBJC_ASSOCIATION_RETAIN, 给关联对象指定原子强引用, 相当于@property(atomic, strong)或@property(atomic, retain)

- OBJC_ASSOCIATION_COPY, 给关联对象指定原子copy特性, 相当于@property(atomic, copy)


