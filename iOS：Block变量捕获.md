---
title: iOS：Block变量捕获
date: 2019-04-06 09:41:10
tags: [iOS,Block]
---

这篇博客我们从一个很常见的题目入手。

<!-- more -->

```objc
int age = 10;   
void (^myblock)(void) =  ^{
  NSLog(@"%d",age);
};
age  = 20;
myblock();
```

这个题目就涉及到了block内访问外部变量，block有个变量捕获机制，

我们新建一个mac的命令行工程，把上面代码写进去，然后用clang把main.m文件编译为cpp的文件看一下。

具体的block底层结构上一篇文章我们已经说过了，这里我们针对结构就不在赘述，直接说核心点。

#### auto类型局部变量

```objc
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

可以看到，在block的内部，多了一个age变量。而且从初始化的函数来看，这个`__main_block_impl_0`中age的值是外面传进来赋给他的。我们看一下main函数中。

```objc
 int age = 10;

void (*myblock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));

age = 20;

((void (*)(__block_impl *))((__block_impl *)myblock)->FuncPtr)((__block_impl *)myblock);
```

可以看出来在这个block初始化的过程中，就把age的值，也就是10，传了进去，赋值给了block内部的age变量。

那我再看一下打印的时候的age是什么情况。


```objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age; // bound by copy
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_6l_80sfw0tn35bg4dxc51jlq_bw0000gn_T_main_3b5bf4_mi_0,age);
}
```
打印的这个age就是block自己内部的这个ege变量（值为10），所以我们在执行block之前，改变外面的age值为20，其实改变的不是内部要打印的这个age了，所以打印出来结果还是10。


好的，我们定义的这个age就是一个普通的局部变量，其实就是auto类型的局部变量（C语言基础）。那我们下面改变一下这个变量的属性，尝试一下静态变量（static）和全局变量。

#### static类型局部变量

首先是静态变量。

```objc
//定义变量时加static标识
static int age = 10;
```

运行结果是20。我们依然要看一下clang编译一下，看c++代码。

```objc
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *age;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

我们来分析一下两次的异同，首先block都对变量进行了捕获，但不同的是static类型的变量是捕获了变量的指针，那我们应该就可以理解了，一般情况下这种基本数据类型如果是传指针，就意味要修改值的。

我们从main函数中也可以看出来，在初始化block的时候传入了外部age的指针，

```objc
static int age = 10;
void (*myblock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &age));
```

而且在打印的时候也是打印了这个指针指向的数据。

```objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *age = __cself->age; // bound by copy

  NSLog((NSString *)&__NSConstantStringImpl__var_folders_6l_80sfw0tn35bg4dxc51jlq_bw0000gn_T_main_d618bf_mi_0,(*age));
}
```
所以我们在执行block之前更改了age的值，在执行block的时候打印的也是同一块内存的值，所以值改变了。


最后我们在试一下全局变量。

#### 全局变量

在定义age的时候，定义成一个全局变量。（拿到main方法外面）。

```objc
int age = 10;


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
从编译后的代码可以看出，在底层其实也是生成了一个age=10的全局变量，然后在block内部并没有捕获这个变量。

在不管是重新赋值，还是输出打印，都是操作的这个全局的age，所以这个值也是能被改变的。

可能有人会问如果是

```objc
static int age = 10;
```

这个样子的全局变量呢？

全局变量加不加static都是一样的。这里就不上代码细说了。

**小结**

局部变量：

1. 会被block捕获到内部 
2. auto类型的是值传递，内部不能修改值
3. static是指针传递，可以修改值。

全局变量：

1. 不会被捕获到block内部
2. 可以修改值


-----

上面我们使用的是基本数据类型，那对象是不是也一样呢？

其实对象类型的也是一样的，但是有一个小点我们需要了解一下。

看下面这个例子吧。

```objc
NSMutableArray * arr = [NSMutableArray new];
   
void (^myblcok)(void) = ^{
  [arr addObject:@"1"];
  NSLog(@"%@",arr);
};
   
myblcok();
```

按照我们上面总结的这种auto类型的局部变量是要被捕获到内部的，但是应该不可以修改值。

上面代码执行后打印数组，是有@“1”这个元素的，这里我们就要搞清楚，我们调用`[arr addObject:@"1"];`并没有修改arr的值，只是只用了这个指针。

如果我们在block的回调中让`arr=nil;`，这算是改变arr的值，但是xcode就会报错的了。

