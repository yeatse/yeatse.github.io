---
title: UIWebView 与 3D Touch 的自定义交互
categories:
  - 笔记
date: 2016-10-08 20:50:40
tags:
  - iOS
  - UIKit
  - 3D Touch
---

## 0x00 3D Touch API 处理 UIWebView 的局限

从 iOS 9 开始，UIKit 新增了 3D Touch 相关接口，如果使用苹果推荐的 storyboard 搭建 UI，勾选了 `Preview & Commit Segues` 选项之后就可以零代码实现系统级的 3D Touch 效果；用代码实现也很简单，只要实现 `UIViewControllerPreviewingDelegate` 协议，然后调用 `- [UIViewController registerForPreviewingWithDelegate:sourceView:]` 方法，就可以对任意 UIView 进行 Peek 和 Pop 操作了。

实际上，尽管 `- [UIViewController registerForPreviewingWithDelegate:sourceView:]` 方法的第二个参数接受的是任意的 UIView，但在 UIWebView 和 WKWebView 上按压的操作却是没有效果的。虽然苹果针对这两个 WebView 提供了 `allowsLinkPreview` 属性做了特殊处理，但这也仅仅是调用了 Safari 打开链接，实际应用中经常要针对某些特殊的链接进行应用内跳转，这是 `allowsLinkPreview` 无论如何也完成不了的。

<!-- more -->

那么为什么在其他 View 上都正常的 3D Touch 在 WebView 上却无效了呢？以 UIWebView 为例，在实际应用中可以发现，如果把 WebView 的 `userInteractionEnabled` 值设为 NO，或者按压空白位置，`UIViewControllerPreviewingDelegate` 中的方法还是可以正常回调的，所以原因很可能是 UIWebView 处理链接点击事件的手势与 3D Touch 的手势发生了冲突。要让 UIWebView 和其他 View 一样支持 Peek & Pop，就要完成以下三个步骤：
 
1. 取出 UIWebView 中处理点击事件的 gesture recognizer；
2. 取出 UIWebView 中处理 3D Touch 的 gesture recognizer；
3. 对 `1` 中的每一个手势监听器，调用 `- [UIGestureRecognizer requireGestureRecognizerToFail:]` 方法，保证 UIKit 优先处理 3D Touch 事件。

## 0x01 UIWebView 的点击与 3D Touch 手势的冲突解决

通过断点等方法可以得出，UIWebView 的内部视图层级结构和继承关系是这样的：

```
+--- UIWebView → UIView
|   |
|   +--- _UIWebViewScrollView → UIWebScrollView → UIScrollView → UIView
|       |
|       +--- UIWebBrowserView → UIWebDocumentView → UIWebTiledView → UIView
|           |
|           (...)
```

UIWebBrowserView 是显示 WebView 内部元素的容器，处理链接点击事件也应该由它来做，于是在 Xcode 中打个断点偷窥一下成员变量，果然在它的父类 UIWebDocumentView 里看到了一组与 gesture 有关的私有成员：

![](/images/2016/14759362518041.jpg)

看这些单词的意思很明显了，与 3D Touch 冲突的手势可能是 `_singleTapGestureRecognizer`、`highlightLongPressGestureRecognizer`、`_longPressGestureRecognizer`。在 runtime 面前一切私有成员都是纸老虎，通过 KVC 把这三个手势取出来：

```objc
UIView* browserView = [self valueForKeyPath:@"internal.browserView"]; // internal.browserView 是一个能获取内部 UIWebBrowserView 的私有 keyPath
UIGestureRecognizer* singleTapGesture = [browserView valueForKey:@"singleTapGestureRecognizer"];
UIGestureRecognizer* longPressGesture = [browserView valueForKey:@"longPressGestureRecognizer"];
UIGestureRecognizer* highlightLongPressGesture = [browserView valueForKey:@"highlightLongPressGestureRecognizer"];
```

同样地，检查调用 `registerForPreviewingWithDelegate:sourceView:` 前后 UIWebView 的 gesture 变化，发现注册了 Previewing Delegate 之后，UIWebView 的父类 `UIView` 多出了三个手势监听器：

![](/images/2016/14759368882844.jpg)

那么这三个监听器自然也就与 3D Touch 相关了。接下来调用 `requireGestureRecognizerToFail:` 为上边取出的手势添加依赖：

```objc
for (UIGestureRecognizer* gesture in self.webView.gestureRecognizers) {
    [singleTapGesture requireGestureRecognizerToFail:gesture];
    [longPressGesture requireGestureRecognizerToFail:gesture];
    [highlightLongPressGesture requireGestureRecognizerToFail:gesture];
}
```

这样一来 UIWebView 就可以和普通的 UIView 一样使用 `UIViewControllerPreviewingDelegate` 进行 Peek 和 Pop 了：

![screenshot](/images/2016/20161009-screenshot.gif)


## 0x02 3D Touch 过程中监听器的状态变化

按上面的方法我们虽然可以正常使用 Peek 和 Pop，但是正常的链接点击却也受到了影响。当`长按`链接并松手之后，即使没有`重按`调出 Peek 界面，链接点击事件也依然没有触发，这跟 UIButton 和 UITableViewCell 等行为不一样，所以还需要进一步的处理。

