
#### 前言

最近打算重新梳理一遍iOS底层的知识，尽量把所有的底层知识点都搞懂搞透彻，碍于iOS不开源，有很多东西并不能很直观的去学习，所以可能有瑕疵，希望大家可以理解，并一起交流，笔者也尽可能做到尽善尽美吧。

#### KVO概述

KVO的底层是如何实现的呢？

对于这个问题，我想大家都可以简单的聊上这么几句。

> 对某个实例的某一个属性添加KVO监听后，系统会利用runtime的运行时特性，生成一个临时的类NSKVONotifying_xxx，然后把该实例的isa指针指向NSKVONotifying_xxx，监听哪个属性，就重写NSKVONotifying_xxx中此属性的set方法，然后在重写的set方法中实现监听和通知。

简单的来说就是这样，但是这太笼统了，下面我们通过例子，一步一步的来分析。

#### 探究

##### 1. 为什么会想到可能是类发生了变化？

```objc
Person * person1 = [[Person alloc] init];
Person * person2 = [[Person alloc] init];
    
[person1 addObserver:self
          forKeyPath:@"age"
             options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
             context:nil];
    
person1.age = 10;
person2.age = 20;
```

这是一个最基本的KVO使用，在回调中只有person1的值改变被监听了，但是我们在赋值的时候都是调用了age的set方法，如果我们在Person类中实现setAge：的方法并debug，这两次赋值都会走setAge方法，问题不是出在setAge这里，所以我们推测可能是类发声了某些变化。（此处应该有runtime的知识基础，runtime有能力对类做一些动态的改变）。
    
所以我们可以获取一下这两个实例的类型
    
```objc
// 输出 person1:NSKVONotifying_Person
NSLog(@"person1:%@",object_getClass(person1));
    
// 输出 person2:Person
NSLog(@"person2:%@",object_getClass(person2));
```


到这里我们就可以确定确实是生成了一个中间类。并且让person1的isa指针指向了这个类（object_getClassName方法就是返回isa的指向）。
    
**注：此处为什么要用runtime的api，因为runtime的api调用后的结果更加接近本质**


##### 2. NSKVONotifying_Person类中做了什么处理？

首先我们先看一下这面这个图，其实这就是添加了KVO之后类的类型结构
    
![kvo_1](img/kvo_1.jpg)

关于NSKVONotifying_Person类实现的方法，我们是怎么样得到的呢，这里我们可以借助runtime的api窥探一下。
    
```objc
[self printMethodList:object_getClass(person1)];
// 下面是方法实现
- (void)printMethodList:(Class)cls {
    unsigned int count;
    Method * methodList = class_copyMethodList(cls, &count);
    for (unsigned int i = 0; i < count; i++) {
        Method method = methodList[i];
        NSLog(@"method(%d) : %@", i, NSStringFromSelector(method_getName(method)));
    }
    free(methodList);
}
```
    
输出结果
    
```objc
method(0) : setAge:
method(1) : class
method(2) : dealloc
method(3) : _isKVOA
```

到这一步，我们可以先做一下小总结：
    
person2的isa指针指向Person类，所以在setAge的时候，就直接调用了Person中实现的setAge：方法，正常的赋值操作，没有触发KVO。但是person1的isa动态改变，指向了NSKVONotifying_Person，同时NSKVONotifying_Person中又重新实现了setAge：方法，所以在给person1的age赋值时，首先调用的是NSKVONotifying_Person中的setAge：方法，但是我们在之前的debug中发现，Person中的setAge：方法也会调用，其实这很容易理解，这应该是在NSKVONotifying_Person的setAge：实现中又调用了Person的setAge，毕竟NSKVONotifying_Person的isa指向Person（请自行验证）。

##### 3. NSKVONotifying_Person中的setAge：的实现
    
个人感觉，挖掘setAge：的实现是比较难的。

我们通过下面的方法打印一下方法IMP的地址
    
