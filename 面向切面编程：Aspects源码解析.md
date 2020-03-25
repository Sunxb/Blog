---
title: 面向切面编程：Aspects源码解析
date: 2019-05-23 14:19:09
tags: [iOS,Runtime,Aspects]
---

#### 面向切面编程

所谓的面向切面编程（AOP），原理就是在不更改正常业务的流程的前提下，通过一个动态代理类，实现对目标对象嵌入的附加的操作。

简单说，就是在不影响我们现在正常业务的情况下，对某些类的某些方法嵌入操作。我们可以很通俗的理解一个方法可以有方法前和方法后这两个切面，当然还可以把方法执行过程看过一个整的切面去hook。

在我们的iOS开发中，AOP的实现方法就是使用Runtime的Swizzling Method改变selector指向的实现，在新的实现中添加新的操作，执行完新实现之后，再处理之前的实现逻辑。

#### Aspects

Aspects是iOS平台比较成熟的AOP的框架，这次我们主要来研究一下这个库的源码。

**基于Aspects 1.4.1版本。**

##### 总览

Aspects给出了两个方法，一个类方法一个实例方法，使用起来非常简单。

```objc
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;


- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                           withOptions:(AspectOptions)options
                            usingBlock:(id)block
                                 error:(NSError **)error;
```

传参也很容易理解，selector自然就是我们要hook的方法，options使我们要hook的位置，下面具体再说，block是一个回调，也就是我们所说的要嵌入的代码逻辑，error就是hook失败，当然了失败的原因较多，我们下面会提到。

```objc
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// Called after the original implementation (default)
    AspectPositionInstead = 1,            /// Will replace the original implementation.
    AspectPositionBefore  = 2,            /// Called before the original implementation.
    
    AspectOptionAutomaticRemoval = 1 << 3 /// Will remove the hook after the first execution.
};
```

options也是一个枚举的类型，看一下里面定义的字段就很容易明白了，AspectPositionAfter是表示嵌入的放大要在被hook方法原来逻辑之前之后执行，AspectPositionBefore是之前执行，AspectPositionInstead表示要用嵌入的代码替换掉之前的逻辑，AspectOptionAutomaticRemoval表示hook执行后，移除hook。

因为是NS_OPTIONS类型，可多选。

##### 重要的类

除了上述核心的方法是通过NSObject的Category的方式给出，还有以下几个类比较重要。

`AspectsContainer`: 一个对象或者类的所有的Aspects整体情况

`AspectIdentifier`: 一个Aspects的具体内容，这里主要包含了单个的 aspect 的具体信息，包括执行时机，要执行 block 所需要用到的具体信息：包括方法签名、参数等等

`AspectInfo`: 一个 Aspect 执行环境，主要是 NSInvocation 信息。


##### 核心方法aspect_add

Aspects给出的两个方法最终都是调用了`aspect_add`。

```objc
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        // 是否允许hook -
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            //取到sel对应的container (一个类不同sel对应不同的container?)
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception拦截.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```


```objc
__block AspectIdentifier *identifier = nil;
```
每次在添加hook的时候，都会先创建一个AspectIdentifier。
__block是为了能在下面的block中修改identifier。


`aspect_performLocked`是封装了一个自旋锁。

在自旋锁中会有一个if语句来判断selector是否能被hook。那我们就先来看一下是否能被hook的判断方法`aspect_isSelectorAllowedAndTrack`

###### aspect_isSelectorAllowedAndTrack

1. 黑名单
    
    `aspect_isSelectorAllowedAndTrack`方法中维护了一个NSSet，在初始化的时候加入了一些方法名，在源码中是下面这些。
    
    ```objc
    [NSSet setWithObjects:@"retain", @"release", @"autorelease", @"forwardInvocation:", nil];
    ```
    
    这说明了后面这几个方法是不允许被hook的，如果hook了这些方法会有错误的信息提示。

2. dealloc方法

    在hook的方法中，dealloc属于一个特殊情况，因为这个方法是在对象要被销毁的时候创建，所以Aspects为了安全起见，在hook dealloc方法的时候。options只允许时AspectPositionBefore，也就是插入的逻辑只能在dealloc原有逻辑之前处理，不允许替换或者在dealloc之后。

