---
title: 关于 WKWebView 适配
date: 2017-08-06 18:42:55
categories: [工与匠, iOS]
tags: [WebView]
math: true
mermaid: true
---

# 一、分析
在iPhone 6s、iOS 10.3.2中，对 **http://www.qq.com** 进行10次请求，得到如下数据：

| 次数 | UIWebView 内存消耗 | WKWebView 内存（APP）消耗 | UIWebView 请求耗时  | WKWebView 请求耗时 |
| :--: | :--: | :--: | :--: | :--: |
| 1 | 67.47 MB | 0.81 MB | 4.13 s | 0.80 s |
| 2 | 58.23 MB | 0.86 MB | 1.16 s | 0.54 s |
| 3 | 57.83 MB | 0.50 MB | 1.14 s | 0.56 s |
| 4 | 59.38 MB | 0.88 MB | 1.08 s | 1.07 s |
| 5 | 59.70 MB | 0.75 MB | 1.07 s | 0.71 s |
| 6 | 64.05 MB | 0.83 MB | 1.47 s | 0.65 s |
| 7 | 59.45 MB | 0.81 MB | 1.11 s | 0.63 s |
| 8 | 57.55 MB | 0.45 MB | 1.15 s | 0.64 s |
| 9 | 58.47 MB | 0.77 MB | 1.17s | 0.75 s |
| 10 | 58.89 MB | 0.84 MB | 1.11 s | 0.70 s |

UIWebView平均内存消耗：54.13 MB
WKWebView平均（APP）内存消耗：**0.75 MB**
UIWebView平均请求耗时：1.46 s
WKWebView平均请求耗时：**0.7 s**

**综上可得**：WKWebView在请求耗时上为UIWebView的50%左右，内存上更是完胜。但实际上 WKWebView是一个多进程组件，**网络请求**以及**UI渲染**在其它进程中执行。仔细观察会发现：加载时，App进程内存消耗虽非常小甚至反而大幅下降，但Other Process的内存占用会增加。所以：在UIWebView上当内存占用太大的时候，App Process会crash；而在WKWebView上当总体的内存占用比较大的时候，WebContent Process会crash，从而出现白屏现象。
<!-- more -->
> Tip: 在一些用`webGL`渲染的复杂页面，使用WKWebView总体的内存占用（App Process Memory + Other Process Memory）不见得比UIWebView少很多。

考虑全部替换WKWebView风险过高，可通过Server端在APP启动时下发URL列表的方式实现WKWebView的灰度能力。通过封装继承 `UIView` 的 `SFWebView` ，实现UIWebView与WKWebView双核能力WebView。

# 特性
关于WKWebView特性：
+ 在性能、稳定性、功能方面有很大提升；
+ 允许JavaScript的Nitro库加载并使用（UIWebView中限制）；
+ 支持了更多的HTML5特性；
+ 高达60fps的滚动刷新率以及内置手势；
+ 将UIWebView 和 UIWebViewDelegate 重构成了14类与3个协议；

# 二、一些问题及解决方案
## 2.1 白屏问题
在UIWebView上当内存占用太大的时候，App Process会crash；而在WKWebView上当总体的内存占用比较大的时候，WebContent Process会crash，从而出现白屏现象。
> 实验链接： http://people.mozilla.org/~rnewman/fennec/mem.html

这个时候WKWebView.URL会变为nil, 简单的 `reload` 刷新操作已经失效，对于一些长驻的H5页面影响比较大。
解决方案：

