---
title: iOS AOP：事务拦截 异常处理 HotFix
date: 2018-08-20 23:01:58
tags: Refactoring
---



**AOP**（ Aspect-OrientedProgramming）：  面向切面编程

**何为面向切面编程？**

在运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程。



**OOP**（Object-oriented programming）：面向对象编程

**何为面向对象编程？**

面向对象编程是一种实现的方法，在这种方法中，程序被组织成许多组互相协作的对象，每个对象代表某个类的一个实例，而类则属于一个通过继承关系形成的层次结构。



**AOP&OOP**

面向对象的特点是继承、多态和封装。而封装就要求将功能分散到不同的对象中去，这在软件设计中往往称为职责分配。实际上也就是说，让不同的类设计不同的方法。这样代码就分散到一个个的类中去了。这样做的好处是降低了代码的复杂程度，使类可重用。

一般而言，我们管切入到指定类指定方法的代码片段称为切面，而切入到哪些类、哪些方法则叫切入点。有了AOP，我们就可以把几个类共有的代码，抽取到一个切片中，等到需要时再切入对象中去，从而改变其原有的行为。

这样看来，AOP其实只是OOP的补充而已。OOP从横向上区分出一个个的类来，而AOP则从纵向上向对象中加入特定的代码。有了AOP，OOP变得立体了。



#### 事务拦截，安全可变容器

iOS中有各类容器的概念，容器分可变容器和非可变容器，可变容器一般内部在实现上是一个链表，在进行各类（insert 、remove、 delete、 update ）难免有空操作、指针越界的问题。
最粗暴的方式就是在使用可变容器的时间，每次操作都必须手动做空判断、索引比较这些操作:

```objective-c
 NSMutableDictionary *dic = [[NSMutableDictionary alloc] init];
    if (obj) {
        [dic setObject:obj forKey:@"key"];
    }

 NSMutableArray *array = [[NSMutableArray alloc] init];
    if (index < array.count) {
        NSLog(@"%@",[array objectAtIndex:index]);
    }
```

在代码中大量的使用这鞋操作实在是太过于繁琐了，试想如果可变容器自身如何能做这些兼容岂不是更好。可能会想到继承的方法来解决，但是项目中尽可能的避免过多的派生（至于派生的弊端这里就不多说了）；或者想到分类，这里也不尽人意。

Method Swizzling 移花接木

runtime 这里就不多多说了（swift里面已经对这个概念的说法从心转变成了 Reflection<反射>），objective c中每个方法的名字(SEL)跟函数的实现(IMP)是一一对应的，Swizzle的原理只是在这个地方做下手脚，将原来方法名与实现的指向交叉处理，就能达到一个新的效果。

废话少说，直接上代码：

这里使用NSMutableArray 做实例，为NSMutableArray追加一个新的方法

```
@implementation NSMutableArray (safe)

+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        id obj = [[self alloc] init];
        [obj swizzleMethod:@selector(addObject:) withMethod:@selector(safeAddObject:)];
        [obj swizzleMethod:@selector(objectAtIndex:) withMethod:@selector(safeObjectAtIndex:)];
        [obj swizzleMethod:@selector(insertObject:atIndex:) withMethod:@selector(safeInsertObject:atIndex:)];
        [obj swizzleMethod:@selector(removeObjectAtIndex:) withMethod:@selector(safeRemoveObjectAtIndex:)];
        [obj swizzleMethod:@selector(replaceObjectAtIndex:withObject:) withMethod:@selector(safeReplaceObjectAtIndex:withObject:)];
    });
}

- (void)safeAddObject:(id)anObject
{
    if (anObject) {
        [self safeAddObject:anObject];
    }else{
        NSLog(@"obj is nil");
        
    }
}

- (id)safeObjectAtIndex:(NSInteger)index
{
    if(index<[self count]){
        return [self safeObjectAtIndex:index];
    }else{
        NSLog(@"index is beyond bounds ");
    }
    return nil;
}
```

```objective-c
- (void)swizzleMethod:(SEL)origSelector withMethod:(SEL)newSelector
{
    Class class = [self class];
    
    Method originalMethod = class_getInstanceMethod(class, origSelector);
    Method swizzledMethod = class_getInstanceMethod(class, newSelector);
    
    BOOL didAddMethod = class_addMethod(class,
                                        origSelector,
                                        method_getImplementation(swizzledMethod),
                                        method_getTypeEncoding(swizzledMethod));
    if (didAddMethod) {
        class_replaceMethod(class,
                            newSelector,
                            method_getImplementation(originalMethod),
                            method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

这里唯一可能需要解释的是 class_addMethod 。要先尝试添加原 selector 是为了做一层保护，因为如果这个类没有实现 originalSelector ，但其父类实现了，那 class_getInstanceMethod 会返回父类的方法。这样 method_exchangeImplementations 替换的是父类的那个方法，这当然不是你想要的。所以我们先尝试添加 orginalSelector ，如果已经存在，再用 method_exchangeImplementations 把原方法的实现跟新的方法实现给交换掉。

safeAddObject 代码看起来可能有点奇怪，像递归不是么。当然不会是递归，因为在 runtime 的时候，函数实现已经被交换了。调用 objectAtIndex: 会调用你实现的 safeObjectAtIndex:，而在 NSMutableArray: 里调用 safeObjectAtIndex: 实际上调用的是原来的 objectAtIndex: 。

如此以来，一直担心的问题就迎刃而解了，不仅在可变数组、可变字典等容器内都可以做自己想做的事情。



#### Aspects 一个基于Objective-c的AOP开发框架



* **Aspects 使用姿势：**

```objective-c
[UIViewController aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {

    NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);

} error:NULL];

