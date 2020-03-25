---
title: iOS:断点下载
date: 2017-04-24 14:43:54
tags: iOS
---


最近想到断点下载这个技术点, 然后学习尝试了一下,下面分享自己的一些心得.

上网找了很多demo, 大多用的是老版本的AFN, 很少有基于3.x版本的AFN来实现的, 有一些3.X版本的实现也是做得在理想状态下的断点下载, 没有做杀死进程这类特殊情况的处理.

单纯的在理想状态下的断点下载是很容易实现的, 利用AFNetworking抛出的接口就可以实现. 

    NSURLSessionDownloadTask * task = [[AFHTTPSessionManager manager] downloadTaskWithRequest:request progress:^(NSProgress * _Nonnull downloadProgress) {
    
    //        NSLog(@"完成: %lld------ 一共: %lld",downloadProgress.completedUnitCount/1024,downloadProgress.totalUnitCount/1024);
        } destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
            // 制定下载完成后的路径
            return [NSURL fileURLWithPath:filePath];
            
        } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
            NSLog(@"======================\nreponse: %@ \n=======================\nerror: %@ \n=================\n path: %@",response,[error.userInfo objectForKey:NSURLSessionDownloadTaskResumeData],filePath);
        }];
        
        [task resume];


封装一个方法, 将这个task返回, 然后对这个task进行操作, 即可实现暂停或者继续. 

AFN在下载一个文件的时候, 会先在沙盒的`tmp`文件中生成一个临时文件, 在完全下载完毕, 才会将文件移动到你指定的文件夹内.(下载成功后的返回路径就是你设置的路径) 

<!-- more -->

**一点思考**
用上面的方法, 我在下载的过程中将任务暂停, 过一会之后会提示失败,error之后就不能再继续了, 可能是我网络问题, 或者别的问题, 现在还没思考出来, 所以感觉这个不太适合做断点续传,  我查了一下, 有一篇文章中说到, 在过程中提示error之后, 其实已经下载过的数据是可以得到的, `[error.userInfo objectForKey:NSURLSessionDownloadTaskResumeData]`就可以拿到data数据, 然后可以结合`NSURLSession`类的`downloadTaskWithResumeData`方法 (这个方法描述的第一句话就是*Creates a download task to resume a previously canceled or failed download.*) 来实现断点下载, 这可能是一个思路. 


-------
### 使用系统原生类实现断点下载
简单的说几句咱们原生的类:**NSURLSession**

`NSURLSession` 指的也不仅是同名类 `NSURLSession`，还包括一系列相互关联的类。`NSURLSession` 包括了与之前相同的组件，`NSURLRequest` 与 `NSURLCache`，以及 `NSURLSession`、`NSURLSessionConfiguration` 以及 `NSURLSessionTask` 的 3 个子类：`NSURLSessionDataTask`，`NSURLSessionUploadTask`，`NSURLSessionDownloadTask`。

`NSURLsessionTask` 是一个抽象类，其下有 3 个实体子类可以直接使用：`NSURLSessionDataTask`、`NSURLSessionUploadTask`、`NSURLSessionDownloadTask`。这 3 个子类封装了现代程序三个最基本的网络任务：获取数据，比如 JSON 或者 XML，上传文件和下载文件。

### 参考demo
![demo](http://upload-images.jianshu.io/upload_images/1491333-e0f74be3b4091ed7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
朋友发给我的一个网上的demo, 写的很不错, 满足了断点下载该有的要求. 具体的代码我就不往文章里面粘了, 文件我会直接放到百度云盘, 大家可以下载来看看代码,了解一下思路.(链接在文末)

其实主要的实现很简单, 下面是几个代码段.

首先就是根据已经下载的大小来设置请求头

    NSMutableURLRequest *requestM = [NSMutableURLRequest requestWithURL:URL];
        // 设置请求头
    [requestM setValue:[NSString stringWithFormat:@"bytes=%lld-", (long long int)[self hasDownloadedLength:URL]] forHTTPHeaderField:@"Range"];
        NSURLSessionDataTask *dataTask = [self.urlSession dataTaskWithRequest:requestM];

再就是这几个代理方法

1. 刚开始收到数据的时候调用
`- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSHTTPURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler`

2.  收到数据调用, 调用多次
`- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data`

3.  完成时调用
`- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error`


上面这两部分是实现下载的部分, 其他一些代码就是辅助实现断点下载这一特性的, 这份代码的作者下载类抛出的方法和属性很具体, 包括开始暂停, 全部开始全部暂停,   删除, 可同时下载数, 这部分很代码麻烦但也是逻辑重点. 

-------

[demo -- demo -- demo  下载](https://pan.baidu.com/s/1nvqKJkH) 
**提取码:vc9n**

------
附:其他一些用到的类

NSInputStream 与 NSOutputStream 都继承于 NSStream, NSStream 是一个抽象的基类, 规定了Stream共有的一些行为…

什么是Stream

Stream翻译成为流，它是对我们读写文件的一个抽象。 你可以这样想象，当你读文件和写文件的时候，文件的内容就像水流一样向你流过来或者流给别人。 而Stream就帮我们做了这样的事情， 实际上，它是把文件的内容，一小段一小段的读出或写入，来到达这样的效果

NSStream

NSStream 是Cocoa平台下对流这个概念的实现类， NSInputStream 和 NSOutputStream 则是它的两个子类，分别对应了读文件和 写文件。



