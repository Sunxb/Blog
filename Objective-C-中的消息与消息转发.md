---
title: Objective-C 中的消息与消息转发
date: 2019-02-28 09:36:37
tags: [iOS,Runtime]
---

大家都知道OC是一门动态语言，其动态性由底层的runtime库来支撑实现。OC所有的方法都是通过runtime来发送消息，当我们探讨消息发送，其实也就是在探讨OC方法的调用过程。

<!-- more -->

```objc
[receiver message];
```

这是我们很熟悉的一个OC方法的调用，大家都知道这个方法最终会被编译器转换为消息发送函数

```objc
objc_msgSend(receiver, @selector(message));
```

首先声明咱们这篇文章不去讲解具体的class的数据结构一类的细节问题，我们主要关注的是这个过程。

很遗憾，`objc_msgSend`的实现是用汇编写的，我并不能看懂。但是从runtime的源码中我发现了一个关键的方法：

```c
//查找方法的实现
extern IMP lookUpImpOrForward(Class, SEL, id obj, bool initialize, bool cache, bool resolver);
```

我们要执行一个方法，其实最重要的就是找到这个方法的实现。下面我们来看一下这个`lookUpImpOrForward`函数的源码。

#### lookUpImpOrForward

通过源码中的注释可以看出来，主要的流程分为以下几个阶段：

##### 无锁状态下从缓存中查找方法的实现

```c
runtimeLock.assertUnlocked();
// Optimistic cache lookup
if (cache) {
    imp = cache_getImp(cls, sel);
    if (imp) return imp;
}
```

##### 类的实现和初始化

```c
// 判断类是否已经实现
if (!cls->isRealized()) {
    realizeClass(cls);
}
    
// 是否初始化
if (initialize  &&  !cls->isInitialized()) {
    runtimeLock.unlock();
    _class_initialize (_class_getNonMetaClass(cls, inst));
    runtimeLock.lock();

```


在执行这两段代码之前还有两行代码，一个是`runtimeLock.lock();`，注释里面解释是通过加锁在遇到并发实现类的时候保护实现的过程。

另一个是`checkIsKnownClass(cls);`。看名字是检测这个类是不是我们已知的类，什么情况下可能出现我们未知的类呢？我猜测是通过NSClassFromString()来获取类的时候，如果class的string写错了，就会生成一个未知的类，这时候肯定找不到方法的实现，直接就crash掉行了。
    
我们的Class在底层其实是一个名为objc_class的结构体，大家可以去源码中看一下，这里我们不细说。执行`realizeClass`方法其实等于对类的第一次初始化，包括配置类的读写空间（class_rw_t）并且返回类的正确的结构体，就相当于搭好了这个类的框架。`_class_initialize`则是让类执行我们熟悉的`+initialize`方法。

##### 加锁，查找

###### 加锁

下面就要开始真正的IMP查找了，查找之前有一个`runtimeLock.assertLocked();`加锁。

注释中给出了解释，runtimeLock在查找方法的时候加锁，是为了保持method-lookup（查找方法）和 cache-fill（缓存填充）这两种方法的原子性，

```
// Otherwise, a category could be added but ignored indefinitely because
// the cache was re-filled with the old value after the cache flush on
// behalf of the category.
```

这几行注释我没读明白啥意思，感觉是说如果不加锁，有添加category的情况时，会导致缓存被冲洗掉。

###### 在类中查找

通过代码可以看出，首先依然是从缓存中查找，然后在当前的类中查找，最后通过一个循环来从各级父类中查找。

这部分代码我们就不展示出来了，大家自己可以看源码，如果找到了这个方法的实现，我们看到会调用`log_and_fill_cache`这样一个方法，其实就是把这次查找缓存起来，方便下次使用。

如果没有找到就进入到下一个环节。

##### 动态决议
这个名称大家应该是耳熟的，我们都知道消息转发机制，也就是说在我们调用了一个未实现的方法时，并不会直接crash掉，然后报`unrecognized selector sent to instance`错误。我们的系统会给我们两次机会，第一次就是动态决议。

```c
if (resolver  &&  !triedResolver) {
    runtimeLock.unlock();
    _class_resolveMethod(cls, sel, inst);
    runtimeLock.lock();
    // Don't cache the result; we don't hold the lock so it may have 
    // changed already. Re-do the search from scratch instead.
    triedResolver = YES;
    goto retry;
}
```

在OC层面最直接的表现形式就是看我们是否实现了`+ (BOOL)resolveClassMethod:(SEL)sel`或者`+ (BOOL)resolveInstanceMethod:(SEL)sel`方法（对应类方法和实例方法）。

```c
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    if (! cls->isMetaClass()) {//不是元类
        // try [cls resolveInstanceMethod:sel]
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
```

根据cls是否为元类来调用`_class_resolveInstanceMethod`或`_class_resolveClassMethod`。

```c
/***********************************************************************
* lookUpImpOrNil.
* Like lookUpImpOrForward, but returns nil instead of _objc_msgForward_impcache
**********************************************************************/
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    // 这个imp或者是一个正经的IMP指针,或者是一个汇编的入口
    //_objc_msgForward_impcache 是一个汇编的入口  (看名字应该是消息转发相关的)
    // 本方法是不进行消息转发的 ~  所以如果获取到的IMP是这个入口,就直接return nil
    
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
```

