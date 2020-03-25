---
title: iOS:CoreLocation实现定位当前城市
date: 2016-05-09 15:01:02
tags: iOS
---


#### 首先导入头文件
    #import <CoreLocation/CoreLocation.h>

#### 在info.plist文件中添加:
![](http://upload-images.jianshu.io/upload_images/1491333-6c102cc631be5136?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注:NSLocationAlwaysUsageDescription可以不添加**

<!-- more -->

#### 下面是具体用法的demo:
**注意**:
1.[CLLocationManager locationServicesEnabled]判断定位是否开启是判断的整个手机系统的定位是否打开,并不是针对这一应用.
2.具体在本应用中的是否已经授权定位要通过代理方法判断
如果定位失败(未授权)则会执行此代理方法

	- (void)locationManager:(CLLocationManager *)manager didFailWithError:(NSError *)error {
	
	}

###### 具体代码:

	@interface ViewController ()<CLLocationManagerDelegate>
	{
	    CLLocationManager * locationManager;
	    NSString * currentCity; //当前城市
	}
	@end
	@implementation ViewController
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    [self locate]; 
	}
	
	- (void)locate { 
         //判断定位功能是否打开
	    if ([CLLocationManager locationServicesEnabled]) {
	        locationManager = [[CLLocationManager alloc] init];
	        locationManager.delegate = self;
	//        [locationManager requestAlwaysAuthorization];
	        currentCity = [[NSString alloc] init];
	        [locationManager startUpdatingLocation];    
	    }
	    
	}
	
	#pragma mark CoreLocation delegate	

	//定位失败则执行此代理方法
    //定位失败弹出提示框,点击"打开定位"按钮,会打开系统的设置,提示打开定位服务
	- (void)locationManager:(CLLocationManager *)manager didFailWithError:(NSError *)error {
	    UIAlertController * alertVC = [UIAlertController alertControllerWithTitle:@"允许\"定位\"提示" message:@"请在设置中打开定位" preferredStyle:UIAlertControllerStyleAlert];
	UIAlertAction * ok = [UIAlertAction actionWithTitle:@"打开定位" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
	        //打开定位设置
	        NSURL *settingsURL = [NSURL URLWithString:UIApplicationOpenSettingsURLString];
	        [[UIApplication sharedApplication] openURL:settingsURL];
	    }];
	    UIAlertAction * cancel = [UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
	        
	    }];
	    [alertVC addAction:cancel];
	    [alertVC addAction:ok];
	    [self presentViewController:alertVC animated:YES completion:nil];
	    
	}
    //定位成功
	- (void)locationManager:(CLLocationManager *)manager didUpdateLocations:(NSArray<CLLocation *> *)locations {
	    [locationManager stopUpdatingLocation];
	    CLLocation *currentLocation = [locations lastObject];
	    CLGeocoder * geoCoder = [[CLGeocoder alloc] init];
	    
          //反编码
	    [geoCoder reverseGeocodeLocation:currentLocation completionHandler:^(NSArray<CLPlacemark *> * _Nullable placemarks, NSError * _Nullable error) {     
	        if (placemarks.count > 0) {
	            CLPlacemark *placeMark = placemarks[0];
	            currentCity = placeMark.locality;  
	            if (!currentCity) {
	                currentCity = @"无法定位当前城市";
	            } 
	            NSLog(@"%@",currentCity); //这就是当前的城市
	            NSLog(@"%@",placeMark.name);//具体地址:  xx市xx区xx街道
	        }
	        else if (error == nil && placemarks.count == 0) {
	            NSLog(@"No location and error return");
	        }
	        else if (error) {
	            NSLog(@"location error: %@ ",error);
	        }
      
	    }];	    
	}
	@end


