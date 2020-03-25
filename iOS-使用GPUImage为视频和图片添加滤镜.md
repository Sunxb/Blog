---
title: iOS使用GPUImage为视频和图片添加滤镜
date: 2018-07-10 18:35:17
tags: [iOS,音视频]
---


#### 前言

这次的文章是我对滤镜效果一个学习，文章的文字比较少，花的主要功夫其实都在代码里面。demo的链接在下文已经给出了。

#### 使用场景

下面是我们常用的使用滤镜的场景

1. 相机录像添加实时滤镜
2. 相册内视频添加滤镜处理
3. 相机拍照添加实时滤镜
4. 给已有的图片/照片添加滤镜

#### 滤镜实现方案

我简单的查了一下，可以使用CoreImage，也可以使用GPUImage来实现。
我这次主要说的是是使用GPUImage来实现滤镜效果。

#### GPUImage

GPUImage这个框架有多强大我就不再赘述了，想必大家也了解到了，毕竟这是当前做滤镜此类功能最流行的框架。

我们从一个例子来入手吧，下面是给已有图片添加滤镜的例子。大家可以先看一眼这个图。
![流程](https://raw.githubusercontent.com/Sunxb/blog_img/master/03-iOS%E4%B8%BA%E8%A7%86%E9%A2%91%E5%92%8C%E5%9B%BE%E7%89%87%E6%B7%BB%E5%8A%A0%E6%BB%A4%E9%95%9C/1.png)

这张图呢，是给已有的图片添加滤镜效果的一个大概流程图，我们在拿到一个原始的图片，先用它来生成一个GPUImagePicture，然后创建我们需要的滤镜，然后用滤镜来处理我们GPUImagePicture，最后，可以从这个filter类中取到处理后的图片。就这么简单

我简单的写了一下代码，大家请看

<!-- more -->

```objc
// 原图
    UIImage *inputImage = [UIImage imageNamed:@"WID-small.jpg"];
    
    // 生成GPUImagePicture
    _sourcePicture = [[GPUImagePicture alloc] initWithImage:inputImage smoothlyScaleOutput:YES];
    
    // 随便用一个滤镜
    _sepiaFilter = [[GPUImageTiltShiftFilter alloc] init];
    
    // 如果要显示话,得创建一个GPUImageView来进行显示
    GPUImageView * imageView = [[GPUImageView alloc] initWithFrame:self.view.bounds];
    self.view = imageView;
    
    //
    [_sepiaFilter forceProcessingAtSize:imageView.sizeInPixels];
 
    // 个人理解,这个add其实就是把_sourcePicture给_sepiaFilter来处理
    [_sourcePicture addTarget:_sepiaFilter];
    // 用这个imageView来显示_sepiaFilter处理的效果
    [_sepiaFilter addTarget:imageView];
    
    // 开始!
    [_sourcePicture processImage];
```

一个滤镜效果就这样简单的实现了。

其他的场景跟这种情况的原理也是相同的，整个添加滤镜的过程大家可以理解成分为三部分。

第一部分，就是输入的数据，不管是实时的数据，还是已经存在的数据，都算作输入。

第二部分就是滤镜，GPUImage这个框架给我们提供了大量的滤镜效果了，我们可以根据我们的需要选择使用，如果所有的不满足，滤镜还可以自定义。滤镜拿到我们的输入的原始数据进行处理。

第三部分就是输出，我们在做滤镜的效果时，肯定是要让用户能看到添加了滤镜之后的效果是什么样子的，所以我们需要显示处理后的数据。

这三部分的实现GPUImage为我们封装好了对应的类，我们只需要按需使用就可以。

#### DEMO

我参考了网上的文章和GPUImage的官方demo之后，把几种常用场景的是实现汇总了一下。

[大家可以去下载这个代码，算是GPUImage使用的简单入门。](https://github.com/Sunxb/GPUImageDemo)

https://github.com/Sunxb/GPUImageDemo

demo里面包括我们这这常用场景的实现，然后我还写了混合滤镜的使用，还有一些其他细节的东西。

![实现的效果](https://raw.githubusercontent.com/Sunxb/blog_img/master/03-iOS%E4%B8%BA%E8%A7%86%E9%A2%91%E5%92%8C%E5%9B%BE%E7%89%87%E6%B7%BB%E5%8A%A0%E6%BB%A4%E9%95%9C/2.png)



#### 结束语
GPUImage入门很简单，但毕竟这么强大的框架，要学透还是要我们在日常的使用中继续的探索了。一起加油。


