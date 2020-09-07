

#### 前言
最近公司需要我们使用XMPP协议，实现一个简单的IM模块。在此之前没有接触过IM相关技术，仅了解iOS可以通过集成XMPPFramework来快速的实现某些需求。本系列文章旨在记录使用XMPPFramework过程中遇到的问题。

#### 正文

首先先聊一下XMPP实现IM，在查过一些资料后，我粗略的认为其实整体来说IM就是一个长连接（暂时抛开优化等深层次的东西），XMPP协议则是约束了用户端和服务端信息交互的规则。当然随着对IM和XMPP等的深入研究，我可能会慢慢改变自己的看法，至少目前来是这样子认识的。

回到正题，我们都知道XMPP是开源的，iOS端使用XMPPFramework，服务端一般都采用Openfire，而大多公司会在此基础上进行二次开发，我们公司就是这样。

关于用户登录的细节网上代码很多，我简单的贴一点。当我们使用我们的JID连接到服务器的时候，我们下一步需要验证密码。在默认的情况下，当我们发送了验证密码的请求，下一次服务端返回的报文是关于验证结果状态的。大家简单看一眼下面的代码：

```swift
// 拼接JID然后连接
xmppStream.myJID = XMPPJID(user: UserInfo.sysAccount, domain: domain, resource: "iOS")
try? xmppStream.connect(withTimeout: timeOut)
```

然后在连接成功的回调方法中验证密码：

```swift
func xmppStreamDidConnect(_ sender: XMPPStream) {
    try? xmppStream.authenticate(withPassword: self.password ?? "")
}
```

如果服务端没有改动，那么我们在验证之后收到的报文应该是下面这种：

```swift
// 验证成功
<success xmlns="urn:ietf:params:xml:ns:xmpp-sasl"/>
// 验证失败
<failure xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><not-authorized/><code>xxxxx</code><msg>xxxxx</msg></failure>
```

在确保账号和密码都是正确的情况下，我一直收到验证失败的回调。因为我们的服务端对验证过程作了一些改动，在验证密码成功之后，代表用户登录成功，这时会在验证结果状态返回之前，插入返回一条<iq>标签的报文，包裹着用户资料。

我们通过debug源码，看了一下整个验证密码的过程。

具体的总是提示验证失败的原因要定位到`XMPPPlainAuthentication.m`文件的`- (XMPPHandleAuthResponse)handleAuth:(NSXMLElement *)authResponse`方法中。

```swift
- (XMPPHandleAuthResponse)handleAuth:(NSXMLElement *)authResponse
{
	XMPPLogTrace();
	if ([[authResponse name] isEqualToString:@"success"])
	{
		return XMPPHandleAuthResponseSuccess;
	}
	else
	{
		return XMPPHandleAuthResponseFailed;
	}
}
```

这是XMPPFramework里面源码，在处理验证的时候，直接判断这个标签是不是success，是就返回`XMPPHandleAuthResponseSuccess`，不是的话，不管是什么都返回`XMPPHandleAuthResponseSuccess`。

因为我们会提前收到一条用户信息的报文，所以我们需要在处理此条报文的时候不要处理为失败。唯一的方法就是要对源码做一些改动了。下面是改动后的代码。

```swift
- (XMPPHandleAuthResponse)handleAuth:(NSXMLElement *)authResponse
{
	XMPPLogTrace();
	
	// We're expecting a success response.
	// If we get anything else we can safely assume it's the equivalent of a failure response.
	
#warning sunxb add
    // 添加iq判断 解决验证时候多条报文的问题
	if ([[authResponse name] isEqualToString:@"success"])
	{
		return XMPPHandleAuthResponseSuccess;
	}
    // add 
    else if ([XMPPAuthUtils judgeUserInfoWith:authResponse]) {
        return XMPPHandleAuthResponseContinue;
    }
	else
	{
		return XMPPHandleAuthResponseFailed;
	}
}
```

我们增加了一个条件判断，else if 中的`judgeUserInfoWith`是我们自己的业务逻辑判断，我们的用户信息是通过<iq>标签返回的，为了区别与其他的报文，我们给他加上了不一样的命名空间（xmlns）（不同公司业务不同判断条件也不一样，总之我们的判断逻辑是此条报文是<iq>，同时携带用户信息，才返回true）。

