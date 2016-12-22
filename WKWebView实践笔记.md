####  WKWebView 实践笔记

###### 1⃣️. 性能

>  UIWebView 这个自iOS2开始使用的网页加载器一直是开发的心病：加载速度慢，占用内存多，优化困难。如果加载网页多，还可能因为过量占用内存而给系统kill掉。

> WKWebView 适用于iOS8及iOS8以后。在性能、稳定性、功能方面有很大提升（最直观的体现就是加载网页是占用的内存，模拟器加载百度与开源中国网站时，WKWebView占用23M，而UIWebView占用85M）。


###### 2⃣️. webView请求成功失败等代理


```
#pragma mark - WKNavigationDelegate

// 页面开始加载时调用
- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation;
{
    NSLog(@"%s",__func__);
}

// 当内容开始返回时调用
- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation
{
    NSLog(@"%s",__func__);
}

// 页面加载完成之后调用
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation
{
    NSLog(@"%s",__func__);
}

// 页面加载失败时调用
- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(WKNavigation *)navigation
{
    NSLog(@"%s",__func__);
}

```


###### 3⃣️. webView弹框

> 在UIWebView中，网页代码编写的弹框，不需要我们自己处理，UIWebView自己就会自动处理。但是在WKWebView中，我们需要实现<WKUIDelegate>代理，并在代理中主动实现UIAletView或者UIAlertController。

<img src="emoji/smile" width="18"/>**温馨提示**<img src="emoji/smile" width="18"/>
[欢迎使用PublicMethodTool](https://github.com/CoderHJZhao/PublicMethodTool)
```
如果使用了WKWebView，还是建议使用UIAlertController，因为UIAletView弹出的时候会在缘由的keywindow上新建一个window,从而篡改跟控制器的效果，非常的不安全，笔者GitHub上的「PublicMethodTool」中已经有对UIAlertController的封装，大家可以参考使用。
```



以下是WKUIDelegate中要实现的方法，仅供参考！


```
#pragma mark - WKUIDelegate（提示窗口）

- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler();
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
    
}

- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL))completionHandler{
    //    DLOG(@"msg = %@ frmae = %@",message,frame);
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(NO);
    }])];
    [alertController addAction:([UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(YES);
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
}

- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:prompt message:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addTextFieldWithConfigurationHandler:^(UITextField * _Nonnull textField) {
        textField.text = defaultText;
    }];
    [alertController addAction:([UIAlertAction actionWithTitle:@"完成" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(alertController.textFields[0].text?:@"");
    }])];
    
}

```

###### 4⃣️. 拦截WebView中的请求URL


实现以下代理即可


```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler{
    //navigationAction.request.URL.absoluteString  这里就是WebView中请求的URL，大家可以按需截取URL后面跟的参数，WebView 中网页跳转都会经过这个方法，包括WebView第一次加载的时候。
}

```

###### 5⃣️️.WKWebView Cookie 写入

>  UIWebView Cookie: 同一个应用，不同UIWebView之间的Cookie是自动同步的。并.     且可以被其他网络类访问比如NSURLConnection,AFNetworking。它们都是保存在NSHTTPCookieStorage容器中。 当UIWebView加载一个URL的时候，在加载完成时候，Http Response，对Cookie进行写入,更新或者删除，结果更新Cookie到NSHTTPCookieStorage存储容器中。[request setHTTPShouldHandleCookies:YES];


> WKWebView不会再从NSHTTPCookieStorage取cookie，而是需要在初始化时通过WKUserScript写入。

在讲接下来的内容之前，先熟悉下这个类
```
WKUserScript：在WKUserContentController中，所有使用到WKUserScript。WKUserContentController是用于与JS交互的类，而所注入的JS是WKUserScript对象。
```
```


WKWebView Cookie 写入 三种方式

JS注入1（推荐）
```
WKUserContentController* userContentController = WKUserContentController.new;
WKUserScript * cookieScript = [[WKUserScript alloc] initWithSource: @"document.cookie ='TeskCookieKey1=TeskCookieValue1';document.cookie = 'TeskCookieKey2=TeskCookieValue2';"injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
 
[userContentController addUserScript:cookieScript];
WKWebViewConfiguration* webViewConfig = WKWebViewConfiguration.new;
webViewConfig.userContentController = userContentController;
WKWebView * webView = [[WKWebView alloc] initWithFrame:CGRectMake(/*set your values*/) configuration:webViewConfig];
```

JS注入2

```
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    [webView evaluateJavaScript:@"document.cookie ='TeskCookieKey1=TeskCookieValue1';" completionHandler:^(id result, NSError *error) {
        //...
    }];
}
```

NSMutableURLRequest（测试无效，暂时不清楚什么原因）

```
NSMutableURLRequest *request= [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http://dev.skyfox.org/cookie.php"]];
//[request setHTTPShouldHandleCookies:YES];
[request setValue:[NSString stringWithFormat:@"%@=%@",@"testwkcookie", @"testwkcookievalue"] forHTTPHeaderField:@"Cookie"];
```

注意：JS注入的Cookie,比如PHP代码在Cookie容器中取是取不到的,直接js document.cookie能读取到

NSMutableURLRequest 注入的PHP等动态语言直接能从$_COOKIE对象中获取到

###### 6⃣️️️.WKWebView Cookie 读取


```
WKWebsiteDataStore在iOS 9和OS X 10.11中引入，是一个新的API，它用于管理一个网站站点存储的数据，例如cookies，它是你网页的 WKWebViewConfiguration上的一个可读写的属性。你可以根据类型或者时间来删除数据，例如cookies和缓存，你可以用非持久性数 据存储来改变配置。



- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation{
    WKWebsiteDataStore *dataStore = [WKWebsiteDataStore defaultDataStore];
    [dataStore fetchDataRecordsOfTypes:[WKWebsiteDataStore allWebsiteDataTypes]
                     completionHandler:^(NSArray<WKWebsiteDataRecord *> * __nonnull records) {
                         for (WKWebsiteDataRecord *record  in records)
                         {
                             NSLog(@"WKWebsiteDataRecord:%@",[record description]);
                         }
                     }];
    
}
```

###### 7⃣️.结合WKWebViewJavascriptBridge的使用，实现OC和网页交互


初始化方法


```
 //配置WKWebViewJavascriptBridge
    [WKWebViewJavascriptBridge enableLogging];
    //和webview建立桥接
    _bridge = [WKWebViewJavascriptBridge bridgeForWebView:self.webView];
    //设置代理
    [_bridge setWebViewDelegate:self];
    //加载网页
    [webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"www.baidu.com"]]];

```

JS调用OC


```
//首先注册Js方法requestData，然后回调中调用本地OC方法
    [_bridge registerHandler:@"APPJsBridge.requestData" handler:^(id data, WVJBResponseCallback responseCallback) {
    //调用成功回调本地的OC方法
        
    }];

```

OC调用JS

```
//OC调用JS方法，发送数据给网页
 [_bridge callHandler:@"APPJsBridge.requestData" data:data responseCallback:^(id responseData) {
       //调用成功回调，具体看前端如何定义逻辑
    }];
```

（更多iOS开发干货，欢迎关注  [微博@3W_狮兄 ](http://weibo.com/hanjunzhao/) ）

----------
Posted by  [微博@3W_狮兄 ](http://weibo.com/hanjunzhao/))  
原创文章，版权声明：自由转载-非商用-非衍生-保持署名 |


