---
title: iOS 键盘事件
date: 2018-03-09 22:31:48
tags: iOS
---

在涉及到表单输入的界面中，我们通常需要监听一些键盘事件，并根据实际需要来执行相应的操作。如，键盘弹起时，要让我们的`UIScrollView`自动收缩，以能看到整个`UIScrollView`的内容。为此，在`UIWindow.h`中定义了如下`6`个通知常量，来配合键盘在不同时间点的事件处理：

```
UIKeyboardWillShowNotification			// 键盘显示之前
UIKeyboardDidShowNotification			// 键盘显示完成后
UIKeyboardWillHideNotification			// 键盘隐藏之前
UIKeyboardDidHideNotification			// 键盘消息之后
UIKeyboardWillChangeFrameNotification	// 键盘大小改变之前
UIKeyboardDidChangeFrameNotification	// 键盘大小改变之后
```

这几个通知的`object`对象都是`nil`。而`userInfo`字典都包含了一些键盘的信息，主要是键盘的位置大小信息，我们可以通过使用以下的`key`来获取字典中对应的值：

```
// 键盘在动画开始前的frame
let UIKeyboardFrameBeginUserInfoKey: String

// 键盘在动画线束后的frame
let UIKeyboardFrameEndUserInfoKey: String

// 键盘的动画曲线
let UIKeyboardAnimationCurveUserInfoKey: String

// 键盘的动画时间
let UIKeyboardAnimationDurationUserInfoKey: String
```

在此，我感兴趣的是键盘事件的调用顺序和如何获取键盘的大小，以适当的调整视图的大小。

从定义的键盘通知的类型可以看到，实际上我们关注的是三个阶段的键盘的事件：显示、隐藏、大小改变。在此我们设定两个`UITextField`，它们的键盘类型不同：一个是普通键盘，一个是数字键盘。我们监听所有的键盘事件，并打印相关日志(在此就不贴代码了)，直接看结果。

1) 当我们让`textField1`获取输入焦点时，打印的日志如下：

```
keyboard will change
keyboard will show
keyboard did change
keyboard did show
```

2) 在不隐藏键盘的情况下，让`textField2`获取焦点，打印的日志如下：

```
keyboard will change
keyboard will show
keyboard did change
keyboard did show
```

3) 再收起键盘，打印的日志如下：

```
keyboard will change
keyboard will hide
keyboard did change
keyboard did hide
```

从上面的日志可以看出，不管是键盘的显示还是隐藏，都会发送大小改变的通知，而且是在`show`和`hide`的对应事件之前。而在大小不同的键盘之间切换时，除了发送`change`事件外，还会发送`show`事件(不发送`hide`事件)。

另外还有两点需要注意的是：

1. 如果是在两个大小相同的键盘之间切换，则不会发送任何消息
2. 如果是普通键盘中类似于中英文键盘的切换，只要大小改变了，都会发送一组或多组与上面2)相同流程的消息

了解了事件的调用顺序，我们就可以根据自己的需要来决定在哪个消息处理方法中来执行操作。为此，我们需要获取一些有用的信息。这些信息是封装在通知的`userInfo`中，通过上面常量`key`来获取相关的值。通常我们关心的是`UIKeyboardFrameEndUserInfoKey`，来获取动画完成后，键盘的`frame`，以此来计算我们的`scroll view`的高度。另外，我们可能希望`scroll view`高度的变化也是通过动画来过渡的，此时`UIKeyboardAnimationCurveUserInfoKey`和`UIKeyboardAnimationDurationUserInfoKey`就有用了。

我们可以通过以下方式来获取这些值：

```
if let dict = notification.userInfo {
    var animationDuration: NSTimeInterval = 0
    var animationCurve: UIViewAnimationCurve = .EaseInOut
    var keyboardEndFrame: CGRect = CGRectZero

    dict[UIKeyboardAnimationCurveUserInfoKey]?.getValue(&animationCurve)
    dict[UIKeyboardAnimationDurationUserInfoKey]?.getValue(&animationDuration)
    dict[UIKeyboardFrameEndUserInfoKey]?.getValue(&keyboardEndFrame)
    ......
}
```