---
title: __weak对引用计数的影响
date: 2018-08-10 14:40:25
tags: [iOS,内存管理]
---


>问题：打印__weak修饰的变量，引用计数+1 ? 

从一个例子入手吧

```objc
UIView * v = [[UIView alloc] init];
UIView * __weak v1 = v;
NSLog(@"retain count = %ld\n",CFGetRetainCount((__bridge CFTypeRef)(v)));
NSLog(@"retain count = %ld\n",CFGetRetainCount((__bridge CFTypeRef)(v1)));
```

我们初始化了一个view，并用v来指向它，然后在创建一个用__weak修饰的v1，跟v指向同一个内存空间。

按照我们的理解，打印v的引用计数是1，这个毫无疑问的，因为v1是使用__weak修饰，只有指向，但并不持有，所以按理说打印v1的引用计数应该也是1.

下面是结果

```objc
retain count = 1
retain count = 2
```

v1打印的引用计数竟然是2 ？

我查了查之后得出了结论：

当我们把__weak修饰的变量传进NSLog方法中打印，这个方法需要持有这个变量，为了安全起见嘛，如果不强引用一下，万一还没打印的被释放了呢 ？ 所以会对v1调用`objc_loadWeakRetained`, 这时候v1的引用计数就会+1，在NSLog结束是，会调用`objc_release`, 然后引用计数-1。

为了辅证这个结论，我们可以把代码转成汇编来看一下。
![](https://raw.githubusercontent.com/Sunxb/blog_img/master/05-iOS-__weak%E5%AF%B9%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E7%9A%84%E5%BD%B1%E5%93%8D/1.jpg)


然后我们直接搜索nslog，看第二个
![](https://raw.githubusercontent.com/Sunxb/blog_img/master/05-iOS-__weak%E5%AF%B9%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E7%9A%84%E5%BD%B1%E5%93%8D/2.jpg)
确实是这个样子~ 那就不用多解释了 ~ 



