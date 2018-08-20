---
title: iOS UIPanGestureRecognizer 判断手势滑动方向
date: 2017-08-18 23:14:27
tags: iOS
---

> UIPanGestureRecognizer 判断手势滑动方向 , 可用于导航栏缩放等



```objective-c
- (void)addGestureWithView:(UIView *)view{
    UIPanGestureRecognizer *recognizer = [[UIPanGestureRecognizer alloc]initWithTarget:self action:@selector(handlePan:)];
    [self.view addGestureRecognizer:recognizer];
}

/** 平移手势响应事件  */
- (void)handlePan:(UIPanGestureRecognizer *)swipe {
    if (swipe.state == UIGestureRecognizerStateChanged) {
        [self commitTranslation:[swipe translationInView:self.view]];
    }
}

/** 判断手势方向  */
- (void)commitTranslation:(CGPoint)translation {
    CGFloat absX = fabs(translation.x);
    CGFloat absY = fabs(translation.y);
    // 设置滑动有效距离
    if (MAX(absX, absY) < 10)
        return;
    if (absX > absY ) {
        if (translation.x<0) {//向左滑动
        }else{//向右滑动
        }
    } else if (absY > absX) {
        if (translation.y<0) {//向上滑动
        }else{ //向下滑动
        }
    }
}
```

