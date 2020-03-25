---
title: iOS：Block 循环引用问题
date: 2019-04-07 10:56:48
tags: [iOS,Block]
---

循环引用是一个比较常见的问题，之前面试的时候也会被问到，如何解决循环引用问题，其实大家都知道使用__block,__weak这些修饰符可以解决循环引用问题，那今天我们要讨论的就是他们是怎么样解决了循环引用问题的。

<!-- more -->

#### __weak

其实__weak是比较好理解的，它的作用就是在两方相互强引用的时候，把其中一个引用变为弱引用，打破这个循环引用的圈。

我们通过代码看一下。

```objc
MyPerson * person = [[MyPerson alloc] init];
person.age = @"10";
__weak typeof(person) weakPerson = person;
person.block = ^{
    NSLog(@"age is %@", weakPerson.age);
};
        
```

MyPerson类里面有一个block，一个string类型的age，在执行block的时候，打印了age，如果不用weakPerson的话，就会产生循环引用，这种用法想必大家都很熟悉。

那我们看一下编译后的cpp文件。

```objc
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  MyPerson *__weak weakPerson;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, MyPerson *__weak _weakPerson, int flags=0) : weakPerson(_weakPerson) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

可以看到block内部捕获到的是`MyPerson *__weak weakPerson;`，所以不会产生强引用，自然也就不会出现循环引用问题。

__weak只在ARC环境下使用。


#### __block

最开始我以为__block消除循环引用的方式跟__weak是一样的。

```objc
//这种用法ARC环境下是错的 MRC可以
MyPerson * person = [[MyPerson alloc] init];
person.age = @"10";   
__block typeof(person) weakPerson = person;
person.block = ^{
    NSLog(@"age is %@", weakPerson.age);
};
```

我们现在开发一直都是在ARC环境下，首先自己检讨一下，我一直都以为__block可以这么用，而且关键是这样用了确实编译器就没有了关于循环引用的警告了。

但是我们如果重写一下MyPerson类的dealloc方法，让对象释放时打印点东西，你会发现如果使用__weak，在main函数结束时，person会调用dealloc释放，但是如果像上面一样用__block，person不会释放，还是存在循环引用。

我强调了这种用法在ARC环境下不可以，但是在MRC环境下是可以的，因为MRC环境下block不会对__block修饰的属性强引用。


**下面是ARC环境正确的__block使用方式。**

如果就按照__weak的使用方法使用，在block内部把weakPerson置为nil，同时这个block**必须要调用**。

```objc
MyPerson * person = [[MyPerson alloc] init];
person.age = @"10";
        
__block typeof(person) weakPerson = person;
person.block = ^{
    NSLog(@"age is %@", weakPerson.age);
    weakPerson = nil;
};
person.block();//必须有这个代码
```

也可以这样写：

```objc
__block MyPerson * person = [[MyPerson alloc] init];
person.age = @"10";
person.block = ^{
    NSLog(@"age is %@", person.age);
    person = nil;
};
        
person.block();
```

两种写法都一样，必须手动置为nil，然后必须执行block。下面我们说一下原理。

首先呢，我们上面已经说了，如果不手动置为nil的话，使用__block依然有循环引用，我们结合cpp的代码分析一下具体循环引用在什么地方。

我们编译下面这种写法。

```objc
MyPerson * person = [[MyPerson alloc] init];
__block typeof(person) weakPerson = person;
person.age = @"10";
person.block = ^{
    NSLog(@"age is %@", weakPerson.age);
};
```

我们来分析一下下面的代码

```objc
struct __Block_byref_weakPerson_0 {
  void *__isa;
__Block_byref_weakPerson_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 typeof (person) weakPerson;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_weakPerson_0 *weakPerson; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_weakPerson_0 *_weakPerson, int flags=0) : weakPerson(_weakPerson->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

其实这个__block属性的作用咱们之前已经说过了，就是在block内部把修饰的属性包装成一个对象，也就是这个`__Block_byref_weakPerson_0`。`__Block_byref_weakPerson_0`内部有我们的weakPerson属性，`typeof (person) weakPerson`，这里只是叫weakPerson，他的持有方式还是strong的。

所以说我们可以分析出来，__block属性持有我们的person变量，person持有block，block内部持有这个__block属性，就像下面这个图示一样。

![__block循环引用.png](https://raw.githubusercontent.com/Sunxb/blog_img/master/14/2.jpg)


我们通过置为nil解决它循环引用的方式，就是打断一条强引用。如下图

![](https://raw.githubusercontent.com/Sunxb/blog_img/master/14/3.jpg)



#### __unsafe_unretained

ARC环境下__unsafe_unretained与__weak使用方法相同。

MRC环境下，与__block MRC环境下的使用一样。

**__unsafe_unretained和__weak对比：**

__weak：不会产生强引用，指向的对象销毁时，会自动让指针置为nil

__unsafe_unretained：不会产生强引用，不安全，指向的对象销毁时，指针存储的地址值不变


