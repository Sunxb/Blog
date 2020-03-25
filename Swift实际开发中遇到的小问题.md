---
title: Swift实际开发中遇到的小问题
date: 2016-12-02 16:49:55
tags: Swift
---

从上周开始, 正式使用Swift语言进行实际项目的开发, 虽然之前Swift的语法已经了解过, 并且写过几个简单的小Demo, 但是在实际应用到了公司项目中还是遇到了一些小问题. 主要是Swift与OC语法对比下的一些用法不同, 还有一些就是混编的问题. 

1. OC项目, 新建Swift文件,没有自动生成桥接文件
  **这个问题只基于本人公司项目的实际情况进行说明.(OC项目添加Swift文件)**
  -  打开工程文件->BuildSetting 检查是否已经存在了Objective-C Bridging Header
![1](http://upload-images.jianshu.io/upload_images/1491333-84a1f4feb5bf82bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)     这个是要导入了桥接文件才会生成的, 所以如果之前你的项目中创建过Swift文件, 也就是生成过了桥接文件, 即时你之后把桥接文件删掉了, 再次导入Swift文件, 它是不会给你重新自动生成桥接文件的.
 
   -  解决:  
    - 把Objective-C Bridging Header 后面对应的路径删除, 然后在重新创建Swift文件, 这时就会提示创建桥接文件了. 
    -  自己手动创建一个桥接文件, 然后手动更改Objective-C Bridging Header后面的为路径为你手动创建的桥接文件的路径.

2. Swift调用OC的Category
  先说一下在混编时OC类和Swift类的互相调用:
  Swift调用OC --- 在创建的桥接文件中导入OC类的.h文件  **#import "xxx.h"**
  OC调用Swift --- 要被OC调用的Swift类要做一个声明,用到`@objc` 
    下面是新建的一个Swift类 , 如果要使得此类能被OC累调用, 需要添加`@objc(TestClass)`(括号内为类名)
       import UIKit
       @objc(TestClass)
       class TestClass: NSObject {
       }
  
  <!-- more -->
  
  **下面说回重点 ** 
  这个问题我用实际代码来说可能会清楚一些.下面是我的一个OC的Category
       //  NSDictionary+Handle.h
       //  Created by sunxb on 16/12/2.
       //  Copyright © 2016年 sunxb. All rights reserved.
       //
       #import <Foundation/Foundation.h>
       @interface NSDictionary (Handle)
       - (void)handleData;
       @end
  这是一个NSDictionary的Category,  按照之前的Swift调用OC类的规则, 我们应该在桥接文件中`#import "NSDictionary+Handle.h"`, 然后编译一下, 然后在Swift中实例一个字典,然后调用. 
  注意: Swift的类型安全问题, 我们在Swift中创建的是Dictionary, 虽然系统在底层做了Dictionary与NSDictionary的桥接, 我们在实际使用时仍要做手动的类型转换, 就像这样`let dict = ["key":"value"] as! NSDictionary` , 如果不强制转为NSDictionary是无法调用OC Category中的方法的. 或者新建一个OC的类, 把Category中的方法在封装一次, Dictionary实例的对象当做参数传进去.
       #import "DealDictionary.h"
       #import "NSDictionary+Handle.h"
       @implementation DealDictionary
       + (void)dealDictionary:(NSDictionary *)dict {
           [dict handleData];
       }
       @end
  直接调用这个类的类方法, 把需要处理的字典直接传进来就ok, 字典也无需做强转, 不过有点麻烦, 建议一些简单的Category就直接用Swift重新写成这个类的extention, 类型转化页ok, 随心情.

3. pod ‘SwiftyJSON’ 遇到的问题
  需要在Podfile文件中添加use_frameworks! ,导入成功后编译项目会报错--.h文件找不到 . (影响了之前导入的三方库的使用)
**原因: cocoapods 里面不使用 use_frameworks!, 则是通过static libraries 这个方式来管理pod的代码. 而如果使用了use_frameworks!, 则cocoapods 使用frameworks 来取代static libraries 方式. **
 **解决方法:** 
 - 全工程里面更改某些库的导入方式, 具体分析是 #import "" #import <>这两种导入方式的哪一种  
 - 不用cocoapods, 直接把项目拖进工程里面. 
  
4. 一些Swift与OC不同的使用方法
 OC:  isKindOfClass --- Swift : 直接用 is
 OC: boolValue等     --- Swift: 先把String转为NSString类型, 在.booValue
------

持续更新中


