---
title: tableView下拉头部放大
date: 2016-07-04 17:14:39
tags: iOS
---

这个效果很多见得, 想自己实现一下试试
![效果.gif](http://upload-images.jianshu.io/upload_images/1491333-550260b530e5b44b.gif?imageMogr2/auto-orient/strip)


**主要的就是一个计算问题**

#### 直接附上代码
	#import "ViewController.h"
	
	#define KScreen_Width [UIScreen mainScreen].bounds.size.width
	#define KScreen_Height [UIScreen mainScreen].bounds.size.height
	#define imgHeight 200
	
	@interface ViewController ()<UITableViewDelegate,UITableViewDataSource,UIScrollViewDelegate>
	{
	    UITableView * rootTableView;
	    UIImageView * headerImg;
	}
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    	    
	    rootTableView = [[UITableView alloc] initWithFrame:CGRectMake(0, 0, KScreen_Width, KScreen_Height)];
	    rootTableView.delegate = self;
	    rootTableView.dataSource = self;
	    rootTableView.contentInset = UIEdgeInsetsMake(imgHeight, 0, 0, 0);
	    [self.view addSubview:rootTableView];
	    
	    headerImg = [[UIImageView alloc] initWithFrame:CGRectMake(0, -imgHeight, KScreen_Width, imgHeight)];
	    headerImg.image = [UIImage imageNamed:@"1.jpg"];
	    headerImg.contentMode = UIViewContentModeScaleAspectFill;// !!
	    [rootTableView addSubview:headerImg];
	    
	    // Do any additional setup after loading the view, typically from a nib.
	}
	
	- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
	    return 40;
	}
	
	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	    static NSString * ID = @"cell";
	    UITableViewCell * cell = [tableView dequeueReusableCellWithIdentifier:ID];
	    if (!cell) {
	        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:ID];
	    }
	    cell.textLabel.text = [NSString stringWithFormat:@"数据---%ld",indexPath.row];
	    return cell;
	}
		
	- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
	    CGFloat offsetY = scrollView.contentOffset.y;
	    NSLog(@"%f",offsetY);
	    CGFloat offsetH = imgHeight + offsetY;// offsetH是相对的偏移量(偏移量应该是scrollView.contentOffset.y 但是rootTableView.contentInset = UIEdgeInsetsMake(imgHeight, 0, 0, 0)  tableView提前设置便宜了一段距离  所以现在相对的偏移量应该是offsetH)
	    if (offsetH < 0) {//下拉偏移为负数
	        CGRect frame = headerImg.frame;
	        frame.size.height = imgHeight - offsetH;//下拉后图片的高度应变大
	        frame.origin.y = -imgHeight + offsetH;// 下边界是一定的  高度变大了  起始的Y应该上移
	        headerImg.frame = frame;
	    }
	}
	
	- (void)didReceiveMemoryWarning {
	    [super didReceiveMemoryWarning];
	    // Dispose of any resources that can be recreated.
	}
	
	@end



#### 最重要的部分就是下面这段计算
	- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
	    CGFloat offsetY = scrollView.contentOffset.y;
	    NSLog(@"%f",offsetY);
	    CGFloat offsetH = imgHeight + offsetY;// offsetH是相对的偏移量(偏移量应该是scrollView.contentOffset.y 但是rootTableView.contentInset = UIEdgeInsetsMake(imgHeight, 0, 0, 0)  tableView提前设置偏移了一段距离  所以现在相对的偏移量应该是offsetH)
	    if (offsetH < 0) {//下拉偏移为负数
	        CGRect frame = headerImg.frame;
	        frame.size.height = imgHeight - offsetH;//下拉后图片的高度应变大
	        frame.origin.y = -imgHeight + offsetH;// 下边界是一定的  高度变大了  起始的Y应该上移
	        headerImg.frame = frame;
	    }
	}
**具体的分析在代码后面已经做了注释**

#####当然了,还有关于tableView的一些设置,比如
	rootTableView.contentInset = UIEdgeInsetsMake(imgHeight, 0, 0, 0);

##### 还有头部图片的设置
	headerImg.contentMode = UIViewContentModeScaleAspectFill;// !!

**注:图片是直接加在tableview上面的,并不是tableView的header**

