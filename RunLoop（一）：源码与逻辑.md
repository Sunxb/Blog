---
title: RunLoop（一）：源码与逻辑
date: 2019-03-06 15:41:39
tags: [iOS,RunLoop]
---


#### 简述
什么是RunLoop？顾名思义RunLoop是一个运行循环，它的作用是使得程序在运行之后不会马上退出，保持运行状态，来处理一些触摸事件、定时器时间等。RunLoop可以使得线程在有任务的时候处理任务，没有任务的时候休眠，以此来节省CPU资源，提高程序性能。

那RunLoop是怎样保持程序的运行状态，到底处理了哪些事件？下面我们就从源码的层面来了解一下RunLoop。

<!-- more -->

#### RunLoop

##### 获取runloop对象

NSRunLoop和CFRunLoopRef都代表RunLoop对象，NSRunLoop是对CFRunLoopRef的封装。

**Foundation**
>[NSRunLoop currentRunLoop]; // 获得当前线程的RunLoop对象
[NSRunLoop mainRunLoop]; // 获得主线程的RunLoop对象

**Core Foundation**
>CFRunLoopGetCurrent(); // 获得当前线程的RunLoop对象
CFRunLoopGetMain(); // 获得主线程的RunLoop对象

##### RunLoop相关类

从源码的代码结构中我们可以找出来一下5个跟RunLoop相关的结构

```
CFRunLoopRef
CFRunLoopModeRef
CFRunLoopSourceRef
CFRunLoopObserverRef
CFRunLoopTimerRef
```

下面是CFRunLoopRef的结构代码

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

变量很多，我们不需要全部看，只需要注意这两个

```objc
CFRunLoopModeRef _currentMode;
CFMutableSetRef _modes;
```

每一个runloop里面有很多mode（存在一个set集合里面），然后之后后一个mode叫做currentMode，也就是说runloop一次只能处理一种mode。

然后我们再看`CFRunLoopModeRef`的结构，我已经给大家省略了里面那些我们不需要关注的变量

```objc
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFStringRef _name;
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
};
```
根据上面这些我们大概的可以概括出来RunLoop这些相关类的关系。