3. 没找到方法

    这个就没啥疑问了，如果我们hook了一个根本不存在的方法也会有错误提示。

4. 方法只允许hook一次（元类相关）
    
    这个有点麻烦，因为从错误提示的枚举来看，他是对应`AspectErrorSelectorAlreadyHookedInClassHierarchy`这一项的，从字面意思来看是说方法已经被hook了。
    
    最开始我对这个类的层级不是很明白，我的最初理解的类的层级是父类和子类不能同时hook，经过验证这种理解是错误的。
    
    后来我仔细地看了一下源码，他的元类判断中是这么写的：
    
    ```objc
     if (class_isMetaClass(object_getClass(self))) {}
    ```
    
    判断的是object_getClass(self)，通过runtime的源码我们可以知道，object_getClass得到的是传参的isa指针指向的结构，意识就是self如果是对象，object_getClass(self)得到的是对应的类，如果self是类，那就得到了元类。

    那什么时候会提示这个错误呢，我举一个例子吧。我创建了一个`TestHookViewController`类，继承自`UIViewController`，如果我在`TestHookViewController`中像下面这样写就会有错误提示了。
    
    ```objc
    [[UIViewController class] aspect_hookSelector:@selector(viewWillAppear:) withOptions:0 usingBlock:^(id<AspectInfo> info, BOOL animated) {
        NSLog(@"%s",__func__);
    } error:NULL];
    
    [[TestHookViewController class] aspect_hookSelector:@selector(viewWillAppear:) withOptions:0 usingBlock:^(id<AspectInfo> info, BOOL animated) {
        NSLog(@"%s",__func__);
    } error:NULL];
    ```

    当然了，这两个hook的前后位置不同，打印台输出的提示也是不一样的，虽然都是一个类层级方法只允许hook一次的错误原因。这个大家自行尝试一下。
    
    if语句里面就是关于方法是否重复hook的判断逻辑，这里牵扯到一个相关类，AspectTracker。我们现在就来看一下这个类。
    
    
###### AspectTracker类

虽然是说AspectTracker类，但是代码结构还是接着上面的咱们说到的位置继续往下走。

因为AspectTracker主要就是用在方法只允许hook一次的判断中。

Aspects维护了一个字典，来储存被hook方法的类和对应的AspectTracker。

```objc
Class currentClass = [self class];
AspectTracker *tracker = swizzledClassesDict[currentClass];
```

通过上面的方式取到对应的AspectTracker。这里提一句，这里的代码我们应该先看下面的，要先了解AspectTracker是怎么存储到字典里面的。

这里我们要看下面这个do-while循环

```objc
currentClass = klass;
AspectTracker *subclassTracker = nil;
do {
    tracker = swizzledClassesDict[currentClass];
    if (!tracker) {
        tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass];
        swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
    }
    if (subclassTracker) {
        [tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];
    } else {
        [tracker.selectorNames addObject:selectorName];
    }

    // All superclasses get marked as having a subclass that is modified.
    subclassTracker = tracker;
}while ((currentClass = class_getSuperclass(currentClass)));
```

如果最开始没有tracker，会初始化一个，然后存到字典中。最开始的时候subclassTracker为nil，所以selector会add到tracker.selectorNames。

然后currentClass重新赋值

```objc
currentClass = class_getSuperclass(currentClass)
```

再次执行do逻辑里面的代码，这次subclassTracker就有值了（上一次循环的tracker），就会执行`[tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];`

我们进到`addSubclassTracker`源码中可以看到，tracker被存到`selectorNamesToSubclassTrackers`这个字典中，关键的是这个字典的key是selectorName，value是一个集合，tracker是放在这个集合里面的。为什么要通过集合来存tracker呢？

因为这里是有子类的情况的，一个类的子类可能有多个，如果在不同的子类中hook了这个父类的一个方法，也就是父类中的这一个selector被多次hook，所以也会有不同的tracker，所以使用一个集合来储存。

其实说到这个地方大家就差不多可以理解了，如果满足了
`if (class_isMetaClass(object_getClass(self)))`这个判断，我们会把这个类hook的方法通过封装为`AspectTracker`来进行记录，当然包括他的层层父类，都对对应一个`AspectTracker`，而且父类中的还会记录子类中hook的方法。这部分代码最好是debug跟一下，会明显一点。

