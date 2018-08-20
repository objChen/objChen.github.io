---
title: iOS NSSetUncaughtExceptionHandler
date: 2018-04-23 21:52:04
tags: iOS
---

Foundation里面提供了一个NSSetUncaughtExceptionHandler函数，可以设置一个顶层异常处理函数，让我们在程序发生异常并终止前，有最后的机会来捕获并输出异常信息，如下代码所示。

```
void UncaughtExceptionHandler(NSException *exception) {
    
    NSArray *symbols = [exception callStackSymbols];
    NSString *reason = [exception reason];
    NSString *name = [exception name];
    
    NSLog(@"reason = %@", reason);
    NSLog(@"name = %@", name);
    NSLog(@"symbols = %@", symbols);
}

@implementation AppDelegate


- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);
    
    return YES;
}

@end
```

这个函数的参数是一个函数指针，指向的函数其签名是：void NSUncaughtExceptionHandler(NSException *exception)。可以看到这个函数有参数是一个NSException对象，通过这个对象我们就可以获取到异常的信息。假定发生数组越界异常时，会有如下输出。

```
2016-09-20 11:55:36.719 Test111[5548:199035] reason = *** -[__NSSingleObjectArrayI objectAtIndex:]: index 10 beyond bounds [0 .. 0]
2016-09-20 11:55:36.720 Test111[5548:199035] name = NSRangeException
2016-09-20 11:55:36.720 Test111[5548:199035] symbols = (
	0   CoreFoundation                      0x0000000106cef34b __exceptionPreprocess + 171
	1   libobjc.A.dylib                     0x000000010675021e objc_exception_throw + 48
	2   CoreFoundation                      0x0000000106d47bdf -[__NSSingleObjectArrayI objectAtIndex:] + 111
	3   Test111                             0x000000010617d87b -[AppDelegate application:didFinishLaunchingWithOptions:] + 235
	4   UIKit                               0x000000010710968e -[UIApplication _handleDelegateCallbacksWithOptions:isSuspended:restoreState:] + 290
	5   UIKit                               0x000000010710b013 -[UIApplication _callInitializationDelegatesForMainScene:transitionContext:] + 4236
	6   UIKit                               0x00000001071113b9 -[UIApplication _runWithMainScene:transitionContext:completion:] + 1731
	7   UIKit                               0x000000010710e539 -[UIApplication workspaceDidEndTransaction:] + 188
	8   FrontBoardServices                  0x000000010a2ee76b __FBSSERIALQUEUE_IS_CALLING_OUT_TO_A_BLOCK__ + 24
	9   FrontBoardServices                  0x000000010a2ee5e4 -[FBSSerialQueue _performNext] + 189
	10  FrontBoardServices                  0x000000010a2ee96d -[FBSSerialQueue _performNextFromRunLoopSource] + 45
	11  CoreFoundation                      0x0000000106c94311 __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17
	12  CoreFoundation                      0x0000000106c7959c __CFRunLoopDoSources0 + 556
	13  CoreFoundation                      0x0000000106c78a86 __CFRunLoopRun + 918
	14  CoreFoundation                      0x0000000106c78494 CFRunLoopRunSpecific + 420
	15  UIKit                               0x000000010710cdb6 -[UIApplication _run] + 434
	16  UIKit                               0x0000000107112f34 UIApplicationMain + 159
	17  Test111                             0x000000010617db4f main + 111
	18  libdyld.dylib                       0x000000010928a68d start + 1
	19  ???                                 0x0000000000000001 0x0 + 1
)
```

不过这个函数有效范围局限于异常，还有很多错误是无法处理的，如EXC_BAD_ACCESS内存访问错误，这类错误抛出的是Signal，需要专门做Signal处理。