---
layout: post
title: 'React Native 一处诡异 crash: RCTFont SIGABRT'
categories:
  - 笔记
date: 2017-09-29 01:10:29
tags:
  - iOS
  - React Native
---

目前在做的一个项目迁移到 React Native 已经一年多了，也意味着踩了一年的坑，感觉光填上各种奇怪的坑都会让自己的水平提升不少。最近解决了一个占比接近 10% 的崩溃，在这里记录一下。

<!-- more -->

![屏幕快照](/assets/images/2017/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-07-31%20%E4%B8%8B%E5%8D%889.38.38.png)

崩溃的方法是
`+[RCTFont updateFont:withFamily:size:weight:style:variant:scaleMultiplier:]`，在这个位置会抛出 `mutex lock failed: Invalid argument` 的异常。

查了下崩溃栈，大概长这样：

```
Thread 24 Crashed:
0   libsystem_kernel.dylib              __pthread_kill + 8
1   libsystem_c.dylib                   abort + 140
2   libc++abi.dylib                     __cxa_bad_cast + 0
3   libc++abi.dylib                     default_terminate_handler() + 280
4   libobjc.A.dylib                     _objc_terminate() + 140
5   libc++abi.dylib                     std::__terminate(void (*)()) + 16
6   libc++abi.dylib                     __cxxabiv1::exception_cleanup_func(_Unwind_Reason_Code, _Unwind_Exception*) + 0
7   libc++.1.dylib                      std::__1::__throw_system_error(int, char const*) + 88
8   bee                                 +[RCTFont updateFont:withFamily:size:weight:style:variant:scaleMultiplier:] + 780
9   bee                                 -[RCTShadowText _attributedStringWithFontFamily:fontSize:fontWeight:fontStyle:letterSpacing:useBackgroundColor:foregroundColor:backgroundColor:opacity:] + 580
10  bee                                 -[RCTShadowText attributedString] + 192
11  bee                                 -[RCTShadowText recomputeText] + 28
12  bee                                 -[RCTTextManager uiBlockToAmendWithShadowViewRegistry:] + 612
13  bee                                 -[RCTComponentData uiBlockToAmendWithShadowViewRegistry:] + 96
14  bee                                 -[RCTUIManager _layoutAndMount] + 220
15  bee                                 __36-[RCTBatchedBridge batchDidComplete]_block_invoke + 52
16  libdispatch.dylib                   _dispatch_call_block_and_release + 24
17  libdispatch.dylib                   _dispatch_client_callout + 16
18  libdispatch.dylib                   _dispatch_queue_serial_drain + 928
19  libdispatch.dylib                   _dispatch_queue_invoke + 884
20  libdispatch.dylib                   _dispatch_root_queue_drain + 540
21  libdispatch.dylib                   _dispatch_worker_thread3 + 124
22  libsystem_pthread.dylib             _pthread_wqthread + 1096
23  libsystem_pthread.dylib             start_wqthread + 4
```

