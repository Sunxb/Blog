---
title: iOS:tableView的类型改为Grouped组头出现默认的高度
date: 2016-05-18 20:24:23
tags: iOS
---


#### 背景介绍:
项目中的一个tableView要添加一个头部视图,组头是要跟着cell一起滚动的...

#### 分析解决问题思路:

1. 首先在代理方法中
 
		-(UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section{
		    
		}
添加组头

2. 组头跟着cell一起滚动,所以tableView的类型改为Grouped类型


#### 出现的问题:
- 类型改为grouped之后组头出现了一段默认的高度,如下图所示

![图片来自网络.jpg](http://upload-images.jianshu.io/upload_images/1491333-a74e3a5ac6dd75f9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!-- more -->

#### 尝试解决:
我从网络上搜的一些办法一般都是针对没有头部视图的,他们只需要实现这个代理方法

	-(CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section{
	    return 0.1;
	}
但是像我们这种项目里面有用到组头的,此方法pass.



还有一种方法需要说一下

     self.tableView.contentInset = UIEdgeInsetsMake(-35, 0, 0, 0); 

这个方法作用只是把tableView整体进行偏移,并不能实质性的解决问题.如果你的tableView有下来加载时的动画,你就会发现这个方法pass

#### 最终解决办法:
**首先要声明这个方法只是相对较好,如果有更好的办法希望分享一下**
    
    //把tableView的类型改回plain类型,然后创建头部视图
    tableView.tableHeaderView = bannerView;//bannerView为创建的头部视图

