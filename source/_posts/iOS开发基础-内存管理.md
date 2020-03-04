---
title: iOS开发基础-内存管理
date: 2019-07-25 11:17:36
tags:
    - 基础知识
categories:
    - iOS开发-基础
---

内存管理是开发的基本功之一, 下面将从以下几个方面来聊一下iOS开发的内存管理.

# 1 内存分配
关于内存分配可以看[之前的总结](http://hchong.net/2016/09/18/%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D/);

# 2 引用计数原理
ARC是编译器（时）特性，而不是运行时特性，更不是垃圾回收器(GC). Objc是通过引用计数来管理内存, Cocoa为我们提供了这些内存管理的准则:

* 自己生成的对象, 自己持有.
* 非自己生成的对象, 自己也能持有.
* 不在需要自己持有对象的时候, 释放.
* 非自己持有的对象无需释放.

[Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)详细讲解了引用计数的原理. 简单来说就是ARC在代码编译阶段, 会自动在代码的上下文中成对插入retain以及release, 保证引用计数能够正确管理内存. 如果对象不是强引用类型, 那么ARC的处理也会进行相应的改变.

## 2.1 引用计数如何存储

* 有些对象如果支持使用 TaggedPointer, 苹果会直接将其指针值作为引用计数返回.
* 如果当前设备是 64 位环境并且使用 Objective-C 2.0, 那么"一些"对象会使用其 isa 指针的一部分空间来存储它的引用计数.
* 否则 Runtime 会使用一张散列表(哈希表)来管理引用计数.

后面的改变引用计数和获取引用计数, 都是以存储引用计数为基础的.

### 2.1.1 TaggedPointer
判断当前对象是否在使用 TaggedPointer 是看标志位是否为 1. 

```
#if SUPPORT_MSB_TAGGED_POINTERS
#   define TAG_MASK (1ULL<<63)
#else
#   define TAG_MASK 1

inline bool 
objc_object::isTaggedPointer() 
{
#if SUPPORT_TAGGED_POINTERS
    return ((uintptr_t)this & TAG_MASK);
#else
    return false;
#endif
}
```
id 其实就是 `objc_object *` 的简写（`typedef struct objc_object *id;`），它的 `isTaggedPointer()` 方法经常会在操作引用计数时用到, 因为这决定了存储引用计数的策略.

### 2.1.2 isa指针
用 64 bit 存储一个内存地址显然是种浪费，毕竟很少有那么大内存的设备。于是可以优化存储方案，用一部分额外空间存储其他内容。isa 指针第一位为 1 即表示使用优化的 isa 指针.

在 64 位环境下，优化的 isa 指针并不是就一定会存储引用计数. has_sidetable_rc 的值如果为 1，那么引用计数会存储在一个叫 SideTable 的类的属性中，后面会详细讲.

### 2.1.3 散列表(哈希表)
散列表（Hash table，也叫哈希表), 是根据键（Key）而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表.

散列表(哈希表)来存储引用计数具体是用 DenseMap 类来实现，这个类中包含好多映射实例到其引用计数的键值对，并支持用 DenseMapIterator 迭代器快速查找遍历这些键值对.

## 2.2 引用计数表
引用计数表就是个是个散列表(哈希表).

### 2.2.1 SideTable
再介绍下 SideTable 这个类，它用于管理引用计数表(哈希表)和 weak 表，并使用 spinlock_lock 自旋锁来防止操作表结构时可能的竞态条件。它用一个 64*128 大小的 uint8_t 静态数组作为 buffer 来保存所有的 SideTable 实例。并提供三个公有属性:

``` C
spinlock_t slock;//保证原子操作的自选锁
RefcountMap refcnts;//保存引用计数的散列表
weak_table_t weak_table;//保存 weak 引用的全局散列表
```

还提供了一个工厂方法，用于根据对象的地址在 buffer 中寻找对应的 SideTable 实例:

```C
static SideTable *tableForPointer(const void *p)
```

### 2.2.2 weak表
weak 表的作用是在对象执行 dealloc 的时候将所有指向该对象的 weak 指针的值设为 nil，避免悬空指针。苹果使用一个全局的 weak 表来保存所有的 weak 引用。并将对象作为键，weak_entry_t 作为值。weak_entry_t 中保存了所有指向该对象的 weak 指针。

这是 weak 表的结构:

