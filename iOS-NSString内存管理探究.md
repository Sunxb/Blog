---
title: iOS:NSString内存管理探究
date: 2018-07-27 14:48:07
tags: [iOS,内存管理]
---

>本文主要包括NSString的内存管理问题，按照string的初始化方法来分析。还包括使用__weak后的影响

在我们所认识的内存管理的规则下看下面的代码

    NSObject * __weak ob = [[NSObject alloc] init];
    NSLog(@"%@",ob);

输出应该是null才对，但是最近发现了NSString这个类的一些初始化问题很有意思，拿出来跟大家探讨一下。

首先我们先定义一个宏，打印字符串的值+内存地址+所属类名

    #define KLog(_c) NSLog(@"%@ -- %p -- %@",_c,_c,[_c class]);

我把字符串的初始化方式分为了两类，一类是后面是带withString的(直接=@"xx"这种归入此类)，一类是后面带withFormat的

**第一种：**

```onjc
NSString * __weak s1 = @"111";
NSString * __weak s2 =[[NSString alloc] initWithString:@"222"];
NSString * __weak s3 = [NSString stringWithString:@"33"];
KLog(s1)
KLog(s2)
KLog(s3)
```

输出结果是
![结果1](https://upload-images.jianshu.io/upload_images/1491333-2fb6e5d3c59b6488.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!-- more -->

这几种方式创建出来的string，他实际属于__NSCFConstantString类，__NSCFConstantString代表的是一个字符串常量，retain和copy操作都还是自己，没有任何变化。（可以打印一下引用计数，数值很大，没啥作用了~ ）。所以说__NSCFConstantString类的字符串是创建了就不能被我们释放的。所以即使我们前面加了__weak, 在下面用NSLog输出s1,s2,s3的时候也是有值的。（按正常的内存管理规则来说前面加了__weak会让变量立即释放掉的~）

如果__NSCFConstantString类的字符串的内容都是一样的，那么不论怎么创建，他们的地址都是一样的，可以理解为此类为一个单例。


**第二种：**

```objc
NSString *  s1 = [NSString stringWithFormat:@"123456789"];
KLog(s1)

NSString *  s2 = [NSString stringWithFormat:@"1234567890"];
KLog(s2)

NSString *  s3 = [[NSString alloc] initWithFormat:@"123456789"];
KLog(s3)

NSString *  s4 = [[NSString alloc] initWithFormat:@"1234567890"];
KLog(s4)
```



在进行试验之前我从网上查到，这种初始化方式的字符串有一个阈值，字符串长度>9的时候，字符创的类型是__NSCFString，<=9时是NSTaggedPointerString类型。所以我在测试时，分别穿进去了长度为9和10两种情况。

注： 这个长度的阈值到底是多少，我也是在是不太清楚，只是发现网上大多都是使用9为界限，所以暂且也是用这个阈值。


下面是运行结果
![结果2](https://upload-images.jianshu.io/upload_images/1491333-4b2acf4c8946cb61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


结果符合我们的预期。

那我们现在就要稍微了解一下__NSCFString和NSTaggedPointerString这两种类型了

__NSCFString 表示对象类型的字符串，我们可以把他理解为正常的符合内存管理规则的那种字符串。该种类型字符串通过format方式创建。

NSTaggedPointerString 类型的字符串是对__NSCFString类型的一种优化，在运行时创建字符串时，会对字符串内容及长度作判断，若内容由ASCII字符构成且长度较小（具体要多小暂时不太清楚），这时候创建的字符串类型就是 NSTaggedPointerString （标签指针字符串），字符串直接存储在指针的内容中。

上面的这两种描述都是我通过查过资料得出的， 其实我个人理解就是NSTaggedPointerString是一种对__NSCFString的优化，在一些内存的使用上面有所优化和改变。

NSTaggedPointerString类也不符合内存管理的规则。

对此我们多做一步测试，在上面的代码上面加上__weak, 看看具体的内存情况。

```objc

    NSString * __weak s1 = [NSString stringWithFormat:@"123456789"];
    KLog(s1)

    NSString * __weak s2 = [NSString stringWithFormat:@"1234567890"];
    KLog(s2)

    NSString * __weak s3 = [[NSString alloc] initWithFormat:@"123456789"];
    KLog(s3)

    NSString * __weak s4 = [[NSString alloc] initWithFormat:@"1234567890"];
    KLog(s4)
    
```
   
    
结果：

![结果3](https://upload-images.jianshu.io/upload_images/1491333-4cd36812bac3becf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


确实NSTaggedPointerString类的都没有释放，但是又出现了别的问题，那按理说__NSCFString类的应该都是null了，怎么第二个，也就是类方法+stringWithFormat创建出来的也没释放呢 ? 

我查了一下，原因是因为+stringWithFormat: 返回的是一个autorelease的string, 所以并不会立即释放，也不需要我们手动释放。


-----
参考资料:

https://blog.csdn.net/shifang07/article/details/54409763
https://stackoverflow.com/questions/3898974/stringwithformat-vs-initwithformat-on-nsstring



