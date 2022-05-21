---
layout: post
title: View Controller 转场实现机制分析
categories:
  - 笔记
date: 2016-09-10 12:33:51
tags:
  - iOS
---

众所周知，iOS 的 View Controller 的转场效果本质上是基于“当前视图消失和下一视图出现”所进行的动画。如果是自己实现的 UIViewControllerAnimatedTransitioning 协议，那么此动画就由 `- animateTransition:` 方法来提供；如果没有实现此协议则此动画由系统提供。以结构如下图的 Navigation Controller 为例：

![Navigation Controller 结构](/assets/images/2016/14734732613835.jpg)

<!-- more -->

在页面 B → A 切换的过程中，应用的视图层级结构如下图所示：
![Navigation Controller B → A 切换](/assets/images/2016/14734073322458.jpg)

最终运行的转场动画，不论是系统默认的平移动画还是通过 UIViewControllerAnimatedTransitioning 来实现的动画，都作用在 View Controller A & B 共同的父视图 UIViewControllerWrapperView 内部，而这个共同的父视图即是 `- [UIViewControllerContextTransitioning containerView]` 所返回的那个容器 View。

-------

下面再说说可交互式动画的实现机制。

一般情况下，实现可交互式动画需要先实现 UIViewControllerInteractiveTransitioning 协议，通过`- startInteractiveTransition:` 来启动转场，然后不断调用 `- updateInteractiveTransition:` 更新转场进度，最后调用 `-  finishInteractiveTransition` 或 `- cancelInteractiveTransition` 来完成或取消转场。

`- updateInteractiveTransition:` 方法接受一个表示`完成百分比`的参数，显然这个百分比是用来更新转场动画 CAAnimation 的进度的，但是 Core Animation 框架并没有提供直接改变动画进度的接口，所以一直以来我都以为苹果利用它的特权调用了某些私有方法来完成这件事。直到有一天，我在一个视图中通过 `- [CALayer convertTime:fromLayer:]` 来定期检查当前 layer 的相对时间：

```objectivec
CFTimeInterval time = [self.layer convertTime:CACurrentMediaTime() fromLayer:nil];
NSLog(@"Current layer time: %@", @(time));
```

在通过左滑手势操作当前页滑动返回时，打出的 Log 是这样的：

```
Current layer time: 261910.875040995
Current layer time: 261911.377889105
Current layer time: 0.02057165431515606
Current layer time: 0.05607890069196765
Current layer time: 0.1110305915132237
Current layer time: 0.2074074031074266
Current layer time: 0.2620772946859903
Current layer time: 0.3263285024154589
Current layer time: 261917.375051694
Current layer time: 261917.879273495
```

当前 layer 的时间竟然变成了返回动画的相对时间！根据 Apple 文档说明，这个时间会被 CAMediaTiming 协议的属性所影响（如 speed），并且一个 layer 的时间发生改变，则此 layer 层级树中的子孙 layer 的时间全部发生同样的改变。所以我决定找出导致 layer 的 `speed` 值改变的元凶：

```objectivec
for (UIView* view = self; view; view = view.superview) {
    if (view.layer.speed != 1) {
        NSLog(@"Speed changed in the layer of view: %@, speed: %@, time offset: %@", view.class, @(view.layer.speed), @(view.layer.timeOffset));
    }
}
```

输出：

```
Speed changed in the layer of view: UIViewControllerWrapperView, speed: 0, time offset: 0.1144122469252434
Speed changed in the layer of view: UIViewControllerWrapperView, speed: 0, time offset: 0.1479468599033816
Speed changed in the layer of view: UIViewControllerWrapperView, speed: 0, time offset: 0.172463768115942
Speed changed in the layer of view: UIViewControllerWrapperView, speed: 0, time offset: 0.2020531400966183
```

于是可交互动画的实现机制就变得明朗了：

1. 调用 `- startInteractiveTransition:` 方法，实际上是将 UIViewControllerWrapperView 的 `layer.speed` 值改成 0 以暂停动画；
2. 调用 `- updateInteractiveTransition:` 方法，实际上是通过 `duration` 和百分比换算出一个合适的 `timeOffset`，更新到 UIViewControllerWrapperView 的 layer 上面，以模拟动画进度更新的效果；
3. 调用 `- finishInteractiveTransition`，则是将 `layer.speed` 值改回 1，让 Core Animation 继续完成剩下的动画。

这个 UIViewControllerWrapperView 就是上面图中那个共同的父 View，在不同的实现中，它的类名可能会有变化，但都是 `- [UIViewControllerContextTransitioning containerView]` 返回的那个容器 View。由于父 layer 的动画时间改变是会影响到所有子 layer 的，所以有时候即使不用 `transitionCoordinator`，一些看上去与转场过程毫无关联的动画依然会受滑动返回手势的影响：

![screenshot](/assets/images/2016/20160910-screenshot.gif)


最后一个问题，是调用 `- cancelInteractiveTransition` 之后，系统会自动将动画逆转，以回退到切换之前的状态，这个又是如何实现的呢？

[网上一些文章](http://blog.devtang.com/2016/03/13/iOS-transition-guide/) 介绍了通过一帧帧调整 timeOffset 值来模拟逆转动画的效果，还用了 CADisplayLink 来同步屏幕刷新率，讲道理我是不信苹果会用这么蠢的办法的。通过一系列断点和 Log 调试发现，实际上苹果做的事非常巧妙：将所有 CAAnimation 的 `autoreverses` 属性设定为 `YES`，然后将 `beginTime` 设定为一个在 `(- 2 * duration, - duration)` 区间的一个合适的值，再用 CAAnimationGroup 包裹起来，以保证 CAAnimation 只运行想要的那部分动画。这样一来由于 `autoreverses` 的作用，Core Animation 层自己就会将动画回放了。


