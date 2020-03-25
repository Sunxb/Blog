---
title: iOS:内存管理知识点
date: 2017-09-11 11:17:32
tags: [iOS,内存管理]
---
### PART1:  ARC的修饰符

ARC主要提供了4种修饰符，他们分别是:
`__strong` `__weak` `__autoreleasing` ` __unsafe_unretained`

#### __strong
表示引用为强引用。对应在定义property时的"strong"。所有对象只有当没有任何一个强引用指向时，才会被释放。
注意：如果在声明引用时不加修饰符，那么引用将默认是强引用。当需要释放强引用指向的对象时，需要将强引用置nil。

#### __weak
表示引用为弱引用。对应在定义property时用的"weak"。弱引用不会影响对象的释放，即只要对象没有任何强引用指向(弱引用不会持有对象)，即使有100个弱引用对象指向也没用，该对象依然会被释放。不过好在，对象在被释放的同时，指向它的弱引用会自动被置nil，这个技术叫zeroing weak pointer。这样有效得防止无效指针、野指针的产生。__weak一般用在delegate关系中防止循环引用或者用来修饰指向由Interface Builder编辑与生成的UI控件。

#### __autoreleasing
表示在autorelease pool中自动释放对象的引用，和MRC时代调用autorelease的用法相同。定义property时不能使用这个修饰符，任何一个对象的property都不应该是autorelease型的。

一个常见的误解是，在ARC中没有autorelease，因为这样一个“自动释放”看起来好像有点多余。这个误解可能源自于将ARC的“自动”和autorelease“自动”的混淆。其实你只要看一下每个iOS App的main.m文件就能知道，autorelease不仅好好的存在着，并且变得更fashion了：不需要再手工被创建，也不需要再显式得调用[drain]方法释放内存池。

以下两行代码的意义是相同的。

<!-- more -->

    NSString *str = [[[NSString alloc] initWithFormat:@"hehe"] autorelease]; // MRC
    NSString *__autoreleasing str = [[NSString alloc] initWithFormat:@"hehe"]; // ARC

__autoreleasing在ARC中主要用在参数传递返回值（out-parameters）和引用传递参数（pass-by-reference）的情况下。
__autoreleasing

 > is used to denote arguments that are passed by reference (id *
) and are autoreleased on return.

比如常用的NSError的使用：
    
    NSError *__autoreleasing error; 
    if (![data writeToFile:filename options:NSDataWritingAtomic error:&error]) { 
    　　NSLog(@"Error: %@", error); 
    }

在上面的writeToFile方法中error参数的类型为(NSError *__autoreleasing *)

**注意**，如果你的error定义为了strong型，那么，编译器会帮你隐式地做如下事情，保证最终传入函数的参数依然是个__autoreleasing类型的引用。

    NSError *error; 
    NSError *__autoreleasing tempError = error; // 编译器添加 
    if (![data writeToFile:filename options:NSDataWritingAtomic error:&tempError]) { 
    　　error = tempError; // 编译器添加 
    　　NSLog(@"Error: %@", error); 
    }


所以为了提高效率，避免这种情况，我们一般在定义error的时候将其声明为__autoreleasing类型的：
    
    NSError *__autoreleasing error;

在这里，加上__autoreleasing之后，相当于在MRC中对返回值error做了如下事情：
    
    *error = [[[NSError alloc] init] autorelease];

\*error指向的对象在创建出来后，被放入到了autoreleasing pool中，等待使用结束后的自动释放，函数外error的使用者并不需要关心\*error指向对象的释放。

**另外一点，在ARC中，所有这种指针的指针 （NSError \*\*）的函数参数如果不加修饰符，编译器会默认将他们认定为__autoreleasing类型.**


#### __unsafe_unretained
忽略~ 

#### 补充一点
苹果的文档中明确地写道：

>You should decorate variables correctly. When using qualifiers in an object variable declaration,
the correct format is :
ClassName * qualifier variableName;

按照这个说明，要定义一个weak型的NSString引用，它的写法应该是：

    NSString * __weak str = @"xx";

而不应该是：

    __weak NSString *str = @"xx";  




-----

### PART2: __weak使用注意

访问附有__weak修饰符的变量时，该变量会被自动注册到autoreleasepool中,  这种等价是编译器自动做的转换，原因是__weak修饰符支持有对象的弱引用，在访问引用对象的过程中，该对象可能被释放。而如果将该对象加入到autoreleasepool中，在pool被释放之前, 对该对象的引用都是有效的。

但是大量的使用__weak修饰符的变量, 注册到autoreleaspool的对象也会大大增加, 因此在使用__weak修饰符的变量时, 最好先暂时赋值给__strong修饰符的变量后在使用

![示例图](http://upload-images.jianshu.io/upload_images/1491333-268c8b0ce2e4ae1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

-----
### PART3: NSArray array/new/@[]/alloc-init的区别

所有这些方法的结果其实都是一样的，你得到一个新的空的不可变数组. 但不同的方法有不同的内存管理方式. 现在我们都是使用ARC, 所以说最终是没有区别的, 但在ARC之前我们要正确适时的调用`retain`, `release`, `autorelease`方法.

**[NSArray new]** 和 **[[NSArray alloc] init]**返回一个数组,并且引用计数+1. 在ARC之前, 你需要自己`release`或者`autorelease`数组, 否则会导致内存泄漏.

**[NSArray array]** 和 **@[]** 返回了一个已经调用了autorelease的数组, 引用计数为0(不持有), 你必须手动调用`retain`方法来保留它，否则将在当前自动释放池销毁时被释放。
**[NSMutableArray array]**相当于**[[[NSMutableArray alloc] init] autorelease]**

----



http://www.cnblogs.com/flyFreeZn/p/4264220.html
https://stackoverflow.com/questions/33297171/difference-between-nsarray-array-new-alloc-init

