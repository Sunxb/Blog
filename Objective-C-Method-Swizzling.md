---
title: Objective-C Method Swizzling
date: 2019-05-15 18:43:36
tags: [iOS,Runtime]
---

Method Swizzling已经被聊烂了，都知道这是Objective-C的黑魔法，可以交换两个方法的实现。今天我也来聊一下Method Swizzling。

#### 使用方法

我们先贴上这一大把代码吧

```objc
@interface UIViewController (Swizzling)

@end

@implementation UIViewController (Swizzling)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(swizzling_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        BOOL success = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (success) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)swizzling_viewWillAppear:(BOOL)animated {
    [self swizzling_viewWillAppear:animated];
    NSLog(@"==%@",NSStringFromClass([self class]));
}

@end
```

好的，上面就是Method Swizzling的使用方法，将方法`- (void)swizzling_viewWillAppear:(BOOL)animated`和系统级方法`- (void)viewWillAppear:(BOOL)animated`交换。常用的场景就是埋点，这个咱就不细说了。

这里我们说一个点就是实现代码的位置问题。

我们的交换代码必须只能调用一次，如果执行多次的，那不就把交换的实现又换回来了吗，所以我们必须找一个只会调用一次的地方来写实现交换的代码。
    
我们都知道`+load`会在加载类的时候执行，而且只执行一次，但是为了进一步保障他只能执行一次，我们使用了dispatch_once（因为会有人手动调用+load方法）。
    
此外`+load`方法还有一个非常重要的特性，那就是子类、父类和分类中的`+load`方法的实现是被区别对待的。换句话说在 Objective-C runtime 自动调用`+load`方法时，分类中的`+load`方法并不会对主类中的`+load`方法造成覆盖。

#### Method Swizzling 实现分析

在取到了SEL和Method之后，执行了下面这句代码

```objc
 BOOL success = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
```

然后根据success来做不同的处理。

这里我们先说**结论**，就拿我们上面的代码作为例子。

如果我们的主类，也就是UIViewController（我们是给UIViewController创建的Category）实现了`viewWillAppear:`方法，success为NO，如果没有实现，则为YES。在我们的例子中的UIViewController肯定是实现了`viewWillAppear:`方法的，所以success肯定为NO。

如果我们这里给一个自定义的VC创建Category实现Swizzling，并且VC没有显式的实现`viewWillAppear:`（继承父类的），这时success就是YES了。

大家可以自己创建不同类和类别的试一试。

##### class_addMethod

我们通过runtime的源码来看一下`class_addMethod`内部做了什么操作。

```objc
BOOL 
class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    mutex_locker_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}
```

`class_addMethod`返回值是对`addMethod`返回值取反。这个地方稍微有点别扭。我们可以看一下`addMethod`方法，返回值是IMP，所以说：

**主类实现了被Swizzling的方法，success为NO，即class_addMethod返回NO，addMethod返回值不为空。**

**主类未实现了被Swizzling的方法，success为YES，即class_addMethod返回YES，addMethod返回值为空。**


```objc
// addMethod方法的主要代码

method_t *m;

if ((m = getMethodNoSuper_nolock(cls, name))) {
    // already exists
    if (!replace) {//replace==NO (class_addMethod)
        result = m->imp;
    } else {//(class_replaceMethod)
        result = _method_setImplementation(cls, m, imp);
    }
}
    
else {
    // fixme optimize
    method_list_t *newlist;
    newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
    newlist->entsizeAndFlags = 
        (uint32_t)sizeof(method_t) | fixed_up_method_list;
    newlist->count = 1;
    newlist->first.name = name;
    newlist->first.types = strdupIfMutable(types);
    newlist->first.imp = imp;

    prepareMethodLists(cls, &newlist, 1, NO, NO);
    cls->data()->methods.attachLists(&newlist, 1);
    flushCaches(cls);

    result = nil;
}


```

我们结合源码来梳理上面提到的两种情况。

首先method_t其实就是一个储存方法的细节的结构体。
通过`m = getMethodNoSuper_nolock(cls, name)`查找类中对应的方法的信息，包括方法名，实现等等。

1. 如果主类中实现了被swizzling的方法

    调用`m = getMethodNoSuper_nolock(cls, name)`查找，这里应该是可以找到的，`class_addMethod`方法中调用`addMethod`的时候relpace传的是NO，所以会执行`result = m->imp;`，其实就是给result赋了个值，让他不为空。`class_addMethod`的返回值是`addMethod`返回值取反，所以此时`class_addMethod`返回为NO。
    
    继续往下执行`method_exchangeImplementations(originalMethod, swizzledMethod);`，这就很好理解了，就是直接把需要交换的两个方法的实现直接交换。
    
    
2. 如果主类中没有实现被swizzling的方法

    `getMethodNoSuper_nolock`找不到，m还是为空，所以会执行else下面的代码，这里面的代码其实很明显，就是动态的为我们的主类创建实现了需要被swizzling的方法，当然了，因为此时我们传入`class_addMethod`方法的sel是需要被swizzling的方法，但是实现已经是传了需要替换后的实现，所以执行完else里面的代码之后，我们的需要被swizzling的方法的实现，已经指向了替换后的实现，也就是`viewWillAppear:`的IMP其实此时已经指向了`swizzling_viewWillAppear:`的IMP。
    
    最后result置为nil，所以`class_addMethod`返回值就是YES，success为YES。
    
    最后再执行`class_replaceMethod`方法。
    
    从源码中来看，`class_replaceMethod`和`class_addMethod`都是调用了`addMethod`，但有两点不同，一来是`class_replaceMethod`在调用`addMethod`时，参数replace传YES，再者就是因为`class_replaceMethod`的返回值是一个IMP，所以和`addMethod`是一致的，直接return了`addMethod`的返回值。
    
    在通过`class_replaceMethod`调用`addMethod`的时候，虽然我们主类之前没实现要被swizzling的方法，但是在上一步中，我们已经动态的添加了，所以此时`getMethodNoSuper_nolock`是能找到的。
    
    最终执行`result = _method_setImplementation(cls, m, imp);`。`_method_setImplementation`内部实现很简单，先用一个old记录下m->imp  然后再把m->imp设置为传入的imp，随后返回old，其实也就是返回了没交换前的m->imp。
    
    这样我们就通过`_method_setImplementation`方法把我们的`swizzling_viewWillAppear:`的IMP指向了`viewWillAppear:`的IMP。完成了`viewWillAppear:`和`swizzling_viewWillAppear:`两个方法实现的交换。

-----

#### 参考资料

http://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/

