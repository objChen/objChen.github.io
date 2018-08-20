---
title: iOS NSString的copy Or Strong
date: 2018-03-4 19:27:34
tags:iOS
---



> NSString属性什么时候用copy，什么时候用strong?



我们在声明一个`NSString`属性时，对于其内存相关特性，通常有两种选择(基于`ARC`环境)：`strong`与`copy`。那这两者有什么区别呢？什么时候该用`strong`，什么时候该用`copy`呢？让我们先来看个例子。

### 示例

我们定义一个类，并为其声明两个字符串属性，如下所示：

```
@interface TestStringClass ()
@property (nonatomic, strong) NSString *strongString;
@property (nonatomic, copy) NSString *copyedString;
@end
```

上面的代码声明了两个字符串属性，其中一个内存特性是`strong`，一个是`copy`。下面我们来看看它们的区别。

首先，我们用一个不可变字符串来为这两个属性赋值，

```
- (void)test {
    NSString *string = [NSString stringWithFormat:@"abc"];
    self.strongString = string;
    self.copyedString = string;

    NSLog(@"origin string: %p, %p", string, &string);
    NSLog(@"strong string: %p, %p", strongString, &strongString);
    NSLog(@"copy string: %p, %p", copyedString, &copyedString);
}
```

其输出结果是：

```
origin string: 0x7fe441592e20, 0x7fff57519a48
strong string: 0x7fe441592e20, 0x7fe44159e1f8
copy string: 0x7fe441592e20, 0x7fe44159e200
```

我们要以看到，这种情况下，不管是`strong`还是`copy`属性的对象，其指向的地址都是同一个，即为`string`指向的地址。如果我们换作`MRC`环境，打印`string`的引用计数的话，会看到其引用计数值是`3`，即`strong`操作和`copy`操作都使原字符串对象的引用计数值加了`1`。

接下来，我们把`string`由不可变改为可变对象，看看会是什么结果。即将下面这一句

```
NSString *string = [NSString stringWithFormat:@"abc"];
```

改成：

```
NSMutableString *string = [NSMutableString stringWithFormat:@"abc"];
```

其输出结果是：

```
origin string: 0x7ff5f2e33c90, 0x7fff59937a48
strong string: 0x7ff5f2e33c90, 0x7ff5f2e2aec8
copy string: 0x7ff5f2e2aee0, 0x7ff5f2e2aed0
```

可以发现，此时`copy`属性字符串已不再指向`string`字符串对象，而是深拷贝了`string`字符串，并让`_copyedString`对象指向这个字符串。在`MRC`环境下，打印两者的引用计数，可以看到`string`对象的引用计数是`2`，而`_copyedString`对象的引用计数是`1`。

此时，我们如果去修改string字符串的话，可以看到：因为_strongString与string是指向同一对象，所以`_strongString`的值也会跟随着改变(需要注意的是，此时`_strongString`的类型实际上是`NSMutableString`，而不是`NSString`)；而`_copyedString`是指向另一个对象的，所以并不会改变。

### 结论

由于`NSMutableString`是`NSString`的子类，所以一个`NSString`指针可以指向`NSMutableString`对象，让我们的`strongString`指针指向一个可变字符串是OK的。

而上面的例子可以看出，当源字符串是`NSString`时，由于字符串是不可变的，所以，不管是`strong`还是`copy`属性的对象，都是指向源对象，`copy`操作只是做了次**浅拷贝**。

当源字符串是`NSMutableString`时，`strong`属性只是增加了源字符串的引用计数，而`copy`属性则是对源字符串做了次**深拷贝**，产生一个新的对象，且`copy`属性对象指向这个新的对象。另外需要注意的是，这个`copy`属性对象的类型始终是`NSString`，而不是`NSMutableString`，因此其是不可变的。

这里还有一个性能问题，即在源字符串是`NSMutableString`，`strong`是单纯的增加对象的引用计数，而`copy`操作是执行了一次深拷贝，所以性能上会有所差异。而如果源字符串是`NSString`时，则没有这个问题。

所以，在声明`NSString`属性时，到底是选择`strong`还是`copy`，可以根据实际情况来定。不过，一般我们将对象声明为`NSString`时，都不希望它改变，所以大多数情况下，我们建议用`copy`，以免因可变字符串的修改导致的一些非预期问题。