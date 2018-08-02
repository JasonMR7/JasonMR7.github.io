---
layout: post
title: UITableView将指定位置滚动至Window键盘上方 - JasonMR7
date: 2018-05-11 18:49:00.000000000 +09:00
---



## 背景

在Window上弹起键盘，UITableView需要将指定的位置滚动到键盘上方，如微信朋友圈点击评论时，需要将cell的底部滚动至键盘上方，如下：![键盘滚动](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2018-05-11-UITableView根据评论滚动/键盘滚动.gif "键盘滚动")



## 正文

我们首先要解决键盘问题，键盘什么时候开始弹起，弹起多高，动画有多久？

键盘的几个事件，了解一下？

### 键盘事件

```objective-c
// Each notification includes a nil object and a userInfo dictionary containing the
// begining and ending keyboard frame in screen coordinates. Use the various UIView and
// UIWindow convertRect facilities to get the frame in the desired coordinate system.
// Animation key/value pairs are only available for the "will" family of notification.
// 通知将带一个userInfo的字典，包括了开始和结束时的frame，动画时间等信息

// 键盘将要展示
UIKIT_EXTERN NSNotificationName const UIKeyboardWillShowNotification __TVOS_PROHIBITED;
// 键盘已经展示
UIKIT_EXTERN NSNotificationName const UIKeyboardDidShowNotification __TVOS_PROHIBITED;
// 键盘将要隐藏
UIKIT_EXTERN NSNotificationName const UIKeyboardWillHideNotification __TVOS_PROHIBITED;
// 键盘已经隐藏
UIKIT_EXTERN NSNotificationName const UIKeyboardDidHideNotification __TVOS_PROHIBITED;
```



userInfo中的信息

```objective-c
// 开始的frame
UIKIT_EXTERN NSString *const UIKeyboardFrameBeginUserInfoKey        NS_AVAILABLE_IOS(3_2) __TVOS_PROHIBITED; // NSValue of CGRect
// 结束的frame
UIKIT_EXTERN NSString *const UIKeyboardFrameEndUserInfoKey          NS_AVAILABLE_IOS(3_2) __TVOS_PROHIBITED; // NSValue of CGRect
// 动画的时间
UIKIT_EXTERN NSString *const UIKeyboardAnimationDurationUserInfoKey NS_AVAILABLE_IOS(3_0) __TVOS_PROHIBITED; // NSNumber of double
// 动画UIViewAnimationCurve类型
UIKIT_EXTERN NSString *const UIKeyboardAnimationCurveUserInfoKey    NS_AVAILABLE_IOS(3_0) __TVOS_PROHIBITED; // NSNumber of NSUInteger (UIViewAnimationCurve)
// 是否为本地用户键盘
UIKIT_EXTERN NSString *const UIKeyboardIsLocalUserInfoKey           NS_AVAILABLE_IOS(9_0) __TVOS_PROHIBITED; // NSNumber of BOOL
```





### 取指定内容坐标

将指定位置滚动到键盘上方当时是获取指定位置点

这里的位置点是Cell的底部任意一点，就取最左边的点吧

点击评论我们可以获取到指定cell的NSIndexPath，通过NSIndexPath我们可以获取到整个Cell的Rect：

```objective-c
CGRect rect = [self.tableView rectForRowAtIndexPath:indexPath];
CGPoint point = CGPointMake(0, rect.origin.y + rect.size.height);
```



好了，位置点已经找到，但是这个位置是相对于UITableView的，而键盘如果是在Window上的，怎么转化呢？



### 转化坐标

为了解决上述问题，我们需要了解几个UIView的方法：

```objective-c
// Converts a point from the receiver’s coordinate system to that of the specified view.
// 将一个点从调用方的坐标系转换为指定视图的坐标系，指定视图如果为nil则默认为Window
- (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view;
// Converts a point from the coordinate system of a given view to that of the receiver.
// 将一个点从指定视图的坐标系转换为调用方的坐标系，指定视图如果为nil则默认为Window
- (CGPoint)convertPoint:(CGPoint)point fromView:(nullable UIView *)view;

// Converts a rectangle from the receiver’s coordinate system to that of another view.
// 将一个矩形从调用方的坐标系转换为指定视图的坐标系，指定视图如果为nil则默认为Window
- (CGRect)convertRect:(CGRect)rect toView:(nullable UIView *)view;
// Converts a rectangle from the coordinate system of another view to that of the receiver.
// 将一个矩形从指定视图的坐标系转换为调用方的坐标系，指定视图如果为nil则默认为Window
- (CGRect)convertRect:(CGRect)rect fromView:(nullable UIView *)view;
```

什么意思呢？举个例子：![坐标转化](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2018-05-11-UITableView根据评论滚动/坐标转化.png "坐标转化")

图中方块的层级关系是这样的：绿色方块在红色方块上，origin为(0,0)；红色方块在蓝色方块上，origin为(100,100)，那么如果将绿色方块直接放到蓝色方块上，并且视觉上位置不发生变化，那么绿色方块的origin应该是多少呢，我们可以直接用上面的convert方法算出来：

```objective-c
CGPoint newGreenOrigin = [redRect convertPoint:greenRect.frame.origin toView:blueView];
// 或者
CGPoint newGreenOrigin = [blueView convertPoint:greenRect.frame.origin fromView:redRect];
// 两种方法算出来都是(100,100)
```



```objective-c
/**
 接收键盘通知

 @param notification 通知体
 */
- (void)__keyboardWillShow:(NSNotification *)notification {
    // 获取结束时的frame
    CGRect keyboardEndFrame = [notification.userInfo[UIKeyboardFrameEndUserInfoKey] CGRectValue];
    // 还要加上一个toolBar(输入框)的高度，_point为目标位置点，在window上计算偏差值
    CGFloat y = [self.tableView convertPoint:CGPointMake(0, keyboardEndFrame.origin.y - _toolBar.height) fromView:nil].y - _point.y;
    // 根据偏差值算出真正的左上角的目标点
    CGPoint targetPoint = CGPointMake(0, self.tableView.contentOffset.y - y);
    // 值得要注意的是，当y小于0的时候，不要把顶部多余的空白拉下来了
    if (targetPoint.y < 0) {
        targetPoint.y = 0;
    }
    [self.tableView setContentOffset:targetPoint animated:YES];
}
```



### 总结

坐标转化的时候头脑要清晰，转来转去别转晕了就好了
