---
title: RunLoop（二）：实际应用
date: 2019-03-12 17:30:18
tags: [iOS,RunLoop]
---


前不久我们我们对RunLoop的底层有了简单的了解，那我们现在就要把我们学到的这些东西，实际应用到我们的项目中。

<!-- more -->

#### Timer定时器问题

我们在vc中创建一个定时器，然后在view上面添加一个滚动视图，比如说scrollView，可以发现在scrollView滚动的时候，timer定时器会卡住，停止滚动之后才重新生效。

这个问题比较简单，也是我们经常遇到的。

因为定时器默认是添加在了RunLoop的NSDefaultRunLoopMode模式下，scrollView在滚动的时候会进入UITrackingRunLoopMode，RunLoop在同一时间只能处理一种mode，所以在滚动的时候，自然定时器就没法处理，卡住。

解决方法就是我们创建了timer之后，把他add到RunLoop的NSRunLoopCommonModes，NSRunLoopCommonModes其实并不是一种真实的模式，他只是一个标志，意味着timer在标记为common的模式下都能使用 (标记为common 也就是_commonModes数组)。

这个地方多说一句，这个标记为common是啥意思。我们得看回RunLoop结构体的源码

```objc
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

可以看到里面有一个set类型的变量，`CFMutableSetRef _commonModes;`，被放到这个set中的mode就等于是被标记为了common。NSDefaultRunLoopMode和UITrackingRunLoopMode都在里面。

下面是我们创建timer的正确姿势 ~ 

```objc
    //我们平时可能都是用scheduledTimerWithTimeInterval这个方法创建，这个会默认把timer添加到runloop的defalut模式下，所以我们使用timerWithTimeInterval创建
    NSTimer * timer = [NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"%d",++ count);
    }];

    //NSRunLoopCommonModes 并不是一个真的模式 他只是一个标记,意味着timer在标记为common的模式下都能使用 (标记为common 也就是_commonModes数组)
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

#### 线程保活

线程保活并不是所有的项目都用的到，他适应于那种一直有任务需要处理的场景，而且注意，一定要是**串行**的任务。这种情况下保活一条线程，就可以免去线程创建和销毁的开销，提高性能。

具体怎么保活线程，我下面先直接把我的代码贴出来，然后针对一些点在做一系列的说明。（模拟的项目场景是进入到一个vc中，开一条线程，然后用这条线程来执行任务，当然vc销毁时，线程也要销毁。）

下面是全部代码，大家可以先跳过代码看下面的一些解析。

```objc
#import "SecondViewController.h"

@interface MyThread : NSThread
@end
@implementation MyThread
- (void)dealloc {
    NSLog(@"%s",__func__);
}
@end

@interface SecondViewController ()
@property (nonatomic, strong) MyThread * thread;
@property (nonatomic, assign, getter=isStopped) BOOL stopped;
@end

@implementation SecondViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    
    self.stopped = NO;
    
    UIButton * btn = [UIButton buttonWithType:UIButtonTypeCustom];
    btn.frame = CGRectMake(40, 100, 100, 40);
    btn.backgroundColor = [UIColor blackColor];
    [btn setTitle:@"停止" forState:UIControlStateNormal];
    [btn addTarget:self action:@selector(stopThread) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:btn];
    
    __weak typeof(self) weakSelf = self;
    // 初始化thread
    self.thread = [[MyThread alloc] initWithBlock:^{
        NSLog(@"--begin--");
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
        
        while (weakSelf && !weakSelf.isStopped) {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        
        NSLog(@"--end--");
    }];
    [self.thread start];
    
}

- (void)stopThread {
    if (!self.thread) return;
    
    // waitUntilDone YES
    [self performSelector:@selector(__stopThread) onThread:self.thread withObject:nil waitUntilDone:YES];
}

// 执行这个方法必须要在我们自己创建的这个线程中
- (void)__stopThread {    
    // 标识
    self.stopped = YES;
    // 停止runloop
    CFRunLoopStop(CFRunLoopGetCurrent());
    //
    self.thread = nil;
}


#pragma mark - 添加touch事件  (每点击一次 让线程处理一次事件)
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    if (!self.thread) return;

    [self performSelector:@selector(threadDoSomething) onThread:self.thread withObject:nil waitUntilDone:NO];
}

- (void)threadDoSomething {
    NSLog(@"work--%@",[NSThread currentThread]);
}


#pragma mark - dealloc
- (void)dealloc {
    NSLog(@"%s",__func__);
    [self stopThread];
}

@end

```

最顶部新建了一个继承自NSThread的MyThread类，目的就是为了重写`-dealloc`方法，在内部有打印内容，方便我调试线程是否被销毁。在我们真是的项目中，可以不需要这部分。

##### 初始化线程，开启RunLoop

```objc
    __weak typeof(self) weakSelf = self;
    // 初始化thread
    self.thread = [[MyThread alloc] initWithBlock:^{
        NSLog(@"--begin--");
        //往runloop里面添加source/timer/observer
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
        
        while (weakSelf && !weakSelf.isStopped) {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        
        NSLog(@"--end--");
    }];
    [self.thread start];
```

这部分是初始化我们的线程，线程的初始化我们一般用的多的是`self.thread = [[MyThread alloc] initWithTarget:self selector:@selector(runThread) object:nil];`这样的方法，我是觉得这样把self传进线程内部，可能造成一些循环引用问题，最后影响vc和thread的销毁，所以我是用了block的形式。

`initWithBlock`的意思也就是线程初始化完毕会执行block内的代码。一个子线程默认是没有RunLoop的，RunLoop会在第一次获取的时候创建，所以我们先`[NSRunLoop currentRunLoop]`获取RunLoop，也就是创建了我们当前线程的RunLoop。