```objc
NSLog(@"person1添加KVO之前的两个setAge地址: \n -person1:%p -- person2:%p",
      [person1 methodForSelector:@selector(setAge:)],
      [person2 methodForSelector:@selector(setAge:)]);
    
[person1 addObserver:self
          forKeyPath:@"age"
             options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
             context:nil];
    
NSLog(@"person1添加KVO之后的两个setAge地址: \n -person1:%p -- person2:%p",
    [person1 methodForSelector:@selector(setAge:)],
    [person2 methodForSelector:@selector(setAge:)]);
```

输出:
    
```objc
person1添加KVO之前的两个setAge地址: 
-person1:0x1005ee850 -- person2:0x1005ee850
 
person1添加KVO之后的两个setAge地址: 
-person1:0x7fff25623f0e -- person2:0x1005ee850
```

我们可以看到，在添加KVO监听前后，person2的setAge实现的地址没有发生变化，但是person1的变了，我们在打印台用lldb命令打印一下0x7fff25623f0e

```objc
(lldb) p (IMP)0x7fff25623f0e
(IMP) $1 = 0x00007fff25623f0e (Foundation`_NSSetIntValueAndNotify)
```

可以看到setAge的实现其实就是调用了Foundation框架的_NSSetIntValueAndNotify方法。那具体的_NSSetIntValueAndNotify内部实现是怎么样的呢？因为Foundation不开源，我们只能猜测，并对我们的猜测做出相应的验证。
    
下面是我看过一些大神的分析之后猜测的_NSSetIntValueAndNotify实现的**伪代码**(特此鸣谢我们的MJ老师)
    
```objc
void _NSSetIntValueAndNotify() {
    [self willChangeValueForKey:@"age"];
    [super setAge:age];
    [self didChangeValueForKey:@"age"];
}


- (void)didChangeValueForKey:(NSString *)keyPath {
    // 通知监听者,已经修改完毕
    [observer observeValueForKeyPath:keyPath ofObject:self change:nil context:nil];
} 
```
    
如何验证一下我们的猜测呢？
    
我们知道NSKVONotifying_Person类中没有实现willChangeValueForKey和didChangeValueForKey这两个方法，所以我们可以在NSKVONotifying_Person的父类，也就是Person类型重写这两个方法，改造完之后的Person类里面应该是下面这样子：
    
```objc  
@interface Person : NSObject
@property (nonatomic, assign) int age;
@end
@implementation Person
- (void)setAge:(int)age {
    _age = age;
    NSLog(@"age:%d",age);
}
    
- (void)willChangeValueForKey:(NSString *)key {
    [super willChangeValueForKey:key];
    NSLog(@"willChangeValueForKey");
}
    
- (void)didChangeValueForKey:(NSString *)key {
    NSLog(@"didChangeValueForKey - begin");
    [super didChangeValueForKey:key];
    NSLog(@"didChangeValueForKey - end");
}
@end  
```
    
person1在添加了KVO监听，并设置值`person1.age = 10;`之后，输出如下：
    
```objc
2020-09-07 23:47:13.787066+0800 KVO[8122:119707] willChangeValueForKey
2020-09-07 23:47:13.787229+0800 KVO[8122:119707] age:10
2020-09-07 23:47:13.787334+0800 KVO[8122:119707] didChangeValueForKey - begin
2020-09-07 23:47:13.787592+0800 KVO[8122:119707] <Person: 0x60000288c190> -- age -- {
    kind = 1;
    new = 10;
    old = 0;
}
2020-09-07 23:47:13.787732+0800 KVO[8122:119707] didChangeValueForKey - end
```

输出结果与我们的猜测一致。
    
#### 其他小知识点
    
NSKVONotifying_Person也重写了class方法，使用[person1 class]的时候返回的是Person，其实也很容易理解，只是为了隐藏NSKVONotifying_Person这个类，尽量隐藏KVO的内部实现。
    
大家也可以看一下我下面附上的参考文章，写的很不错。
    
#### 结束语
    
经过上面的层层分析，我们探究了KVO的实现原理，有不缜密的地方还请指点。
感谢阅读。

#### 参考
[mikeash.com: just this guy, you know?](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)
[自己实现 KVO](https://tech.glowing.com/cn/implement-kvo/)

