---
title: iOS 再谈TableView性能优化
date: 2017-06-30 21:33:48
tags: iOS
---



> 优化TableView性能



1、cell复用

复用很简单，这或许是所有iOS开发者最为熟知的一个优化内容，如下代码

```objective-c
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    static NSString *Identifier = @"cell";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:ID];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:ID];
    }
    return cell;
}

```

但是，这样重用就完美了吗？

我们经常在注意cellForRowAtIndexPath：中为每一个cell绑定数据，实际上在调用cellForRowAtIndexPath：的时候cell还没有被显示出来，为了提高效率我们应该把数据绑定的操作放在cell显示出来后再执行，可以在tableView：willDisplayCell：forRowAtIndexPath：（以后简称willDisplayCell）方法中绑定数据。

注意willDisplayCell在cell 在tableview展示之前就会调用，此时cell实例已经生成，所以不能更改cell的结构，只能是改动cell上的UI的一些属性（例如label的内容等）。

2、cell高度的计算

这边我们分为两种cell，一种是定高的cell，另外一种是动态高度的cell。

* 定高的cell，应该采用如下方式：

```
self.tableView.rowHeight = 88;
```

这个方法指定了所有cell高度都是88的tableview，rowHeight默认的值是44，所以一个空的TableView会显示成这个样子。对于定高cell，直接采用上面方式给定高度，不需要实现tableView:heightForRowAtIndexPath:以节省不必要的计算和开销。

* 动态高度的cell

我们需要实现它的代理，来给出高度：

```objective-c
- (CGFloat)tableView:(UITableView *)tableViewheightForRowAtIndexPath:(NSIndexPath *)indexPath{
 return xxx
}

```

这个代理方法实现后，上面的rowHeight的设置将会变成无效。在这个方法中，我们需要提高cell高度的计算效率，来节省时间。

自从iOS8之后有了self-sizing cell的概念，cell可以自己算出高度，使用self-sizing cell需要满足以下三个条件：

（1）使用Autolayout进行UI布局约束（要求cell.contentView的四条边都与内部元素有约束关系）。

（2）指定TableView的estimatedRowHeight属性的默认值。

（3）指定TableView的rowHeight属性为UITableViewAutomaticDimension

```
- (void)viewDidload {
    self.myTableView.estimatedRowHeight = 44.0;
    self.myTableView.rowHeight = UITableViewAutomaticDimension;
}
```

除了提高cell高度的计算效率之外，对于已经计算出的高度，我们需要进行缓存，对于已经计算过的高度，没有必要进行计算第二次。

3、异步化UI，不要阻塞主线程

我们时常会看到这样一个现象，就是加载时整个页面卡住不动，怎么点都没用，仿佛死机了一般。原因是主线程被阻塞了。所以对于网路数据的请求或者图片的加载，我们可以开启多线程，将耗时操作放到子线程中进行，异步化操作。这个或许每个iOS开发者都知道的知识，不必多讲。

4、滑动时按需加载对应的内容

如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。

```objective-c
-(void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inoutCGPoint *)targetContentOffset{
    NSIndexPath *ip = [self indexPathForRowAtPoint:CGPointMake(0,targetContentOffset->y)];
    NSIndexPath  *cip=[[self indexPathsForVisibleRows] firstObject];
    NSIntegerskipCount=8;
    if(labs(cip.row-ip.row)>skipCount){
        NSArray *temp=[self indexPathsForRowsInRect:CGRectMake(0,targetContentOffset->y,self.width,self.height)];
        NSMutableArray *arr=[NSMutableArray arrayWithArray:temp];
        if(velocity.y<0){
            NSIndexPath *indexPath=[temp lastObject];
            if(indexPath.row+33){
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-3 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-1 inSection:0]];
            }
        }
        [needLoadArr addObjectsFromArray:arr];
    }
}

```

记得在tableView:cellForRowAtIndexPath:方法中加入判断：

```
if(needLoadArr.count>0&&[needLoadArrindexOfObject:indexPath]==NSNotFound{
    [cellclear];
    return;
}
```

滑动很快时，只加载目标范围内的cell，这样按需加载（配合SDWebImage），极大提高流畅度。