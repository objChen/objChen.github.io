---
title: iOS 清除UIWebView缓存
date: 2017-06-17 20:40:45
tags: iOS
---



> 清除webview cookies&缓存



//清除cookies

```objective-c
NSHTTPCookie *cookie;
NSHTTPCookieStorage *storage = [NSHTTPCookieStorage sharedHTTPCookieStorage];
for (cookie in [storage cookies]){
	[storage deleteCookie:cookie];
}

```



//    清除webView的缓存

```objective-c
- (void)cleanCache{
//清除UIWebView的缓存
NSURLCache *cache = [NSURLCache sharedURLCache];
	[cache removeAllCachedResponses];
	[cache setDiskCapacity:0];
	[cache setMemoryCapacity:0];
}

```

