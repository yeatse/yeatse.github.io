---
title: 在 iOS APP 崩溃时弹出友好提示框
categories:
  - 笔记
date: 2016-07-30 14:57:57
tags:
  - iOS
  - RunLoop
---

昨天补了 iOS RunLoop 相关的基础知识，在一部[讨论 RunLoop 实现细节的视频](http://v.youku.com/v_show/id_XODgxODkzODI0.html)的最后面，@sunnyxx 讲到了一个很有意思的黑科技————“让 App 在 Crash 的时候回光返照”，内容大致如下：

> ```objectivec
//取当前 run loop
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
//取 run loop 所有运行的 mode
NSArray *allModes = CFBridgingRelease(CFRunLoopCopyAllModes(runLoop));
while (1) {
    for (NSString *mode in allModes) {
    //在每个 mode 中轮流运行至少 0.001 秒
        CFRunLoopRunInMode((CFStringRef)mode, 0.001, false);
    }
}
> ```
> 对于因为接收到 crash 的 signal 而挂掉的程序，可以在接收到 crash 的信号之后重新起一个 run loop 然后跑起来。但是这个并不能保证 app 能像原来一样能正常运行，只能是利用它来在奄奄一息的状态下弹出一些友好的错误信息。

<!-- more -->

自己写了个 Demo 测试了一下，首先随便触发一个 unrecognized selector 错误：

```objectivec
UIView* view = (id)[NSObject new];
view.hidden = YES;
```

捕获到的崩溃栈如下图：
![](/images/2016/14698640741585.jpg)

可以看到，RunLoop 在 Source0 中处理点击事件，调用了未定义的 selector 之后经过一系列消息转发，最后调用 objc_exception_throw 抛出了异常。

Foundation 提供的 NSSetUncaughtExceptionHandler 方法可以截获到这个异常。通过它设置一个回调函数，在里面展示一个 UIAlertViewController，再按照上面的方式手动启动一个 RunLoop 来监听手势事件，用 Swift 实现如下：

```swift
NSSetUncaughtExceptionHandler { (exception) in
    var shouldRun = true
    
    let runLoop = CFRunLoopGetCurrent()
    
    let alertCtrl = UIAlertController(title: "Oops", message: "Your app crashed! OAO", preferredStyle: .Alert)
    alertCtrl.addAction(UIAlertAction(title: "OK", style: .Default, handler: { (_) in
        shouldRun = false
    }))
    
    guard let rootViewController = UIApplication.sharedApplication().keyWindow?.rootViewController else {
        return
    }
    
    rootViewController.presentViewController(alertCtrl, animated: true, completion: nil)
    
    let allModesAO = CFRunLoopCopyAllModes(runLoop) as [AnyObject]
    guard let allModes = allModesAO as? [CFStringRef] else {
        return
    }
    
    while (shouldRun) {
        for mode in allModes {
            CFRunLoopRunInMode(mode, 0.001, false)
        }
    }
}
```

这样就可以在程序 Crash 之前弹出一个提示框了。

一个需要注意的地方是，如果程序使用了第三方 SDK 做崩溃收集的话，很可能由于第三方 SDK 也注册了 UncaughtExceptionHandler 导致自己注册的函数被覆盖掉。另外由于注册的回调[只对 Foundation 对象的异常有效](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-exception.mm#L673-L682)，所以这个方法只对 NSException 起作用，对于 BAD_ACCESS、std::terminate() 等非 Foundation 异常还是没有办法的。至于用 @throw 还是 c++ 的 throw 这个倒是影响不大，亲自尝试了之后发现两者都是抛 NSException 有效，抛其他对象 (包括非 NSException 的 NSObject)无效的。


