---
title: YYCache 源码学习（一）：YYMemoryCache
date: 2018-12-10 17:35:38
tags: [iOS,YYCache]
---
>其实最近是在重新熟练Swift的使用，我想出了一个比较实用的方法，那就是一边看OC的项目，看懂之后用Swift实现一遍。这样既学习了优秀的源码又练习了Swift，一举两得。

>之前看过几篇文章是剖析YYKit里面的一些小模块，对源码对一些解读。不得不说作者ibireme的设计思维和技术细节的处理都非常的棒。所以就选了YYKit里面的一些小模块入手。

YYCache主要分为了两部分：YYMemoryCache内存缓存和磁盘缓存YYDiskCache。平常使用的时候我们一般都只直接操作YYCache这个类，他是对内存缓存和磁盘缓存的封装。

这篇文章主要是讲解YYCache模块里面的YYMemoryCache部分。

<!-- more -->

#### API

我们先可以看一下YYMemoryCache的.h文件，浏览一起属性和方法。大多数的都可以见名知意的。

```objc
@interface YYMemoryCache : NSObject

#pragma mark - Attribute
///=============================================================================
/// @name Attribute
///=============================================================================

@property (nullable, copy) NSString *name;
@property (readonly) NSUInteger totalCount;
@property (readonly) NSUInteger totalCost;

#pragma mark - Limit
///=============================================================================
/// @name Limit
///=============================================================================

@property NSUInteger countLimit;
@property NSUInteger costLimit;
@property NSTimeInterval ageLimit; //过期时间
@property NSTimeInterval autoTrimInterval;//自动处理的间隔时间
@property BOOL shouldRemoveAllObjectsOnMemoryWarning;
@property BOOL shouldRemoveAllObjectsWhenEnteringBackground;
@property (nullable, copy) void(^didReceiveMemoryWarningBlock)(YYMemoryCache *cache);
@property (nullable, copy) void(^didEnterBackgroundBlock)(YYMemoryCache *cache);
@property BOOL releaseOnMainThread;
@property BOOL releaseAsynchronously;

#pragma mark - Access Methods
///=============================================================================
/// @name Access Methods
///=============================================================================
- (BOOL)containsObjectForKey:(id)key;
- (nullable id)objectForKey:(id)key;
- (void)setObject:(nullable id)object forKey:(id)key;
- (void)setObject:(nullable id)object forKey:(id)key withCost:(NSUInteger)cost;
- (void)removeObjectForKey:(id)key;
- (void)removeAllObjects;

#pragma mark - Trim
///=============================================================================
/// @name Trim
///=============================================================================

- (void)trimToCount:(NSUInteger)count;

- (void)trimToCost:(NSUInteger)cost;

- (void)trimToAge:(NSTimeInterval)age;

@end
```

我把乱七八糟的注释都删掉了，这样可以直观的来看，api分为四个部分，前两部分都是一些属性，后面两个是方法。

第一个Attribute部分是YYMemoryCache类储存的一些基本的属性：name，totalCount（储存对象的总个数），totalCost（储存的总占内存）。Limit部分是一些限制条件，就不一一的说了，单说一个`releaseOnMainThread`这个属性，可能会有因为，如果如果能异步释放，为什么还要强制去主线程释放呢 ? 这是因为有一些类，像UIView/CALayer这种是要在主线程中释放的，源码注释中也有提到。

第三部分就是一些跟储存相关的方法，最后一部分就是根据限制条件修剪处理内存的方法了 ~ 


#### .m代码剖析
##### LRU 缓存淘汰算法
YYMemoryCache是提供了内存修剪的方法的，既然有修剪，那么我们得有一个算法来确定是修剪掉哪一些。YYMemoryCache 和 YYDiskCache 都是实现的 LRU (least-recently-used) ，即最近最少使用淘汰算法。具体怎么样实现我们往后再说。

##### 实现缓存方式
.m的最前面是实现了两个内部类_YYLinkedMap和_YYLinkedMapNode。 可以看出具体的缓存方法是通过一个双向列表和散列容器来实现的。

_YYLinkedMap中给出来操作结点的方法

```objc
- (void)insertNodeAtHead:(_YYLinkedMapNode *)node;
- (void)bringNodeToHead:(_YYLinkedMapNode *)node;
- (void)removeNode:(_YYLinkedMapNode *)node;
- (_YYLinkedMapNode *)removeTailNode;
- (void)removeAll;
```

##### 具体细节剖析

