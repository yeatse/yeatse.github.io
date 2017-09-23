---
title: 使用 Objective-C Runtime 解决 unrecognized selector 错误
date: 2016-04-25 23:41:08
tags:
  - iOS
  - Objective-C
  - Runtime
categories:
  - 笔记
---

## 0x01 前言
`NSOperation` 类有一个属性 `name`，用以标记一个 NSOperation 对象。苹果提供这个属性的本意是为了调试方便，但实际上通过它我们还可以简便地实现一些业务需求，比如加入 `NSOperationQueue` 前检查去重和排序什么的。

但很可惜，这个属性是 `NS_AVAILABLE(10_10, 8_0)` 的，换言之如果在 iOS 7 以下的系统上使用这个属性的话，控制台会打印这样一行错误：

> unrecognized selector sent to instance

如果项目需要兼容 iOS 7 系统的话，我们就需要寻找一种方法，在 iOS 7 上也能方便地标记一个 NSOperation 。

<!-- more -->

## 0x02 Associated Objects
扩展一个已有的 Objective-C 类一般有两种方法： `Subclass` 和 `Category` 。这里我们选择 Category ，这样可以在扩展 NSOperation 的同时也扩展 NSBlockOperation 等它的子类。

使用 OC Runtime 的两个函数 `objc_getAssociatedObject` 和 `objc_setAssociatedObject` ，可以方便地在 Category 中给一个类增加属性。代码大概像这样：

```objectivec
- (NSString*)xxx_name {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)xxx_setName:(NSString*)name {
    objc_setAssociatedObject(self, @selector(xxx_name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
```

在头文件里声明 `xxx_name` 属性以后，在需要调用 `name` 的地方都改成 `xxx_name`，就可以完美兼容 iOS 7 以上的所有机型了。

## 0x03 IMP, SEL, Method
以上方法虽然实现了功能，但实际上我们抛弃了苹果提供的接口，这实在跟标题的**优雅**沾不上边。所以还需要继续使用 OC Runtime 的黑魔法，来尝试实现『安全地在低版本上调用高版本才有的API，同时完全不影响高版本API的功能』这个目的。

OC Runtime 有三个基础类型`IMP`，`SEL`和`Method`，它们的内容如下表：

名词 | 定义 | 说明
---|---|---
Selector | typedef struct objc_selector *SEL | 表示一个 OC 对象方法的方法名
Implementation | typedef id (*IMP)(id, SEL, ...) | 实际上是一个函数指针，指向了 Selector 对应的方法的具体实现
Method | typedef struct objc_method *Method | 封装了从 `Selector` 到 `Implementation` 的映射关系

三者之间的联系，可以用[NShipster](http://nshipster.com/method-swizzling/)的一段话来总结：

> A class (Class) maintains a dispatch table to resolve messages sent at runtime; each entry in the table is a method (Method), which keys a particular name, the selector (SEL), to an implementation (IMP), which is a pointer to an underlying C function.

在 NSOperation 的 Category 中，我们尝试调用与之相关的 OC Runtime 函数来实现以上目的：

```objectivec
+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL nameSEL = @selector(name);
        SEL setNameSEL = @selector(setName:);
        
        Method nameMethod = class_getInstanceMethod(class, nameSEL);
        Method setNameMethod = class_getInstanceMethod(class, setNameSEL);
        
        if (!nameMethod)
        {
            SEL xxxNameSEL = @selector(xxx_name);
            Method xxxNameMethod = class_getInstanceMethod(class, xxxNameSEL);
            class_addMethod(class, nameSEL, method_getImplementation(xxxNameMethod), method_getTypeEncoding(xxxNameMethod));
        }
        
        if (!setNameMethod)
        {
            SEL xxxSetNameSEL = @selector(xxx_setName:);
            Method xxxSetNameMethod = class_getInstanceMethod(class, xxxSetNameSEL);
            class_addMethod(class, setNameSEL, method_getImplementation(xxxSetNameMethod), method_getTypeEncoding(xxxSetNameMethod));
        }
    });
}

- (NSString*)xxx_name
{
    return objc_getAssociatedObject(self, @selector(xxx_name));
}

- (void)xxx_setName:(NSString*)name
{
    objc_setAssociatedObject(self, @selector(xxx_name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}
```

在以上代码中，`+ (void)load` 函数调用的时候，我们通过运行时的操作，来实现以下两个步骤：

1. 如果 NSOperation 本身有 name 属性，则什么也不做；
2. 如果 NSOperation 没有 name 属性，则在运行时动态添加 `name` 和 `setName:` 方法，使用我们自己的实现。

把这个 Category 加入工程以后，我们就可以安全地在 iOS 7 以上使用 NSOperation 的 name 属性了，好像它原生支持了低版本的 iOS 一样。

## 0x04 总结
本文演示了通过 OC Runtime 来优雅地为 `NSOperation` 的 `name` 属性增加了 iOS 7 以下的支持。实际上不止是 `NSOperation`，通过这个方法，很多高版本 iOS 新增的 API（比如 `[NSString containsString:]` 等）都可以用同样的方法移植到低版本系统上，只需要我们自己模拟实现相应的功能，然后通过 Category 提供给相应的 Selector 就可以了。