从崩溃栈上可以看出来是 RN 库 RCTFont 模块出的问题，除此之外再也找不到其他信息，放 google 搜了一圈，只能找到别人提的同样的问题 -- [RCTFont SIGABRT crash](https://github.com/facebook/react-native/issues/13588)、[App crashes for "mutex lock failed: Invalid argument"](https://github.com/facebook/react-native/issues/14526)，却没人提出解决方法，看来只能自己解了。

首先把可执行文件拉到 [Hopper](https://www.hopperapp.com) 里，定位到崩溃处，看一下对应的汇编指令：

![屏幕快照 2017-08-01 下午3.03.38](/assets/images/2017/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-01%20%E4%B8%8B%E5%8D%883.03.38.png)

可以得知应用在 RCTFont 内部使用 std::mutex 加锁的时候抛出了异常，对应于 [RCTFont.mm 第 103 行](https://github.com/facebook/react-native/blob/6ce42441ec98bb8543e8eff8849ce50e076ce520/React/Views/RCTFont.mm#L103)：

```objc
{
    std::lock_guard<std::mutex> lock(fontCacheMutex); ///< 在这里挂掉了
    if (!fontCache) {
      fontCache = [NSCache new];
    }
    font = [fontCache objectForKey:cacheKey];
} 
```

对比收集到的各种崩溃样本，可以总结出以下几处共同点：

- 主线程都有 `handleApplicationDeactivationWithScene` → `+[_UIAlertManager hideAlertsForTermination]` → `exit` 的调用；
- crash 都发生在后台；
- 都在 `mutex::lock` 的时候抛出了异常。

给 `handleApplicationDeactivationWithScene` 等方法下个断点，发现只有在用户手动 kill 掉 app 时这些方法才会被调用，猜测这个时候系统可能正在做一些清理工作，这时候如果有其他线程调用了 `mutex::lock` 可能就会导致异常。

假设上面的猜测为真，那么解决问题的关键是让应用进程结束时不调用 `mutex::lock` 方法。查看 React Native 源码，crash 处的代码只被一个方法调用，即上面提到的 `+[RCTFont updateFont:withFamily:size:weight:style:variant:scaleMultiplier:]`

![屏幕快照 2017-08-09 上午11.45.36](/assets/images/2017/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-09%20%E4%B8%8A%E5%8D%8811.45.36.png)

这个方法的调用者有多个，但最终都走到了 React Native 模块的各个属性的设置方法里，在 React Native 线程里，由 `RCTBatchedBridge` → `RCTJSCExecutor` 驱动

![屏幕快照 2017-08-23 下午12.25.22](/assets/images/2017/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-23%20%E4%B8%8B%E5%8D%8812.25.22.png)

注意到 js 每次调用时都会检查 `_valid` 属性，所以只需要在进程结束时把 `_valid` 置为 false，crash 处的代码就不会被执行。`RCTBatchedBridge` 和它的包装类 `RCTBridge` 刚好暴露出了 `invalidate` 方法，可以把 `_valid` 置为 false：

```objc
// -[RCTBridge invalidate]
- (void)invalidate
{
  RCTBridge *batchedBridge = self.batchedBridge;
  self.batchedBridge = nil;
 
  if (batchedBridge) {
    RCTExecuteOnMainQueue(^{
      [batchedBridge invalidate];
    });
  }
}
```

```objc
// -[RCTBatchedBridge invalidate]
- (void)invalidate
{
  if (!_valid) {
    return;
  }
 
  _loading = NO;
  _valid = NO;
 
  // Invalidate modules
  for (RCTModuleData *moduleData in _moduleDataByID) {
    id<RCTBridgeModule> instance = moduleData.instance;
    [instance invalidate];
    [moduleData invalidate];
  }
}
```

用户杀掉 app 时，系统会调用 App Delegate 的 `applicationWillTerminate` 方法，所以我们需要在这里调用一下 `-[RCTBridge invalidate]`，使 `RCTBridge` 失效，这样就不会再触发导致 crash 的代码了。

但问题是，`-[RCTBridge invalidate]` 方法是异步的，`applicationWillTerminate` 一返回马上就进入 `exit` 函数，这时候程序还来不及干掉 `RCTBridge`，crash 处的代码还是会执行。所以这里还需要借用 runloop 让 `applicationWillTerminate` 卡一会儿，直到 `RCTBridge` 完全停止：

```objc
- (void)applicationWillTerminate:(UIApplication *)application {
    RCTBridge *batchedBridge = [self.bridge valueForKey:@"batchedBridge"];
    [self.bridge invalidate];
    
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    NSArray<NSRunLoopMode> *allModes = CFBridgingRelease(CFRunLoopCopyAllModes(runLoop.getCFRunLoop));
    while (batchedBridge.moduleClasses) {
        for (NSRunLoopMode mode in allModes) {
            [runLoop runMode:mode beforeDate:[NSDate dateWithTimeIntervalSinceNow:0.1]];
        }
    }
}
```

这里用到了几处 trick：

- 在这里阻塞 `applicationWillTerminate` 要用 runloop，而不是简单的 sleep，原因是 invalidate 方法内部向主线程分发了一些事要做（见 [RCTBatchedBridge.mm 源码](https://github.com/facebook/react-native/blob/6ce42441ec98bb8543e8eff8849ce50e076ce520/React/Base/RCTBatchedBridge.mm#L714-L735)），需要主线程有处理事件的能力；
- invalidate 方法最后一步是置空 `RCTBatchedBridge` 的 `moduleClasses` 属性，所以可以通过它是否为空来确定 RCTBridge 完全停止的时机。 batchedBridge 是私有属性所以需要 kvc 来拿到。

加入工程发版之后，这个 crash 就消失了，撒花。

![屏幕快照 2017-08-23 下午1.05.44](/assets/images/2017/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-23%20%E4%B8%8B%E5%8D%881.05.44.png)

---

## One more thing

为什么主线程调用了 `exit` 之后，其他线程调用 `mutex::lock` 方法时会抛异常？

```objc
static NSCache *fontCache;
static std::mutex fontCacheMutex;
 
NSString *cacheKey = [NSString stringWithFormat:@"%.1f/%.2f", size, weight];
UIFont *font;
{
  std::lock_guard<std::mutex> lock(fontCacheMutex);
  if (!fontCache) {
    fontCache = [NSCache new];
  }
  font = [fontCache objectForKey:cacheKey];
}
```

回过来看崩溃位置代码，第 2 行声明了一个局部变量 fontCacheMutex，通过汇编指令可以看出，它在创建的时候通过 `__cxa_atexit` 方法注册了一个销毁函数：

![DX-20170823@2x](/assets/images/2017/DX-20170823@2x.png)

主线程调用 `exit` 方法时，会通过 `__cxa_finalize` 逐个调用之前注册的销毁函数（参考 [atexit.c 源码](https://opensource.apple.com/source/Libc/Libc-1158.50.2/stdlib/FreeBSD/atexit.c.auto.html)），这个静态变量 fontCacheMutex 随之销毁，之后再调用这个销毁过的 mutex 对象的方法自然会 crash 了。