```

前插、后插、替换某个方法都可以。使用类的方式很简单，NSClassFromString 即可，Selector 也一样 NSSelectorFromString，这样就能通过外部传入 String，内部动态构造 Class 和 Selector 来达到 Fix 的效果了。

这种方式的安全性在于：

1. 不需要中间 JS 文件，准备工作全部在 Native 端完成。
2. 没有使用 App Store 不友好的类/方法。



假设线上运行这这样一个 Class，由于疏忽，没有对参数做检查，导致特定情况下会 Crash。

```objective-c
@interface MightyCrash: NSObject
- (float)divideUsingDenominator:(NSInteger)denominator;
@end
@implementation MightyCrash
// 传一个 0 就 gg 了
- (float)divideUsingDenominator:(NSInteger)denominator{
    return 1.f/denominator;
}
@end
```



现在我们要避免 Crash，就可以通过这种方式来修复

```objective-c
[Felix fixIt];
NSString *fixScriptString = @"\
fixInstanceMethodReplace('MightyCrash', 'divideUsingDenominator:', function(instance, originInvocation, originArguments){\
    if (originArguments[0] == 0) {\
        console.log('zero goes here');\
    } else {\
        runInvocation(originInvocation);\
    }\
});\
";
[Felix evalString:fixScriptString];
```



运行一下看看

```objective-c
MightyCrash *mc = [[MightyCrash alloc] init];
float result = [mc divideUsingDenominator:3];
NSLog(@"result: %.3f", result);
result = [mc divideUsingDenominator:0];
NSLog(@"won't crash");

// output
// result: 0.333
// Javascript log: zero goes here
// won't crash
```



It Works, 是不是有那么点意思了。以下是可以正常运行的代码，仅供参考。

```objective-c
#import <Aspects.h>
#import <objc/runtime.h>
#import <JavaScriptCore/JavaScriptCore.h>

@interface Felix: NSObject
+ (void)fixIt;
+ (void)evalString:(NSString *)javascriptString;
@end

@implementation Felix
+ (Felix *)sharedInstance
{
    static Felix *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}
+ (void)evalString:(NSString *)javascriptString
{
    [[self context] evaluateScript:javascriptString];
}
+ (JSContext *)context
{
    static JSContext *_context;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _context = [[JSContext alloc] init];
        [_context setExceptionHandler:^(JSContext *context, JSValue *value) {
            NSLog(@"Oops: %@", value);
        }];
    });
    return _context;
}
+ (void)_fixWithMethod:(BOOL)isClassMethod aspectionOptions:(AspectOptions)option instanceName:(NSString *)instanceName selectorName:(NSString *)selectorName fixImpl:(JSValue *)fixImpl {
    Class klass = NSClassFromString(instanceName);
    if (isClassMethod) {
        klass = object_getClass(klass);
    }
    SEL sel = NSSelectorFromString(selectorName);
    [klass aspect_hookSelector:sel withOptions:option usingBlock:^(id<AspectInfo> aspectInfo){
        [fixImpl callWithArguments:@[aspectInfo.instance, aspectInfo.originalInvocation, aspectInfo.arguments]];
    } error:nil];
}
+ (id)_runClassWithClassName:(NSString *)className selector:(NSString *)selector obj1:(id)obj1 obj2:(id)obj2 {
    Class klass = NSClassFromString(className);
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    return [klass performSelector:NSSelectorFromString(selector) withObject:obj1 withObject:obj2];
#pragma clang diagnostic pop
}
+ (id)_runInstanceWithInstance:(id)instance selector:(NSString *)selector obj1:(id)obj1 obj2:(id)obj2 {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    return [instance performSelector:NSSelectorFromString(selector) withObject:obj1 withObject:obj2];

#pragma clang diagnostic pop
}
+ (void)fixIt
{
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionBefore instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionInstead instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionAfter instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionBefore instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionInstead instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionAfter instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };
    self context = ^id(NSString *className, NSString *selectorName) {
        return [self _runClassWithClassName:className selector:selectorName obj1:nil obj2:nil];
    };
    self context = ^id(NSString *className, NSString *selectorName, id obj1) {
        return [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:nil];
    };
    self context = ^id(NSString *className, NSString *selectorName, id obj1, id obj2) {
        return [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:obj2];
    };
    self context = ^(NSString *className, NSString *selectorName) {
        [self _runClassWithClassName:className selector:selectorName obj1:nil obj2:nil];
    };
    self context = ^(NSString *className, NSString *selectorName, id obj1) {
        [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:nil];
    };
    self context = ^(NSString *className, NSString *selectorName, id obj1, id obj2) {
        [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:obj2];
    };
    self context = ^id(id instance, NSString *selectorName) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:nil obj2:nil];
    };
    self context = ^id(id instance, NSString *selectorName, id obj1) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:nil];
    };
    self context = ^id(id instance, NSString *selectorName, id obj1, id obj2) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:obj2];
    };
    self context = ^(id instance, NSString *selectorName) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:nil obj2:nil];
    };
    self context = ^(id instance, NSString *selectorName, id obj1) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:nil];
    };
    self context = ^(id instance, NSString *selectorName, id obj1, id obj2) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:obj2];
    };
    self context = ^(NSInvocation *invocation) {
        [invocation invoke];
    };
    // helper
    [[self context] evaluateScript:@"var console = {}"];
    self context[@"log"] = ^(id message) {
        NSLog(@"Javascript log: %@",message);
    };
}
@end
```



* **业务埋点、日志打印分离**

重构代码时经常会从一些问题入手，例如轻量级controller、MVVM等，这些无非是对原有逻辑进一步抽象、区分、分离，重新抽象数据模型、viewmodel；相关代码放入分类；考虑业务层次抽取剥离父类；mananger、factory等。经历一大翻工作controller 中代码终于减少了，但是仍旧留下一堆的埋点、日志log的相关代码。

Aspects是一个很不错的 AOP 库，封装了 Runtime ， Method Swizzling 这些黑色技巧，只提供两个简单的API：

```
+ (id<aspecttoken>)aspect_hookSelector:(SEL)selector
                         withOptions:(AspectOptions)options
                      usingBlock:(id)block
                           error:(NSError **)error;
