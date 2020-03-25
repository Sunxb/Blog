---
title: ReactiveCocoa学习
date: 2017-04-11 14:51:02
tags: [iOS,ReactiveCocoa]
---
![版本](http://upload-images.jianshu.io/upload_images/1491333-80aec16ab7edd8db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
到我写这篇文章为止,ReactiveCocoa版本为5.0.1,搜了很多博客来了解ReactiveCocoa的基础用法,并不是很多,所以这篇文章算是自己对搜索资源的一个汇总,并加入一些自己在学习中遇到的问题和解决.

RAC 5.0 相比于 4.0 有了巨大的变化，不仅是受 swift 3.0 大升级的影响，RAC 对自身项目结构的也进行了大幅度的调整。这个调整就是将 RAC 拆分为四个库：**ReactiveCocoa**, **ReactiveSwift**, **ReactiveObjC**, **ReactiveObjCBridge**. 


#### 在项目里现在到底要引入哪些
如果你的项目是纯 OC 项目，你需要使用的是 ReactiveObjC 。这个库里面包含原来 RAC 2 的全部代码。
如果你只是纯 swift 项目，你继续使用ReactiveCocoa 。但是 RAC 依赖于 ReactiveSwift ，等于你引入了两个库。
如果你的项目是 swift 和 OC 混编，你需要同时引用 ReactiveCocoa 和 ReactiveObjCBridge 。但是 ReactiveObjCBridge 依赖于 ReactiveObjC ，所以你就等于引入了 4 个库。

-------

### ReactiveCocoa 试图解决什么问题

<!-- more -->

1. 传统 iOS 开发过程中，状态以及状态之间依赖过多的问题
2. 传统 MVC 架构的问题：Controller 比较复杂，可测试性差
3. 提供统一的消息传递机制

#### 传统 iOS 开发过程中，状态以及状态之间依赖过多的问题
我们在开发 iOS 应用时，一个界面元素的状态很可能受多个其它界面元素或后台状态的影响。
例如，在用户帐户的登录界面，通常会有 2 个输入框（分别输入帐号和密码）和一个登录按钮。如果我们要加入一个限制条件：当用户输入完帐号和密码，并且登录的网络请求还未发出时，确定按钮才可以点击。通常情况下，我们需要监听这两个输入框的状态变化以及登录的网络请求状态，然后修改另一个控件的`enabled`状态。
RAC 通过引入信号（Signal）的概念，来代替传统 iOS 开发中对于控件状态变化检查的代理（delegate）模式或 target-action 模式。因为 RAC 的信号是可以组合（combine）的，所以可以轻松地构造出另一个新的信号出来，然后将按钮的enabled状态与新的信号绑定。

    RAC(self.loginBtn,enabled) = [RACSignal combineLatest:@[self.nameTF.rac_textSignal,
                                                            self.passwordTF.rac_textSignal
                                                            ]
                                                    reduce:^(NSString *nameSignal,NSString *pwdSignal){
                                                                return @(nameSignal.length>=0 && pwdSignal.length>=0);
                                                            }];

简单的解释一下代码:
这是将loginBtn的enable属性和帐号和密码两个输入框绑定,当两个输入框的文本都不为空的时候,loginBtn才可以点击. 注意一点,reduce后面的block并不会自动生成所有的返回值,需要根据自己在前面绑定的几个信号自己补全,然后直到这部分代码完全写完中间可能一直在报错,不要理他.从RAC的源码中可以看出来,前面绑定的信号不同后面的block返回值类型也是不同的.


#### 统一消息传递机制
iOS 开发中有着各种消息传递机制，包括 KVO、Notification、delegation、block 以及 target-action 方式。各种消息传递机制使得开发者在做具体选择时感到困惑. 在引入 RAC 之后，以前散落在action-target或 KVO 的回调函数中的判断逻辑被统一到了一起.

    // KVO
    [RACObserve(self, username) subscribeNext:^(id x) {
        NSLog(@" 成员变量 username 被修改成了：%@", x);
    }];
    // target-action
    self.button.rac_command = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
        NSLog(@" 按钮被点击 ");
        return [RACSignal empty];
    }];
    // Notification
    [[[NSNotificationCenter defaultCenter] 
        rac_addObserverForName:UIKeyboardDidChangeFrameNotification         
                        object:nil] 
        subscribeNext:^(id x) {
            NSLog(@" 键盘 Frame 改变 ");
        }
    ];
    // Delegate
    [[self rac_signalForSelector:@selector(viewWillAppear:)] subscribeNext:^(id x) {
        debugLog(@"viewWillAppear 方法被调用 %@", x);
    }];

**基础用法理解参考下面这篇文章吧,是翻译过来的,写得很棒,例子层层深入,慢慢读,就理解RAC的signal了.不过个人感觉想要深入了解,还是得自己多用.**
[ReactiveCocoa入门教程——第一部](http://benbeng.leanote.com/post/ReactiveCocoaTutorial-part1)

#### 试图解决 MVC 框架的问题
对于传统的 Model-View-Controller 的框架，Controller 很容易变得比较庞大和复杂。由于 Controller 承担了 Model 和 View 之间的桥梁作用，所以 Controller 常常与对应的 View 和 Model 的耦合度非常高，这同时也造成对其做单元测试非常不容易，对 iOS 工程的单元测试大多都只在一些工具类或与界面无关的逻辑类中进行。
RAC 的信号机制很容易将某一个 Model 变量的变化与界面关联，所以非常容易应用 Model-View-ViewModel 框架。通过引入 ViewModel 层，然后用 RAC 将 ViewModel 与 View 关联，View 层的变化可以直接响应 ViewModel 层的变化，这使得 Controller 变得更加简单，由于 View 不再与 Model 绑定，也增加了 View 的可重用性。

**MVVM 的作用和问题**
MVVM 在实际使用中，确实能够使得 Model 层和 View 层解耦，但是如果你需要实现 MVVM 中的双向绑定的话，那么通常就需要引入更多复杂的框架来实现了。
对此，MVVM 的作者 John Gossman 的 批评 应该是最为中肯的。John Gossman 对 MVVM 的批评主要有两点：

第一点：数据绑定使得 Bug 很难被调试。你看到界面异常了，有可能是你 View 的代码有 Bug，也可能是 Model 的代码有问题。数据绑定使得一个位置的 Bug 被快速传递到别的位置，要定位原始出问题的地方就变得不那么容易了。
第二点：对于过大的项目，数据绑定需要花费更多的内存。

附:
之前在上家公司用过一个MBMvc的框架来帮助分层,很好用,不过我看了一下github上已经很久没有更新了,怀疑是不是有什么问题停更了呢, 大家有兴趣的可以了解一下这个框架.

------

摘自:
[唐巧 : ReactiveCocoa - iOS开发的新框架](http://blog.devtang.com/2014/02/11/reactivecocoa-introduction/)




