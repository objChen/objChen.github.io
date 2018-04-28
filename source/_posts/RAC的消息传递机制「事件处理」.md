---
title: RAC的消息传递机制「事件处理」
date: 2018-04-26 14:11:28
tags:
---



> ReactiveCocoa事件处理

#### Event(按钮点击)

```objective-c
[[button rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(__kindof UIControl * _Nullable x) {
    NSLog(@"button click");
}];
```

<!-- more -->

#### KVO

```objective-c
[[View rac_valuesAndChangesForKeyPath:@"x" options:NSKeyValueObservingOptionNew observer:nil] subscribeNext:^(id x) {
    NSLog(@"view的x值发生了改变");
}];

// 通过RAC提供的宏快速实现观察者模式
// RACObserve(self, name):监听某个对象的某个属性,返回的是信号。
[RACObserve(View,x) subscribeNext:^(id x) {
    NSLog(@"view的x值发生了改变");
}];
```

#### Notification

```objective-c
[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillShowNotification object:nil] subscribeNext:^(id x) {
    NSLog(@"键盘将要出现");
}];
```

#### TextField

```objective-c
// 监听文本框的文字改变
[[_textField rac_textSignal] subscribeNext:^(NSString *x) {
    NSLog(@"文本框文字发生了改变：%@",x);
}];
    
// 通过RAC提供的宏快速实现textSingel的监听
// RAC(TARGET, [KEYPATH, [NIL_VALUE]]):用于给某个对象的某个属性绑定。
// 当textField的文字发生改变时，label的文字也发生改变
RAC(self.textLabel,text) = self.textField.rac_textSignal;
```

#### TapGestureRecognizer

```objective-c
// 监听手势
UITapGestureRecognizer *tapGesture = [[UITapGestureRecognizer alloc] init];
[[tapGesture rac_gestureSignal] subscribeNext:^(id x) {
    NSLog(@"view被触发tap手势");
}];
[view addGestureRecognizer:tapGesture];
```

#### 过滤器filter

```objective-c
// 过滤器
[[self.textField.rac_textSignal filter:^BOOL(NSString *value) {
    return value.length >= 3;
}] subscribeNext:^(id x) {
    NSLog(@"%@",x);
}];
```

#### 定时器

- 延时执行

```objective-c
[[RACScheduler mainThreadScheduler]afterDelay:5 schedule:^{
   NSLog(@"五秒后执行一次");
}];
```

- 定时执行

```
//每一秒执行一次,这里要加上释放信号,否则控制器推出后依旧会执行,看具体需求吧
[[[RACSignal interval:1 onScheduler:[RACScheduler scheduler]]takeUntil:self.rac_willDeallocSignal ] subscribeNext:^(id x) {
    NSLog(@"%@",[NSDate date]);
}];
```