- (id<aspecttoken>)aspect_hookSelector:(SEL)selector
                     withOptions:(AspectOptions)options
                      usingBlock:(id)block
                           error:(NSError **)error;

//使用 Aspects 提供的 API，我们之前的例子会进化成这个样子

@implementation UIViewController (Logging)+ (void)load
{
   [UIViewController aspect_hookSelector:@selector(viewDidAppear:)
                             withOptions:AspectPositionAfter
                              usingBlock:^(id<aspectinfo> aspectInfo) {        NSString *className = NSStringFromClass([[aspectInfo instance] class]);
       [Logging logWithEventName:className];
                              } error:NULL];
}
```

相对来说如果想要捕捉到viewDidAppear 的log打印，或者是页面PV的统计上报，我们从原有的业务中将这部分代码剥离出来，掉换IMP指向以后我们可以在usingBlock 内做自己想做的事情， 这样就能达到上述想要的目的了。

* **Aspects扩展使用：**

页面的PV统计，事件点击统计，可以事先写在配置文件里面：

```
{        @"MainViewController": @{
           GLLoggingPageImpression: @"page imp - main page",
           GLLoggingTrackedEvents: @[
               @{
                   GLLoggingEventName: @"button one clicked",
                   GLLoggingEventSelectorName: @"buttonOneClicked:",
                   GLLoggingEventHandlerBlock: ^(id<aspectinfo> aspectInfo) {
                       [Logging logWithEventName:@"button one clicked"];
                   },
               },
               @{
                   GLLoggingEventName: @"button two clicked",
                   GLLoggingEventSelectorName: @"buttonTwoClicked:",
                   GLLoggingEventHandlerBlock: ^(id<aspectinfo> aspectInfo) {
                       [Logging logWithEventName:@"button two clicked"];
                   },
               },
          ],
       },        @"DetailViewController": @{
           GLLoggingPageImpression: @"page imp - detail page",
       }
@implementation AppDelegate (Logging)
+ (void)setupLogging{
     [AppDelegate setupWithConfiguration:config];
}

+ (void)setupWithConfiguration:(NSDictionary *)configs
{    // Hook Page Impression
   [UIViewController aspect_hookSelector:@selector(viewDidAppear:)
                             withOptions:AspectPositionAfter
                              usingBlock:^(id<aspectinfo> aspectInfo) {                                       NSString *className = NSStringFromClass([[aspectInfo instance] class]);
                                   [Logging logWithEventName:className];
                              } error:NULL];    // Hook Events
   for (NSString *className in configs) {
       Class clazz = NSClassFromString(className);        NSDictionary *config = configs[className];        if (config[GLLoggingTrackedEvents]) {            for (NSDictionary *event in config[GLLoggingTrackedEvents]) {
               SEL selekor = NSSelectorFromString(event[GLLoggingEventSelectorName]);
               AspectHandlerBlock block = event[GLLoggingEventHandlerBlock];
               [clazz aspect_hookSelector:selekor
                              withOptions:AspectPositionAfter
                               usingBlock:^(id<aspectinfo> aspectInfo) {
                                   block(aspectInfo);
                               } error:NULL];
           }
       }
   }
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {    // Override point for customization after application launch.
   [self setupLogging];    return YES;
}
```