上面我们先看了tracker是怎样被存起来的，接来下再来看关于只能被hook一次的判断。

首先要判断子类中是否hook

```objc
if ([tracker subclassHasHookedSelectorName:selectorName]) {
    //内部省略    
}
```

`subclassHasHookedSelectorName`内部实现很简单

```objc
- (BOOL)subclassHasHookedSelectorName:(NSString *)selectorName {
    return self.selectorNamesToSubclassTrackers[selectorName] != nil;
}
```

就是查询一下selectorNamesToSubclassTrackers这个字典中，通过seletorName是否能取到值，上面已经说过了，这个字典中通过key：selectorName value: set的方法储存了子类的tracker。

如果能取到值，就说明子类中已经hook了这个方法了，父类中就不能在hook。

如果没能取到值，说明当前类可能就是个子类，此时需要看一下他的父类中是否hook了这个selector，所以就会执行下面的的do-while循环。 此处的代码就不展示了。

但这个位置初步的关于方法能否被hook就已经判断完了。如果可以seletor可以被hook，继续if里面的代码。

##### Swizzling Method

我们直接说Swizzling Method这一最重要的逻辑吧。

Swizzling Method主要有两部分，一个是对对象的 forwardInvocation 进行 swizzling，另一个是对传入的 selector 进行 swizzling。

我们来看`aspect_prepareClassAndHookSelector`方法的源码。

###### 替换forwardInvocation

首先是`Class klass = aspect_hookClass(self, error);`

```objc
static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

 //
 // 省略一部分代码
 //
 //
	if (subclass == nil) {
        //动态创建子类+
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }
        //替换forwardInvocation方法
		aspect_swizzleForwardInvocation(subclass);
        // subclass的class方法交替换  替换为statedClass的class方法   subclass元类也替换
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
		objc_registerClassPair(subclass);
	}
    // 改变当前类的isa指针指向
	object_setClass(self, subclass);
	return subclass;
}
```

我们从源码中可以看出逻辑，主要是动态创建了一个subclass，名为subclass，其实最终把我们hook的类的isa指针指向了这个subclass，实为是一个父类。

```objc
aspect_swizzleForwardInvocation(subclass);
```

上面这个方法是替换了`forwardInvocation:`方法。

当然了，也是替换的这个subclass的`forwardInvocation:`方法，把`forwardInvocation:`替换为`__ASPECTS_ARE_BEING_CALLED__`这个方法，主要的hook后的代码执行处理逻辑都在这个`__ASPECTS_ARE_BEING_CALLED__`中。

在替换了`aspect_hookClass`方法之后，同时修改了 subclass以及其subclass metaclass的class方法，使他返回当前对象的class。这个地方有点绕，其实最终目的就是把所有的swizzling都放到了这个subclass中处理，不影响原来的类，而且对于外部的使用者，又可以把它当做原对象使用。

###### 替换selector

执行完`aspect_hookClass`完之后，`forwardInvocation:`方法已经被替换，下面会执行swizzling selector 的代码。

在swizzling selector的时候，将selector指向了消息转发IMP，同时生成一个aliasSelector，指向原方法的IMP。

这里代码就不往外粘了。

###### 处理forwardInvocation

其实上面已经把整个过程分析完了，我们也知道，最后转发的代码最终会在`__ASPECTS_ARE_BEING_CALLED__ `函数的处理中。所以最后我们来看看这个函数就可以了。


```objc
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];  //根据aliasSelector找到之前的逻辑 执行
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    // 没有找到之前方法的实现 - 消息转发
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

从源码中很容易看出来，分别处理不同的hook点，然后中间有根据
aliasSelector找到之前方法的实现，然后执行。

#### 小结

初步的源码分析就是这个样子，没有太关注一些细节，也存在一些自己现在还不是很熟悉的处理方式，毕竟涉及到太多的swizzling，消息转发一类的方法，这一块的只是需要后期在多研究runtime来提高。

代码中有一些其他的比较小的方法没有讲到，大家自己自行看一下。


#### 参考

https://wereadteam.github.io/2016/06/30/Aspects/

http://blog.ypli.xyz/ios/aop-mian-xiang-qie-mian-bian-cheng-yu-aspects-yuan-ma-jie-xi

https://blog.csdn.net/weixin_34375233/article/details/87974740



