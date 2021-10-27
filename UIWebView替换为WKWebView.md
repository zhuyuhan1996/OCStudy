- didFailLoad = didFailNavigation
- didFinishLoad = didFinishNavigation
- didStartLoad = didStartPrivision
- shouldStartLoad = decidePolicyForNavigation

```
// OC:
// ref: https://stackoverflow.com/a/26583062 实现 scalesPageToFit 的效果

NSString *jScript = @"var meta = document.createElement('meta'); meta.setAttribute('name', 'viewport'); meta.setAttribute('content', 'width=device-width'); document.getElementsByTagName('head')[0].appendChild(meta);";
WKUserScript *wkUScript = [[WKUserScript alloc] initWithSource:jScript injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:YES];
WKUserContentController *wkUController = [[WKUserContentController alloc] init];
[wkUController addUserScript:wkUScript];
WKWebViewConfiguration *wkWebConfig = [[WKWebViewConfiguration alloc] init];
wkWebConfig.userContentController = wkUController;

// Swift:
let jsScript = "var meta = document.createElement('meta'); meta.setAttribute('name', 'viewport'); meta.setAttribute('content', 'width=device-width'); document.getElementsByTagName('head')[0].appendChild(meta);"
let wkUserScript = WKUserScript(source: jsScript, injectionTime: .atDocumentEnd, forMainFrameOnly: true)
let wkUserController = WKUserContentController()
wkUserController.addUserScript(wkUserScript)
let wkConfig = WKWebViewConfiguration().then {
    $0.userContentController = wkUserController
}
```

- http://liuyanwei.jumppo.com/2015/10/17/ios-webView.html
- https://stackoverflow.com/questions/37509990/migrating-from-uiwebview-to-wkwebview/37513449