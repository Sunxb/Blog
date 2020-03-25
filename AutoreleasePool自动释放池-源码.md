---
title: AutoreleasePool自动释放池-源码
date: 2019-02-18 16:08:58
tags: [iOS,AutoreleasePool]
---

AutoreleasePool相关的内容是在面试中比较容易被问到的。之前呢，谈到Autoreleasepool只能粗浅的了解到自动释放池与内存的管理有关，具体是怎么样来管理和释放对象，并没有深入的学习，本文是笔者在深入学习AutoreleasePool之后的总结和心得，希望对大家有帮助。

<!-- more -->

#### main函数
首先我们从main函数开始，main函数是我们应用的入口。

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

可以看出，我们整个iOS应用都是包含在一个自动释放池中。
而且在现在的ARC环境中，自动释放池的用法就是这样子` @autoreleasepool {}`。


#### @autoreleasepool
这个`@autoreleasepool{}`大家都会用，我们的代码直接写在这个大括号内即可。我们代码中的对象是怎样加到自动释放池中的，最后又是怎么样被释放的呢 ？

我们要先知道这个`@autoreleasepool`到底是什么。

从网上的一些博客中可以学到的，在命令行使用`clang -rewrite-objc main.m `让编译器重新改写main函数所在的这个文件。当然了这一步我并没有操作，直接“盗用”了大家的结果。

