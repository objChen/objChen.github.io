---
title: iOS block导致EXC_BAD_ACCESS
date: 2018-04-12 23:43:35
tags: iOS
---

> 在调用block时，如果这个block为nil，则程序会崩溃，报类似于EXC_BAD_ACCESS(code=1, address=0xc)异常[32位下的结果，如果是64位，则address=0x10]。



在定义一个block时，编译器会在栈上创建一个结构体，类似于图2的结构体。

```
struct Block_layout {
	void *isa;
	int flags;
	int reserved;
	void (*invoke)(void *, ...);
	struct Block_descriptor *descriptor;
	/* Imported variables. */
}
```

block就是指向这个结构体的指针。其中的invoke就是指向具体实现的函数指针。当block被调用时，程序最终会跳转到这个函数指针指向的代码区。而当block为nil时，程序就会试图去读取0xc地址的信息，而这个地址什么都不会有(duff address)，于是抛出一个segmentation fault。在32位系统下，之所以是0xc，是因为invoke前面的三个成员变量的大小正好是12。

所以我们在使用block时，应该首先去判断block是否为空。一种比较优雅的写法是：

```
!block ?: block()
```

