---
title: Swift中String与CChar数组的转换
date: 2018-03-29 20:01:02
tags: iOS
---

在现阶段`Swift`的编码中，我们还是有很多场景需要调用一些`C`函数。在`Swift`与`C`的混编中，经常遇到的一个问题就是需要在两者中互相转换字符串。在`C`语言中，字符串通常是用一个`char数组`来表示，在`Swift`中，是用`CChar数组`来表示。从`CChar`的定义可以看到，其实际上是一个`Int8`类型，如下所示：

```
/// The C 'char' type.
///
/// This will be the same as either `CSignedChar` (in the common
/// case) or `CUnsignedChar`, depending on the platform.
public typealias CChar = Int8
```

如果我们想将一个`String`转换成一个`CChar数组`，则可以使用`String`的`cStringUsingEncoding`方法，它是`String`扩展中的一个方法，其声明如下：

```
/// Returns a representation of the `String` as a C string
/// using a given encoding.
@warn_unused_result
public func cStringUsingEncoding(encoding: NSStringEncoding) -> [CChar]?
```

参数指定的是编码格式，我们一般指定为`NSUTF8StringEncoding`，因此下面这段代码：

```
let str: String = "abc1个"

// String转换为CChar数组
let charArray: [CChar] = str.cStringUsingEncoding(NSUTF8StringEncoding)!
```

其输出结果是：

```
[97, 98, 99, 49, -28, -72, -86, 0]
```

可以看到`"个"`字由三个字节表示，这是因为`Swift`的字符串是`Unicode`编码格式，一个字符可能由1个或多个字节组成。另外需要注意的是`CChar`数组的最后一个元素是`0`，它表示的是一个字符串结束标志符`\n`。

我们知道，在`C`语言中，一个数组还可以使用指针来表示，所以字符串也可以用`char *`来表示。在`Swift中`，指针是使用`UnsafePointer`或`UnsafeMutablePointer`来包装的，因此，`char指针`可以表示为`UnsafePointer<CChar>`，不过它与`[CChar]`是两个不同的类型，所以以下代码会报编译器错误：

```
// Error: Cannot convert value of type '[CChar]' to specified type 'UnsafePointer<CChar>'
let charArray2: UnsafePointer<CChar> = str.cStringUsingEncoding(NSUTF8StringEncoding)!
```

不过有意思的是我们可以直接将`String`字符串传递给带有`UnsafePointer<CChar>`参数的函数或方法，如以下代码所示：

```
func length(s: UnsafePointer<CChar>) {
    print(strlen(s))
}

length(str)

// 输出：7\n
```

而`String`字符串却不能传递给带有`[CChar]`参数的函数或方法，如以下代码会报错误：

```
func length2(s: [CChar]) {
    print(strlen(s))
}

// Error: Cannot convert value of type 'String' to expected argument type '[CChar]'
length2(str)
```

实际上，在`C`语言中，我们在使用数组参数时，很少以数组的形式来定义参数，则大多是通过指针方式来定义数组参数。

如果想从`[CChar]`数组中获取一上`String`字符串，则可以使用`String`的`fromCString`方法，其声明如下：

```
/// Creates a new `String` by copying the nul-terminated UTF-8 data
/// referenced by a `CString`.
///
/// Returns `nil` if the `CString` is `NULL` or if it contains ill-formed
/// UTF-8 code unit sequences.
@warn_unused_result
public static func fromCString(cs: UnsafePointer<CChar>) -> String?
```

从注释可以看到，它会将`UTF-8`数据拷贝以新字符串中。如下示例：

```
let chars: [CChar] = [99, 100, 101, 0]
let str2: String = String.fromCString(chars)!

// 输出：cde
```

这里需要注意的一个问题是，`CChar`数组必须以`0`结束，否则会有不可预料的结果。在我的`Playground`示例代码中，如果没有`0`，报了以下错误：

```
Execution was interrupted. reason: EXC_BAD_INSTRUCTION
```

还有可能出现的情况是`CChar`数组的存储区域正好覆盖了之前某一对象的区域，这一对象有一个可以表示字符串结尾的标识位，则这时候，`str2`输出的可能是`"cde1一"`。

### 小结

在`Swift`中，`String`是由独立编码的`Unicode`字符组成的，即`Character`。一个`Character`可能包括一个或多个字节。所以将`String`字符串转换成`C`语言的`char *`时，数组元素的个数与`String`字符的个数不一定相同(即在`Swift`中，与`str.characters.count`计算出来的值不一定相等)。这一点需要注意。另外还需要注意的就是将`CChar`数组转换为`String`时，数组最后一个元素应当为字符串结束标志符，即`0`。