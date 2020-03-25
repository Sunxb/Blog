---
title: MJExtension源码学习(一)
date: 2018-06-01 15:18:33
tags: [iOS,MJExtension]
---

继续进行优秀开源框架的源码学习，这次打算学习一些常用的model解析的框架，比如YYModel，MJExtension，Mantle等。我自己用过YYModel和MJExtension，比较简单易用，看过别人用Mantle的代码，个人感觉稍微繁琐一些，所以这次就先学习MJExtension吧。

本次的学习我分为了两个过程：

1. 初步了解MJExtension的原理，并通过第一版的代码学习基本的逻辑和思路。
2. 阅读学习新版本的代码，加深对MJExtension的认识。

**本文主要是记录第一个过程中的学习和心得。**

MJExtension从最初到现在，也已经更新了几十个版本了。所以在开始之前，我们先查阅一些别的资料，从大概上来了解一下MJExtension的实现原理和一些学习的点。

<!-- more -->

[参考文章](https://juejin.im/post/5a93c171f265da4e747fdbfb)

就拿最简单的json转model来说，我的个人观点，其实主要是**运行时机制**和**递归思想**相结合来实现。通过运行时机制，我们可以获取到一个类的所有属性，然后通过遍历来对每一个属性进行赋值，如果该属性又是一个自定义的类，那就用到递归的思想这样一级级的解析下去，直到解析完成。数组也是一样的，只是多了一个对数组遍历的环节。

现在让我们具体来看MJExtension第一版本的代码（以下所提到MJExtension都是指的它的第一版，特殊情况会单独指出）

MJExtension中最主要的就是`NSObject+MJKeyValue`这个类，他通过分类的形式向我们提供了dict->model的方法，然后其中重要的方法`- (instancetype)setKeyValues:(NSDictionary *)keyValues`

#### 赋值部分

下面是其中的代码：

    - (instancetype)setKeyValues:(NSDictionary *)keyValues
    {
        MJAssert2([keyValues isKindOfClass:[NSDictionary class]], self);
        
        [[self class] enumerateIvarsWithBlock:^(MJIvar *ivar, BOOL *stop) {
            // 1.取出属性值
            id value = keyValues ;
            for (NSString *key in ivar.keys) {
                value = value[key];
            }
            if (!value || value == [NSNull null]) return;
            
            // 2.如果是模型属性
            MJType *type = ivar.type;
            Class typeClass = type.typeClass;
            if (!type.isFromFoundation && typeClass) {
                value = [typeClass objectWithKeyValues:value];
            } else if (typeClass == [NSString class]) {
                if ([value isKindOfClass:[NSNumber class]]) {
                    // NSNumber -> NSString
                    value = [_numberFormatter stringFromNumber:value];
                } else if ([value isKindOfClass:[NSURL class]]) {
                    // NSURL -> NSString
                    value = [value absoluteString];
                }
            } else if ([value isKindOfClass:[NSString class]]) {
                if (typeClass == [NSNumber class]) {
                    // NSString -> NSNumber
                    value = [_numberFormatter numberFromString:value];
                } else if (typeClass == [NSURL class]) {
                    // NSString -> NSURL
                    value = [NSURL URLWithString:value];
                }
            } else if (ivar.objectClassInArray) {
                // 3.字典数组-->模型数组
                value = [ivar.objectClassInArray objectArrayWithKeyValuesArray:value];
            }
            
            // 4.赋值
            [ivar setValue:value forObject:self];
        }];
        
        // 转换完毕
        if ([self respondsToSelector:@selector(keyValuesDidFinishConvertingToObject)]) {
            [self keyValuesDidFinishConvertingToObject];
        }
        
        return self;
    }
    
可以看的出这个方法里面的主要代码就是`enumerateIvarsWithBlock` 回调里面的代码，回调中返回了一个`MJIvar`的类，MJIvar其实就是对你的model类里面的每一个成员变量做的进一步的封装，封装后每一个成员变量对应对封装成一个MJIvar的实例。

MJIvar中的大多数字段都是起到了一个标识和记录的作用，比如说成员变量的名、属于哪个类等，其中还包含一个MJType类型的属性，其实也是做一些标识的作用，大家点进去看看就一目了然了。

做这一层的封装主要是为了之后的处理值时使用。

上述代码的中间部分大篇的if，else if的判断就是在做值的分类处理

    MJType *type = ivar.type;
    Class typeClass = type.typeClass;
    if (!type.isFromFoundation && typeClass) {
       value = [typeClass objectWithKeyValues:value];
    }
这第一个判断，就是用于如果model中的某个成员变量还是一个自定义类的情况，type中的isFromFoundation字段就是标识改成员变量的类是否是自定义的类，如果是自定义的类，把这个类存进type.typeClass下面。
`value = [typeClass objectWithKeyValues:value];`
这句代码也就是递归思想的提现，如果这个成员变量是一个自定义类的，那么该成员变量对应的值应该也是一个model，所以用这个二级的model类继续调用`objectWithKeyValues`方法，继续解析下去。

如果是数组，调用`objectArrayWithKeyValuesArray`这个方法，原理相同，只是多一层的遍历。

所有的解析，最后都调用了

    // 4.赋值
    [ivar setValue:value forObject:self];
这个是MJIvar中的方法

    - (void)setValue:(id)value forObject:(id)object
    {
        if (_type.KVCDisabled) return;
        [object setValue:value forKey:_propertyName];
    }

这里_propertyName的也是MJIvar的一个字段，就是记录封装成MJIvar之前的这个成员变量的名字，然后使用
`setValue:forKey`为一个类的成员变量赋值。

#### 获取类的成员变量，封装MJIvar

上面的是所有的成员变量已经封装成MJIvar之后，遍历所有的MJIvar并最终赋值的过程，还有一个方法也很重要，他实现了获取所有的成员变量并封装的这个过程的。下面看`+ (void)enumerateIvarsWithBlock:(MJIvarsBlock)block`这个方法

```objc
+ (void)enumerateIvarsWithBlock:(MJIvarsBlock)block
{
    static const char MJCachedIvarsKey;
    // 获得成员变量
    NSMutableArray *cachedIvars = objc_getAssociatedObject(self, &MJCachedIvarsKey);
    if (cachedIvars == nil) {
        cachedIvars = [NSMutableArray array];
        
        [self enumerateClassesWithBlock:^(__unsafe_unretained Class c, BOOL *stop) {
            // 1.获得所有的成员变量
            unsigned int outCount = 0;
            Ivar *ivars = class_copyIvarList(c, &outCount);
            
            // 2.遍历每一个成员变量
            for (unsigned int i = 0; i<outCount; i++) {
                MJIvar *ivar = [MJIvar cachedIvarWithIvar:ivars[i]];
                ivar.key = [self ivarKey:ivar.propertyName];
                // 如果有多级映射
                ivar.keys = [ivar.key componentsSeparatedByString:@"."];
                // 数组中的模型类
                ivar.objectClassInArray = [self ivarObjectClassInArray:ivar.propertyName];
                ivar.srcClass = c;
                [cachedIvars addObject:ivar];
            }
            
            // 3.释放内存
            free(ivars);
        }];
        objc_setAssociatedObject(self, &MJCachedIvarsKey, cachedIvars, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    // 遍历成员变量
    BOOL stop = NO;
    for (MJIvar *ivar in cachedIvars) {
        block(ivar, &stop);
        if (stop) break;
    }
}
```

大家可以看到，它主要的部分，又是一个遍历之后的回调， 我们那顺便贴出来这个遍历的代码

```objc
+ (void)enumerateClassesWithBlock:(MJClassesBlock)block
{
    // 1.没有block就直接返回
    if (block == nil) return;
    
    // 2.停止遍历的标记
    BOOL stop = NO;
    
    // 3.当前正在遍历的类
    Class c = self;
    
    // 4.开始遍历每一个类
    while (c && !stop) {
        // 4.1.执行操作
        block(c, &stop);
        
        // 4.2.获得父类
        c = class_getSuperclass(c);
        
        if ([MJFoundation isClassFromFoundation:c]) break;
    }
}
```
很容易看出来，这是为了处理那种父类也是**自定义类**的情况，那我们实际开发中来说，一般后台返回的数据都是有固定形式的，比如说status，message，code这种字段是每个接口都返回的，所以这些字段我一般都写在一个父类里面，然后这个循环就是遍历出父类，为从父类中继承的字段赋值。

说回上一个遍历方法，其实就是一个封装过程，从代码中可以看出来，使用了一些runtime
的api来获取了类的成员变量，然后通过循环对每个变量进行了封装。这一步的话，光这样干很难体验什么，大家可以跑一个简单的例子，然后debug跟一下，看看MJIvar中每个属性代表什么。

#### 映射问题
有些情况可能要映射字段名，比如id属于关键字，可能公司要求model中不允许用id作为成员变量名， 所以要做映射处理，还有如果数组中如果包含别的model，这个组数中的model类名我们也应该告诉MJExtension。

MJExtension都抛出了方法，需要映射名称使用`replacedKeyFromPropertyName`,数组包含模型使用`objectClassInArray`,我们根据自己的需要的重写相应方法。

举个例子：

成员变量属于是数组的情况下， 这个数组里面包含model类型要存在ivar.objectClassInArray下面

    // 数组中的模型类
    ivar.objectClassInArray = [self ivarObjectClassInArray:ivar.propertyName];

这是`ivarObjectClassInArray`的实现

```objc
+ (Class)ivarObjectClassInArray:(NSString *)propertyName
{
    if ([self respondsToSelector:@selector(objectClassInArray)]) {
        return self.objectClassInArray[propertyName];
    } else {
        // 为了兼容以前的对象方法
        id tempObject = self.tempObject;
        if ([tempObject respondsToSelector:@selector(objectClassInArray)]) {
            id dict = [tempObject objectClassInArray];
            return dict[propertyName];
        }
        return nil;
    }
    return nil;
}
```
在赋值的时候会先判断是否`respondsToSelector`，然后根据情况赋值。

第一版的代码比较简单，我就简单的dict->model说了一下，model->dict大家可以自己再去看看，当然其他还有一些细节处理的东西， 大家也可以通过代码来进一步学习。

对MJExtension学习的第一步就先到这，慢慢我会继续看它的新的代码，学习他的一些优化和封装。




