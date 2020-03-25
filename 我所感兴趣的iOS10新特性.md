---
title: 我所感兴趣的iOS10新特性
date: 2016-09-20 11:49:01
tags: iOS
---


### SiriKit
Siri API 的开放自然是 iOS 10 SDK 中最激动人心也是亮眼的特性。SiriKit 为我们提供一全套从语音识别到代码处理，最后向用户展示结果的流程。Apple 加入了一套全新的框架 Intents.framework 来表示 Siri 获取并解析的结果。你的应用需要提供一些关键字表明可以接受相关输入，而 Siri 扩展只需要监听系统识别的用户意图 (intent)，作出合适的响应，修改以及实际操作，最后通过 IntentsUI.framework 提供反馈。整个过程非常清晰明了，但是这也意味着开发者所能拥有的自由度有限。
在 iOS 10 中，我们能用 SiriKit 来做六类事情，分别是：
- 语音和视频通话
- 发送消息
- 发送或接收付款
- 搜索照片
- 约车
- 管理健身
**(具体可了解滴滴出行等软件iOS10的适配)**

<!-- more -->

### Xcode8
Xcode8有很多的新特性,这里不详说,可以再自己的日常使用中慢慢的去发现.
我想说的是Xcode8中对证书的管理.下面引用喵神的话:
>在 app 签名方面，Apple 终于意识到了他们在 Xcode 7 中所犯得错误。我想可能不止一个人被证书和描述文件出问题时的 "Fix Issue" 按钮坑过。这个按钮不仅不会修正问题，反而会直接注销现有的开发者证书，然后“自作主张”地重新申请。大多数情况下，这让事情变得更加糟糕。特别是对于新加入的开发者，他们并不理解 Apple 的证书系统，错误的操作和处置，往往让开发环境变得不可挽回。Xcode 8 中，同一个开发者帐号现在允许多个开发证书，而完全重做的 app 签名系统也足够好用，并且避免了误操作的可能性。在兼顾自动配置的基础上，也为大型项目和复杂的 CI 环境提供了足够灵活的配置空间，这绝对值得点赞。
另外 Xcode 终于提供了进行代码编辑器扩展的能力。现在开发者可以创建 XCSourceEditorExtension
 来对 Xcode 的功能进行扩展了，在没有文档帮助和官方支持的情况下摸索着为 Xcode 制作插件的历史也即将结束。

**哦对了,还要说一下关于Xcode的插件问题,在新版的Xcode中是不允许使用第三方的插件的,也就是说以前的插件要失效了,当然苹果并不是禁止开发者使用插件,而且对插件的开发要进行统一的管理,以后的插件也就成了官方插件了,像VVDocumenter这样的插件以后会默认收进Xcode里面,也就是不用再额外安装了,具体的情况在[VVDocumenter的github issues](https://github.com/onevcat/VVDocumenter-Xcode/issues/228)上面有一些讨论,大家可以去看看**

----[解决插件失效](https://github.com/inket/update_xcode_plugins)


### APP Extension
 iOS10 如下的全新 7 种 App Extension：
- Call Directory（VoIP回调）
- Intents(接Siri、Apple map等服务)
- Intents  UI(接Siri、Apple map等服务的自定义界面)
- Messages（iMessage拓展）
- Notification Content（内容通知）
- Notification  Service （服务通知）
- StickerPack（iMessage表情包）

[了解App Extension](http://www.jianshu.com/p/bbc6a95d9c54)

### CallKit
关于CallKit简单的说几句,可以使VoIP apps 在锁屏界面接听 VoIP电话(新版QQ)，还可以在app extensions中进行**来电拦截和来电识别**（这个是苹果和腾讯手机管家合作开发的,朋友圈不经允许就做广告...不过不得不佩服鹅厂）

----

此外还有User Notifications(**项目中有Push的应该注意一下**),iMessage Apps,Swift 3等的一些新特性就不做详述了.大家可以去官方文档浏览一下

--[iOS 10.0](https://developer.apple.com/library/content/releasenotes/General/WhatsNewIniOS/Articles/iOS10.html)
--[iOS 10.0 API Differences](https://developer.apple.com/library/content/releasenotes/General/iOS10APIDiffs/index.html)

----

#### 升级到iOS10的一些适配问题
1. Xib文件的注意事项
简单的来说Xcode8改完的Xib文件Xcode7打开会报错,虽然说有解决办法, 不过我还是希望我们开发者能跟上苹果的步伐,使用最近的工具
2. 权限以及相关设置
我们需要打开info.plist文件添加相应权限的说明
>麦克风权限：Privacy – Microphone Usage Description 是否允许此App使用你的麦克风？
相机权限： Privacy – Camera Usage Description 是否允许此App使用你的相机？
相册权限： Privacy – Photo Library Usage Description 是否允许此App访问你的媒体资料库？通讯录权限： Privacy – Contacts Usage Description 是否允许此App访问你的通讯录？
蓝牙权限：Privacy – Bluetooth Peripheral Usage Description 是否许允此App使用蓝牙？
语音转文字权限：Privacy – Speech Recognition Usage Description 是否允许此App使用语音识别？
日历权限：Privacy – Calendars Usage Description 是否允许此App使用日历？
定位权限：Privacy – Location When In Use Usage Description 我们需要通过您的地理位置信息获取您周边的相关数据
定位权限: Privacy – Location Always Usage Description 我们需要通过您的地理位置信息获取您周边的相关数据

[更多的适配问题](http://ios.jobbole.com/88982/)

----
特此感谢
[OneV's Den-开发者所需要知道的 iOS 10 SDK 新特性](https://onevcat.com/2016/06/ios-10-sdk/)
[iOS10新特性及开发者要注意什么](http://codecloud.net/10909.html)
[揭秘 iOS App Extension 开发 —— Today 篇](http://www.jianshu.com/p/bbc6a95d9c54)

