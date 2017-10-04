---
title: 检测 iOS 项目中的内存泄漏
categories:
  - 笔记
date: 2016-07-19 21:36:50
tags:
  - Objective-C
  - iOS
---

一般来说，在 ARC 环境下，只要在使用 delegate、NSTimer、block 的时候注意一下不要出现循环引用，那么 Objective-C 对象的内存泄漏问题就可以轻松避免。

但是在实际项目中，一些错误的结构设计可能会导致难以发现的泄漏问题，比如像 `A -> B -> C -> ... -> A` 这种长环的循环引用，或者一个实例被一个 单例 持有，在 review 的时候可能会漏掉这些问题，这时就需要流程化的方式来检测了。

<!-- more -->

一个很方便的检测方法是重写 dealloc 方法：

```objectivec
- (void)dealloc {
    NSLog(@"%s", __func__);
}
```

只要目标对象有 dealloc 的 log 输出，就表示这里没有出现循环引用问题。

对于拿不到源文件的类，也可以通过类似的方法来实现：

```objectivec
// DeallocationObserver.h
#import <Foundation/Foundation.h>

@interface DeallocationObserver : NSObject

+ (instancetype)attachObserverToObject:(id)object;

@end



// DeallocationObserver.m
#import "DeallocationObserver.h"
#import <objc/runtime.h>

static const char ObserverTag;

@interface DeallocationObserver ()

- (instancetype)initWithParent:(id)parent;

@property (nonatomic, copy) void(^deallocationBlock)();

@end

@implementation DeallocationObserver

+ (instancetype)attachObserverToObject:(id)object {
    return [[self alloc] initWithParent:object];
}

- (instancetype)initWithParent:(id)parent {
    self = [super init];
    if (self) {
        NSString* deallocMsg = [NSString stringWithFormat:@"deallocated: %@", parent];
        self.deallocationBlock = ^{
            NSLog(@"%@", deallocMsg);
        };
        objc_setAssociatedObject(parent, &ObserverTag, self, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return self;
}

- (void)dealloc {
    if (self.deallocationBlock) {
        self.deallocationBlock();
    }
}

@end


// Usage:
NSObject* testObj = [NSObject new];
[DeallocationObserver attachObserverToObject:testObj];
testObj = nil;  // Output - deallocated: <NSObject: 0x7fce1a412c10>

```

因为 NSObject 对象在 dealloc 的时候也会把 objc_setAssociatedObject 关联的对象也一并 release 掉，通过监听 DeallocationObserver 的销毁时机，我们就可以检测到目标对象的销毁事件了。

由于 ARC 只对 NSObject 有效，所以对于 Core Foundation、Core Graphics 等非 NSObject 对象，就需要苹果提供的 Instruments 来检测内存泄漏问题了。 

按照 Instruments 的[官方文档](https://developer.apple.com/library/ios/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/FindingLeakedMemory.html)中的步骤，测试一下这段代码：

```objectivec
- (void)testMemoryLeak {
    CFMutableDataRef data = CFDataCreateMutable(kCFAllocatorDefault, 0);
    CGDataConsumerRef consumer = CGDataConsumerCreateWithCFData(data);
}
```

打开 Instruments - Leaks，选择目标设备和应用，然后点击🔴按钮，时间线面板就开始记录当前内存的使用情况：
![QQ20160719-1@2x](/images/2016/QQ20160719-1@2x.png)

可以看出，图中 28 s 的位置出现了内存泄漏，泄漏点刚好在 testMemoryLeak 方法上。

修改 Details 栏的 Leaks 选项，切换到 Call Tree，<kbd>⌘ + 2</kbd> 键切换到 Display Settings，然后勾选右边设置栏中的 Invert Call Tree 和 Hide System Libraries 选项，可以看到泄漏点具体的调用栈：
![QQ20160720-0@2x](/images/2016/QQ20160720-0@2x.png)

双击其中一个方法，Instruments 还会把出错的具体代码标识出来：
![QQ20160720-1@2x](/images/2016/QQ20160720-1@2x.png)

问题果然出现在 CFDataCreateMutable 和 CGDataConsumerCreateWithCFData 上，根据 Core Foundation 中 [关于方法命名的约定](https://developer.apple.com/library/ios/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html#//apple_ref/doc/uid/20001148-SW3)，含有 `Copy` 和 `Create` 的方法返回的对象需要调用 CFRelease 来释放，Core Graphics / Core Text 也一样，所以需要在 testMemoryLeak 方法中加入这两行，以解决这里的内存泄漏问题：

```objectivec
    CGDataConsumerRelease(consumer);
    CFRelease(data);
```



