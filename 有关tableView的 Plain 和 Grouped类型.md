---
title: 有关tableView的 Plain 和 Grouped 类型
date: 2016-04-18 17:25:55
tags: iOS
---

之前封装了一个组件,里面的tableView全部定义的plain类型,新版本迭代需要用到了grouped类型.

plain类型下的组头是固定的,要组头移动就要改为grouped格式.

但是,改为group格式,如果没有设置组头高度,会默认有一个组头的高度,其他只有一个分组的tableView也出现了组头.

<!-- more -->

解决方法:

改为grouped类型后,在代理方法中设置组头的高度,注意,不要设置为0,设置为0.1(我设置为0不管用,0.1起作用了) (组尾应该也是一样的)

    - (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section {
    
      return 0.1;
    
    }


