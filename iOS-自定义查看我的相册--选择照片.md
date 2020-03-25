---
title: iOS:自定义查看我的相册--选择照片
date: 2016-08-21 11:20:24
tags: iOS
---

感觉发微博添加照片的页面很不错,就自己尝试一下. 整体的UI是根据自己的感觉做的,有不足之处,可能不太美观,现阶段以完善功能为重,UI可以后期再进行调整 ~ 

 **图片是直接录制的手机屏幕,因为有隐私所以屏幕都没滑动,也看不到手势具体点的哪里,转的gif图为了小一点,可能效果欠佳,很抱歉**

-------

下面是照片的展示和选择照片时候的一个小动画


![照片选择](http://upload-images.jianshu.io/upload_images/1491333-46755aa4a2857215.gif?imageMogr2/auto-orient/strip)

>我在这里只取到了系统相簿的缩略图,没有使用原图,也没有拿其他自定义相册的数据,不过取原图和其他相册数据的方法都写在代码里面了,只是没有调用.

	#pragma mark 缩略图
	- (void)getThumbnailImages {
	//    // 获得所有的自定义相簿
	//    PHFetchResult<PHAssetCollection *> *assetCollections = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeAlbum subtype:PHAssetCollectionSubtypeAlbumRegular options:nil];
	//    // 遍历所有的自定义相簿
	//    for (PHAssetCollection *assetCollection in assetCollections) {
	//        [self enumerateAssetsInAssetCollection:assetCollection original:NO];
	//    }
	    
	    _thumbnailArr = [[NSMutableArray alloc] init];
	    // 获得系统相册
	    PHAssetCollection *cameraRoll = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeSmartAlbum subtype:PHAssetCollectionSubtypeSmartAlbumUserLibrary options:nil].lastObject;
	    [self enumerateAssetsInAssetCollection:cameraRoll original:NO containerArr:_thumbnailArr];
	    _displayView.photoArr = [[NSMutableArray alloc] initWithArray:_thumbnailArr];
	    [_displayView loadPhotoAlbumView];
	    
	}
	
	
	- (void)getOriginalImages {
	    // 获得所有的自定义相簿
	    PHFetchResult<PHAssetCollection *> *assetCollections = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeAlbum subtype:PHAssetCollectionSubtypeAlbumRegular options:nil];
	    // 遍历所有的自定义相簿
	    for (PHAssetCollection *assetCollection in assetCollections) {
	        [self enumerateAssetsInAssetCollection:assetCollection original:YES  containerArr:nil];
	    }
	    
	    // 获得相机胶卷
	    PHAssetCollection *cameraRoll = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeSmartAlbum subtype:PHAssetCollectionSubtypeSmartAlbumUserLibrary options:nil].lastObject;
	    // 遍历相机胶卷,获取大图
	    [self enumerateAssetsInAssetCollection:cameraRoll original:YES containerArr:nil];
	}
	
	
	- (void)enumerateAssetsInAssetCollection:(PHAssetCollection *)assetCollection original:(BOOL)original containerArr:(NSMutableArray *)containerArr {
	//    NSLog(@"相簿名:%@", assetCollection.localizedTitle);
	    
	    PHImageRequestOptions *options = [[PHImageRequestOptions alloc] init];
	    // 同步获得图片, 只会返回1张图片
	    options.synchronous = YES;
	    
	    // 获得某个相簿中的所有PHAsset对象
	    PHFetchResult<PHAsset *> *assets = [PHAsset fetchAssetsInAssetCollection:assetCollection options:nil];
	    for (PHAsset *asset in assets) {
	        // 是否要原图
	        CGSize size = original ? CGSizeMake(asset.pixelWidth, asset.pixelHeight) : CGSizeZero;
	        
	        // 从asset中获得图片
	        [[PHImageManager defaultManager] requestImageForAsset:asset targetSize:size contentMode:PHImageContentModeDefault options:options resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
	            PhotoModel * photoMod = [[PhotoModel alloc] init];
	            photoMod.displayImg = result;
	            photoMod.isSelected = NO;
	            [containerArr addObject:photoMod];
	        }];
	    }
	}

**这部分代码是从网上借鉴的一篇博客的,不过忘记网址了,特此感谢**

<!-- more -->

整体的的展示使用的collectionView,选中时候的动画使用的POP,选择超过9张时候的提示用的MBProgressHUD.

每张照片的选择与否,选择的是第几张照片,我解决的思路是每一个照片对应一个model, model中有下面几个字段

	@property (nonatomic,strong) UIImage * displayImg;//照片
	@property (nonatomic,assign) BOOL isSelected;//是否已选
	@property (nonatomic,assign) NSInteger photoIndex;//位置

通过model来存储选择的状态信息.

选完之后要把照片传出来 ,如图:

![选中的图片传出](http://upload-images.jianshu.io/upload_images/1491333-fc92065ec1ed83c8.gif?imageMogr2/auto-orient/strip)


具体的传值block,delegate,notification都有用到.

-------

其他的就是一些逻辑代码了,我就不一一往上贴了,留下个链接,大家可以去下了看一下.
[github点这里哦](https://github.com/Sunxb/PhotoAlbum)


-------

最后说一句,代码里面有一些处理导航条上控件显示的代码,是因为用的系统的导航条,会有一些控件显示还是隐藏的问题,一般自己自定义的viewController和导航来的话,应该不太需要处理.


