---
title: 聊聊iOS中的PromiseKit
date: 2018-04-20 21:38:20
tags: Refactoring
---



> 项目中使用Block频次很高，经常用到回调闭包多级的嵌套，所以最近对Promise比较感兴趣。
>
> 很巧，今天技术群里有人抛出一个问题。一个非常常见的需求，如果是你，你会怎么做？会不会使用一种更优雅的方式？



PM：我们要实现一个下载音频的功能，下载前需要做这些处理。

1. 如果网络异常，直接提示网络异常并退出下载；
2. 如果是4G网络，需要弹窗提醒用户选择是否继续下载，用户点击下载后进入下载流程，取消后直接退出下载；
3. 如果 Wi-Fi 直接进入下载；
4. 下载时需要获取下载地址，获取到下载地址后，进入下载；
5. 下载完成需要刷新UI；

<!-- more -->

梳理完逻辑后是这样的：

![PromiseKitR](https://raw.githubusercontent.com/objChen/StaticResources/master/PromiseKitResources.png)

这个好办，不一会就写完了，结果是这样的

```objective-c
- (void)downloadCallBack:(void(^)(BOOL isSuccess))callback {
    if ([self isBadNetwork]){
        callback(NO);
        return;
    }
    if ([self is4GNetwork]){
        [self showAlert:^(NSInteger index) {
            if (index == 1){
                [self requestDownloadUrl:^(NSString *url) {
                    [self startDownloadWithURLString:url callback:^(BOOL isSuccsee) {
                        callback(YES);
                    }];
                }];
            }
        }];
    }else {
        [self requestDownloadUrl:^(NSString *url) {
            [self startDownloadWithURLString:url callback:^(BOOL isSuccsee) {
                callback(YES);
            }];
        }];
    }
}
```

这种写法主要有回调地狱和代码冗余问题，后期维护困难。

何为优雅？如何优雅？

Promise！

```objective-c
- (void)downloadForPromiseCallBack:(void(^)(BOOL isSuccess))callback {
    // 检查是否可以下载
    AnyPromise *checkPromise = [self checkCanDownloadPromise];
    checkPromise.then(^(){
        // 请求下载地址
        return [self requestUrlPromise];
    }).then(^(NSString *downloadUrl){
        // 开始下载音频
        return [self downloadPromiseWithURLString:downloadUrl];
    }).then(^(){
        // 下载成功
        if (callback){
            callback(YES);
        }
    }).catch(^(NSError *error){
        // 所有的异常
        callback(NO);
    });
}
```

#### Promise背景简介

Promise最初被提出是在 [E语言](http://erights.org/elib/distrib/pipeline.html)中， 它是基于并列/并行处理设计的一种编程语言。

Promise更多应用于 JavaScript，主要是为了解决 JavaScript 的 Callback Hell(回调地狱) 问题。 简单来说就是， JS 中很异步函数都是用 callback 的形式返回结果，而开发者经常会连续的使用这些异步调用，最后就导致回调嵌套的层级越来越深，严重影响代码的结构和可读性。Promise 的出现，就是用来解决 Callback Hell 这个问题的(用线性调用的接口来封装回调嵌套)。

#### Promise应用

- GCD异步下载图片

当我们需要异步下载一张图片并赋给Image 用GCD实现，请看下面这一坨OC：

```objective-c
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
    NSData * imageData = [NSData dataWithContentsOfURL:imgUrl];
    dispatch_async(dispatch_get_main_queue(), ^{
        imgv.image = [UIImage imageWithData:imageData];
    });
});
```

用 Promise 中的 dispatch_promise ：

```objective-c
dispatch_promise(^{
    return [NSData dataWithContentsOfURL:imgUrl];
}).then(^(NSData * imageData){
    imgv.image = [UIImage imageWithData:imageData];
});
```

dispatch_promise做了什么事情？看源码不难发现dispatch_promise中为我们建好了队列并在then中返回了MainThread。

- 用catch处理Error

让我们再看看文章开头的那个例子

```
- (void)downloadForPromiseCallBack:(void(^)(BOOL isSuccess))callback {
    // 检查是否可以下载
    AnyPromise *checkPromise = [self checkCanDownloadPromise];
    checkPromise.then(^(){
        // 请求下载地址
        return [self requestUrlPromise];
    }).then(^(NSString *downloadUrl){
        // 开始下载音频
        return [self downloadPromiseWithURLString:downloadUrl];
    }).then(^(){
        // 下载成功
        if (callback){
            callback(YES);
        }
    }).catch(^(NSError *error){
        // 所有的异常
        callback(NO);
    });
}
```

在之前需要在检查是否可以下载，请求下载地址和开始下载中分别对Error进行处理

catch是怎样做的？在传递promise的链中，一旦中间任何一环产生了错误，都会传递到catch去执行Error Handler。

- always

```objective-c
[self login].then(^(){
    if (callback){
       callback(YES);
    }
}).catch(^(NSError *error){
    if (callback){
       callback(NO);
    }
}).always(^{
    // 结束
    NSLog(@"over");
});
```

在所有的then和catch执行完还会执行always，根据业务选择是否实现。

- 用PMKWhen处理多异步任务作为前置任务

比如有一个这样的需求，需要从服务端下载两张图片，然后比较图片的size

```objective-c
PMKWhen(@[[self downloadFirstPicture], [self downloadSecondPicture]]).then(^(NSArray *results){
    [self comparePictureSizeWithFirstPicture:results.firstObject
                              sencondPicture:results.lastObject];
}).catch(^{
    // Processing error
});
```

PMKWhen传入数组，数组中存放promise，当数组中所有promise执行完毕才会执行then。

```objective-c
PMKWhen(@{@"firstPicture":[self downloadFirstPicture],
          @"secondPicture":[self downloadSecondPicture]
         }).then(^(NSArray *results){
  [self comparePictureSizeWithFirstPicture:results[@"firstPicture"]
                              sencondPicture:results[@"secondPicture"]];
}).catch(^{
  // Processing error
});
```

PMKWhen还可以传入存放prmise的字典。



##### 一直在追求如何更有效率的开展工作

项目中使用block频次之高，PromiseKit无疑是最优雅的处理CallBack hell的方式之一，学习了PromiseKit源码思想感触很深。异步问题，回调嵌套，回调地狱一直都是项目中的痛点，通过PromiseKit可以相对优雅的解决，使项目逻辑更加清晰。



