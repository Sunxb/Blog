---
title: iOS：Block __block修饰符
date: 2019-04-07 08:55:29
tags: [iOS,Block]
---


#### __block修饰符
上一篇文章中说过，auto类型的局部变量，可以被block捕获，但是不能修改值。

__block可以解决block内部无法修改外部auto变量的问题。

<!-- more -->

```objc
__block int age = 10;
void (^myblock)(void) =  ^{
  NSLog(@"%d",age);
};
age  = 20;
myblock();
```

用法就是这么简单，这样我们修改age为20的时候，打印也是20。

我们看看编译后的代码。

```objc
struct __Block_byref_age_0 {
  void *__isa;
__Block_byref_age_0 *__forwarding;
 int __flags;
 int __size;
 int age;
};

//Block
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_age_0 *age; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

在block内部多了一个指向`__Block_byref_age_0`类型结构体的age指针。上面我也帖上了这个`__Block_byref_age_0`结构体的结构。我们发现int类型的age在这个结构体内部了。

那也就是说，__block修饰的变量，编译器会把它包装成一个对象，然后我们的这个成员变量放到了这个对象的内部。

我们观察一下这个`__Block_byref_age_0`内部，这些变量可能有疑惑的也就是这个`__forwarding`。他是一个指向这个结构体自身的指针。而且我们还可以看出来在打印age的时候，是也是通过__forwarding调用的age（`age->__forwarding->ag`），具体为什么要多加这个字段，我们后面再说。

```objc
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
}
```

`__main_block_desc_0`这个结构体中也多了两个指针，这是与内存管理有关的函数。

底层分析差不多了，那我们还没说到为什么__block修饰的属性，在block内部可以修改，我们看下面的代码

```objc
 __attribute__((__blocks__(byref))) __Block_byref_age_0 age = {(void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};

void (*myblock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));

(age.__forwarding->age) = 20;
```

我们创建了`__Block_byref_age_0`类型的age对象，同时把外部age的值也就是10，传递了进去。然后初始化了block。

关键是下面的修改age的值的时候，直接就是修改的age对象里面的age属性了，然后打印的时候，也是打印的他。

这个地方其实还是挺抽象的了，也不是很好理解。

怎么前面定义的age变量跟后面修改的就不是一个了？

```objc
__block int age = 10;
NSLog(@"%p",&age);
void (^myblock)(void) =  ^{
  NSLog(@"%d",age);
};
NSLog(@"%p",&age);
age  = 20;
myblock();
```

这是最简单的方法，打印出两个age的地址，就是不一样的。那我们怎么去判断就是`__Block_byref_age_0`里面的age呢，大家可以参考下面的做法。

```objc
struct __Block_byref_age_0 {
    void *__isa;
    struct __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age;
};

struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(void);
    void (*dispose)(void);
};

struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

struct __main_block_impl_0 {
    struct __block_impl impl;// 8+4+4+8
    struct __main_block_desc_0* Desc;//8
    struct __Block_byref_age_0 *age;// 8+8+4+4
};

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        __block int age = 10;
        
        void (^block)(void) = ^{
            age = 20;
            NSLog(@"age is %d", age);
        };
        
        struct __main_block_impl_0 *blockImpl = (__bridge struct __main_block_impl_0 *)block;
        
        NSLog(@"%p", &age);
    }
    return 0;
}
```

上面的操作就是我们把底层的一些结构拿出来，然后把我们的block类型桥接成`__main_block_impl_0`类型。然后我们通过debug可以拿到这个blockImpl的地址，然后通过内存中的地址偏移计算出来内部`__Block_byref_age_0`中的age地址，看看和打印出来的age地址是否一致。

我们拿到blockImpl的地址是0x1005002d0（我测试时候的值，每次都不同）。

`__main_block_impl_0`内部第一个属性是一个`__block_impl`结构体，然后`__block_impl`内部两个指针（一个指针8字节）两个int（一个int4字节），共占24字节。
第二个参数是一个指针，8字节。
age在第三个参数指向的结构体中，也就是说`__Block_byref_age_0`类型的age内存地址是0x1005002d0偏移32，也就是0x100500300。然后`__Block_byref_age_0`内部的age变量前面，有两个指针两个int，24字节，0x100500300再偏移24，也就是0x100500318。跟我们`NSLog(@"%p", &age);`打印的一致。

所以可以得出，我们修改的这个age，其实就是底层age对象内部的age变量。


上面我们留下了一个`__forwarding`指针的疑问，我们先不着急解决，先说说block类型。

#### block的类型

block有isa指针，开始我想通过写了几个类型的block，用clang编译看cpp代码，但发现一直是这个样子`impl.isa = &_NSConcreteStackBlock;`。所以我就用最直接的打印[obj class]的方法。

注意，因为在ARC的环境下，编译器给我们做了很多内存相关的工作，所以我在研究block类型的过程中切换到了MRC环境。

我用过的例子就不写了，下面是一个小总结。

一共有三种Block

`__NSGlobalBlock__  内存位于数据区`
`__NSStackBlock__   栈区`
`__NSMallocBlock__  堆区`

具体什么样的block对应哪一种类型？

`__NSGlobalBlock__`：没有访问auto变量
`__NSStackBlock__`：访问了auto变量
`__NSMallocBlock__`： `__NSStackBlock__`调用copy

**提示**
我们在声明一个block属性的时候，习惯用copy关键字，是为了把栈区的block拷贝到堆区，让我们来管理他的生命周期。

ARC环境下会根据情况自动将栈上的block拷贝到堆上。ARC环境下也用copy是为了和MRC环境统一，也可以用strong。


#### __block修饰对象类型

当__block变量在栈上时，不会对指向的变量产生强引用。

当__block变量copy到堆上时，会根据这个变量的修饰符是__strong,__weak,__unsafe_unretained做出相应的操作形成强引用（retain）或者弱引用。（ARC会retain，MRC不会）。


#### __forwarding指针

最后说一下上面的__forwarding指针问题。

![](https://raw.githubusercontent.com/Sunxb/blog_img/master/14/1.jpg)


这个图可以很好地诠释这个问题了。

我们的block在内存中可能位于栈上，可能在堆上。

使用了这个指针之后，让我们在block位于不同内存位置的情况下，访问到相应准确位置的变量。

