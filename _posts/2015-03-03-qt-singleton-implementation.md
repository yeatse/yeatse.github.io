---
layout: post
title: Qt中单例模式的实现
tags:
  - Qt
id: 65
categories:
  - 笔记
date: 2015-03-03 19:13:14 +0800
---

最简单的写法：

```c++
static MyClass* MyClass::Instance()
{
    static MyClass inst;
    return &inst;
}
```

过去很长一段时间一直都这么写，简单粗暴有效。但是直接声明静态对象会使编译出的可执行文件增大，也有可能出现其他的一些问题，所以利用了Qt自带的智能指针`QScopedPointer`和线程锁`QMutex`，改成了需要时才动态初始化的模式：

```c++
static MyClass* MyClass::Instance()
{
    static QMutex mutex;
    static QScopedPointer<MyClass> inst;
    if (Q_UNLIKELY(!inst)) {
        mutex.lock();
        if (!inst) {
            inst.reset(new MyClass);
        }
        mutex.unlock();
    }
    return inst.data();
}
```

既保证了线程安全又防止了内存泄漏，效率也没降低太多，简直完美。

<!-- more -->

可惜每次都要重复这么几行实在麻烦，于是写了一个模板类：

```c++
template <class T>
class Singleton
{
public:
    static T* Instance()
    {
        static QMutex mutex;
        static QScopedPointer<T> inst;
        if (Q_UNLIKELY(!inst)) {
            mutex.lock();
            if (!inst) {
                inst.reset(new T);
            }
            mutex.unlock();
        }
        return inst.data();
    }
};
```

使用的时候直接这样——

```c++
MyClass* inst = Singleton<MyClass>::Instance();
```

除了用模板类，还可以利用c++中强大的宏：

```c++
#define DECLARE_SINGLETON(Class) \
Q_DISABLE_COPY(Class) \
public: \
    static Class* Instance() \
    { \
        static QMutex mutex; \
        static QScopedPointer<Class> inst; \
        if (Q_UNLIKELY(!inst)) { \
            mutex.lock(); \
            if (!inst) inst.reset(new Class); \
            mutex.unlock(); \
        } \
        return inst.data(); \
    }
```

然后声明的时候，填加一行这个宏：

```c++
class MyClass
{
    DECLARE_SINGLETON(MyClass);    // 声明单例模式
    //...
}
```

好评好评。

当然，为了要保证真的是单例模式，还要把构造函数限制为private，不然以后什么时候忘记了这码事，在外面又new了一下就不好了。

另外Qt本身自带了一个宏`Q_GLOBAL_STATIC`，也有类似单例模式的效果，`QThreadPool::globalInstance()`函数的实现就是利用了这个宏。不过它的主要用处是声明全局变量，和Singleton还是有差别的。

