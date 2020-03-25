---
title: 一个有关tableViewCell的复用问题
date: 2016-04-22 18:24:10
tags: iOS
---

背景:

  tableview有两个分组,两个分组中的cell里面控件布局不同....

   手写代码布局cell...

问题:

手写代码的cell复用,上面添加的控件没有移除,会出现重叠.而且最开始用了一个复用ID,也就是默认了整个tableview是一类的cell.所以在页面中,尤其是复用了cell的时候,两种cell 会混乱..

<!-- more -->

尝试办法一:(没起作用)

把布局cell的子控件的代码写到if(!cell){}方法外面 ,也就是每次加载cell的时候,不管是否存在可以服用的cell,都重新加载cell内部的控件布局

结果就是造成cell上的控件重复添加,比如文字字体越来越粗等..

然后我就没继续尝试这个方法,估计在每次加载cell的时候先把cell(也就是cell的contentView的subViews)上面的控件清空应该可以奏效,但是遍历的话会卡顿..过意直接放弃

尝试方法二:(解决了自己的问题)

把cell分类,section = 0 或者section = 1;分别为cell设置不同的复用ID,

这就表明了两个组的cell是不同类型的,不管是复用或者是新建,都根据自己的类型来加载,所以就解决了问题

    UITableViewCell * cell;
    
    switch (indexPath.section) {
    
    case 0:
    
    cell = [tableView dequeueReusableCellWithIdentifier:Identifier3];
    
    break;
    
    case 1:
    
    cell = [tableView dequeueReusableCellWithIdentifier:Identifier4];
    
    break;
    
    default:
    
    break;
    
    }

基本就是这个意思了 ....