```
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

## 2.3 获取引用计数
在非 ARC 环境可以使用 retainCount 方法获取某个对象的引用计数，其会调用 objc_object 的 rootRetainCount() 方法.

在 ARC 时代除了使用 Core Foundation 库的 CFGetRetainCount() 方法，也可以使用 Runtime 的 _objc_rootRetainCount(id obj) 方法来获取引用计数，此时需要引入 <objc/runtime.h> 头文件。这个函数也是调用 objc_object 的 rootRetainCount() 方法.

rootRetainCount()的内部实现也是参考引用计数的存储:

* TaggedPointer 直接通过对象内存地址获取引用计数.
* 64 位环境优化的存储在 isa 指针中
* 其他的是从从 SideTable 的静态方法获取当前实例对应的 SideTable 对象, 然后在引用计数表中用迭代器查找当前实例对应的键值对，获取引用计数值，并在此基础上 +1 并将结果返回.

## 2.4 修改引用计数
### 2.4.1 retain 和 release 
在非 ARC 环境下可以使用 retain 和 release 方法对引用计数进行加一减一操作, 它们分别调用了 _objc_rootRetain(id obj) 和 _objc_rootRelease(id obj) 函数. ARC 环境下是使用_objc_rootRetain(id obj) 和 _objc_rootRelease(id obj) 函数.

实现跟获取引用计数类似，先是看是否支持 TaggedPointer（毕竟数据存在栈指针而不是堆中，栈的管理本来就是自动的），否则去操作 SideTable 中的 refcnts 属性，这与获取引用计数策略类似。sidetable_retain() 将 引用计数加一后返回对象，sidetable_release() 返回是否要执行 dealloc 方法.

### 2.4.2 alloc, new, copy, mutableCopy
根据编译器的约定，这以这四个单词开头的方法都会使引用计数加一。而 new 相当于调用 alloc 后再调用 init.

alloc 和 new 最终都会调用 callAlloc，默认使用 Objective-C 2.0 且忽视垃圾回收和 NSZone，那么后续的调用顺序依次是为：

```C
class_createInstance()
_class_createInstanceFromZone()
calloc()
```
calloc() 函数相比于 malloc() 函数的优点是它将分配的内存区域初始化为0，相当于 malloc() 后再用 memset() 方法初始化一遍。

copy 和 mutableCopy 都是基于 NSCopying 和 NSMutableCopying 方法约定，分别调用各类自己实现的 copyWithZone: 和 mutableCopyWithZone: 方法。这些方法无论实现方式是深拷贝还是浅拷贝，都会增加引用计数。（有些类的策略是懒拷贝，只增加引用计数但并不真的拷贝，等对象内容发生变化时再拷贝一份出来，比如 NSArray）。

retain 方法加符号断点会发现 alloc, new, copy, mutableCopy 这四个方法都会通过 Core Foundation 的 CFBasicHashAddValue() 函数来调用 retain 方法。其实 CF 有个修改和查看引用计数的入口函数 __CFDoExternRefOperation.

## 2.5 Autorelease
[这里](http://hchong.net/2018/03/12/Autoreleasepool%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)可以看到之前关于AutoreleasePool的总结.

[这里](https://www.jianshu.com/p/32265cbb2a26)有一篇Draveness大神关于AutoreleasePool的剖析, 十分详细.

[这里](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)是sunny大神的黑幕背后的Autorelease.

Autorelease: 向一个对象发送延迟释放信息，使得这个对象可以在作用域意外范围被使用；典型的例子就是将一个对象作为返回值给调用者，如果不延迟释放，这个返回值在出了所在函数范围就被立即释放，调用者拿到的永远是nil，因为iOS的内存遵循谁申请谁释放的原则，当向一个对象发送了autorelease消息，实际上就是将该对象放入autoreleasePool池子动，等到延迟到适当的时机（通常是在NSRunloop即将进入休眠或者退出时）进行释放（对池子中的每个对象发送release消息）.

使用new/alloc/copy/mutablecopy等函数实例化的对象不会放入自动释放池，相反，用其他简便构造器获（如+(NSMutableArray *)array）得到的实例则会放入自动释放池.

在没有手加Autorelease Pool的情况下，Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop.

* 自动释放池是由 AutoreleasePoolPage 以双向链表的方式实现的
* 当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
* 调用 AutoreleasePoolPage::pop 方法会向栈中的对象发送 release 消息

# 3 对象的内存分布
`Person *p1 = [Person new]`这行代码主要做了下面的事: 

* 在堆内存中申请1块合适大小的空间
* 在这块内存上根据类模版创建对象。类模版中定义了什么属性就依次把这些属性声明在对象中；对象中还存在一个属性叫做isa，是一个指针，指向对象所属的类在代码段中地址
* 初始化对象的属性。这里初始化有几个原则：a、如果属性的数据类型是基本数据类型则赋值为0；b、如果属性的数据类型是C语言的指针类型则赋值为NULL；c、如果属性的数据类型为OC的指针类型则赋值为nil。
* 返回堆空间上对象的地址

上面程序的内存分配, 如下所示:
![内存分配图](https://user-gold-cdn.xitu.io/2017/5/22/e92671eed683172d1f957822f184a559?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

静定的isa指向图: 
![分析图](https://upload-images.jianshu.io/upload_images/2752872-8bfac339e72aeded.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/550/format/webp)

>程序运行时，运行时系统通过自身的库函数在内存中（推测是代码区或静态区）为源代码中的每个类创建了类对象即meta-class（元类）的实例对象，类对象（类的代表）包含有类函数列表等。
如果我们要创建某个类的实例对象，对象的isa指针指向类对象（类的代表），类对象（类的代表）的isa指针指向根类（NSObject）的类对象（根类的代表），根类的isa指针指向根类的元类，而根类的元类是自身（NSObject）。 根元类的超类是NSObject，而isa指向了自己，而NSObject的超类为nil，也就是它没有超类。

OC的对象内存布局: 
![OC的对象内存布局](http://img.souche.com/f2e/b1ae662279e9e867861ccbea3b5f116f.png)

当添加一个Method的时候，变化为: 
![当添加一个Method的时候，变化为](http://img.souche.com/f2e/646ffe9d22d99bb4abced117da06548b.png)

# 4 内存泄漏
之前总结过关于内存泄漏的文章, 可以看[这里](http://hchong.net/2018/04/04/iOS%E4%B8%AD%E5%B8%B8%E8%A7%81%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F/).

常见的有Block下的循环引用, delegate循环引用问题, NSTimer的循环引用, 非OC对象内存处理(free), 地图类处理, 大次数循环内存暴涨问题.

## 4.1 内存泄漏的排查
内存泄漏(memory leak): 是指申请的内存空间使用完毕之后未回收。 一次内存泄露危害可以忽略，但若一直泄漏，无论有多少内存，迟早都会被占用光，最终导致程序crash。（因此，开发中我们要尽量避免内存泄漏的出现

内存溢出(out of memory): 是指程序在申请内存时，没有足够的内存空间供其使用。
通俗理解就是内存不够用了，通常在运行大型应用或游戏时，应用或游戏所需要的内存远远超出了你主机内安装的内存所承受大小，就叫内存溢出。最终导致机器重启或者程序crash.

目前比较常用的内存泄漏的排查方法有两种，都在Xcode中可以直接使用：

* 静态分析方法（Analyze)
* 动态分析方法（Instrument工具库里的Leaks）。一般推荐使用第二种.

静态内存泄漏分析方法: 

1. 通过Xcode打开项目，然后点击Product->Analyze，开始进入静态内存泄漏分析
2. 等待分析结果
3. 根据分析的结果对可能造成内存泄漏的代码进行排查

动态内存泄漏分析方法:

动态分析法主要使用Instruments工具中的Leaks. 可以参开之前的[总结](http://hchong.net/2017/04/13/Xcode%E7%A5%9E%E5%99%A8-Instruments%E5%A4%A7%E6%B3%95/).

# 5 循环引用
对象 A 和对象 B，相互引用了对方作为自己的成员变量，只有当自己销毁时，才会将成员变量的引用计数减 1。因为对象 A 的销毁依赖于对象 B 销毁，而对象 B 的销毁与依赖于对象 A 的销毁，这样就造成了我们称之为循环引用（Reference Cycle）的问题. 不止两对象存在循环引用问题，多个对象依次持有对方，形式一个环状，也可以造成循环引用问题.

## 5.1 解决方案

* 我明确知道这里会存在循环引用，在合理的位置主动断开环中的一个引用，使得对象得以回收.(主动断开循环引用这种方式常见于各种与 block 相关的代码逻辑中, 主动释放对于 block 的持有，以便打破循环引用).
* 弱引用虽然持有对象，但是并不增加引用计数，这样就避免了循环引用的产生。在 iOS 开发中，弱引用通常在 delegate 模式中使用

注意: CAAnimation的delegate代理是强引用, 因为CAAnimation动画是异步的，如果动画的代理是弱应用不是强应用的话，会导致其随时都可能被释放掉.

## 5.2 弱引用的实现原理
弱引用的实现原理是这样，系统对于每一个有弱引用的对象，都维护一个表来记录它所有的弱引用的指针地址。这样，当一个对象的引用计数为 0 时，系统就通过这张表，找到所有的弱引用指针，继而把它们都置成 nil。

## 5.3 循环引用检测
Xcode 的 Instruments 工具集可以很方便的检测循环引用. Instruments -> leaks.

# 6 Other
这里是内存管理中其他一些常见的基本概念.

## 6.1 Thread Local Storage
Thread Local Storage（TLS）线程局部存储，目的很简单，将一块内存作为某个线程专有的存储，以key-value的形式进行读写.

在返回值身上调用objc_autoreleaseReturnValue方法时，runtime将这个返回值object储存在TLS中，然后直接返回这个object（不调用autorelease）；同时，在外部接收这个返回值的objc_retainAutoreleasedReturnValue里，发现TLS中正好存了这个对象，那么直接返回这个object（不调用retain）。

于是乎，调用方和被调方利用TLS做中转，很有默契的免去了对返回值的内存管理

## 6.2 Tagged Pointer
[这里](http://blog.devtang.com/2014/05/30/understand-tagged-pointer/)有一篇文章写的很详细. 

为了节省内存和提高执行效率，苹果提出了Tagged Pointer的概念。对于 64 位程序，引入 Tagged Pointer 后，相关逻辑能减少一半的内存占用，以及 3 倍的访问速度提升，100 倍的创建、销毁速度提升.

我们先看看原有的对象为什么会浪费内存。假设我们要存储一个 NSNumber 对象，其值是一个整数。正常情况下，如果这个整数只是一个 NSInteger 的普通变量，那么它所占用的内存是与 CPU 的位数有关，在 32 位 CPU 下占 4 个字节，在 64 位 CPU 下是占 8 个字节的。而指针类型的大小通常也是与 CPU 位数相关，一个指针所占用的内存在 32 位 CPU 下为 4 个字节，在 64 位 CPU 下也是 8 个字节。

所以一个普通的 iOS 程序，如果没有Tagged Pointer对象，从 32 位机器迁移到 64 位机器中后，虽然逻辑没有任何变化，但这种 NSNumber、NSDate 一类的对象所占用的内存会翻倍。如下图所示:
![内存图](http://blog.devtang.com/images/tagged_pointer_before.jpg)

由于 NSNumber、NSDate 一类的变量本身的值需要占用的内存大小常常不需要 8 个字节. 拿整数来说，4 个字节所能表示的有符号整数就可以达到 20 多亿（注：2^31=2147483648，另外 1 位作为符号位). 所以我们可以将一个对象的指针拆成两部分，一部分直接保存数据，另一部分作为特殊标记，表示这是一个特别的指针，不指向任何一个地址。所以，引入了Tagged Pointer对象之后64 位 CPU 下 NSNumber 的内存图变成了以下这样：
![Tagged Pointer 内存图](http://blog.devtang.com/images/tagged_pointer_after.jpg)

总结:
由于 NSNumber、NSDate 一类的变量本身的值需要占用的内存大小常常不需要 8 个字节. 当 8 字节可以承载用于表示的数值时，系统就会以Tagged Pointer的方式生成指针，如果 8 字节承载不了时，则又用以前的方式来生成普通的指针.

Tagged Pointer通过在其最后一个 bit 位设置一个特殊标记，用于将数据直接保存在指针本身中。因为Tagged Pointer并不是真正的对象，我们在使用时需要注意不要直接访问其 isa 变量.

## 6.3 Toll-Free Bridge
[这里](http://hchong.net/2018/05/07/OC%E4%B8%AD%E7%9A%84%E6%A1%A5%E6%8E%A5Toll-Free%20Bridging/)可以看到之前总结的关于OC中内存管理的桥接机制.

* __bridge: 通过 __bridge 桥接，id 和 void * 就能够相互转换. __bridge 为直接转换, 不会对引用计数做特殊处理.
* __bridge_retained & CFBridgingRetain: 表示将指针类型转变的同时, 将内存管理的责任由原来的Objective-C交给Core Foundation来处理.
* bridge_transfer & CFBridgingRelease: 与上面的__bridge_retained相反, 它表示将管理的责任由Core Foundation转交给Objective-C, 即将管理方式由MRC转变为ARC.
    
    ```
    id obj = (id)p;
    [obj retain];
    [(id)p release];
    ```

-----

参考资料:
1.[自动释放池的前世今生 ---- 深入解析 Autoreleasepool](https://www.jianshu.com/p/32265cbb2a26)
2.[Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)
3.[闲聊内存管理](http://sindrilin.com/runtime/2016/12/23/%E9%97%B2%E8%81%8A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)
4.[理解 iOS 的内存管理](https://blog.devtang.com/2016/07/30/ios-memory-management/)
5.[iOS内存管理详解](https://juejin.im/post/5abe543bf265da23784064dd)
6.[iOS进阶——iOS（Objective-C）内存管理·二](http://zhoulingyu.com/2017/02/15/Advanced-iOS-Study-objc-Memory-2/)
7.[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
8.[深入理解Tagged Pointer](http://blog.devtang.com/2014/05/30/understand-tagged-pointer/)
9.[Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)
10.[深入浅出ARC(上,中,下)](http://blog.tracyone.com/tags/ARC/)
11.[iOS渐入佳境之内存管理机制（三）：Toll-Free Bridging](https://solacode.github.io/2015/10/20/iOS%E6%B8%90%E5%85%A5%E4%BD%B3%E5%A2%83%E4%B9%8B%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9AToll-Free-Bridging/)