![runloop相关类的关系.png](https://raw.githubusercontent.com/Sunxb/blog_img/master/11/1.png)


##### CFRunLoopModeRef

由上面的源码我们可以稍微总结一下这个CFRunLoopModeRef：

1. CFRunLoopModeRef代表RunLoop的运行模式
2. 一个RunLoop包含多个CFRunLoopModeRef，每个CFRunLoopModeRef又包含多个_sources0，_sources1，_observers，_timers。
3. RunLoop每次只能运行一种mode，切换mode的时候，要先退出之前的mode。
4. 如果mode中没有_sources0、_sources1、_observers、_timers，程序会立刻退出。


常用的两种Mode

kCFRunLoopDefaultMode（NSDefaultRunLoopMode）：App的默认Mode，通常主线程是在这个Mode下运行

UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。


##### CFRunLoopObserverRef

源码中给出了可以监听的RunLoop状态

```objc
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {  
    // 进入RunLoop
    kCFRunLoopEntry = (1UL << 0),
    // 即将处理timers
    kCFRunLoopBeforeTimers = (1UL << 1),
    // 即将处理Sources
    kCFRunLoopBeforeSources = (1UL << 2),
    // 即将休眠
    kCFRunLoopBeforeWaiting = (1UL << 5),
    // 被唤醒
    kCFRunLoopAfterWaiting = (1UL << 6),
    // 退出循环
    kCFRunLoopExit = (1UL << 7),
    // 所有状态
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

具体的怎么样添加observer来监听RunLoop状态我就不贴代码了，网上一搜有很多的。

#### RunLoop的运行逻辑

前面我们已经了解了RunLoop相关的结构的源码，知道了RunLoop大概的数据结构，那RunLoop到底是如何工作的呢？它的运行逻辑是什么?

我们了解过了每个mode中会存放不同的_sources0、_sources1、_observers、_timers，这些我们可以全部统称是RunLoop要处理的东西，那每一种具体对应我们了解的哪写事件呢？

**Source0**
触摸事件处理
performSelector:onThread:

**Source1**
基于系统Port(端口)的线程间通信
系统事件捕捉

**Timers**
NSTimer定时器
performSelector:withObject:afterDelay:

**Observers**
用于监听RunLoop的状态
UI刷新（BeforeWating）
Autorelease Pool （BeforWaiting）

注： UI的刷新并不是即时生效，比如说我们改变了view的backgroundColor，当执行到这行代码是并不是立刻生效，而是先记录下有这么一个任务，然后在RunLoop处理完所有的时间，进入休眠之前UI刷新。

![runloop具体流程.png](https://raw.githubusercontent.com/Sunxb/blog_img/master/11/2.png)


这是大神总结的RunLoop的运行逻辑图，我直接拿过来用了。我们主要是看左边这部分，右边的这些标注是在源码中对应的主要方法名称。

这个图很容易理解，只有从06跳转到08这一步，单从图上看的话不是很清晰，这一块结合源码就比较明了了。第06步，如果存在Source1就直接跳转到08，在代码中使用了goto这个关键字，其实就是跳过了runloop休眠和唤醒这一部分的代码，直接跳转到了处理各种事件的这一部分。

下面我把源码做了一些删减，方便大家可以更清楚的梳理整个过程

```c
// 这个是runloop入口函数
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    
    // 通知Observers 即将进入RunLoop
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // 核心方法
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 通知Observers 即将退出RunLoop
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    return result;
}
```

下面是核心方法

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {

    int32_t retVal = 0;
    do {
        
        //通知Observers 即将处理Timers
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        //通知Observers 即将处理Sources
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        //处理Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        //处理source0,根据返回值决定在处理一次blocks
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        
        // source1相关
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            msg = (mach_msg_header_t *)msg_buffer;
            // 是否有Source1  有的话跳转到handle_msg
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
        }

        didDispatchPortLastTime = false;

        // 通知Observers: 即将休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        //休眠
        __CFRunLoopSetSleeping(rl);
	

        //等待别的消息来唤醒当前线程
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);

        
        __CFRunLoopUnsetSleeping(rl);
        
        // 通知Observers: 即将醒来
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

    // 标识标识 !!!!!
    handle_msg:;
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        //下面根据是什么唤醒的runloop来分别处理
        
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
        
        // 被Timer唤醒（一个timer周期）
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // 处理Timers
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
        // 被Timer唤醒（一个timer周期）
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // 处理Timers
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
        // 被GCD唤醒
        else if (livePort == dispatchPort) {

            // 处理GCD相关
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);

        } else {
            //被Source1唤醒
            //处理Source1
            sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;

	    }


        //在处理一遍BLocks
        __CFRunLoopDoBlocks(rl, rlm);
        
        
        // 设置返回值 决定是否继续循环
        if (sourceHandledThisLoop && stopAfterHandle) {
            retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
        } else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            retVal = kCFRunLoopRunFinished;
        }
        
    } while (0 == retVal);

    return retVal;
}
```

图和源码结合来看，整个流程就清晰了很多。流程里面的有些东西不需要我们太过深入的研究，我们把这个流程掌握一下就OK了。

#### 细节补充

##### 第一点

我们都知道RunLoop有一个优势，那就是可以使线程在有工作的时候工作，没有工作的时候休眠，来减少占用CPU资源，提高程序性能。

这说明代码在执行到

```c
//等待别的消息来唤醒当前线程
__CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
```
的时候，会阻塞当前的线程。但这种阻塞跟我们之前所用到过的阻塞线程不是一回事。

举个例子，我们可以使用`while(1){};`这句代码来阻塞线程，这句代码在底层会转换为汇编的代码，我们的线程一直在重读执行这几句代码，所以他仅仅是阻塞线程，并没有使线程休眠，我们的线程一直在工作。但是runloop，通过`mach_msg`使用了一些内核层的API，真的是实现了线程的休眠，让线程不再占用CPU资源。

##### 第二点
RunLoop与线程的关系？

1. 一个线程对应一个RunLoop对象。
2. RunLoop默认不创建，在第一次获取的时候创建，主线程中的默认存在RunLoop也是因为在底层代码中，提前获取过一次。
3. RunLoop储存在一个全局的字典中，线程是key，RunLoop是value。（源码中有所体现）
4. RunLoop会在线程结束时销毁。




