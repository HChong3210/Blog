---
title: iOS开发总结系列-Runtime
date: 2018-03-07 20:05:46
tags:
    - 基础知识
    - Runtime
categories:
    - 基础知识
---

这篇主要是RunTime相关的知识, 具体关于RunTime可以看[RunTime用法与分析](http://hchong.net/2017/12/11/Runtime%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/)
## 什么时候会报unrecognized selector的异常

涉及到消息的转发阶段知识[Runtime用法与分析](http://hchong.net/2017/12/11/Runtime%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/)

当调用该对象上某个方法, 而该对象上没有实现这个方法的时候, 可以通过“消息转发”进行解决. objc在向一个对象发送消息时, runtime库会根据对象的isa指针找到该对象实际所属的类, 然后在该类中的方法列表以及其父类方法列表中寻找方法运行, 如果在最顶层的父类中依然找不到相应的方法时, 程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX. 但是在这之前, objc的运行时会给出三次拯救程序崩溃的机会.

1. Method resolution阶段. objc运行时会调用`+resolveInstanceMethod:`或者 `+resolveClassMethod:`, 让你有机会提供一个函数实现. 如果你添加了函数, 那运行时系统就会重新启动一次消息发送的过程, 否则, 运行时就会移到下一步, 消息转发(Message Forwarding).
2. Fast forwarding阶段. 如果目标对象实现了`-forwardingTargetForSelector:`方法, Runtime这时就会调用这个方法, 给你把这个消息转发给其他对象的机会. 只要这个方法返回的不是nil和self, 整个消息发送过程会被重启, 当然发送的对象会变成你返回的那个对象. 否则就会继续Normal fowarging. 这里叫做Fast只是为了区别下一步的转发机制, 因为这一步不会创建任何新的对象, 但下一步会转发会创建一个NSInvocation对象, 所以相对更快一点.
3. Normal forwarding阶段. 这一步是Runtime最后给你的挽救机会. 首先会发送`-methodSignatureForSelector:`消息获得函数的参数和返回值类型. 如果`-methodSignatureForSelector:`返回nil, Runtime就会发出`-doesNotRecognizeSelector:`消息, 程序这时候就挂掉了. 如果返回了一个函数签名, Runtime就会创建一个NSInvocation对象并发送`-forwardInvocation:`消息给目标对象. 

## 一个objc对象如何进行内存布局
所有父类的成员变量和自己的成员变量都会存放在该对象所对应的存储空间中.
每一个对象内部都有一个isa指针, 指向他的类对象, 类对象中存放着本对象的对象方法列表, 成员列表的变量, 属性列表.
类的内部也有一个isa指针, 指向原类, 原类内部存放得失类方法列表, 内部也有一个superclass指针, 指向他的父类对象.

## _objc_msgForward函数是做什么的, 直接调用它将会发生什么
_objc_msgForward是 IMP 类型, 用于消息转发: 当向一个对象发送一条消息, 但它并没有实现的时候, _objc_msgForward会尝试做消息转发.

首先我们从消息的查找看起:

```
id objc_msgSend(id self, SEL op, ...) {
    if (!self) return nil;
	IMP imp = class_getMethodImplementation(self->isa, SEL op);
	imp(self, op, ...); //调用这个函数，伪代码...
}
 
//查找IMP
IMP class_getMethodImplementation(Class cls, SEL sel) {
    if (!cls || !sel) return nil;
    IMP imp = lookUpImpOrNil(cls, sel);
    if (!imp) return _objc_msgForward; //_objc_msgForward 用于消息转发
    return imp;
}
 
IMP lookUpImpOrNil(Class cls, SEL sel) {
    if (!cls->initialize()) {
        _class_initialize(cls);
    }
 
    Class curClass = cls;
    IMP imp = nil;
    do { //先查缓存,缓存没有时重建,仍旧没有则向父类查询
        if (!curClass) break;
        if (!curClass->cache) fill_cache(cls, curClass);
        imp = cache_getImp(curClass, sel);
        if (imp) break;
    } while (curClass = curClass->superclass);
 
    return imp;
}
```
我们可以看出, _objc_msgForward 是一个函数指针(和 IMP 的类型一样), 是用于消息转发的: 当向一个对象发送一条消息, 但没有实现的时候, _objc_msgForward会尝试做消息转发.

我们向一个对象发送一条错误的消息, 然后在程序中加入断点或者暂停程序运行, 并在 gdb 中输入下面的命令: `call (void)instrumentObjcMessageSends(YES)`. 之后, 运行时发送的所有消息都会打印到/tmp/msgSend-xxxx文件里了. 终端中输入命令`open /private/tmp`前往文件夹, 找到最新生成的, 双击打开.

在里面我们可以看到整个程序运行时调用的方法, 排除NSObject做的事, 剩下的就是`_objc_msgForward`消息转发做的事(实际也是消息转发的整个完整流程, [Runtime用法与分析](http://hchong.net/2017/12/11/Runtime%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/)). 

1. 调用resolveInstanceMethod:方法 (或 resolveClassMethod:). 允许用户在此时为该 Class 动态添加实现. 如果有实现了, 则调用并返回YES, 那么重新开始objc_msgSend流程. 这一次对象会响应这个选择器, 一般是因为它已经调用过class_addMethod. 如果仍没实现, 继续下面的动作.
2. 调用forwardingTargetForSelector:方法, 尝试找到一个能响应该消息的对象. 如果获取到, 则直接把消息转发给它, 返回非 nil 对象. 否则返回 nil, 继续下面的动作. 注意, 这里不要返回 self, 否则会形成死循环.
3. 调用methodSignatureForSelector:方法, 尝试获得一个方法签名. 如果获取不到, 则直接调用doesNotRecognizeSelector抛出异常. 如果能获取, 则返回非nil: 创建一个 NSlnvocation 并传给forwardInvocation:.
4. 调用forwardInvocation:方法, 将第3步获取到的方法签名包装成 Invocation 传入, 如何处理就在这里面了, 并返回非nil.
5. 调用doesNotRecognizeSelector:, 默认的实现是抛出异常. 如果第3步没能获得一个方法签名, 执行该步骤.

上面前4个方法均是模板方法, 开发者可以override, 由 runtime 来调用. 最常见的实现消息转发: 就是重写方法3和4, 忽略掉一个消息或者代理给其他对象.

一旦调用_objc_msgForward, 将跳过查找 IMP 的过程, 直接触发"消息转发". 具体可以看上面消息查找的具体过程. 可能会导致抛出unrecoginised selector的异常而导致程序crash. 

## 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
不能向编译后的类中增加实例变量, 因为已经注册在runtime中, 类结构体中的objc_ivar_list(实例变量链表)和instance_size(实例变量的内存大小)已经确定, 同时runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用, 所以不能向存在的类中添加实例变量.

可以向运行时创建的类中添加实例变量, 直接调用class_addIvar函数, 但是得在调用 objc_allocateClassPair 之后, objc_registerClassPair 之前, 原因同上.

## +load vs +initialize
+load 方法是当类或分类被添加到 Objective-C runtime 时被调用的, 实现这个方法可以让我们在类加载的时候执行一些类相关的行为. 子类的 +load 方法会在它的所有父类的 +load 方法之后执行, 而分类的 +load 方法会在它的主类的 +load 方法之后执行. 但是不同的类之间的 +load 方法的调用顺序是不确定的.
对 +load 方法进行调用使用的是`(*load_method)(cls, SEL_load)`，而不是使用发送消息 objc_msgSend 的方式. 这样的调用方式就使得 +load 方法拥有了一个非常有趣的特性, 那就是子类、父类和分类中的 +load 方法的实现是被区别对待的. 也就是说如果子类没有实现 +load 方法, 那么当它被加载时 runtime 是不会去调用父类的 +load 方法的. 同理, 当一个类和它的分类都实现了 +load 方法时, 两个方法都会被调用.

+initialize 方法是在类或它的子类收到第一条消息之前被调用的, 这里所指的消息包括实例方法和类方法的调用. 也就是说 +initialize 方法是以懒加载的方式被调用的, 如果程序一直没有给某个类或它的子类发送消息, 那么这个类的 +initialize 方法是永远不会被调用的. 这么做节省了系统资源, 避免浪费.
runtime 使用了发送消息 objc_msgSend 的方式对 +initialize 方法进行调用. 也就是说 +initialize 方法的调用与普通方法的调用是一样的, 走的都是发送消息的流程. 如果子类没有实现 +initialize 方法, 那么继承自父类的实现会被调用; 如果一个类的分类实现了 +initialize 方法, 那么就会对这个类中的实现造成覆盖.

|  | +load | +initialize |
| --- | --- | --- |
| 本质 | (*load_method)(cls, SEL_load) | (*load_method)(cls, SEL_load) |
| 调用时机 | 被添加到runtime时 | 直到给他的某个分类或它的子类发送消息 |
| 调用顺序 | 父类->子类->分类 | 父类->子类 |
| 调用次数 | 1次 | 多次 |
| 是否需要显式调用父类实现 | 不会 | 不会 |
| 是否沿用父类方法 | 否 | 是 |
| 分类中的实现 | 类和分类都执行 | 分类会覆盖类中的方法, 只执行分类的实现 |

## Method swizzling
该特性的实质是, 为类添加一个新的方法, 然后将目标方法和新的方法的IMP进行互换, 结果就是修改selector和IMP的对应关系.

我们可以利用 method_exchangeImplementations 来交换2个方法中的IMP.
我们可以利用 class_replaceMethod 来修改类.
我们可以利用 method_setImplementation 来直接设置某个方法的IMP.
...
归根到底都是更换selector的IMP(函数指针, 指向方法的实现).

## 如何使用runtime hook类方法和实例方法
使用Method Swizzling 方法. swizzling大多时候是在category中的+load方法中使用, 也可以创建hook的管理类, 放在里面使用.

使用`class_getInstanceMethod`来获取实例方法, 使用`class_getClassMethod`来获取类方法. 

* 我们可以利用 `class_addMethod`来检测方法是否存在.
* 我们可以利用 `method_exchangeImplementations` 来交换2个方法中的IMP.
* 我们可以利用 `class_replaceMethod` 来修改类.
* 我们可以利用 `method_setImplementation` 来直接设置某个方法的IMP.

下面是AFN中的写法:

```
static inline void af_swizzleSelector(Class class, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    if (class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod))) {
        class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

## 代理模式
在iOS中代理的本质就是代理对象内存的传递和操作, 我们在委托方设置代理属性后, 实际上只是用一个id类型的指针将代理对象进行了一个弱引用. 委托方让代理方执行协议, 实际上是在委托方中向这个id类型的指针指向的对象(代理方对象)发送消息.

委托方的代理属性本质上就是代理对象自身, 设置委托代理就是代理属性指针指向代理对象, 相当于代理对象只是在委托方中调用自己的方法. 如果方法没有实现就会报错(代理方没有实现委托方中协议中的方法).

协议只是一种语法, 是声明委托方中的代理属性可以调用协议中声明的方法. 如果代理方遵守协议那么协议中方法的实现就会加入到代理方类的objc_protocol_list中去. 委托方的self.delegate实际指向代理方的地址, 消息转发到代理方那里, 调用代理方objc_protocol_list中的实现.

## 为什么category不能添加属性可以添加方法
因为category在runtime中是用一个结构体表示的, 结构体在C中是初始化完成后是不能改变的, 但是方法是一个链表, 他是可以往里面添加新的方法的.

```
struct _category_t {
    const char *name;
    struct _class_t *cls;
    const struct _method_list_t *instance_methoods;
    const struct _method_list_t *class_methods;
    const struct _protocol_list_t *protocols;
    const struct _prop_list_t *properties;
}
```

这里面虽然可以添加property, 但是这些property并不会自动生成Ivar, 也就是不会有 @synthesize 的作用. dyld加载的期间, 这些categories会被加载并patch到相应的类中, 这个过程是动态的, Ivar不能动态添加, 因为表示ObjC类的结构体运行时并不能改变. 

