---
title: iOS 中的单例
date: 2017-09-30 10:18:45
tags: iOS
---

优点：
1、提供了对唯一实例的受控访问。
2、由于在系统内存中只存在一个对象，因此可以节约系统资源，对于一些需要频繁创建和销毁的对象单例模式无疑可以提高系统的性能。
3、允许可变数目的实例。

缺点：
1、由于单利模式中没有抽象层，因此单例类的扩展有很大的困难。
2、单例类的职责过重，在一定程度上违背了“单一职责原则”。
3、滥用单例将带来一些负面问题，如为了节省资源将数据库连接池对象设计为的单例类，可能会导致共享连接池对象的程序过多而出现连接池溢出；如果实例化的对象长时间不被利用，系统会认为是垃圾而被回收，这将导致对象状态的丢失.

以下是相对严谨的做法

```
#import <Foundation/Foundation.h>

@interface Person : NSObject<NSCopying, NSMutableCopying>
+ (instancetype)sharePerson;
@end
```

```objective-c
#import "Person.h"

//全局
static Person *_person;

@implementation Person

//初始化方法
- (instancetype)init{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _person = [super init];
    });
    return _person;
}

//单例方法
+ (instancetype)sharePerson{
    return [[self alloc] init];
}

//alloc会调用allocWithZone，确保使用同一块内存地址
+ (instancetype)allocWithZone:(struct _NSZone *)zone{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _person = [super allocWithZone:zone];
    });
    return _person;
}

//copy的时候会调用copyWithZone
- (id)copyWithZone:(NSZone *)zone{
    return _person;
}

+ (id)copyWithZone:(struct _NSZone *)zone{
    return _person;
}

+ (id)mutableCopyWithZone:(struct _NSZone *)zone{
    return _person;
}

- (id)mutableCopyWithZone:(NSZone *)zone{
    return _person;
}
Person *person1 = [[Person alloc]init];
    Person *person2 = [Person sharePerson];
    Person *person3 = [[Person alloc]init];
    Person *person4 = [person3 mutableCopy];
    Person *person5 = [person4 copy];
    NSLog(@"%p %p %p %p %p", person1, person2, person3, person4, person5);
// 0x170001ad0 0x170001ad0 0x170001ad0 0x170001ad0 0x170001ad0
```