通过`lookUpImpOrNil`的源码我们知道，其实它封装了`lookUpImpOrForward`，不进行`lookUpImpOrForward`中最后的消息转发这一步，而且在这个if中，参数resolver传入了NO，也就是动态决议这一步也不进行，只进行了方法在本类和各级父类中的查找，如果找不到，则跟非元类一样执行`_class_resolveInstanceMethod`。

**else中为什么要再执行这一段if代码，我不是很理解。**

具体的动态决议实现的代码，我们看其中一个就行来看，用`_class_resolveInstanceMethod`当例子吧

```c
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        // 没有找到SEL_resolveInstanceMethod(resolveInstanceMethod)方法
        return;
    }
    // 下面应该是类型转换的代码而已吧？
    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    // 执行实现的+resolveInstanceMethod方法
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    //通过+resolveInstanceMethod动态添加了方法，在进行一次查找
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
```

一开始先通过调用`lookUpImpOrNil`来查找是否已经实现了`+resolveInstanceMethod`，没有实现就直接返回了。

如果实现了，通过`objc_msgSend`来执行实现的`+resolveInstanceMethod`方法。

我们在`+resolveInstanceMethod`方法中都是动态的添加方法，所以在执行完之后在进行一次查找。

##### 消息转发
如果在动态决议之后，依然没有找到方法，那我们还有最后一次机会，那就是消息转发。

```c
imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);
```

很遗憾`_objc_msgForward_impcache`汇编实现看不懂。不过我们还是可以从另一个方向来研究消息转发到底做了什么。

先补充一个骚操作，这也是我这两天刚学到的，就是打印runtime的代码执行日志，这么说可能不太贴切，反正就是能看到我们执行一个方法的过程中调用了哪些方法。

>想要开始打印的地方加上下面代码
extern void instrumentObjcMessageSends(BOOL);
instrumentObjcMessageSends(YES);

>想要关闭的地方加上下面代码
extern void instrumentObjcMessageSends(BOOL);
instrumentObjcMessageSends(NO);

当然还有别的方式，大家可以去查一下。

首先我们新建一个工程，类型就选择macOS->Common Line Tool。
因为我用iOS的工程不管用。

main函数中这样写

```c
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Sark.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        extern void instrumentObjcMessageSends(BOOL);
        instrumentObjcMessageSends(YES);
        
        Sark * test = [[Sark alloc] init];
        [test performSelector:@selector(xxx)];
        
        instrumentObjcMessageSends(NO);
    }
    return 0;
}
```

创建SEL选择子的时候我们故意穿进去一个xxx，就是为了让类找不到方法，然后走消息转发的的这个过程。

执行一下代码，运行时发送的所有消息都会打印到/private/tmp/msgSend-xxxx文件里了。(这是电脑系统的路径)

如果找起来不方便可以直接使用下面的命令行打开该文件。

`open /private/tmp/`

从该路径的msgSend-xxx文件中我们找到了这么一部分代码

```objc
+ Sark NSObject initialize
+ Sark NSObject alloc
- Sark NSObject init
- Sark NSObject performSelector:
+ Sark NSObject resolveInstanceMethod:
+ Sark NSObject resolveInstanceMethod:
- Sark NSObject forwardingTargetForSelector:
- Sark NSObject forwardingTargetForSelector:
- Sark NSObject methodSignatureForSelector:
- Sark NSObject methodSignatureForSelector:
- Sark NSObject class
- Sark NSObject doesNotRecognizeSelector:
- Sark NSObject doesNotRecognizeSelector:
- Sark NSObject class
```

我们可以看到在执行完了`resolveInstanceMethod`之后又执行`forwardingTargetForSelector:`和`methodSignatureForSelector:`，最后才因为找不到方法执行`doesNotRecognizeSelector:`。

现在我们可以大概了解在消息转发的过程中执行了哪些方法了，经过查阅资料我们得出：

全部的消息转发过程做了如下几件事：（包括动态决议+消息转发）

1. 调用resolveInstanceMethod:方法，允许用户在此时为该Class动态添加实现。如果有实现了，则调用并返回。如果仍没实现，继续下面的动作。

2. 调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接转发给它。如果返回了nil，继续下面的动作。

3. 调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。

4. 调用forwardInvocation:方法，将地3步获取到的方法签名包装成Invocation传入，如何处理就在这里面了。

上面这4个方法均是模板方法，开发者可以override，由runtime来调用。

具体如何重写使用这几个方法，大家可以自己查一下。

-----

##### 参考资料
https://blog.ibireme.com/2013/11/26/objective-c-messaging/

https://github.com/draveness/analyze/blob/master/contents/objc/%E4%BB%8E%E6%BA%90%E4%BB%A3%E7%A0%81%E7%9C%8B%20ObjC%20%E4%B8%AD%E6%B6%88%E6%81%AF%E7%9A%84%E5%8F%91%E9%80%81.md



