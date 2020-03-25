---
title: 深入学习Runtime
date: 2019-02-22 18:06:41
tags: [iOS,Runtime]
---


本文的切入点是2014年的一场线下分享会，也就是sunnyxx分享的objc runtime。很惭愧，这么多年了才完整的看了一下这个分享会视频。当时他出了一份试题，并戏称精神病院objc runtime入院考试。

我们今天的这篇文章就是从这个试题中的题目入手，来深入的学习runtime。

<!-- more -->

**源码版本objc4-750**

#### 第一题

```objc
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

第一行的`[self class]`应该是没有疑问的，肯定是`Son`，问题就出在这个`[super class]`。

大家都知道，我们OC的方法在底层会编译为一个`objc_msgSend`的方法（消息发送），`[self class]`符合这个情况，因为self是类的一个隐藏参数。但是`super`并不是一个参数，它是一个关键字，实际上是一个“编译器标示符”，所以这就有点不一样了，经查阅资料，在调用`[super class]`的时候，runtime调用的是`objc_msgSendSuper`方法，而不是`objc_msgSend`。

首先要做的是验证一下是否是调用了`objc_msgSendSuper`。这里用到了clang这个工具，我们可以把OC的代码转成C/C++。

```objc
@implementation Son
- (void)test {
    [super class];
}
@end

```
在终端运行`clang -rewrite-objc Son.m`生成一个Son.cpp文件。

在这个.cpp文件的底部我们可以找到这么一部分代码

```objc
// @implementation Son