![objc-autorelease-main-cpp](https://raw.githubusercontent.com/Sunxb/blog_img/master/09/1.jpeg)


从上图中可以看出，`@autoreleasepool`被转换为了一个`__AtAutoreleasePool`结构体。然后通过在main.cpp中查找找到这个结构体的定义。

![objc-autorelease-main-cpp-struct](https://raw.githubusercontent.com/Sunxb/blog_img/master/09/2.jpeg)


```objc
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

这个结构体在初始化是调用`objc_autoreleasePoolPush()`，在析构时调用`objc_autoreleasePoolPop()`。

经过整理，可以把main函数中实际的代码应该是类似这样的。

```objc
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        // do whatever you want
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

下面就是正式的通过源码来学习Autoreleasepool了

#### Autoreleasepool源码

上面我们提到了mian函数中的`@autoreleasepool`其实最终转成了`objc_autoreleasePoolPush()`和`objc_autoreleasePoolPop()`这两个方法的调用，我们去源码中搜一下这两个函数。

```objc
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```

```objc
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

有源码可以看出，这两个函数就是对AutoreleasePoolPage类的pish和pop方法的封装，所以我们来着重看AutoreleasePoolPage类。

##### AutoreleasePoolPage
简化一下代码，先看类的部分属性

```objc
class AutoreleasePoolPage {

    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

熟悉链表的朋友，看到这个parent和child就差不多能猜出来了，每一个自动释放池其实是一个双向链表，链表的每一个结点就是这个AutoreleasePoolPage，每个AutoreleasePoolPage的大小为4096字节

```
#define I386_PGBYTES 4096
#define PAGE_SIZE I386_PGBYTES
```

##### 自动释放池中的栈（转载）
如果我们的一个`AutoreleasePoolPage` 被初始化在内存的 0x100816000 ~ 0x100817000 中，它在内存中的结构如下：

![objc-autorelease-page-in-memory](https://raw.githubusercontent.com/Sunxb/blog_img/master/09/3.png)


其中有 56 bit 用于存储 AutoreleasePoolPage 的成员变量，剩下的 0x100816038 ~ 0x100817000 都是用来存储加入到自动释放池中的对象。

`begin()` 和 `end()` 这两个实例方法帮助我们快速获取 0x100816038 ~ 0x100817000 这一范围的边界地址。

`next` 指向了下一个为空的内存地址，如果`next`指向的地址加入一个 `object`，它就会如下图所示移动到下一个为空的内存地址中：

![objc-autorelease-after-insert-to-page](https://raw.githubusercontent.com/Sunxb/blog_img/master/09/4.png)


从图片中我们可以看到在AutoreleasePoolPage的栈中出现了一个`POOL_SENTINEL`，我们称之为哨兵对象。

```
#define POOL_SENTINEL nil
```
其实哨兵对象只是nil的别名，他有啥作用呢 ?

每个自动释放池初始化在调用`objc_autoreleasePoolPush`的时候，都会把一个`POOL_SENTINEL`push到自动释放池的栈顶，并且返回这个`POOL_SENTINEL`的地址。

```
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        // do whatever you want
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```
上面这个atautoreleasepoolobj就是一个`POOL_SENTINEL`。
可以看到在调用objc_autoreleasePoolPop时，会传进去这个地址：

 1. 根据传入的哨兵对象地址找到哨兵对象所处的page
 2. 在当前page中，将晚于哨兵对象插入的所有autorelease对象都发送一次- release消息，并向回移动next指针到正确位置
 3. 补充2：从最新加入的对象一直向前清理，可以向前跨越若干个page，直到哨兵所在的page

因为自动释放池是一个双向链表，而且每一个page的空间有限，所以会存在当前page已满的情况，也就出现了一个自动释放池跨越几个page的情况，所以在release的时候，也要顺着链表全部清理掉。
 
##### objc_autoreleasePoolPush 

```objc
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }
```
经查阅，DebugPoolAllocation是来区别调试模式的，我们主要看autoreleaseFast这个函数。

```objc
    static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {
            return page->add(obj);
        } else if (page) {
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
    }
```

hotPage( )我们可以理解为获取当前的AutoreleasePoolPage，获取到当前page之后又根据page是否已满来区别处理。

- 有 hotPage 并且当前 page 不满

    - 调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中
    
- 有 hotPage 并且当前 page 已满
    - 调用 autoreleaseFullPage 初始化一个新的页
    - 调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中

- 无 hotPage
    - 调用 autoreleaseNoPage 创建一个 hotPage
    - 调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中

```objc
    static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        assert(page == hotPage());
        assert(page->full()  ||  DebugPoolAllocation);

        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());

        setHotPage(page);
        return page->add(obj);
    }
```
上面代码中的函数是在page已满的时候调用，从源码中可以看出通过传入的page遍历链表，直到找到一个未满的page，如果遍历到最后一个结点也没有未满的，就新建一个`new AutoreleasePoolPage(page);`。并且要把找到的满足条件的这个page设置为hotPage。

##### objc_autoreleasePoolPop

我们看一下pop的源码，里面内容很多，我们精简了一下。

```objc
static inline void pop(void *token) {
    AutoreleasePoolPage *page = pageForPointer(token);
    id *stop = (id *)token;

    page->releaseUntil(stop);
    
    if (page->child) {
    // 不清楚为什么要用下面这个if分类
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```
 通过token调用`pageForPointer()`方法获取到当前的AutoreleasePoolPage，然后调用`releaseUntil()`释放page中的对象，直到stop，child节点调用`kill()`方法。
 
 ```objc
 void kill() {
    AutoreleasePoolPage *page = this;
    //通过循环先找到最后一个节点
    while (page->child) page = page->child;

    AutoreleasePoolPage *deathptr;
    //通过do-while循环，依次从后往前置为nil
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->unprotect();
            page->child = nil;
            page->protect();
        }
        delete deathptr;
    } while (deathptr != this);
}
 ```
 
 `pageForPointer()`主要是通过内存地址的操作，获取当前指针所在页的首地址， `releaseUntil()`也是通过一个循环来释放所有的对象，具体的源码大家可以自己看一下。
 
##### Autorelease对象什么时候释放

在没有手动加AutoreleasePool的情况下，Autorelease对象都是在当前的runloop迭代结束时释放的，因为系统在每个runloop迭代中都加入了自动释放池Push和Pop。

这个问题又要跟runloop联系到一起了，等我们研究过runloop的源码，对这个问题应该就有更深刻的认识了。

#### 参考文章

[自动释放池的前世今生](https://github.com/draveness/analyze/blob/master/contents/objc/%E8%87%AA%E5%8A%A8%E9%87%8A%E6%94%BE%E6%B1%A0%E7%9A%84%E5%89%8D%E4%B8%96%E4%BB%8A%E7%94%9F.md)
[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
 
 
 
 




