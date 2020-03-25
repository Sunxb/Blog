---
title: GCD 知识总结
date: 2018-03-09 17:17:11
tags: [iOS,GCD]
---



(持续更新中~)
#### GCD
GCD中两个非常重要的概念： **任务** 和 **队列**

任务分为**同步执行sync**和**异步执行async**, 同步和异步的区别在于是否会阻塞当前线程, 其实在GCD中一个任务就是一个block中的代码.

队列分为**串行队列**和**并行队列**,主队列dispatch_get_main_queue( )是串行队列.

我们可以使用dispatch_queue_create来创建新的队列,

<!-- more -->

```objc
//串行队列
  dispatch_queue_t queue = dispatch_queue_create("testQueue", NULL);
  dispatch_queue_t queue = dispatch_queue_create("testQueue", DISPATCH_QUEUE_SERIAL);
  //并行队列
  dispatch_queue_t queue = dispatch_queue_create("testQueue", DISPATCH_QUEUE_CONCURRENT);
```

下面是我自己总结的:

/|同步执行sync|异步执行async
----|----|----
串行队列 |当前线程,一个一个执行|其他一个线程, 一个一个执行
并行队列 |当前线程,一个一个执行|其他一个或者多个线程(取决于任务数), 同时执行 

通过代码来验证一下:

1. 创建一个串行的队列添加4个同步执行的任务

  ```
  //串行队列
    dispatch_queue_t queue = dispatch_queue_create("testQueue", DISPATCH_QUEUE_SERIAL); 
    dispatch_sync(queue, ^{
        NSLog(@"111");
        NSLog(@"111中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"222");
        NSLog(@"222中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"333");
        NSLog(@"333中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"444");
        NSLog(@"444中 %@",[NSThread currentThread]);
    });
   ```
    
打印结果:

  ```objc
2018-03-09 15:34:26.474786+0800 Learning[3385:281473] 111
2018-03-09 15:34:26.475053+0800 Learning[3385:281473] 111中 <NSThread: 0x60000007a2c0>{number = 1, name = main}
2018-03-09 15:34:26.475165+0800 Learning[3385:281473] 222
2018-03-09 15:34:26.475369+0800 Learning[3385:281473] 222中 <NSThread: 0x60000007a2c0>{number = 1, name = main}
2018-03-09 15:34:26.475568+0800 Learning[3385:281473] 333
2018-03-09 15:34:26.476057+0800 Learning[3385:281473] 333中 <NSThread: 0x60000007a2c0>{number = 1, name = main}
2018-03-09 15:34:26.476249+0800 Learning[3385:281473] 444
2018-03-09 15:34:26.476351+0800 Learning[3385:281473] 444中 <NSThread: 0x60000007a2c0>{number = 1, name = main}
   ```

2. 同步dispatch_sync改为异步dispatch_async
    打印结果依然是有序, 但是开启了一个子线程

  ```objc
2018-03-09 16:23:47.634329+0800 Learning[3830:331376] 111
2018-03-09 16:23:47.634739+0800 Learning[3830:331376] 111中 <NSThread: 0x604000274dc0>{number = 4, name = (null)}
2018-03-09 16:23:47.634911+0800 Learning[3830:331376] 222
2018-03-09 16:23:47.635262+0800 Learning[3830:331376] 222中 <NSThread: 0x604000274dc0>{number = 4, name = (null)}
2018-03-09 16:23:47.636508+0800 Learning[3830:331376] 333
2018-03-09 16:23:47.637206+0800 Learning[3830:331376] 333中 <NSThread: 0x604000274dc0>{number = 4, name = (null)}
2018-03-09 16:23:47.637413+0800 Learning[3830:331376] 444
2018-03-09 16:23:47.637680+0800 Learning[3830:331376] 444中 <NSThread: 0x604000274dc0>{number = 4, name = (null)}
```
 
3. 如果把新建的队列改为并行队列, 同步执行

  ```objc
    //并行队列
  dispatch_queue_t queue = dispatch_queue_create("testQueue", DISPATCH_QUEUE_CONCURRENT);
  dispatch_sync(queue, ^{
        NSLog(@"111");
        NSLog(@"111中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"222");
        NSLog(@"222中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"333");
        NSLog(@"333中 %@",[NSThread currentThread]);
    });
    dispatch_sync(queue, ^{
        NSLog(@"444");
        NSLog(@"444中 %@",[NSThread currentThread]);
    });
   ``` 
   
 结果
    
   ```objc
    2018-03-09 16:31:50.498599+0800 Learning[3938:341075] 111
2018-03-09 16:31:50.498836+0800 Learning[3938:341075] 111中 <NSThread: 0x600000063d80>{number = 1, name = main}
2018-03-09 16:31:50.498946+0800 Learning[3938:341075] 222
2018-03-09 16:31:50.499101+0800 Learning[3938:341075] 222中 <NSThread: 0x600000063d80>{number = 1, name = main}
2018-03-09 16:31:50.499227+0800 Learning[3938:341075] 333
2018-03-09 16:31:50.499353+0800 Learning[3938:341075] 333中 <NSThread: 0x600000063d80>{number = 1, name = main}
2018-03-09 16:31:50.499461+0800 Learning[3938:341075] 444
2018-03-09 16:31:50.499609+0800 Learning[3938:341075] 444中 <NSThread: 0x600000063d80>{number = 1, name = main}
   ```
