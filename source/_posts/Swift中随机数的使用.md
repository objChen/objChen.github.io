---
title: Swift中随机数的使用
date: 2018-03-19 22:40:57
tags: iOS
---

> 在我们开发的过程中，时不时地需要产生一些随机数。这里我们总结一下`Swift`中常用的一些随机数生成函数。这里我们将在`Playground`中来做些示例演示。

### 整型随机数

如果我们想要一个整型的随机数，则可以考虑用`arc4random`系列函数。我们可以通过`man arc4random`命令来看一下这个函数的定义：

> The arc4random() function uses the key stream generator employed by the arc4 cipher, which uses 8*8 8 bit S-Boxes. The S-Boxes can be inabout (2^1700) states. The arc4random() function returns pseudo-random numbers in the range of 0 to (2^32)-1, and therefore has twice the range of rand(3) and random(3).

`arc4random`使用了`arc4密码`加密的`key stream`生成器(请脑补)，产生一个`[0, 2^32)`区间的随机数(注意是左闭右开区间)。这个函数的返回类型是`UInt32`。如下所示：

```
arc4random()				// 2,919,646,954
```

如果我们想生成一个指定范围内的整型随机数，则可以使用`arc4random() % upper_bound`的方式，其中`upper_bound`指定的是上边界，如下处理：

```
arc4random() % 10			// 8
```

不过使用这种方法，在`upper_bound`不是2的幂次方时，会产生一个所谓[Modulo bias(模偏差)](http://eternallyconfuzzled.com/arts/jsw_art_rand.aspx)的问题。

我们在控制台中通过`man arc4random`命令，可以查看`arc4random`的文档，有这么一条：

> arc4random_uniform() will return a uniformly distributed random number less than upper_bound. arc4random_uniform() is recommended over constructions like ‘’arc4random() % upper_bound’’ as it avoids “modulo bias” when the upper bound is not a power of two.

因此可以使用`arc4random_uniform`，它接受一个`UInt32`类型的参数，指定随机数区间的上边界`upper_bound`，该函数生成的随机数范围是`[0, upper_bound)`，如下所示：

```
arc4random_uniform(10)		// 6
```

而如果想指定区间的最小值（如随机数区间在`[5, 100)`），则可以如下处理：

```
let max: UInt32 = 100
let min: UInt32 = 5
arc4random_uniform(max - min) + min			// 82
```

当然，在`Swift`中也可以使用传统的`C`函数`rand`与`random`。不过这两个函数有如下几个缺点：

1. 这两个函数都需要初始种子，通常是以当前时间来确定。
2. 这两个函数的上限在`RAND_MAX=0X7fffffff(2147483647)`，是`arc4random`的一半。
3. `rand`函数以有规律的低位循环方式实现，更容易预测

我们以`rand`为例，看看其使用：

```
srand(UInt32(time(nil)))            // 种子,random对应的是srandom
rand()								// 1,314,695,483
rand() % 10							// 8
```

### 64位整型随机数

在大部分应用中，上面讲到的几个函数已经足够满足我们获取整型随机数的需求了。不过我们看看它们的函数声明，可以发现这些函数主要是针对`32`位整型来操作的。如果我们需要生成一个`64`位的整型随机数呢？毕竟现在的新机器都是支持`64`位的了。

目前貌似没有现成的函数来生成`64`位的随机数，不过`jstn`在`stackoverflow`上为我们分享了他的方法。我们一起来看看。

他首先定义了一个泛型函数，如下所示：

```
func arc4random <T: IntegerLiteralConvertible> (type: T.Type) -> T {
    var r: T = 0
    arc4random_buf(&r, UInt(sizeof(T)))
    return r
}
```

这个函数中使用了`arc4random_buf`来生成随机数。让我们通过`man arc4random_buf`来看看这个函数的定义：

> arc4random_buf() function fills the region buf of length nbytes with ARC4-derived random data.

这个函数使用`ARC4`加密的随机数来填充该函数第二个参数指定的长度的缓存区域。因此，如果我们传入的是`sizeof(UInt64)`，该函数便会生成一个随机数来填充`8`个字节的区域，并返回给`r`。那么`64`位的随机数生成方法便可以如下实现：

```
extension UInt64 {
    static func random(lower: UInt64 = min, upper: UInt64 = max) -> UInt64 {
        var m: UInt64
        let u = upper - lower
        var r = arc4random(UInt64)

        if u > UInt64(Int64.max) {
            m = 1 + ~u
        } else {
            m = ((max - (u * 2)) + 1) % u
        }

        while r < m {
            r = arc4random(UInt64)
        }

        return (r % u) + lower
    }
}
```

我们来试用一下：

```
UInt64.random()					// 4758246381445086013
```

当然`jstn`还提供了`Int64`，`UInt32`，`Int32`的实现，大家可以脑补一下。

### 浮点型随机数

如果需要一个浮点值的随机数，则可以使用`drand48`函数，这个函数产生一个`[0.0, 1.0]`区间中的浮点数。这个函数的返回值是`Double`类型。其使用如下所示：

```
srand48(Int(time(nil)))
drand48()						// 0.396464773760275
```

记住这个函数是需要先调用`srand48`生成一个种子的初始值。

### 一个小示例

最近写了一个随机键盘，需要对`0-9`这几个数字做个随机排序，正好用上了上面的`arc4random`函数，如下所示：

```
let arr = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]

let numbers = arr.sort { (_, _) -> Bool in
    arc4random() < arc4random()
}
```

在闭包中，随机生成两个数，比较它们之间的大小，来确定数组的排序规则。