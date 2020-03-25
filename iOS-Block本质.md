---
title: 'iOS：Block本质'
date: 2019-04-03 16:58:01
tags: [iOS,Block]
---

我们项目中经常使用block来进行回调传值，之前我对block的认识也就仅仅的停留在基础的层面，包括简单的使用和一些基本的避免循环引用的方法，这篇博客是我在对block进行了更深一层的学习之后的记录和总结，希望对大家有所帮助。

<!--more-->

#### Block的本质
新建一个命令行项目，写一个简单的block如下面所示。

```objc
void (^myBlock)(void) = ^{
    NSLog(@"11");
};        
myBlock();
```

使用clang工具把main.m文件编译为cpp文件。main函数变成了下面这个样子

```objc
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

        ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);


    }
    return 0;
}
```
里面有很多类型转换的代码，但是不难看出，我们的block被编译成了一个`__main_block_impl_0`类型的变量，我们可以搜索一下，这是一个结构体。

```objc
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
    
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

这个`__main_block_impl_0`结构体有两个属性，一个是`__block_impl`类型的结构体，一个是指向`__main_block_desc_0`结构体的指针。我们一个一个的看。


```objc
//__block_impl 结构体
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```
我们可以看到这个结构体有一个isa指针，同时在`__main_block_impl_0`这个结构体中是直接包含`__block_impl`结构体，从内存结构中其实这个`isa`指针就在`__main_block_impl_0`里面，也就是说block被编译成了一个拥有isa指针的结构体。了解过isa的应该知道，这算是oc对象的一个标志了，那也就是说block其实也是一个oc的对象。

我们继续看`__main_block_impl_0`内部

```objc
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
```
这是一个初始化的方法，因为这是c++代码嘛，c语言的结构体应该是不可以这样写的。

只要是给这个`__block_impl`类型的impl赋值，这里我们先主要看这个fp指针。

从编译之后main函数中看出来，在初始化`__main_block_impl_0`的时候传进去的参数是一个`__main_block_func_0`。

```objc
// __main_block_func_0
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_zx_pr_dx33d329d7xxgk9kgjdw80000gp_T_main_85af40_mi_0);
}
```
可以看出来这是我们block回调执行的方法，也就是说`impl.FuncPtr`指向了block的回调。

`__main_block_impl_0`还有一个Desc参数，我们可以在文件中查一下。

```objc
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
reserved是一个保留字段，Block_size就是记录我们这个block占内存的大小。

**小结**
Block本质就是一个OC对象，内部有isa指针。
Block是封装了函数调用和函数调用的环境的OC对象。

我们画个草图来梳理一下这种最简单的block的内部结构。

![](https://raw.githubusercontent.com/Sunxb/blog_img/master/13/01_block%E5%9B%BE%E7%A4%BA.png)







