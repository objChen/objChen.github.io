---
title: Swift & OC 混编
date: 2016-06-28 21:29:42
tags: iOS
---

### 在Swift工程中导入Objective-C

如果我们需要在一个Swift工程中导入Objective-C代码，需要依托于Objective-C Bridging header(桥接头文件)。在Swift工程中，当我们添加一个Objective-C文件时，如果工程中没有现存的Bridging header文件，则XCode会提示我们是否创建该文件。如果我们点击Yes，则XCode会自动创建头文件，并命名为”工程名-Bridging-Header.h”。

我们需要编辑这个文件，以导入我们的Objective-C代码。

如果是从同一个target中导入Objective-C代码，则我们需要做如下操作

1. 在Bridging header文件中，import所有需要在Swift中使用的头文件

```
#import "XYZCustomCell.h"	
#import "XYZCustomView.h"
#import "XYZCustomViewController.h"
```

1. 在Build Settings中，确保Swift Compiler Code Generation->Objective-C Bridging Header下的头文件的路径是对的。路径必须直接指向文件本身，而不是文件所在的文件夹。

所有在Bridging header中的公有Objective-C头文件在Swift都是可见的，并且其中的所有功能在所有Swift中都是可用的，而不需要任何导入处理。可以像使用Swift代码一样使用Objective-C代码。如下所示

```
let myCell = XYZCustomCell()
myCell.subtitle = "A custom cell"
```

### 在Objective-C中导入Swift

如果我们要在一个Objective-C工程中导入Swift，则需要依托于XCode-generated header文件。这个自动生成的文件是一个Objective-C头文件，其声明了在我们的target中使用的所有Swift接口。