4. 把上面代码中的同步改为异步, 即并行队列, 异步执行

  ```objc
2018-03-09 16:34:13.089611+0800 Learning[3982:344351] 222
2018-03-09 16:34:13.089612+0800 Learning[3982:344350] 333
2018-03-09 16:34:13.089612+0800 Learning[3982:344349] 111
2018-03-09 16:34:13.089639+0800 Learning[3982:344352] 444
2018-03-09 16:34:13.089997+0800 Learning[3982:344350] 333中 <NSThread: 0x60000027d680>{number = 3, name = (null)}
2018-03-09 16:34:13.089997+0800 Learning[3982:344349] 111中 <NSThread: 0x604000271800>{number = 4, name = (null)}
2018-03-09 16:34:13.090004+0800 Learning[3982:344352] 444中 <NSThread: 0x604000271440>{number = 6, name = (null)}
2018-03-09 16:34:13.090031+0800 Learning[3982:344351] 222中 <NSThread: 0x604000271480>{number = 5, name = (null)}
  ```
   
 从结果中可以看得出, 开启了四个不同的现成来执行四个任务.
    
----- 
**注意**
在同步+串行的时候会有一个特殊的情况, 上面也提到了, 主队列也是一个串行队列, 如果当前在主线程中且把任务加到主队列中会如何呢 ?  

```objc
NSLog(@"---前 %@",[NSThread currentThread]);
dispatch_sync(dispatch_get_main_queue(), ^{//dispatch_sync
    NSLog(@"---中 %@",[NSThread currentThread]);
});
NSLog(@"---后 %@",[NSThread currentThread]);
```
运行结果形成死锁.
**思考**: 同样都是串行队列, 为什么任务添加到主队列会造成死锁现象, 新建一个串行队列则可以正常执行 ? 
**解释**:  在执行完第一个NSLog的时候, 遇到dispatch_sync, 会暂时阻塞主线程. 那么我们想一下, 在阻塞之前, 主线程正在执行的是主队列中的哪一个任务呢? 用我们实际的例子来解释, 我们再写上面的代码的时候, 这部分的代码是包裹在一个另方法中的, 就像我自己写的demo, 这段代码就写在viewDidLoad里面, 因为viewDidLoad这些任务也是添加在主队列中的, 所以说阻塞主线程时, 主线程应该是处理的主队中 viewDidLoad这个任务. 
我们调用了dispatch_sync后向主队列中添加了新的任务, 也就是dispatch_sync后面block中的代码. 但是根据队列的FIFO规则, 新添加的block中的任务, 肯定是要排在主队列的最后.
因为是使用的dispatch_sync同步, 所以说必须要等执行dispatch_sync的block中的代码才会返回, 才会继续执行ViewDidLoad中dispatch_sync这个方法下面的方法, 也就是代码中的第三个NSLog, 但是block中的任务被放在了主队列的底部, 他是不可能在viewDidLoad这个任务还没完成的时候就执行到的, 所以就形成了两个任务互相等待的情况, 也就是形成了死锁. 
**所以说这样看来,GCD形成死锁的原因应该是是队列阻塞，而不是线程阻塞.**

#### dispatch_group

解决先执行A和B, AB都执行完再执行C的这种情况.

```
dispatch_group_t group = dispatch_group_create();
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_group_async(group, queue, ^{
    // 1
    sleep(5);
    NSLog(@"task 1");
});

dispatch_group_async(group, queue, ^{
    //2
    NSLog(@"task 2");
});

dispatch_group_notify(group, queue, ^{
        NSLog(@"done ");
    });
```

或者

```
dispatch_group_t group = dispatch_group_create();
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

dispatch_group_enter(group);
dispatch_async(queue, ^{
    NSLog(@"11");
    dispatch_group_leave(group);
});

dispatch_group_enter(group);
dispatch_async(queue, ^{
    sleep(7);
    NSLog(@"22");
dispatch_group_leave(group);
});

dispatch_group_enter(group);
dispatch_async(queue, ^{
    NSLog(@"33");
    dispatch_group_leave(group);
});

dispatch_group_notify(group, queue, ^{
    NSLog(@"done");
});
```

结果都是先执行完上面的在打印这个done, 这个不需要解释了.

有时候可能会碰到dispatch_group_wait, 注意一下, 因为他会阻塞线程, 所以不能放在主线程里面执行, 等待的时间是以group中任务开始执行的时间算起.

#### dispatch_barrier_async
多线程最常见的问题就是读写，比如数据库读写，文件读写，读取是共享的，写是互斥. 

使用Concurrent Dispatch Queue和dispatch_barrier_basync函数可实现高效率的数据库访问和文件访问。
如果允许多个线程进行读操作，当写文件时，阻止队列中所有其他的线程进入，直到文件写完成.

在读取文件时, 直接使用dispatch_async异步读取, 当要写文件是, 使用dispatch_barrier_async. 用代码看一下dispatch_async和dispatch_barrier_async的执行顺序. 

```objc
 dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"task 1");
    });

    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"task 2");
    });
    
    dispatch_barrier_async(queue, ^{
        NSLog(@"barrier -- ");
        sleep(3);
    });
    
    dispatch_async(queue, ^{
        NSLog(@"task 3");
    });
    
    dispatch_async(queue, ^{
        NSLog(@"task 4");
    });
    
    NSLog(@" end ");
```

结果是先打印完task1和2, 在打印完barrier, 最后打印task3和4, 同理到文件读写中, 文件读取直接用dispatch_async, 写入使用dispatch_barrier_async, 则在写入文件时, 不管前面有多少操作都会等待前面的操作完成, 而且在写入的时候, 也没有其他线程访问, 从而达到线程安全. 



