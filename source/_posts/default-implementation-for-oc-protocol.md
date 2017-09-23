---
title: 给 Objective-C 中的 Protocol 加上默认的实现
categories:
  - 笔记
date: 2016-06-20 22:04:44
tags:
  - iOS
  - Objective-C
---

## 0x01 Abstract Class

Java、C++ 等 OOP 语言有一个`抽象类`的概念，即一个类实现了部分方法，另一部分的方法必须由继承它的子类来实现。Objective-C 在设计上没有这个概念，转而提供了用途类似的 `协议`，除了不能给方法加默认实现以外，与抽象类的用法大体相同。但是在实际项目中，让一个协议实现一些共通的方法还是很有必要的，比如很多类都遵守了某一个协议，而这个协议中某一个方法的实现大体上都一样的时候，在每一个子类内部都 copy 一份同样的代码就不太合适了。

<!-- more -->

一种规避 copy 的做法是把它的实现抽离到全局方法中，比如下面的协议：

```objectivec
@protocol MyProtocol <NSObject>

- (void)method1;
- (void)method2;

@end
```

如果所有子类的 `method2` 的实现都差不多，就可以将它抽到一个全局方法(或者一个单例类的方法)中：

```objectivec
void MyProtocolMethod2(id<MyProtocol> instance) {
    // Do with myprotocol...
}
```

另一种办法是抛弃 `@protocol`，直接使用 `@interface`，然后使用文档说明的方式`约定`它是一个抽象类：

```objectivec
// MyBaseClass.h
@interface MyBaseClass : NSObject

/// 这个方法必须由子类重写
- (void)method1;

/// 这个方法可以被子类重写
- (void)method2;

@end


// MyBaseClass.m
@implementation MyBaseClass

- (void)method1 {
    // 如果没有重写就报错...
    NSAssert(method_getImplementation(class_getInstanceMethod(self.class, _cmd)) !=
             method_getImplementation(class_getInstanceMethod([MyBaseClass class], _cmd)),
             @"method1 must be overriden!");
}

- (void)method2 {
    // A default implementation...
}

@end
```

以上两个方法都可以达成目的，但都有一些缺陷：前一种方法把 MyProtocol 相关的代码放到了全局环境中，不优雅；后一种方法在编译阶段没有提示，需要由开发人员仔细阅读文档才能避免误用。[StackOverflow 的一篇答案](http://stackoverflow.com/questions/4330656/how-do-i-provide-a-default-implementation-for-an-objective-c-protocol)还提供了另一个方案：在每一个子类的 `+initialize` 方法中通过 `class_addMethod` 把协议的默认实现加到方法列表当中，但这样也略显繁琐。

## 0x02 EXTConcreteProtocol

一个第三方库 [libextobjc](https://github.com/jspahrsummers/libextobjc) 通过 `EXTConcreteProtocol` 神奇地实现了这个功能，使用方法与原生协议类似：

```objectivec
// MyProtocol.h
@protocol MyProtocol <NSObject>
@required
- (void)method1;

@concrete
- (void)method2;

@end


// MyProtocol.m
@concreteprotocol(MyProtocol)

- (void)method1 {}

- (void)method2 {
    // A default implementation
}

@end
```

这样声明以后，对于任何遵守 MyProtocol 协议的类，如果没有重写 `method2` 方法，都会有一个在 MyProtocol.m 中声明的默认实现。

这个库为什么这么吊，`@concrete` 和 `@concreteprotocol` 到底做了什么。其实 `concrete` 只是 `optional` 的别名，为了提示调用者就算不重写这个方法也一定会有的，重点还是在 `concreteprotocol` 宏上。

查看 EXTConcreteProtocol 源码可以知道，`@concreteprotocol(MyProtocol)` 这一行通过宏定义的方式生成了这样的一个包装类：

```objectivec
@interface MyProtocol_ProtocolMethodContainer : NSObject <MyProtocol>
@end

@implementation MyProtocol_ProtocolMethodContainer

+ (void)load {
    if (!ext_addConcreteProtocol(objc_getProtocol("MyProtocol"), self))
        fprintf(stderr, "ERROR: Could not load concrete protocol %s\n", "MyProtocol");
}

__attribute__((constructor))
static void ext_MyProtocol_inject (void) {
    ext_loadConcreteProtocol(objc_getProtocol("MyProtocol"));
}

@end
```

其中 `ext_addConcreteProtocol` 在 `load` 方法中被调用，它的作用是把将要对 MyProtocol 进行的注入操作缓存到一个全局列表中，除此之外还有一些边界条件的判断和加锁什么的。

`__attribute__((constructor))` 是[ GCC 的一个编译器指令](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html)（其实是 Clang 的指令，但我翻遍了 Clang 的官方文档并没有找到关于 constructor 的描述- -），被它标记的函数会在整个 Objective-C runtime 初始化完毕之后，在 `main()` 函数之前被调用。这时 `ext_loadConcreteProtocol` 函数会遍历 runtime 中所有的 Class，对其中每一个遵从 MyProtocol 协议的 Class 进行缓存过的注入操作：

```objectivec
if (class_getInstanceMethod(metaclass, selector)) {
    // it does exist, so don't overwrite it
    continue;
}

// add this class method to the metaclass in question
IMP imp = method_getImplementation(method);
const char *types = method_getTypeEncoding(method);
if (!class_addMethod(metaclass, selector, imp, types)) {
    fprintf(stderr, "ERROR: Could not implement class method +%s from concrete protocol %s on class %s\n",
    sel_getName(selector), protocol_getName(protocol), class_getName(class));
}
```

虽然调用层级很复杂，但最终还是调用了 `class_addMethod` 方法给 Class 自动加上了默认的实现，原理跟上面的 StackOverflow 给的答案是一样的。


