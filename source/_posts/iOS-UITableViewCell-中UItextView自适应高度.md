---
title: iOS UITableViewCell 中UItextView自适应高度
date: 2017-08-19 23:16:49
tags: iOS
---

1、先新建控制器，在控制器中创建UItableVIew
2、 新建在uitableVIewcell, 在cell添加UITextView控件，使用autolayout，并且设置代理对象为Cell，UITextVIew的代理方法

```objective-c
- (void)textViewDidChange:(UITextView*)textView{

if([self.delegaterespondsToSelector:@selector(tableViewCell:didChangeText:)]) {

[self.delegatetableViewCell:selfdidChangeText:textView.text];

}
```

```objective-c
UITableView*tableView = [selftableView];

CGRectbounds = textView.bounds;

//计算text view的高度

CGSizemaxSize =CGSizeMake(bounds.size.width,CGFLOAT_MAX);

CGSizenewSize = [textViewsizeThatFits:maxSize];

bounds.size= newSize;

textView.bounds= bounds;

//让table view重新计算高度

[tableViewbeginUpdates];

[tableViewendUpdates];

}
- (UITableView*)tableView{

UIView*tableView =self.superview;

while(![tableViewisKindOfClass:[UITableViewclass]] && tableView) {

tableView = tableView.superview;

}
return(UITableView*)tableView;

}
```