static void _I_Son_test(Son * self, SEL _cmd) {
    ((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Son"))}, sel_registerName("class"));
}
// @end
```

看起来乱七八糟，有很多强制类型转换的代码，不用理它，我们只要看到了我们想要的`objc_msgSendSuper`就好。

去源码中看一下这个方法（具体实现好像是汇编，看不懂）

```objc
OBJC_EXPORT void
objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
```

可以看出来这个方法第一个参数是一个`objc_super`类型的结构体，第二个是一个我们常见的SEL，后面的...代表还有扩展参数。

再看一下这个`objc_super`结构体。

```objc
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header  为了兼容老的 */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
```

第一个参数是接收消息的receiver，第二个是super_class（见名知意~ 😆）。我们和上面提到的.cpp中的代码对应一下就会发现重点了，**receiver是self**。

所以，这个`[super class]`的工作原理是，从`objc_super`结构体的`super_class`指向类的方法列表开始查找`class`方法，找到这个方法之后使用`receiver`来调用。

所以，调用`class`方法的其实还是`self`，结果也就是打印`Son`。

----

#### 第二题

下面代码的结果？

```objc
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
```

对于这个问题我们就要从OC类的结构开始说起了。

我们都应该有所了解，每一个Objective-c的对象底层都是一个C语言的结构体，在之前老的源码中体现出，所有对象都包含一个`isa`类型的指针，在新的源码中已经不是这样了，用一个结构体`isa_t`代替了`isa`。这个`isa_t`结构体包含了当前对象指向的类的信息。

我们来看看当前的类的结构，首先从我们的祖宗类NSObject开始吧。

```objc
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

我们的NSObject类有一个Class类型的变量isa，通过源码我们可以了解到这个Class到底是什么

```objc
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

```

上面的代码是我从源码中复制拼到一起来的。可以看出来，Class就是是一个objc_class结构体，objc_class中有四个成员变量`Class superclass`，`cache_t cache`，`class_data_bits_t bits`，和从`objc_object`中继承过来的`isa_t isa`。

当Objc为一个对象分配内存，初始化实例变量后，在这些实例变量的结构体中第一个就是isa。

![1](https://raw.githubusercontent.com/Sunxb/blog_img/master/10/objc-isa-class-object.png)

而且从上面的objc_class的结构可以看出来，不仅仅是实例会包含一个isa结构体，所有的类也会有这个isa。

所以说，我们可以得出这样一个结论：Objective-c中的类也是一个对象。

那现在就有了一个新的问题，类的isa结构体中储存的是什么？这里就要引入一个`元类`的概念。

**知识补充：**
在Objective-c中，每个对象能执行的方法并没有存在这个对象中，因为如果每一个对象都单独储存可执行的方法，那对内存来说是一个很大的浪费，所以说每个对象可执行的方法，也就是我们说的一个类的实例方法，都储存在这个类的`objc_class`结构体中的`class_data_bits_t`结构体里面。在执行方法是，对象通过自己的isa找到对应的类，然后在`class_data_bits_t`中查找方法实现。

关于方法的结构，可以看这篇博客来理解一些。（[跳转链接](https://github.com/draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md)）

引入元类就是来保证了实例方法和类方法查找调用机制的一致性。

所以让一个类的isa指向他的元类，这样的话，对象调用实例方法可以通过isa找到对应的类，然后查找方法的实现并调用，在调用类方法的时候，通过类的isa找到对应的元类，在元类里完成类方法的查找和调用。

下面这种图也是在网上很常见的了，不需要过多解释，大家看一下记住就行了。

![2](https://raw.githubusercontent.com/Sunxb/blog_img/master/10/objc-isa-class-diagram.png)

看到这里我们就要回到我们的题目上了。首先呢，还是要去看一下这个源码中`isKindOfClass:`和`isMemberOfClass:`的实现了。

##### isKindOfClass

先看`isKindOfClass`吧，源码中提供了一个类方法一个实例方法。

```objc
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

总体的逻辑都是一样的，都是先声明一个Class类型的tcls，然后把这个tcls跟cls比较，看是否相等，如果不相等则循环tcls的各级superclass来进行比较，直到为tcls为nil停止循环。

不同的地方就是类方法初始的tcls是`object_getClass((id)self)`，实例方法的是`[self class]`。

`object_getClass((id)self)`其实是返回了这个self的isa对应的结构，因为这个方法是在类方法中调用的，self则代表这个类，那`object_getClass((id)self)`返回的也应该是这个类的元类了。

其实在`-isKindOfClass`这个实例方法中，调用方法的是一个对象，tcls初始等于`[self class]`，也就是对相对应的类。我们可以看出来，在实例方法中这个tcls初始的值也是方法调用者的isa对应的结构，跟类方法中逻辑是一致的。

回到我们的题目中，

```objc
BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
```

`[NSObject class]`也就是NSObject类调用这个`isKindOfClass:`方法（类方法），方法的参数也是NSObject的类。

在第一次循环中，tcls对应的应该是NSObject的isa指向的，也就是NSObject的元类，它跟NSObject类不相等。第二次循环，tcls取自己的superclass继续比较，我们上面的那个图，大家可以看一下，NSObject的元类的父类就是NSObject这个类本身，在与NSObject比较结果是相等。所以res1为YES。


```objc
BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
```

跟上面一样来分析，在第一次循环中，tcls对应的应该是Sark的isa指向的，也就是Sark的元类，跟Sark的类相比，肯定是不相等。第二次循环，tcls取superclass，从图中可以看出，Sark元类的父类是NSObject的元类，跟Sark的类相比，肯定也是不相等。第三次循环，NSObject元类的父类是NSObject类，也不相等。再取superclass，NSObject的superclass为nil，循环结束，返回NO，所以res3是NO。

##### isMemberOfClass

```objc
+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

有了上面isKindOfClass逻辑分析的基础，isMemberOfClass的逻辑我们应该很清楚，就是使用方法调用者的isa对应的结构和传入的cls参数比较。

```objc
BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
```

NSObject类的isa对应的是NSObject的元类，和NSObject类相比不相等，所以res2为NO。

Sark类的isa对应的是Sark的元类，和Sark类相比也是不相等，所以，res4也是NO。


----

#### 第三题
下面的代码会？Compile Error / Runtime Crash / NSLog…?

```objc
@interface NSObject (Sark)
+ (void)foo;
@end
@implementation NSObject (Sark)
- (void)foo {
    NSLog(@"IMP: -[NSObject (Sark) foo]");
}
@end
// 测试代码
[NSObject foo];
[[NSObject new] foo];
```

`[[NSObject new] foo];`这一个代码应该是毫无疑问会调用到`-foo`方法。问题就在这个`[NSObject foo]`，因为在我们的认识中`[NSObject foo]`是调用的类方法，实现的是实例方法，应该不能调用到。

其实这个题的考点跟第二个题差不多，我们已经知道了，一个类的实例方法相关信息储存在类中，类方法的储存在这个类的元类。所以NSObject在调用foo这个方法时，会先去NSObject的元类中找这个方法的IMP，没有找到，那就要去父类中继续查找。上面图已经给出了，NSObject的元类的父类是NSObject类，所以在NSObject中查找方法，可以找到找到方法的IMP，然后执行打印。

-----


#### 第四题
下面的代码会？Compile Error / Runtime Crash / NSLog…?

```objc
@interface Sark : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Sark
- (void)speak {
    NSLog(@"my name's %@", self.name);
}
@end
@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    id cls = [Sark class];
    void *obj = &cls;
    [(__bridge id)obj speak];
}
@end
```

这里我们先上结果：

```objc
my name's <ViewController: 0x7f9454c1c680>
```
不管地址是多少，打印的总是ViewController。

我们先想一下为什么可以成功的调用speak？
`id cls = [Sark class];`创建了一个Sark的class。`void *obj = &cls;`创建一个obj指针指向了cls的地址。最后使用`(__bridge id)obj`把这个obj指针转成一个oc的对象，用对象来调用speak，所以可以调用成功。

我们在方法中输出的是`self.name`，为什么会打印出来ViewController？

经过查阅资料得知，在调用self.name的时候，本质上是self指针在内存向高位地址偏移一个指针。（这个还得以后深入研究）

为了验证一下查到的这个结论，我改写了一下`speak`方法中的代码如下。

```objc
- (void)speak {
    unsigned int count = 0;
    Ivar * ivars = class_copyIvarList([self class], &count);
    for (int i = 0; i < count; i ++) {
        Ivar ivar = ivars[i];
        ptrdiff_t offSet = ivar_getOffset(ivar);
        const char * n = ivar_getName(ivar);
        NSLog(@"%@-----%ld",[NSString stringWithUTF8String:n],offSet);
    }
    
    
    NSLog(@"my name's %@", self.name);
}
```

取到类的各个变量，然后打印出他的偏移。输出结构如下：

```objc
_name-----8
```

偏移了一个指针。

那为什么打印出来了ViewController的地址，我们就要研究各个变量的内存地址位置关系了。

在`iewDidLoad`中变量的压栈顺序如下所示：

![](https://raw.githubusercontent.com/Sunxb/blog_img/master/10/1.png)

第一个参数self和第二个参数_cmd是隐藏参数，第三和第四个参数是执行`[super viewDidLoad]`之后进栈的，之前第一题的时候我们有了解过，super调用的方法在底层编译之后会有一个`objc_super`类型的结构体。在结构体中有receiver和super_class两个变量，receiver就是self。

我在网上查过很多的资料，为什么是super_class比receiver（self）先入栈。这里不太明白。

最后是生成的obj进栈。

所以在打印self.name的时候，是obj的指针向高位偏移了一个指针，也就是self，所以打印出来的是ViewController的指针。




#### 参考
https://github.com/draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md

http://blog.sunnyxx.com/2014/11/06/runtime-nuts/

https://www.jianshu.com/p/743b975b9fee

https://github.com/ming1016/study/wiki/Objc-Runtime#%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F%E4%B8%8E%E5%B1%9E%E6%80%A7






