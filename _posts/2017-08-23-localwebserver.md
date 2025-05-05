---
title: 基于 LocalWebServer 实现 WKWebView 离线资源加载
date: 2017-08-23 16:38:35
categories: [工与匠]
tags: [iOS, WKWebView]
math: true
mermaid: true
---

# 一、背景

笔者在[《WKWebView》](https://smallfan.net/2017/wkwebview/)一文中提到过，**WKWebView** 在独立于 app 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在 **WKWebView** 上直接使用 NSURLProtocol 无法拦截请求。所以如果需要使用到拦截请求，有种可行地方案是使用苹果开源的 Webkit2 源码暴露的私有API（详见原文第3小节：NSURLProtocol问题）。
但使用私有API，必然带来以下几个问题：

+ 审核风险
+ 拦截http/https时，post请求body丢失
+ 如使用ajax hook方式，可能存在 **post header字符长度限制** 、**Put类型请求异常** 等

由此看来，在 iOS11 **WKURLSchemeHandler** [[探究](http://www.jianshu.com/p/8af24e9dc82e)] 到来之前，私有API并不那么完美。
所幸通过寻找发现，iOS系统上具备搭建服务器能力，理论上对实现 **WKWebView 离线资源加载** 存在可能性。

# 二、分析

基于iOS的local web server，目前大致有以下几种较为完善的框架：

+ [CocoaHttpServer](https://github.com/robbiehanson/CocoaHTTPServer) (支持iOS、macOS及多种网络场景)
+ [GCDWebServer](https://github.com/swisspol/GCDWebServer) （基于iOS，不支持 https 及 webSocket）
+ [Telegraph](https://github.com/Building42/Telegraph) （Swift实现，功能较上面两类更完善）

因为目前大部分APP已经支持ATS，且国内大部分项目代码仍采用OC实现，故本文将以 **CocoaHttpSever** 为基础进行实验。
**Telegraph** 是为补充 CocoaHttpSever 及 GCDWebServer 不足而诞生，对于纯Swift项目，推荐使用 **Telegraph** 。

<!-- more -->

# 三、初出茅驴

在 project工程文件 中引入 CocoaHttpServer 之后，
1. 首先实现一个服务管理。
```objectivec
#import "LocalWebServerManager.h"

#import "HTTPServer.h"
#import "MyHTTPConnection.h"

@interface LocalWebServerManager ()
{
    HTTPServer *_httpServer;
}
@end

@implementation LocalWebServerManager

+ (instancetype)sharedInstance {
    static LocalWebServerManager *_sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedInstance = [[LocalWebServerManager alloc] init];
    });
    return _sharedInstance;
}

- (void)start {
    
    _port = 60000;
    
    if (!_httpServer) {
        _httpServer = [[HTTPServer alloc] init];
        [_httpServer setType:@"_http._tcp."];
        [_httpServer setPort:_port];
        NSString * webLocalPath = [[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent:@"Resource"];
        [_httpServer setDocumentRoot:webLocalPath];
        
        NSLog(@"Setting document root: %@", webLocalPath);
        
    }
    
    if (_httpServer && ![_httpServer isRunning]) {
        NSError *error;
        if([_httpServer start:&error]) {
            NSLog(@"start server success in port %d %@", [_httpServer listeningPort], [_httpServer publishedName]);
        } else {
            NSLog(@"启动失败");
        }
    }
    
}

- (void)stop {
    if (_httpServer && [_httpServer isRunning]) {
        [_httpServer stop];
    }
}

@end
```

2. 然后选择其启动时机，一般选择在 `AppDelegate` 中或 `WKWebView` 请求之前。
```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    
    //Setup WKWebView
    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
    WKUserContentController *controller = [[WKUserContentController alloc] init];
    configuration.userContentController = controller;
    configuration.processPool = [[WKProcessPool alloc] init];
    
    _wkWebView = [[WKWebView alloc] initWithFrame:self.view.bounds
                                    configuration:configuration];
    _wkWebView.navigationDelegate = self;
    _wkWebView.UIDelegate = self;
    
    [self.view addSubview:_wkWebView];
    
    //Start local web server
    [[LocalWebServerManager sharedInstance] start];
    
    //Local request which use local resource
    [self loadLocalRequest];
    
    //Remote request which use local resource
//    [self loadRemoteRequest];
}
```

3. 在 project工程 中引入相对资源目录（蓝色文件夹），在该目录中实现一个 `index.html` 和 `hi.js` 资源文件
```html
<!DOCTYPE html>
<html>
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<title>Hello</title>
	<script type="text/javascript" src="http://localhost:60000/hi.js"></script>
</head>
<body bgcolor="#4F8FFF">
    <center>
        <h1><br><br><br><br><br><br>恭喜你 服务器运行成功!</h1>
        <h5><br>点击下面按钮试一下</h5>
        <input type='button' value='调用本地js资源中的方法' onclick='invokeAlert()'/>
    </center>
</body>
</html>
```
```javascript
function invokeAlert() {
    alert('Perfect！')
}
```

4. 通过WKWebView访问 `http://localhost:60000/index.html` 即可。

通过以上4步，即可开启iOS本地http服务，通过WKWebView访问本地的html资源。

不过本文目的是 **“基于LocalWebServer实现WKWebView离线资源加载”**，所谓离线资源加载，指的是：页面资源在远端服务器，而页面中的部分资源在iOS本地沙盒中。
放在上面的例子上，也就是 `index.html` 应该在远程的 nginx 或 apache 等服务上运行，而 `hi.js` 应该存放在APP的资源目录之中。

此举是否可行呢？我们试试看。
将 `index.html` 改名为 `demo.html` 配置在 http://smallfan.net 中，直接进行访问
```objectivec
- (void)loadRemoteRequest {
    if (_wkWebView) {
        [_wkWebView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"http://smallfan.net/demo.html"]]];
    }
}
```
实验结果：可行。
也就是说，通过以下该方式实现：**HTML页面内资源允许请求本地服务器**。
```html
<script type="text/javascript" src="http://localhost:60000/hi.js"></script>
```

# 四、支持https
正如前面所提：支持ATS已经成为一个正确而有效的选择。
> 在 `Safari` 及 `Apple WebKit` 中：在https页面内，不允许http请求存在，否则一概会被block。

因此如果已经支持了https，那么页面中的 `http://localhost:60000/hi.js` 也必须 支持https才行。问题是，页面https我们可以采用CA颁发的合法证书进行双向或单向认证，而localhost并不是一个合法的host，也就是说我们需要为它实现一个自签名证书。

## 4.1 实现local server自签名证书
因为iOS开发使用MacOS，以下行为默认系统已经安装OpenSSL。

首先到 [OpenSSL官网](https://www.openssl.org/source/) 下载最新版本，解压后在`app`目录中找到`CA.sh`，拷贝到根目录，然后运行
```bash
% sh ~/CA.sh -newca
```
运行完后会生成一个demoCA的目录，里面存放了CA的证书和私钥，同时记住自己设置的授权密码。
创建一个目录
```bash
% mkdir server
```
然后创建一个私钥
```bash
% openssl genrsa -out server/server-key.pem 2048
```
创建证书请求
```bash
% openssl req -new -out server/server-req.csr -key server/server-key.pem
```
此时终端会要求你填写地区、姓名等信息，**Common Name** 这一项必须填 localhost 或者 127.0.0.1 ，如果填写127.0.0.1，那么页面中请求只能使用`https://127.0.0.1:60000` 而不能使用 `https://localhost:60000`，反之相同。

自签署证书
```bash
% openssl x509 -req -in server/server-req.csr -out server/server-cert.pem -signkey server/server-key.pem -CA demoCA/cacert.pem -CAkey demoCA/private/cakey.pem -CAcreateserial -days 3650
```
将证书导出成浏览器支持的.p12格式，记得导出密码（本文为: b123456）
```bash
% openssl pkcs12 -export -clcerts -in server/server-cert.pem -inkey server/server-key.pem -out server/server.p12
```
自此，自签名证书生成完成。

## 4.2 配置自签名证书
将p12文件导入 project工程 中的资源目录中，
同时新建一个 `MyHttpConnection` 继承于 `HttpConnection` ，重载 `- (BOOL)isSecureServer` 方法及 `sslIdentityAndCertificates` 方法。
```objectivec
#import "MyHTTPConnection.h"

@implementation MyHTTPConnection

- (BOOL)isSecureServer {
    return YES;
}

- (NSArray *)sslIdentityAndCertificates {
    
    SecIdentityRef identityRef = NULL;
    SecCertificateRef certificateRef = NULL;
    SecTrustRef trustRef = NULL;
    
    //p12文件资源路径
    NSString *thePath = [[NSBundle bundleWithPath:[[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent:@"Resource"]] pathForResource:@"localhost" ofType:@"p12"];
    NSData *PKCS12Data = [[NSData alloc] initWithContentsOfFile:thePath];
    CFDataRef inPKCS12Data = (__bridge CFDataRef)PKCS12Data;
    //p12文件导出密码
    CFStringRef password = CFSTR("b123456");
    const void *keys[] = { kSecImportExportPassphrase };
    const void *values[] = { password };
    CFDictionaryRef optionsDictionary = CFDictionaryCreate(NULL, keys, values, 1, NULL, NULL);
    CFArrayRef items = CFArrayCreate(NULL, 0, 0, NULL);
    
    OSStatus securityError = errSecSuccess;
    securityError =  SecPKCS12Import(inPKCS12Data, optionsDictionary, &items);
    if (securityError == 0) {
        CFDictionaryRef myIdentityAndTrust = CFArrayGetValueAtIndex (items, 0);
        const void *tempIdentity = NULL;
        tempIdentity = CFDictionaryGetValue (myIdentityAndTrust, kSecImportItemIdentity);
        identityRef = (SecIdentityRef)tempIdentity;
        const void *tempTrust = NULL;
        tempTrust = CFDictionaryGetValue (myIdentityAndTrust, kSecImportItemTrust);
        trustRef = (SecTrustRef)tempTrust;
    } else {
        NSLog(@"Failed with error code %d",(int)securityError);
        return nil;
    }
    
    SecIdentityCopyCertificate(identityRef, &certificateRef);
    NSArray *result = [[NSArray alloc] initWithObjects:(__bridge id)identityRef, (__bridge id)certificateRef, nil];
    
    return result;
}

@end
```
将 `HTTPConnetion.m` 的 `startConnection` 方法中
```objectivec
[settings setObject:(NSString *)kCFStreamSocketSecurityLevelNegotiatedSSL forKey:(NSString *)kCFStreamSSLLevel];
```
替换为
```objectivec
[settings setObject:@"2" forKey:GCDAsyncSocketSSLProtocolVersionMin];
[settings setObject:@"2" forKey:GCDAsyncSocketSSLProtocolVersionMax];
```
同时在启动 HTTPServer（调用 startServer ）之前，使用 `setConnectionClass` 方法将 `HTTPConnecdtion` 替换为 `MyHTTPConnection`
```objectivec
[_httpServer setConnectionClass:[MyHTTPConnection class]];
```
自此，https服务配置完成。
## 4.3 WKWebView证书配置
对于自签名证书，WKWebView需要在WKNavagationDelegate方法中允许使用：
```objectivec
-                   (void)webView:(WKWebView *)webView
didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
                completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler {
    
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        NSURLCredential *card = [[NSURLCredential alloc] initWithTrust:challenge.protectionSpace.serverTrust];
        completionHandler(NSURLSessionAuthChallengeUseCredential, card);
    }
}
```
## 4.4 ATS设置
在 `Info.plist` 中，找到 `App Transport Security Settings` 项，在其中增加一项 `Allow Arbitrary Loads in Web Content` 设为 `YES` 
```objectivec
<key>NSAppTransportSecurity</key>
	<dict>
		<key>NSAllowsArbitraryLoadsForMedia</key>
		<true/>
		<key>NSAllowsArbitraryLoadsInWebContent</key>
		<true/>
		<key>NSAllowsArbitraryLoads</key>
		<false/>
	</dict>
```

通过以上配置，即可实现 **https** 请求本地资源
```html
<script type="text/javascript" src="https://localhost:60000/hi.js"></script>
```
```objectivec
- (void)loadRemoteRequest {
    if (_wkWebView) {
        [_wkWebView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"https://smallfan.net/demo.html"]]];
    }
}

- (void)loadLocalRequest {
    if (_wkWebView) {
        [_wkWebView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:[NSString stringWithFormat:@"https://localhost:%ld/index.html", [[LocalWebServerManager sharedInstance] port] ] ]]];
    }
}
```
## 五、小结
实际上这种方式实现 WKWebView 离线加载，属于奇技淫巧。因为 Apple 封闭的生态，很多时候想优化一些既有的东西举步维艰，只能不断探索。
从这个角度来看，还是 **Python** 好玩点，23333 ... 

关于 **local web server** 中有些问题，笔者并未来得及作更深入研究，以下列举一些可能性问题，欢迎大家一起探讨：
+  资源访问权限安全问题
+  APP前后台切换时，服务重启性能耗时问题
+ 服务运行时，电量及CPU占有率问题
+ 多线程及磁盘IO问题

最后，丢个Demo出来，Biu...
Demo地址：[LocalWebServer](https://github.com/Smallfan/LocalWebServer)
