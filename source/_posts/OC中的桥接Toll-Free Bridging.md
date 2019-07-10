---
title: OC中的桥接机制 - Toll-Free Bridging
date: 2018-05-07 23:48:53
tags:
    - 基础知识
categories:
    - 基础知识
---

Core Foundation框架(CoreFoundation.framework)具有一系列概念性的基于Objective-C的基础框架, 以C语言实现了编程的接口, 它们为iOS应用程序提供基本数据管理和服务功能. Core Foundation可以访问低级函数, 原始数据类型和各种集合类型, 并与Foundation框架无缝桥接. 而Toll-free bridging 就是ARC下OC对象和Core Foundation对象之间的桥梁.

在开发iOS应用程序时我们有时会用到Core Foundation对象, 下面简称CF. 例如Core Graphics, Core Text, 并且我们可能需要将CF对象和OC对象进行相互转化.
 
MRC 下的 Toll－Free Bridging 因为不涉及内存管理的转移, 相互之间可以直接交换使用. 当使用 ARC 时, 由于 ARC 能够管理 Objective-C 对象的内存, 却不能管理 CF 对象, 此时编译器不知道该如何处理这个同时有 ObjC 指针和 CFTypeRef 指向的对象, 所以除了转换类型, 还需要指定内存管理所有权的改变, 我们需要使用__bridge, __bridge_retained, __bridge_transfer 修饰符来告诉编译器该如何去做.

## __bridge
使用`__bridge`意味着告诉编译器不做任何内存管理的事情, 编译器负责管理ARC下 ObjC 端的引用计数的事情, 开发者继续管理好 CF 端的事情.

1. 如果OC创建的对象要给CF使用, 那么CF只管自己使用, OC会处理好所有的内存管理相关的东西. 如果OC中创建的对象被销毁, 那么意味着CF所指向的对象也被销毁了.

    ```
    NSString *nsStr = [self createSomeNSString];
    CFStringRef cfStr = (__bridge CFStringRef)nsStr;
    CFUseCFString(cfStr);
    // CFRelease(cfStr); 不需要
    ```
2. 如果是CF创建的对象给OC使用, 那么我们需要在CF端处理好内存管理的相关事情, 需要在bridge之后release对象.
 
    ```
    CFStringRef hello = CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
    NSString *world = (__bridge NSString *)(hello);
    CFRelease(hello); // 需要
    [self useNSString:world];
    ```

## __bridge_retained
以OC创建的对象给CF使用为例: 如果我们创建了一个OC的对象给CF使用, 我们是不需要在CF中管理对象的内存. 当OC的对象被释放之后, 我们仍然使用这个对象的话, 就有可能造成崩溃. 那么如果我们使用`__bridge_retained`, 就相当于CF也copy了一个对象, 在bridge的时候编译器会去retain对象. 因此, 我们如果使用`__bridge_retained`的话, 在CF中我们也是要考虑对象的内存管理.

## __bridge_transfer
`__bridge_transfer`主要用来告诉编译过期, 转交对象的所有权, 内存由使用方管理. `CFBridgingRetain`的作用类似. 我们以CF创建的对象给OC使用为例: 在使用__bridge_transfer修饰符转换后CF指针不再持有对象.
```
//写法一
CFStringRef hello = CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
NSString *world = (__bridge_transfer NSString *)(hello);
// CFRelease(hello); 不需要
[self useNSString:world];

//写法二
NSString *world = (__bridge_transfer NSString *)CFStringCreateWithCString(kCFAllocatorDefault, "hello", kCFStringEncodingUTF8);
[self useNSString:world];
```

---------
参考资料:
1.[Toll-Free Bridging](http://gracelancy.com/blog/2014/04/21/toll-free-bridging/)

2.[Toll-Free Bridging](https://www.jianshu.com/p/bec56131eaeb)


