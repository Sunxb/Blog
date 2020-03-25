---
title: JSPatch的使用
date: 2016-06-03 14:56:42
tags: [iOS,JSPatch]
---


> JSPatch 是一个 iOS 动态更新框架，只需在项目中引入极小的引擎，就可以使用 JavaScript 调用任何 Objective-C 原生接口，获得脚本语言的优势：为项目动态添加模块，或替换项目原生代码动态修复 bug。

------
**因为还要部署线上的js文件,所以直接用JSPatch SDK,该平台可以帮助管理补丁文件,加密等**
[点击去JSPatch平台注册](http://jspatch.com)
**根据文档的提示把SDK集成进自己的项目,文档讲的很详细,此处略**

------

<!-- more -->

####具体代码
1. 导入  #import <JSPatch/JSPatch.h>

2. 在上线之前需要对脚本进行本地测试，看看运行是否正常。SDK 提供了方法 `+testScriptInBundle` 用于发布前的测试：
注意在 JSPatch 平台的规范里，JS脚本的文件名必须是` main.js`


	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	    [JSPatch testScriptInBundle];//为了线下测试js文件的可用性,线上代码是不同的,下面有写
	 }    
调用这个方法后，JSPatch 会在当前项目的 bundle 里寻找 main.js 文件执行，效果与最终线上用户下载脚本执行一样，测试完后就可以准备上线这个脚本。

![示例图1](http://upload-images.jianshu.io/upload_images/1491333-9946ff3f0d2171d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#下面是重点 -- js文件中写什么
举个例子吧,下面是我的某一个控制中的代码
	#import "JPViewController.h"
	
	@interface JPViewController ()
	{
	 
	}
	@end
	
	@implementation JPViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    self.view.backgroundColor = [UIColor whiteColor];
	    [self loadButton];
	    // Do any additional setup after loading the view.
	}
	
	- (void)loadButton {
	     UIButton *tipBtn = [[UIButton alloc] initWithFrame:CGRectMake(10, 50, 200, 30)];
	    [tipBtn setTitleColor:[UIColor redColor] forState:UIControlStateNormal];
	    [tipBtn setTitle:@"hello_jspatch" forState:UIControlStateNormal];
	    [tipBtn addTarget:self action:@selector(clickedBtn:) forControlEvents:UIControlEventTouchUpInside];
	    [self.view addSubview:tipBtn];
	    
	}
	
	- (void)clickedBtn:(UIButton *)sender {
	    sender.backgroundColor = [UIColor redColor];
	}
	
	
	@end
页面上有一个按钮,文字为"hello_jspatch",点击事件是让自己的背景颜色变为红色
**假设这是我线上的版本,但这是有错误的,我的文字不应该是"hello_jspatch",应该是"success_jspatch",点击事件不该是变红而是变绿色**
(这时我们就可以通过JSPatch热更新,具体原理不解释.)
利用运行时特性处理这个控制器中的`loadButton`和`clickedBtn:`这两个方法
**js文件中的语法不会?不要紧,JSPatch的作者还有一个开源项目,直接把我们需要的oc代码转为需要的js**

[这里是链接--JSPatchConvertor](https://github.com/bang590/JSPatchConvertor)

![示例图2](http://upload-images.jianshu.io/upload_images/1491333-84c0b93429a8b64b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**把我们需要更改的两处代码, 改为"succcess_jspatch"和greenColor()**

------
##注意
**现在的代码也不一定是一定能用的,因为JSPatch作者对语法做了一些规定,有好多地方这个转换器并不能帮助完美的转换**
**这就要去github中看具体的规定**
[点此跳转到github--JSPatch-Wiki](https://github.com/bang590/JSPatch/wiki)

####下面附上修改完的代码
	require('UIButton,UIColor');
	defineClass('JPViewController', {
	    loadButton: function() {
	        var tipBtn = UIButton.alloc().initWithFrame({x:10, y:50, width:200, height:30});
	        tipBtn.setTitleColor_forState(UIColor.redColor(), 0);
	        tipBtn.setTitle_forState("success_jspatch", 0);
	        tipBtn.addTarget_action_forControlEvents(self, "clickedBtn:", 1<<6);
	        self.view().addSubview(tipBtn);
	
	    },
	    clickedBtn: function(sender) {
	        sender.setBackgroundColor(UIColor.greenColor());
	    }
	});

** 我修改了1. initWithFrame后面的CGRectMake()的样式 2. UIControlStateNormal  UIControlEventTouchUpInside 这类枚举改为对应枚举值   这在[Wiki](https://github.com/bang590/JSPatch/wiki)中都是有提到的,一定要仔细看**

###现在就可以运行了~~~~(但是不要忘了,咱们这是在线下测试呢)
####线上的版本appdelegate中是这样的才对

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {	    
	    [JSPatch startWithAppKey:@"你自己的appkey"];//JSPatch SDK 平台上添加应用得到的key
	    [JSPatch sync];
        //自动去平台下载补丁包
	}

####最后,把那个js文件传到平台就ok了,注意版本号

![示例图3](http://upload-images.jianshu.io/upload_images/1491333-1dfebd07bcfde68d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####注:还有一些安全问题,大家可以根据SDK文档研究一下
####文中有不对的地方希望可以提出,一起进步