+ 借助 WKNavigtionDelegate（仅适用iOS9以上）
```objectivec
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11), ios(9.0));
```
当WKWebView总体内存占用过大，页面即将白屏的时候，系统会调用上面的回调函数，我们在该函数里执行 `[webView reload]` (这个时候webView.URL取值尚不为nil）解决白屏问题。在一些高内存消耗的页面可能会频繁刷新当前页面，H5侧也要做相应的适配操作。

+ 检测 webView.title 是否为空（需在初始化时设置默认webView.title）

并不是所有H5页面白屏的时候都会调用上面的回调函数，比如在一个高内存消耗的H5页面上present 系统相机，拍照完毕后返回原来页面的时候出现白屏现象（拍照过程消耗了大量内存，导致内存紧张，WebContent Process 被系统挂起），但上面的回调函数并没有被调用。在WKWebView白屏的时候，另一种现象是webView.title会被置空, 因此，可以在viewWillAppear的时候检测webView.title是否为空来 `reload` 页面。

综合以上两种方法可以解决绝大多数的白屏问题。
## 2.2 Cookie问题
### 2.2.1 Cookie的私有存储问题
业界普遍认为WKWebView拥有自己的私有存储，不会将cookie存入到标准的cookie容器`NSHTTPCookieStorage`中。
实践发现：在iOS 8上，当页面跳转的时候，当前页面的cookie会写入 `NSHTTPCookieStorage` 中，而在iOS 10上，JS执行 `document.cookie` 或服务器 `set-cookie` 注入的cookie会很快同步到 `NSHTTPCookieStorage` 中.
> FireFox工程师曾建议通过 `reset` WKProcessPool来触发cookie同步到 `NSHTTPCookieStorage` 中，实践发现不起作用，并可能会引发当前页面 `session cookie` 丢失等问题。

### 2.2.2 请求不会自动带上容器中Cookie问题
WKWebView发起的请求不会自动带上存储于 `NSHTTPCookieStorage` 容器中的cookie。
比如，`NSHTTPCookieStorage` 中存储了一个cookie:
```JSON
name=Nicholas;value=test;domain=www.smallfan.net;expires=Sat, 02 May 2019 23:38:25 GMT；
```
通过UIWebView发起请求http://www.smallfan.net，则请求头会自动带上cookie: Nicholas=test；
而通过WKWebView发起请求http://www.smallfan.net，请求头不会自动带上cookie: Nicholas=test。
解决方案：

+ A. WKWebView `loadRequest` 前，在request的header中设置cookie, 解决首个请求cookie带不上的问题。
```objectivec
WKWebView *webView = [WKWebView new];
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"https://www.fxiaoke.com"]];
[request addValue:@"uid=1000" forHTTPHeaderField:@"Cookie"];
[webView loadRequest:request];
```

+ B.通过·document.cookie·设置cookie解决后续页面(同域)Ajax、iframe请求的cookie问题。
```objectivec
- (NSString *)shareHttpCookieFromStorage:(NSURL *)url {

	NSMutableArray *array = [NSMutableArray array];
	for (NSHTTPCookie *cookie in [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:url]) {

		NSString *value = [NSString stringWithFormat:@"%@=%@", cookie.name, cookie.value];
		value = [NSString stringWithFormat:@"document.cookie = '%@'", value];
		[array addObject:value];
	}

	NSString *header = @"";
	header = [array componentsJoinedByString:@";"];

	return header;
}

- (void)addCookiesWithUrl:(NSURL *)url {
	WKUserContentController *userContentController = [WKUserContentController new]; 
	WKUserScript *cookieScript = [[WKUserScript alloc] initWithSource:[self shareHttpCookieFromStorage]  injectionTime:WKUserScriptInjectionTimeAtDocumentStart
                                                         forMainFrameOnly:NO];
	[controller addUserScript:cookieScript];
	[userContentController addUserScript:cookieScript];
	_wkWebView = [[WKWebView alloc] initWithFrame:self.bounds configuration:configuration];
	...
}
```
**注意：**因为NSHTTPCookieStorage是整个APP共享的单例，包含了所有domain的cookie，在WKWebView初始化时，务必提前获取URL载入对应的cookie，防止因cookies泄漏造成可模仿登录等安全漏洞。

B方案无法解决302请求（跨域）的cookie问题，可以拦截页面每次跳转都会调用的回调函数：
`- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler`来实现复制request对象，在request header中带上cookie并重新 `loadRequest` 。
```objectivec
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {

	NSMutableURLRequest *newReq = [navigationAction.request mutableCopy];
	NSMutableArray *array = [NSMutableArray array];
	for (NSHTTPCookie *cookie in [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:navagationAction.request.URL]) {
		NSString *value = [NSString stringWithFormat:@"%@=%@", cookie.name, cookie.value];
		[array addObject:value];
	}

	NSString *cookie = [array componentsJoinedByString:@";"];
	[newReq setValue:cookie forHTTPHeaderField:@"Cookie"];
	[webView loadRequest:newReq];
}
```
**缺陷：**依然解决不了页面iframe跨域请求的cookie问题，毕竟-[WKWebView loadRequest:]只适合加载mainFrame请求。
### 2.2.3 WKProcessPool无法本地化保存（离线缓存）
苹果开发者文档对WKProcessPool的定义是：**A WKProcessPool object represents a pool of Web Content process.** 通过让所有WKWebView共享同一个WKProcessPool实例，可以实现多个WKWebView之间共享cookie数据。不过WKProcessPool实例在app杀进程重启后会被重置，导致WKProcessPool中的cookie、session cookie数据丢失，目前也无法实现WKProcessPool实例本地化保存。
**注意：**由于WKWebView在请求过程中用户可能退出界面销毁对象，当请求回调时由于接收处理对象不存在，造成Bad Access crash，所以可将WKProcessPool设为单例
附使用方式：
```objectivec
static WKProcessPool *_sharedWKProcessPoolInstance = nil;
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
	_sharedWKProcessPoolInstance = [[WKProcessPool alloc] init];
});

self.processPool = _sharedWKProcessPoolInstance;

WKWebViewConfiguration *configuration1 = [[WKWebViewConfiguration alloc] init];
configuration1.processPool = self.processPool;
WKWebView *webView1 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration1];
...
WKWebViewConfiguration *configuration2 = [[WKWebViewConfiguration alloc] init];
configuration2.processPool = self.processPool;
WKWebView *webView2 = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration2];
...
```
## 2.3 NSURLProtocol问题
WKWebView在独立于 app 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在WKWebView上直接使用 `NSURLProtocol` 无法拦截请求。苹果开源的 `Webkit2` 源码暴露了私有API：
```objectivec
+ [WKBrowsingContextController registerSchemeForCustomProtocol:]
```
通过注册 http(s) scheme 后WKWebView将可以使用 `NSURLProtocol` 拦截http(s)请求：
```objectivec
//仅iOS8.4以上可用
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.4)  {
	Class cls = NSClassFromString(@"WKBrowsingContextController”); 
	SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");

	if ([(id)cls respondsToSelector:sel]) {
		#pragma clang diagnostic push
		#pragma clang diagnostic ignored "-Warc-performSelector-leaks"

			// 注册http(s) scheme, 把 http和https请求交给 NSURLProtocol处理 
			[(id)cls performSelector:sel withObject:@"http"];
			[(id)cls performSelector:sel withObject:@"https"];

		#pragma clang diagnostic pop
		}
	}
}
```
这种方案目前存在以下严重缺陷：
**post请求body数据被清空**
由于WKWebView在独立进程里执行网络请求。一旦注册http(s) scheme后，网络请求将从Network Process发送到App Process，这样 `NSURLProtocol` 才能拦截网络请求。在webkit2的设计里使用MessageQueue进行进程之间的通信，Network Process会将请求encode成一个Message,然后通过 IPC 发送给 App Process。出于性能的原因，encode的时候HTTPBody和HTTPBodyStream这两个字段被丢弃掉了。
因此，**如果通过 `registerSchemeForCustomProtocol` 注册了http(s) scheme, 那么由WKWebView发起的所有http(s)请求都会通过 IPC 传给主进程 `NSURLProtocol` 处理，导致post请求body被清空。**
解决方案：

### 2.3.1 如果没有开启ATS
可以注册`customScheme`, 比如`smallfan://`, 因此希望使用离线功能又不使用**post**方式的请求可以通过 `customScheme` 发起请求，比如 `smallfan://webCache/HelloWorld` ，然后在App进程 `NSURLProtocol` 拦截这个请求并加载离线数据。不足：使用**post**方式的请求该方案依然不适用，同时需要HTML5侧修改请求scheme以及CSP规则。

### 2.3.2 如果开启ATS
因为：一旦打开ATS开关：**Allow Arbitrary Loads**选项设置为NO，同时通过 `registerSchemeForCustomProtocol` 注册了http(s) scheme，WKWebView发起的所有非https网络请求将被阻塞（即便将**Allow Arbitrary Loads in Web Content**选项设置为YES）。
可通过hook所有的post请求的方式解决：

+ 对于Ajax post请求，思路是通过XMLHttpRequest send及open方法，将http body内容拼装在http header中并正常请求，App进程 `NSURLProtocol` 拦截这个请求，将header中的BODY内容取出置于body，发送请求，并将结果返回WKWebView（可借助 [WebViewProxy](https://github.com/marcuswestin/WebViewProxy) 完成）。
JS文件：
```javascript
var s_ajaxListener = new Object();
s_ajaxListener.tempOpen = XMLHttpRequest.prototype.open;
s_ajaxListener.tempSend = XMLHttpRequest.prototype.send;
s_ajaxListener.tempSetRequestHeader = XMLHttpRequest.prototype.setRequestHeader;

XMLHttpRequest.prototype.open = function(a,b) {
  this._method = a;
  s_ajaxListener.tempOpen.apply(this, arguments);
}

XMLHttpRequest.prototype.send = function(a,b) { 
  if (this._method && this._method.toLowerCase() == 'post') {
    a = encodeURIComponent(a)
    s_ajaxListener.tempSetRequestHeader.apply(this, ['BODY', a])
  }
  return s_ajaxListener.tempSend.apply(this, arguments);
}
```
APP拦截请求：
```objectivec
[WebViewProxy handleRequestsWithHttpHeader:@"BODY" handlerHash:[self hash] handler:^(NSURLRequest *req, WVPResponse *res) {

	NSURLSession *session = [NSURLSession sharedSession];
	NSMutableURLRequest *request = [req mutableCopy];
	request.HTTPMethod = @"POST";
	NSString *postData = request.allHTTPHeaderFields[@"BODY"];
	NSString *decodePostData = (__bridge_transfer NSString *)CFURLCreateStringByReplacingPercentEscapesUsingEncoding(NULL, (__bridge CFStringRef)postData, CFSTR(""), CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding));
	NSString *httpBody = decodePostData;
	request.HTTPBody = [httpBody dataUsingEncoding:NSUTF8StringEncoding];
	NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
		NSDictionary *headers = [(NSHTTPURLResponse *)response allHeaderFields];
		[res respondWithNSData:data mimeType:response.MIMEType header:headers statusCode:((NSHTTPURLResponse *)response).statusCode];

	}];
	[dataTask resume];
}];
```

+ 对于form请求，解决方式类似Ajax，只是将BODY转编码后拼接到URL中，APP进程处理方式一致。

JS文件：
```javascript
var s_formListener = window.onsubmit;
window.onsubmit = function (e) {
var node = e.srcElement;
if (node && node.tagName && node.tagName.toLowerCase() === 'form') {
	if (node.method && node.method.toLowerCase() === 'post') {
		var elements = [].slice.call(node.elements);
		var tempData = [], entryName, entryValue;
		for (var i = 0, l = elements.length; i < l; ++i) {
			entryName = elements[i].name;
			entryValue = elements[i].value;
			if (entryValue.toString() === '[object File]') {
				entryValue = entryValue.name;
			}
			tempData.push(encodeURIComponent(entryName) + '=' + encodeURIComponent(entryValue));
		}
		tempData = tempData.join('&');

		var action = node.action || location.href;
		var hashIndex = action.indexOf('#');
		if (hashIndex >= 0) {
			action = action.substring(0, hashIndex);
		}
		var queryIndex = action.indexOf('?');
		if (queryIndex >= 0) {
			action = action + '&POST_DATA=' + encodeURIComponent(tempData);
		} else {
			action = action + '?POST_DATA=' + encodeURIComponent(tempData);
		}

		node.action = action;
	}
}
if (s_formListener && s_formListener.apply) {
	s_formListener.apply(this, arguments);
}
}
```
APP拦截请求：
```objectivec
[WebViewProxy handleRequestsWithHttpHeader:@"POST_DATA" handler:^(NSURLRequest *req, WVPResponse *res) {

	NSURLSession *session = [NSURLSession sharedSession];
	NSMutableURLRequest *request = [req mutableCopy];
	request.HTTPMethod = @"POST";
	NSString *postData = request.allHTTPHeaderFields[@"POST_DATA"];
	NSString *decodePostData = (__bridge_transfer NSString *)CFURLCreateStringByReplacingPercentEscapesUsingEncoding(NULL, (__bridge CFStringRef)postData, CFSTR(""), CFStringConvertNSStringEncodingToEncoding(NSUTF8StringEncoding));
	NSString *httpBody = decodePostData;
	request.HTTPBody = [httpBody dataUsingEncoding:NSUTF8StringEncoding];
	NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {

		NSDictionary *headers = [(NSHTTPURLResponse *)response allHeaderFields];
		[res respondWithNSData:data mimeType:response.MIMEType header:headers statusCode:((NSHTTPURLResponse *)response).statusCode];

	}];
	[dataTask resume];
}];
```
**缺陷**：Http header value及url字符有长度限制。在**WWDC 2017**上，提到iOS 11将开放一个 `WKURLSchemeHandler` 注册，提供custom response的能力，拭目以待。
## 2.4 JavaScript交互
### 2.4.1 WKWebView调用JavaScript
```objectivec
[_wkWebView evaluateJavaScript:@"Hello" completionHandler:^(NSString result, NSError * _Nullable error) {
	if([result isEqualToString:@"Hi"]) {
	}
}];
```
### 2.4.2 JavaScript调用WKWebView
```objectivec
 WKWebViewConfiguration * Configuration = [[WKWebViewConfiguration alloc] init];
 WKUserContentController *userContentController = [[WKUserContentController alloc] init];

//注册一个name为HelloNative的js方法
[userContentController addScriptMessageHandler:self  name:@"HelloNative"];

Configuration.userContentController = userContentController;
_wkWebView = [[WKWebView alloc] initWithFrame:CGRectMake(0, 0, 300,500) configuration:Configuration];
```
```objectivec
#pragma mark WKScriptMessageHandler
//设置WKWebView的WKScriptMessageHandler代理方法
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(nonnull WKScriptMessage *)message {
	if ([message.name isEqualToString:@"HelloNative"]) {
		// 打印所传过来的参数，只支持NSNumber, NSString, NSDate, NSArray, NSDictionary, NSNull类型
		NSLog(@"%@", message.body);
	}
}
```
JS调用：
```JavaScript
window.webkit.messageHandlers.HelloNative.postMessage(message);
```
**注意**：关闭web页时，需要调用removeScriptMessageHandlerForName以防止内存泄漏。
```objectivec
- (void)dealloc {
	_wkWebView.UIDelegate = nil;
	_wkWebView.navigationDelegate = nil;
	[[_wkWebView configuration].userContentController removeScriptMessageHandlerForName:@"HelloNative"];
}
```
## 2.5 Crash问题
### 2.5.1 JS调用window.alert()函数引起的crash
当JS调用alert函数时，`WKWebView`使用如下方法回调：
```objectivec
+ (void)presentAlertOnController:(nonnull UIViewController*)parentController title:(nullable NSString*)title message:(nullable NSString *)message handler:(nonnull void (^)())completionHandler;
```
主要原因是上述 `completionHandler` 没有被调用导致的。在适配WKWebView的时候，我们需要自己实现该回调函数，window.alert()才能调起alert框。
解决方案：
```objectivec
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler {
	if (/*UIViewController of WKWebView has finish push or present animation*/) { 
		completionHandler();
		return;
	} 
	UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"" message:message preferredStyle:UIAlertControllerStyleAlert];
	[alertController addAction:[UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleCancel handler:^(UIAlertAction *action) { completionHandler(); }]];
	if (/*UIViewController of WKWebView is visible*/)
		[self presentViewController:alertController animated:YES completion:^{}];
	else
		completionHandler();
}
```
### 2.5.2 WKWebView 退出前调用`evaluateJavaScript:completionHandler:`引起的crash
主要原因WKWebView退出并被释放后导致 `completionHandler` 变成野指针，而此时 JavaScriptCore还在执行JS代码，待JavaScriptCore执行完毕后会调用 `completionHandler()` ，导致crash。
**这个crash只发生在iOS 8系统上，iOS9以上主要是对completionHandler block做了copy。**

解决方案：
通过在`completionHandler`里`retain``WKWebView`防止`completionHandler`被过早释放。
```objectivec
+ (void) load {
	[self jr_swizzleMethod:NSSelectorFromString(@"evaluateJavaScript:completionHandler:") withMethod:@selector(altEvaluateJavaScript:completionHandler:) error:nil];
}
/*
 * fix: WKWebView crashes on deallocation if it has pending JavaScript evaluation 
 */
- (void)altEvaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^)(id, NSError *))completionHandler {
	id strongSelf = self;
	[self altEvaluateJavaScript:javaScriptString completionHandler:^(id r, NSError *e)  {
		[strongSelf title];
		if (completionHandler) {
			completionHandler(r, e);
		}
	}];
}
```
## 2.6 进度条问题
在UIWebView中，进度条一直是个缺陷问题，尽管有 `NJKWebViewProgress` 一类开源组件，但在精确处理加载完成上还有些许不足（部分页面可见webViewDidStartLoad:与webViewDidFinishLoad:不成对回调，导致进度条加载结果计算有误）。而WKWebView上增加了一个[estimatedProgress](https://developer.apple.com/documentation/webkit/wkwebview/1415007-estimatedprogress)属性，通过KVO可实现精准进度条控制。[iOS WKWebView添加类似微信的进度条](http://www.jianshu.com/p/54f2d0b0736b)
通过对微信的观察，进度条的精确结果并不是首要的，良好的用户心态预期才是其重点。所以，可以考虑以如下方式实现“虚拟”的进度条。
```objectivec
- (void)startProgress {
	if (_hideProgress) {
		return;
	}

	if (_progress == 0) {
		_progress = 0.9;

		[_progressLayer removeAllAnimations];//清除所有的动画

		CGRect frame = _progressView.frame;
		CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
		//设置关键帧数组
		animation.values=@[[NSValue valueWithCGPoint:CGPointMake(0, 0)],
						   [NSValue valueWithCGPoint:CGPointMake(0.7 * frame.size.width, .0)],
						   [NSValue valueWithCGPoint:CGPointMake(0.9 * frame.size.width, .0)],
						   [NSValue valueWithCGPoint:CGPointMake(1.0 * frame.size.width, .0)]];
		//设置每个关键帧对应的时间点，取值为0~1
		animation.keyTimes = @[[NSNumber numberWithFloat:.0],
							   [NSNumber numberWithFloat:.3],
							   [NSNumber numberWithFloat:.7],
							   [NSNumber numberWithFloat:1.]];
		animation.removedOnCompletion = YES;
		animation.fillMode = kCAFillModeForwards;
		animation.duration = 20;
		animation.delegate = self;
		[_progressLayer addAnimation:animation forKey:@"startProgress"];
	}
}

- (void)completeProgress {
	if (_hideProgress) {
		return;
	}

	CGPoint point = _progressLayer.presentationLayer.position;//当前动画的position
	if (round(point.x) == 0 && !_progress) {
		return;
	}
	[_progressLayer removeAnimationForKey:@"startProgress"];

	_progress = 1.0;

	CGRect frame = _progressView.frame;
	CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
	//设置关键帧数组
	animation.values=@[[NSValue valueWithCGPoint:point],
					   [NSValue valueWithCGPoint:CGPointMake(1.0 * frame.size.width, .0)]];
	//设置每个关键帧对应的时间点
	animation.keyTimes = @[[NSNumber numberWithFloat:.0],
						   [NSNumber numberWithFloat:1.]];
	animation.duration = .27;
	animation.removedOnCompletion = YES;
	animation.fillMode = kCAFillModeForwards;
	[_progressLayer addAnimation:animation forKey:@"completeProgress"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
	// 当WKWebView回调webView:didFinishNavigation:时，页面实际上渲染并未完成
	// 监听loading属性变化，可精确判断请求完成+渲染完成
	if ([keyPath isEqualToString:@"loading"]) {

		BOOL oldLoading = [[change objectForKey:NSKeyValueChangeOldKey] boolValue];
		BOOL newLoading = [[change objectForKey:NSKeyValueChangeNewKey] boolValue];

		if (newLoading) {
			dispatch_async(dispatch_get_main_queue(), ^{
				[self startProgress];
			});
		} else if (!newLoading) {
			dispatch_async(dispatch_get_main_queue(), ^{
				[self completeProgress];
				_progress = 0;
			});
		}
	} else {
		[super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
	}
}

- (void)dealloc {
	_wkWebView.UIDelegate = nil;
	_wkWebView.navigationDelegate = nil;

	//记得在销毁时释放监听
	[_wkWebView removeObserver:self forKeyPath:@"loading"];
}

```
## 2.7 截屏问题
```objectivec
- (UIImage*)imageSnapshot {
	UIGraphicsBeginImageContextWithOptions(self.bounds.size,YES,self.contentScaleFactor);
	[self drawViewHierarchyInRect:self.bounds afterScreenUpdates:YES];
	UIImage* newImage = UIGraphicsGetImageFromCurrentImageContext();
	UIGraphicsEndImageContext();
	return newImage;
}
```
使用以上方法对webGL页面的截屏结果不是空白就是纯黑图片。

解决方案：约定一个JS接口，让H5实现该接口，具体是通过 `canvas getImageData()` 方法取得图片数据后返回 `base64` 格式的数据，客户端在需要截图的时候，调用这个JS接口获取 `base64 String` 并转换成 `UIImage` 。
## 2.8 其他问题
### 2.8.1 iOS8.2以下音频无法停止
解决方案多种：
```objectivec
//第一种：返回时请求一个空页面
 [NSURL URLWithString:@"about:blank"]

//第二种：返回时加载一个空的HTML String
[_wkWebView loadHTMLString:@"<html/>" baseURL:nil]

//第三种：在viewDidDisappear方法里播放无声的音频再暂停
```
### 2.8.2 视频没有自动播放
解决方案：
WKWebView需要通过 `WKWebViewConfiguration.mediaPlaybackRequiresUserAction` 设置是否允许自动播放，但一定要在WKWebView初始化之前设置，在WKWebView初始化之后设置无效。
### 2.8.3 页面回退问题

+ 业务上的需求，当最后只有一条历史，直接 `pop` 回去，需要如下改写。
```objectivec
- (BOOL)canGoBack {
	if (self.backForwardList.backList.count <= 1) {
		return NO;
	}
	return YES;
}
```

+ WKWebView上调用 `-[WKWebView goBack]` , 回退到上一个页面后不会触发 `window.onload()` 函数、不会执行JS。

### 2.8.4 页面回退时，字体变大
解决方案：
在页面 `webView:didFinishNavigation:` 中执行以下JavaScript将webkit中字体还原到100%
```objectivec
[self evaluateJavaScript:@"document.getElementsByTagName('body')[0].style.webkitTextSizeAdjust= '100%'" completionHandler:nil];
```

参考文献：
1.[ 腾讯Bugly团队【WKWebView 那些坑】](https://mp.weixin.qq.com/s/rhYKLIbXOsUJC_n6dt9UfA)
