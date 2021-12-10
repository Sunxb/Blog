#### 边下边播总结(一)

### 概述

 最近修改了项目中的视频播放功能, 由之前的全量下载完再播, 改为了边下边播的方式. 由于我们项目中的视频在发出时都进行了加密, 所以整个过程其实就是边下载边解密边播放.

 边下边播的技术方案, 网上的博客很容易搜到, 不外乎两种方式, `内置本地代理服务器`和`AVAssetResourceLoader`. 我们采取了系统提供的`AVAssetResourceLoader`这一方案.

### 方案原理

 具体的`AVAssetResourceLoader`实现原理网上可以找到很多逻辑图, 如下图(来自网络)所示.
 
 ![image-1](./Img/2021/12/10-1.png)

 这里结合我们的实际代码简单的介绍一个这个图片.

 在平时使用AVPlayer播放url时, 我们会这样创建一个播放器(简略)

 ```swift
let videoAsset = AVURLAsset(url: "http://resource_url/xxxxx")
let item = AVPlayerItem(asset: videoAsset)
let player = AVPlayer(playerItem: item)
 ```
 如果我们这样设置播放, 整个播放的内部流程其实都我们都是不可见的, 视频的下载和缓存等, 我们只能通过已知的一些方法,来控制播放器的播放暂停等. 

 如果想要实现我们项目中想要的效果, 边下载边播放, 同时, 我们可能需要接手视频的缓存这一模块, 所以我们就必须得能进入到整个播放流程中, `AVAssetResourceLoader`其实就算是苹果给我们留的一个小口子, 然后通过设置遵守`AVAssetResourceLoaderDelegate`这一协议的代理对象, 接手数据处理的这一过程(包括获取数据和向播放器填充数据).

 ```swift
 videoAsset.resourceLoader.setDelegate(self, queue: queue)
 ```

**注意事项**

1. 要进入到 `AVAssetResourceLoader`的代理回调, 除了要给videoAsset.resourceLoader设置delegate之外, 还需要把我们的url改为不能识别的scheme. 我们一遍的资源路径都是http或者https, 我们需要把url的scheme改为不能识别的(私有的), 比如`http://resource/xxx/xxx.mp4`改为`http-prefix://reource/xxxx/xxx.mp4`
2. url路径的最后必须要有视频的后缀, 类似.mp4, 我之前使用的资源路径是没有后缀的, 导致了播放器无法起播.

### AVAssetResourceLoaderDelegate

`AVAssetResourceLoaderDelegate`有两个常用的回调方法如下

```swift
// MARK: - AVAssetResourceLoaderDelegate
func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {}

func resourceLoader(_ resourceLoader: AVAssetResourceLoader, didCancel loadingRequest: AVAssetResourceLoadingRequest) {}
```

当播放器开始播放的时候, 会通过`shouldWaitForLoadingOfRequestedResource`这个回调方法向我们索要数据, 具体所要数据的信息细节都封装在`loadingRequest`里面. 
因为这个回调会走很多次, 上图中表示的是要保存起来每一次的loadingRequest, 但在实际项目中, 我使用了不太一样的策略, 我把每一次loadingRequest都对应一个worker对象来处理, 这样每次索要数据, 都有一个单独的worker来处理相对应的网络请求(暂不考虑缓存), 这样比较条理. 同时我们也需要保存起我们的worker, 因为如果播放器需要支持进度条拖动时, 需要手动seek到某一个位置, 这样会触发`didCancel`这个回调, 所以我们也需要把我们对应的worker内部停掉.

 ![image-2](./Img/2021/12/10-2.png)

### 回调处理

当我们收到一个回调时, 我们主要关注这个AVAssetResourceLoadingRequest类型的loadingRequest.

他内部有一个dataRequest属性, dataRequest中有requestedOffset, requestedLength等一些有用信息. 我们通过requestedOffset和requestedLength构建出我们的Range, 塞到请求头里面去, 获取相应range的数据.

当我们的player开始播放时, 收到的第一个回调, requestedOffset=0,requestedLength=2, 也就是索要0-1这两个字节, 这次请求其实可以理解为一个嗅探请求, 目的是为了得到视频的相关信息, 文件大小, 类型等.

```swift
guard request.contentInformationRequest == nil else {
    if request.dataRequest?.requestsAllDataToEndOfResource == false {
        request.contentInformationRequest?.contentLength = totalLen
    } else {
        request.contentInformationRequest?.contentLength = Int64(data.count)
   }
   request.contentInformationRequest?.isByteRangeAccessSupported = true
   request.contentInformationRequest?.contentType = "video/mp4"
   request.finishLoading()
   return
}
```

上述代码就是第一个嗅探请求的处理方式, 通过`request.contentInformationRequest==nil`, 判断出是第一个嗅探请求, 然后我们需要填充request的contentInformationRequest, 然后填充信息结束调用`finishLoading()`, 当前的loadingRequest就结束了.

第一个嗅探请求结束后, 如果我们返回的没有问题, 那播放器会立刻进行下一个回调, 开始所要视频数据, 我在项目中测试时, 第二个请求一般都是0-xxx(文件大小-1), 索要整个文件, 这时我们dataTask类型的请求,等待服务器一片一片的返回数据, 没收到一部分数据后调用`dataRequest.respond(with: data)`, 全部收取完毕之后调用`request.finishLoading()`.

其实这就是最基本的数据填充的逻辑, 除了第一个嗅探请求特殊处理一下, 后面的就是收到数据, 就填充回dataRequest, 索要的数据全部填充完毕, 调用finishLoading. 

在我们请求真个文件的过程中, 有时候会发现一种现象, 就是respond一部分数据之后, loadingRequest被cancel了, 然后又开始索要很后面的range的数据, 其实这可以理解为一个寻找文件的moov的过程, 文件的moov可能在文件头, 也可能在文件尾部.moov里面定义视频的时间尺度,时长,显示特性以及每个轨道信息等, 这一部分可以通过了解mp4文件头格式来多做一下了解.

我们不管他索要的是那一部分数据, 只要我们请求到对应的数据, respond回去就没问题.

**补充**

那这么简单的逻辑对于我们自己的项目来说难点是什么呢,这里简单描述一下. 

前面有说到我们项目中的资源都是经过加密的, 使用了AES的加密算法, 这样我们在接受到数据之后, 是不能直接返回给`dataRequest`的, 需要我们先解密, 然后简单的说我们使用的加密策略是每16字节是一个加密片段, 但请求返回的数据并不能保证每次都是16倍数, 所以我们处理16的倍数才能进行解密这一个问题, 然后还有一个range的修正问题, 打比方我们需要1-10这10个字节的数据, 但是我请求头的range是不能直接写1-10的, 因为按照我们每16个字节是一个加密片段, 我们需要的1-10, 在0-15这个片段中, 所以我们必须要先请求下来0-15这一个片段, 然后解密, 再从中拿出1-10, 填充回去. 当然了还有一些细节就不展开叙述了, 等有机会集合项目单独聊一聊AES这个解密方法.

### 总结
上面就是在实现边下边播过程中总结到的一些小点, 当然每个人在实际项目可能会遇到不一样的问题. 同时本文没有涉及到数据的缓存, github上也有很多不错的缓存方案, 大家可以看看.

感谢阅读.