最终处理目的就是如果收到了用户消息的报文，不要返回`XMPPHandleAuthResponseFailed`，先返回`XMPPHandleAuthResponseContinue`。（其他的乱七八糟的报文跟之前一样当做fail处理）

那`XMPPHandleAuthResponseContinue`这个类型有什么不一样呢？我们还得看源码。

我们定位到`XMPPStream.m`中的`- (void)handleAuth:(NSXMLElement *)authResponse`方法，大家不要混了，都是叫`handleAuth`，下面这个方法中调用了我们上面提到的方法，而且上面的那个方法是有返回值的。大家不用太仔细的看代码，只要知道这个方法，是根据上面的`handleAuth`方法的返回值，分别做了处理。

为了让代码紧凑一些，我删除了一些注释。

```swift
- (void)handleAuth:(NSXMLElement *)authResponse
{
	NSAssert(dispatch_get_specific(xmppQueueTag), @"Invoked on incorrect queue");
	
	XMPPLogTrace();
	
	XMPPHandleAuthResponse result = [auth handleAuth:authResponse];
	
	if (result == XMPPHandleAuthResponseSuccess)
	{
		[self setIsAuthenticated:YES];
		
		BOOL shouldRenegotiate = YES;
		if ([auth respondsToSelector:@selector(shouldResendOpeningNegotiationAfterSuccessfulAuthentication)])
		{
			shouldRenegotiate = [auth shouldResendOpeningNegotiationAfterSuccessfulAuthentication];
		}
		
		if (shouldRenegotiate)
		{
			[self sendOpeningNegotiation];
			
			if (![self isSecure])
			{
				
				[asyncSocket readDataWithTimeout:TIMEOUT_XMPP_READ_START tag:TAG_XMPP_READ_START];
			}
		}
		else
		{
			state = STATE_XMPP_CONNECTED;
			
			[multicastDelegate xmppStreamDidAuthenticate:self];
		}
		auth = nil;
		
	}
	else if (result == XMPPHandleAuthResponseFailed)
	{
		state = STATE_XMPP_CONNECTED;
		
		// Notify delegate
		[multicastDelegate xmppStream:self didNotAuthenticate:authResponse];
		
		// Done with auth
		auth = nil;
		
	}
	else if (result == XMPPHandleAuthResponseContinue)
	{
		// Authentication continues.
		// State doesn't change.
	}
	else
	{
		XMPPLogError(@"Authentication class (%@) returned invalid response code (%i)",
		           NSStringFromClass([auth class]), (int)result);
		
		NSAssert(NO, @"Authentication class (%@) returned invalid response code (%i)",
		             NSStringFromClass([auth class]), (int)result);
	}
}
```

从代码中可以看到，如果失败会直接调用验证失败的回调，但是如果结果是`XMPPHandleAuthResponseContinue`，没有做任何处理。

所以我们需要在验证过程中对收到的用户信息报文，返回continue状态。因为用户信息属于有用的报文（需要储存），所以这个地方我们也要做一些修改，把我们的用户信息传出去。

```swift
else if (result == XMPPHandleAuthResponseContinue)
	{
#warning sunxb add: when auth response is continue, just send authResponse to delegate
        [multicastDelegate xmppStream:self didReceiveCustomElement:authResponse];
	}
```

**注： 所有修改的源码我都会加一个#warning，方法以后定位自己修改过得源码。**

做完上面的处理之后，我们的代理方法回调就正常了。

```swift
func xmppStreamDidAuthenticate(_ sender: XMPPStream) {
    // 此处收到验证成功回调
}
    
func xmppStream(_ sender: XMPPStream, didReceiveCustomElement element: DDXMLElement) {
    // 用户消息的报文走这个回调，我们需要再判断一下是否是用户消息报文
    // 是否是用户消息
    if XMPPAuthUtils.judgeUserInfo(with: element) {
        JMClientUtils.storageUserInfo(with: element)
    }
}
```

#### 总结

本文是使用了XMPPFramework后遇到的第一个小问题，特此记录。随着XMPPFramework的使用可能会遇到其他的问题，到时候会继续记录分享。

因为要对源码做修改，XMPPFramework我采用了手动集成的方式，但XMPPFramework的依赖库依然是使用了Cocoapods来管理。（如果你想把所有的相关库都用手动集成，那需要修改的地方太多了，不推荐）

感谢阅读。