先总起来说一下这些实现的代码，其实很容易读懂，就是通过一个链表的形式来处理缓存的数据，添加缓存的时候，就往链表的尾部添加一个节点，（通过节点来表示我们实际要储存的数据），如果要根据限制条件修剪内存的话，也是循环的删除尾部的那个节点，直到符合限制条件。那我们的LRU 缓存淘汰算法具体怎么使用，我发现在每一个读取了一个数据之后，会把这个数据在链表中对应的结点移动到头部，这样在大概率的情况下使用频率高的缓存数据会在链表的前面。所以修剪的时候可以从尾部修剪。

在具体的实现代码中，作者有很多很亮眼的操作，我们来欣赏一下。

 1.定时修剪内存

```objc
// 根据限制条件修剪内存的占用  并根绝设定的时间递归调用
- (void)_trimRecursively {
    __weak typeof(self) _self = self;
    //注意这个dispatch_after后面使用的dispatch_get_global_queue  异步
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(_autoTrimInterval * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
        __strong typeof(_self) self = _self;
        if (!self) return;
        [self _trimInBackground];
        [self _trimRecursively];
    });
}
```

这个就是通过一个延时来调用修剪内存的方法，然后在递归调用本身。注意的是dispatch_after后面使用的dispatch_get_global_queue 来进行异步操作。

2.修剪内存的逻辑

```objc
- (void)_trimToCost:(NSUInteger)costLimit {
    BOOL finish = NO;
    pthread_mutex_lock(&_lock);
    if (costLimit == 0) {
        [_lru removeAll];
        finish = YES;
    } else if (_lru->_totalCost <= costLimit) {
        finish = YES;
    }
    pthread_mutex_unlock(&_lock);
    if (finish) return;
    
    NSMutableArray *holder = [NSMutableArray new];
    while (!finish) {
        if (pthread_mutex_trylock(&_lock) == 0) {
            if (_lru->_totalCost > costLimit) {
                _YYLinkedMapNode *node = [_lru removeTailNode];
                if (node) [holder addObject:node];
            } else {
                finish = YES;
            }
            pthread_mutex_unlock(&_lock);
        } else {
            usleep(10 * 1000); //10 ms
        }
    }
    //释放
    if (holder.count) {
        dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
        dispatch_async(queue, ^{
            [holder count]; // release in queue
        });
    }
}
```

我们以这个按照占内存大小的限制来修剪内存为例子来看一下具体的实现方法。

开始先做几个判断，看是否超过限制需要修剪，不需要修剪就直接return。

修剪的过程就跟上面说到的是一样的，判断是否超过内存限制，超过就删掉尾部的结点，如此循环操作，只要符合限制要求。

**异步释放资源：**

**我们可以看到在removeNode的时候，会使用一个holder数组来接收被移除的这些node**，然后最后释放这些结点。为什么要这样做？其实这就是通过异步释放这些资源来减少主线程资源的开销。这里作者在异步中调用了[holder count]; 其实最开始我也不知道这个是什么意思，但是作者标注了release in queue，我猜测是**通过调用你这个holder的随便一个方法，让这个异步的线程来管理这个holder，进而通过此异步线程来实现holder中对象的释放**。

**锁的使用：**

还有一个比较重要的点，**为什么使用pthread_mutex_trylock这个方式加锁，然后在失败之后，线程要sleep。**

这个问题就需要我们去研究一下各种锁了。很惭愧我对锁的了解不是很深刻，但是通过看了大神的博客，有了一些了解。（下面内容引用自大神的博客，文末有地址）

作者都是使用的pthread_mutex_t互斥锁，这个锁有一个特性，在多个线程竞争一个资源的时候，除了竞争成功的线程，其他的线程都会被动挂起状态，当竞争成功的线程解锁是，会去主动将挂起的其他线程激活，这个过程包含了上下文切换，CPU抢占，信号发送等开销，很明显，开销有些大。

所以作者使用了pthread_mutex_trylock()尝试解锁，若解锁失败该方法会立即返回，让当前线程不会进入被动的挂起状态（也可以说阻塞），在下一次循环时又继续尝试获取锁。这个过程很有意思，感觉是手动实现了一个自旋锁。而自旋锁有个需要注意的问题是：死循环等待的时间越长，对 cpu 的消耗越大。所以作者做了一个很短的睡眠 usleep(10 * 1000)，有效的减小了循环的调用次数，至于这个睡眠时间的长度为什么是 10ms， 作者应该做了测试。

##### 其他部分
其他部分就不一一细说了，作者整体思路很清晰，然后代码逻辑也很好懂，像上面提到的一些细节的处理可见作者的技术水平了。

#### 参考
https://www.jianshu.com/p/408d4d37bcbd



