---
title: iOS报错:Library not found -lPods-AFNetworking
date: 2016-05-14 17:17:13
tags: iOS
---

#### 每个人出现的此问题的原因不同,直接上我的问题的解决方法

1. 找到Other Linker Flags
![1.jpg](http://upload-images.jianshu.io/upload_images/1491333-09052230fbf772be.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 修改Other Linker Flags
    先删除所有已   **-l"Pods-**   开头的行,然后添加一行** $(inherited) **,并移到最顶部.
    以我们公司的项目为例, 修改完的为:
![2.png](http://upload-images.jianshu.io/upload_images/1491333-5367c74534b7fe22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. clean一下,重新运行



### 如果此方法不能解决,可以点击下面的链接,查看更多的解决方法.
[跳转到Stack Overflow](http://stackoverflow.com/questions/23539147/xcode-ld-library-not-found-for-lpods)



