---
title: iOS UIImageView gif
date: 2018-04-02 22:12:04
tags: iOS
---

UIImageView显示gif图片有两种方式。当然前提都是先将gif中的每一帧取出来放到一个个UIImage对象中，将这些对象放到一个数组中，如下代码所示。

```
CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)data, NULL);
size_t count = CGImageSourceGetCount(source);
    
NSMutableArray *images = [NSMutableArray array];
for (size_t i = 0; i < count; i++) {
   CGImageRef image = CGImageSourceCreateImageAtIndex(source, i, NULL);
   [images addObject:[UIImage imageWithCGImage:image scale:[UIScreen mainScreen] orientation:UIImageOrientationUp]];
   CGImageRelease(image);
}
    
CFRelease(source)
```

一种方式是将这些UIImage对象通过UIImage的类方法`+animatedImageWithImages:duration:`组合成一个UIImage对象，然后赋值给UIImageView对象的image属性。

第二种方式是将UIImage对象的数组赋值给UIImageView对象的`animationImages`属性，然后调用UIImageView对象的`startAnimating`方法来启动动画。

当然，两种方式都需要计算`duration`。