在了解RunLoop底层的时候我们了解到，如果一个RunLoop没有timer、observer、source，就会退出。我们新创建的RunLoop这些都是没有的，如果我们不手动的添加，那我们的RunLoop一跑起来就这就会退出的。所以就等于说我们必须手动给RunLoop添加点事情做。

在代码中我们使用了`addPort:forMode`这个方法，向当前RunLoop添加一个端口让RunLoop监听。RunLoop有工作做了，自然就不会退出的。

我们在开启线程的时候，用了一个while循环，通过一个属性`stopped`来控制是否跳出循环，然后循环内部使用了`- (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;`这个方法开启RunLoop。有人有可能会问了，这里的开启RunLoop为什么不直接使用`- (void)run; `这个方法。这里我稍微解释一下：

查阅一下苹果的文档可以了解到，这个run方法，内部其实也是循环的调用了runMode这个方法的，但是这个循环是永远不会停止的，也就是说我们使用run方法开启的RunLoop是永远都不会停下来的，我们调用了stop之后，也只会停止当前的这一次循环，他还是会继续run起来的。所以文档中也提到，如果我们要创建一个可以停下来的RunLoop，用runMode这个方法。所以我们用这个while循环模拟run的运行原理，但是呢，我们通过stopped这个属性可以控制循环的停止。

while里面的条件`weakSelf && !weakSelf.isStopped`为什么不仅仅使用stopped判断，而是还要判断weakSelf是否有值？我们下面会提到的。

##### 两个stopThread方法


```objc
- (void)stopThread {
    if (!self.thread) return;
    
    // waitUntilDone YES
    [self performSelector:@selector(__stopThread) onThread:self.thread withObject:nil waitUntilDone:YES];
}

// 执行这个方法必须要在我们自己创建的这个线程中
- (void)__stopThread {    
    // 标识置为YES，跳出while循环
    self.stopped = YES;
    // 停止runloop的方法
    CFRunLoopStop(CFRunLoopGetCurrent());
    // RunLoop退出之后，把线程置空释放，因为RunLoop退出之后就没法重新开启了
    self.thread = nil;
}

```

`stopThread`是给我们的停止button调用的，但是实际的停止RunLoop操作在`__stopThread`里面。在`stopThread`中调用`__stopThread`一定要使用`performSelector:onThread:`这一类的方法，这样就可以保证在我们指定的线程中执行这个方法。如果我们直接调用`__stopThread`，就说明是在主线程调用的，那就代表我们把主线程的RunLoop停掉了，那我们的程序就完了。


##### touch模拟事件处理

我们在`touchBegin`方法中，让我们self.thread执行`-threadDoSomething`这个方法，代表每点击一次，我们的线程就要处理一次`-threadDoSomething`中的打印事件。做这个操作是为了检测看我们每次工作的线程是不是都是我们最开始创建的这一个线程，没有重新开新线程。

##### 其他细节

那我们仔细观察的话会发现一个问题，`-threadDoSomething`和`stopThread`这两个方法中都是用下面这个方法来处理线程间通信

`- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait`

但是两次调用传入的wait参数是不一样的。我们要先知道这个`waitUntilDone:(BOOL)wait`代表什么意思。

如果wait传的是YES，就代表我们在主线程用self调用这个performSelector的时候，主线程会等待我们的self.thread这个线程执行他需要执行的方法，等着self.thread执行完方法之后，主线程再继续往下走。那如果是NO，肯定就是主线程不会等了，主线程继续往下走，然后我们的self.thread去调用自己该调用的方法。

那为什么在stop方法中是用的YES？

有这么一个情形，如果我们push进这个vc，线程初始化，然后RunLoop开启，但是我们不想通过点击停止button来停止，当我们点击导航的back的时候，我也需要销毁线程。

所以我们在vc的`-dealloc`方法中也调用了stopThread方法。那如果stopThread中使用
`- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait`
的时候wait不用YES，而是NO，会出现什么情况，那肯定是crash了。

如果wait是NO，代表我们的主线程不会等待self.thread执行`__stopThread`方法。

```objc
#pragma mark - dealloc
- (void)dealloc {
    NSLog(@"%s",__func__);
    [self stopThread];
}
```

但是dealloc中主线程调用完stopThread，之后整个dealloc方法就结束了，也就是我们的控制器已经销毁了。但是呢这个时候self.thread还在执行`__stopThread`方法呢。`__stopThread`中还要self变量，但是他其实已经销毁了，所以这个地方就会crash了。所以在`stopThread`中的wait一定要设置为YES。

在当时写代码的时候，这样确实处理了crash的问题，但是我直接返回值后，RunLoop并没有结束，线程没有销毁。这就要讲到上面说的while判断条件是`weakSelf && !weakSelf.isStopped`的原因了。vc执行了dealloc之后，self被置为nil了，weakSelf.isStopped也是nil，取非之后条件又成立了，while循环还要继续的走，RunLoop又run起来了。所以这里我们加上`weakSelf`这个判断，也就是self必须不为空。


#### 总结

上面就是我实现的线程保活这一功能的代码和细节分析，当然我们在实际的项目中可能有多个位置需要线程保活这一功能，所以我们应该把这一部分做一下简单的封装，来方便我们在不同的地方调用。大家有兴趣的可以自己封装一下，我在写RunLoop相关的代码时，大多用的是OC层的代码，有兴趣的小伙伴可以尝试一下C语言的API。

RunLoop的应用当前不止这么一点，还可以监控应用卡顿，做性能优化，这些以后研究明白了再继续更博客吧，一起加油。

相关的功能代码和封装已经放到github上面了
https://github.com/Sunxb/RunLoopDemo