在上面得到的三个 3D Touch 相关手势当中，仅仅根据类名很难猜出它们的具体作用。于是祭出 KVO，进行操作的同时监听它们的 state 变化，得到结果如下：

#### 单击
```
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 1
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 2
_UIPreviewGestureRecognizer state changed to 5
_UIRevealGestureRecognizer state changed to 5
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 4
```

#### 长按后松手
```
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 1
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 2
_UIRevealGestureRecognizer state changed to 1
_UIPreviewGestureRecognizer state changed to 1
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 4
_UIRevealGestureRecognizer state changed to 3
_UIPreviewGestureRecognizer state changed to 3
```

#### 重按 (Peek)
```
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 1
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 2
_UIRevealGestureRecognizer state changed to 1
_UIPreviewGestureRecognizer state changed to 1
_UIRevealGestureRecognizer state changed to 2
_UIPreviewGestureRecognizer state changed to 2
```

#### 重按后松手
```
_UIRevealGestureRecognizer state changed to 3
_UIPreviewGestureRecognizer state changed to 3
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 4
```

#### Peek 之后 Pop
```
_UIRevealGestureRecognizer state changed to 4
_UIPreviewInteractionTouchObservingGestureRecognizer state changed to 4
_UIPreviewGestureRecognizer state changed to 4
```

可以看出，对于 `_UIPreviewGestureRecognizer` 而言（`_UIRevealGestureRecognizer` 同样），除了单击的时候 state 值会变成 5 (`UIGestureRecognizerStateFailed`) 以外，其余情况都成功触发了按压手势，从而导致链接点击的手势启动失败。这也很容易理解，因为长按也算是重按的一种，力度轻了点而已，通过断点也可以看出 `_UIPreviewGestureRecognizer` 和 `_UIRevealGestureRecognizer` 本身就是继承自 `UILongPressGestureRecognizer` 的。

对比`长按`和`重按`的状态变化可以发现，`重按`的时候，`_UIPreviewGestureRecognizer`的 state 值会变成 2 (`UIGestureRecognizerStateChanged`)，这就是区别`长按`和`重按`的突破口。那么接下来要做的就是，在 `_UIPreviewGestureRecognizer` 的 state 值变成 3 (`UIGestureRecognizerStateEnded`)的时候，检查之前是否有“重按”过，如果没有重按过（state 从 1 直接变成 3），那么就手动触发 `_singleTapGestureRecognizer` 手势，模拟对 UIWebView 的点击事件。

## 0x03 用代码触发 UIWebView 的点击事件

那么问题又来了，如何通过代码来触发一个手势事件呢？

一个自然的想法是构造一个 UITouch 丢给系统，不过构造这个相当麻烦，[一篇 2008 年的文章](http://www.cocoawithlove.com/2008/10/synthesizing-touch-event-on-iphone.html)详细地说明了整个步骤。实际上我们模拟手势的目的只是为了调用对应的 selector 而已，如果能拿到对应的 selector 的话，直接调用 selector 就好了。

通过断点检查 `_singleTapGestureRecognizer` 的成员变量，可以看到里面 `_targets` 那一项很有意思：

![](/images/2016/14759403483679.jpg)

很明显这个单击事件最终是要调用 UIWebBrowserView 的 `_singleTapRecognized:` 方法的，所以我们可以通过 `performSelector:withObject:` 绕过 gesture recognizer 直接调用它。

最后一个问题是 `_singleTapRecognized:` 的参数问题，直接传 `_singleTapGestureRecognizer` 本身会被认为点击了左上角，跟传 nil 效果一样。怀疑 Apple 使用了某个私有方法或变量来确定点击的坐标，于是传一个 NSObject 进去试试，系统弹出这个错误：

```
-[NSObject location]: unrecognized selector sent to instance 0x6000000162e0
```

查看 dump 出的 [UITapGestureRecognizer.h](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/UIKit.framework/UITapGestureRecognizer.h) 文件可以知道，location 是 UITapGestureRecognizer 私有的一个计算型属性，返回类型是 CGPoint。猜想 `_singleTapRecognized:` 就是通过它来确定到底点了哪里的，那么我们只要构造出一个有 location 属性的对象，把 location 存进去再交给这个方法就可以了：

```objc
@interface LocationWrapper : NSObject

@property (nonatomic) CGPoint location;

@end

@implementation LocationWrapper

@end

// ...

LocationWrapper* wrapper = [LocationWrapper new];
wrapper.location = [singleTapGesture locationInView:singleTapGesture.view];
[browserView performSelector:NSSelectorFromString(@"_singleTapRecognized") withObject:wrapper];
```

## 0x04 Talk is cheap ...

按上面的思路，UIWebView 与 Peek & Pop 的冲突就可以比较完美地解决了。我把它封装成了一个 category，通过 method swizzling 的方式尽量简化使用成本，只要把 UIWebView+PeekingSupport.h 和 UIWebView+PeekingSupport.m 拖到工程里，其他什么都不用做，就可以像正常 UIView 一样在使用 UIWebView 上使用 3D Touch 了。项目地址在这里：
https://github.com/yeatse/UIWebView-PeekingSupport

