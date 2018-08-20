---
title: iOS OC数组遍历
date: 2018-04-18 22:32:57
tags: iOS
---

遍历一个数组的方法有几种，for, forin, enumerateObjectsUsingBlock:方法。现在用得比较多的可能是enumerateObjectsUsingBlock:，它能很方便地让我们获取到数组中的元素及对应的索引，然后根据这些信息做一些操作，如下代码所示:

```
NSMutableArray *array = [[NSMutableArray alloc] init];
for (NSInteger index = 0; index < 10; index++) {
  
  Test *t = [[Test alloc] init];
  t.index = index;
  [array addObject:t];
}
   
[array enumerateObjectsUsingBlock:^(Test *  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
  [obj test];
}];
```

不过，如果在循环中，只是想调用元素的某一个方法，则可以考虑使用makeObjectsPerformSelector:或者makeObjectsPerformSelector:withObject:，这两个方法会按元素的顺序向数组中的每个元素发送Selector指定的消息。如下代码所示:

```
NSMutableArray *array = [[NSMutableArray alloc] init];
for (NSInteger index = 0; index < 10; index++) {
  
  Test *t = [[Test alloc] init];
  t.index = index;
  [array addObject:t];
}
   
[array makeObjectsPerformSelector:@selector(test)];
[array makeObjectsPerformSelector:@selector(testWithNumber:) withObject:@10];
```

当然，Selector不能是NULL，否则会抛NSInvalidArgumentException异常。大家如果熟悉runtime的话，应该知道消息机制是如何处理调用不存在方法的。