---
layout: post
title: 扩展 UIButton 和 UICollectionViewCell 的响应区域
categories:
  - 笔记
date: 2016-06-13 20:56:29
tags:
  - iOS
---

## 0x01 前言

问题由一个项目需求引起。设计MM给的图大概像这样：

![snapshot](/assets/images/2016/snapshot.png)

如上图所示，列表的内容由服务器传回，且用户可编辑，显然这样的界面应该用 `UICollectionView` 来搭建。实现关闭按钮则需要在 `UICollectionViewCell` 的右上角添加一个 `UIButton`，并且要将 Cell 的 `clipsToBounds` 属性设置为 `NO` 以避免按钮被切掉一部分。

界面搭建好了，但默认状态下 UIButton 的点击响应范围跟它的显示区域一样小，导致这个按钮很难被点到，因此首先要解决的是这个按钮的热区扩展问题。

<!-- more -->

## 0x02 UIButton 的热区扩展
网络上介绍 UIButton 响应区域扩展的文章有很多，但大多数都是继承 UIButton 然后重写 `pointInside:withEvent:` 方法。实际上这种子类继承的方法在很多场景下是不合适的，因为这意味着每次用到这个功能都要将系统 UIButton 换成自己的子类。一个更符合 *Objective-C Style* 的方法是增加一个 UIButton 的 `Category`，然后在**不影响控件默认的行为**的情况下在 Category 里做文章。

新建一个 UIButton+ExpandHitArea 分类，像这样在头文件中添加一个 `hitTestEdgeInsets` 属性，用来配置热区扩展(根据语意实际上是缩小)的范围：

```objectivec
@interface UIButton (ExpandHitArea)

@property (nonatomic) UIEdgeInsets hitTestEdgeInsets;

@end
```

在 .m 文件中，我们将 `pointInside:withEvent:` 替换成修改过的方法：先将 `hitTestEdgeInsets` 的值与 `UIEdgeInsetsZero` 相比较，如果不等，则调用我们自己的扩展热区的逻辑；如果相等，则调用系统默认的 `pointInside:withEvent:` 方法，就像什么事也没发生过一样。最终代码如下：

```objectivec
@implementation UIButton (ExpandHitArea)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        YTSwizzleMethod([self class], @selector(pointInside:withEvent:), @selector(yt_pointInside:withEvent:));
    });
}

- (UIEdgeInsets)hitTestEdgeInsets {
    NSValue* value = objc_getAssociatedObject(self, _cmd);
    UIEdgeInsets insets = UIEdgeInsetsZero;
    [value getValue:&insets];
    return insets;
}

- (void)setHitTestEdgeInsets:(UIEdgeInsets)hitTestEdgeInsets {
    NSValue* value = [NSValue value:&hitTestEdgeInsets withObjCType:@encode(UIEdgeInsets)];
    objc_setAssociatedObject(self, @selector(hitTestEdgeInsets), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (BOOL)yt_pointInside:(CGPoint)point withEvent:(UIEvent*)event {
    UIEdgeInsets insets = self.hitTestEdgeInsets;
    if (UIEdgeInsetsEqualToEdgeInsets(insets, UIEdgeInsetsZero)) {
        return [self yt_pointInside:point withEvent:event];
    } else {
        CGRect hitBounds = UIEdgeInsetsInsetRect(self.bounds, insets);
        return CGRectContainsPoint(hitBounds, point);
    }
}

@end
```

其中在 `load` 方法里的 `YTSwizzleMethod` 是 [Method Swizzling](http://nshipster.com/method-swizzling/) 的具体实现，因为这段代码经常被用到所以我把它抽了出来，以下是函数的内容：

```objectivec
void YTSwizzleMethod(Class cls, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(cls, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzledSelector);
    
    BOOL didAddMethod = class_addMethod(cls, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    
    if (didAddMethod) {
        class_replaceMethod(cls, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

在工程中引入这个 Category 之后，我们对任意一个 UIButton 设置它的 hitTestEdgeInsets 属性，都可以将它的点击响应区域扩大或缩小。在本文的例子里，扩展 12 pt 差不多是个合适的值：

```objectivec
_closeButton.hitTestEdgeInsets = UIEdgeInsetsMake(-12, -12, -12, -12);
```

## 0x03 UICollectionViewCell 的响应区域修正

![snapshot](/assets/images/2016/snapshot2.png)

如图所示，按照上一段的步骤配置好以后，按钮的响应区域理应变成图中整块高亮的部分，但实际运行后真正的响应区域只有左下角的 A 部分。这是因为 UIKit 在检测点击响应区域时，首先询问的是父控件的 `pointInside:withEvent:` 方法，如果返回 `NO`，那么 UIKit 就认为点击区域在整个控件范围之外，不会继续遍历子控件，因此 Cell 的响应区域也需要跟随按钮一起扩展。除此之外我们还需要屏蔽掉 Cell 本身的事件响应，以防下一个 Cell 覆盖掉了上一个 Button 扩展后的热区 (图中 B 区域)。所以重写 UICollectionViewCell 的两个相关方法，内容如下：

```objectivec
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    CGPoint pointInButton = [self convertPoint:point toView:_closeButton];
    return [_closeButton pointInside:pointInButton withEvent:event] ? _closeButton : nil;
}

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    CGPoint pointInButton = [self convertPoint:point toView:_closeButton];
    return [_closeButton pointInside:pointInButton withEvent:event];
}
```



