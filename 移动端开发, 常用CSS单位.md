---
title: 移动端开发,常用CSS单位
date: 2017-11-09 10:57:09
tags: H5
---
(转)https://www.cnblogs.com/mylove103104/archive/2015/06/18/4584779.html

1. rem
"em" 单位是我们开发中比较常用到的，它表示以当前元素的父元素的单位大小为基准来设置当前元素的大小；“rem” 中的 “r” 代表 “root”，它表示以根（即“html”）元素的单位大小为基准来设置当前元素的单位大小，所以不管当前元素是任意子节点，一旦设单位大小为 “rem” 那么这个元素大小都是以根元素单位为参考的，这里的“em” 和 “rem” 均具有继承性。

<!-- more -->

![image](http://upload-images.jianshu.io/upload_images/1491333-ffe0f1e15a520212.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.vw和vh（移动端开发个人最喜欢的单位属性，也是这次介绍的重点）
传统的响应式开发中，我们常常用百分比来布局，然而这并不是最好的解决方案。例如，你没有办法以body的高度来设置百分比。
"vw" 的全称是 “viewport width” 即视窗的宽度；"vh" 的全称是 “viewport height” 即视窗的高度。
1vw = viewportWidth * 1/100； 1vh = viewportHeight * 1/100；
所以元素使用 “vw” “vh” 作为宽度和高度单位，即可以保证适配不同的设备。

![image](http://upload-images.jianshu.io/upload_images/1491333-4943404717d98db9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](http://upload-images.jianshu.io/upload_images/1491333-e909f6a4af915851.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3\. vmin 和 vmax“vmin” 即 “viewport” 宽度和高度相比较最小的那一个。（也就是说，如果当前元素单位设置了 “vmin” 那么浏览器会去判断宽度和高度的大小，然后继承小的值）“vmax” 同理，继承宽高比较，大的那一个值；即，宽和高谁大，就继承谁的值。这里我们假设：浏览器的宽度为1300px，高度为960px；
50vmin = 960 * (50/100)；
50vmax = 1300 * (50/100)；

![image](http://upload-images.jianshu.io/upload_images/1491333-ab0b24bc46b9316c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](http://upload-images.jianshu.io/upload_images/1491333-f6bce734e51122e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


