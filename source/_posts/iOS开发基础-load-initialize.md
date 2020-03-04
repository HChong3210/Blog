---
title: iOS开发基础-load & initialize
date: 2018-11-8 18:43:43
tags:
    - 基础知识
categories:
    - iOS开发-基础
---

Objective-C 中绝大部分的类都继承自 NSObject 类, 而在 NSObject 类中有两个非常特殊的类方法 +load 和 +initialize, 用于类的初始化. 下面我们就来了解一下这两个方法.

# 1 load
[NSObject Class Reference](https://developer.apple.com/documentation/objectivec/nsobject/1418815-load?language=objc)中是这样描述`+ (void)load`的:

> Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading.

> The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.

> The order of initialization is as follows:

>* All initializers in any framework you link to.
>* All +load methods in your image.
>* All C++ static initializers and C/C++ attribute(constructor) functions in your image.
>* All initializers in frameworks that link to you.

> In addition:

>* A class’s +load method is called after all of its superclasses’ +load methods.
>* A category +load method is called after the class’s own +load method.

> In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet

从这段描述中, 我们可以看出如下几点:

* load方法在这个文件被程序装载时调用, 只要是在Compile Sources中出现的文件总是会被装载，这与这个类是否被用到无关.
* 在一个程序（main 函数）运行之前，所用到的库被加载到 runtime 之后, 被添加到的 runtime 系统的各种类和 category 的+load方法就被调用. 
* 如果父类和子类的+load方法都被调用，父类的调用一定在子类之前，这是系统自动完成的，子类+load中没必要显式调用[super load];
* 如果子类没有实现 +load 方法, 那么当它被加载时 runtime 是不会去调用父类的 +load 方法的.
* 若某个类由一个主类和多个 category 组成, 则允许主类和 category 中各自有自己的+load方法，只是 category 中的+load的执行在主类的+load之后.
* 当有多个类别(Category)都实现了load方法, 这几个load方法都会执行, 执行顺序与Compile Sources中出现的顺序(编译顺序)一致.
* 有多个不同的类的时候, 每个类load 执行顺序与其在Compile Sources出现的顺序一致.

下面我们来看一下, load方法常见的使用方式:

## 1.1 load在method swizzling中的实践
[Method Swizzling 和 AOP 实践](https://tech.glowing.com/cn/method-swizzling-aop/)这篇文章关于Method Swizzling讲解的比较详细, 可以参考这里.

## 1.2 Notification Once
可以参考[这里](http://blog.sunnyxx.com/2015/03/09/notification-once/)
写在`- application:didFinishLaunchingWithOptions:`中的代码都只是为了在程序启动时获得一次调用机会, 多为某些模块的初始化工作, 可以通过load来巧妙的解决:

```
+ (void)load {
    __block id observer =
    [[NSNotificationCenter defaultCenter]
     addObserverForName:UIApplicationDidFinishLaunchingNotification
     object:nil
     queue:nil
     usingBlock:^(NSNotification *note) {
         [self setup]; // Do whatever you want
         [[NSNotificationCenter defaultCenter] removeObserver:observer];
     }];
}
```

* + load方法在足够早的时间点被调用
* block 版本的通知注册会产生一个__NSObserver *对象用来给外部 remove 观察者
* block 对 observer 对象的捕获早于函数的返回，所以若不加__block，会捕获到 nil
* 在 block 执行结束时移除 observer，无需其他清理工作
* 这样，在模块内部就完成了在程序启动点代码的挂载.

注意: 通知是在`- application:didFinishLaunchingWithOptions:`调用完成后才发送的.

# 1.3 load的调用时机
在APP的启动过程中, 在系统内核做好程序准备工作之后, 会交由 dyld 来负责余下的工作. 大致的流程如下:

* 加载dyld到App进程
* 加载动态库（包括所依赖的所有动态库）
* Rebase
* Bind
* 初始化Objective C Runtime
* 其它的初始化代码

当 Objective-C 运行时初始化的时候,在每次有新的镜像加入运行时的时候, 会通过 dyld_register_image_state_change_handler进行回调. 执行 load_images 将所有包含 load 方法的文件加入列表 loadable_classes , 然后从这个列表中找到对应的 load 方法的实现, 调用 load 方法.

ObjC 对于加载的管理, 主要使用了两个列表, 分别是 loadable_classes 和 loadable_categories. 方法的调用过程也分为两个部分, 准备 load 方法和调用 load 方法.

[这篇文章](https://draveness.me/load)关于load讲解的十分详细.

# 2 initialize
[NSObject Class Reference](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize?preferredLanguage=occ)中是这样描述`+initialize`方法:

> The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. Superclasses receive this message before their subclasses.

> The runtime sends the initialize message to classes in a thread-safe manner. That is, initialize is run by the first thread to send a message to a class, and any other thread that tries to send a message to that class will block until initialize completes.

> The superclass implementation may be called multiple times if subclasses do not implement initialize—the runtime will call the inherited implementation—or if subclasses explicitly call [super initialize]. If you want to protect yourself from being run multiple times, you can structure your implementation along these lines:

>```
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```
> Because initialize is called in a blocking manner, it’s important to limit method implementations to the minimum amount of work necessary possible. Specifically, any code that takes locks that might be required by other classes in their initialize methods is liable to lead to deadlocks. Therefore, you should not rely on initialize for complex initialization, and should instead limit it to straightforward, class local initialization.
> 
> *Special Considerations*
> initialize is invoked only once per class. If you want to perform independent initialization for the class and for categories of the class, you should implement load methods.

关于上面这段话, 我们可以看出:

* +initialize方法是在 runtime 时被调用的
* 对于某个类型，其+initialize方法都会在该类型接受其他任何消息（类型方法）之前被调用，包括+alloc
* initialize方法实际上是一种惰性调用, 也就是说如果一个类一直没被用到, 那它的initialize方法也不会被调用.
* 如果父类和子类的+initialize方法都被调用，父类的调用一定在子类之前，这是系统自动完成的，子类+initialize中没必要显式调用[super initialize]
* runtime 系统处理+initialize消息的方式是线程安全的，所以没必要在+initialize中为了保证线程安全而使用 lock、mutex 之类的线程安全工具
* 某个类的+initialize的方法不一定只被调用一次，至少有两种情况会被调用多次：
    * 子类显式调用[super initialize]
    * 子类没有实现+initialize方法
* 不要在+initialize中处理复杂的逻辑, 可以做一些简单的初始化工作. 

注意: 因为在创建子类对象时，首先要创建父类对象，所以会调用一次父类的initialize方法，然后创建子类时，尽管自己没有实现initialize方法，但还是会调用到父类的方法.

虽然initialize方法对一个类而言只会调用一次，但这里由于出现了两个类，所以调用两次符合规则，但不符合我们的需求。正确使用initialize方法的姿势如下

```
+ (void)initialize {
    if (self == [Parent class]) {
        NSLog(@"Initialize Parent, caller Class %@", [self class]);
    }
}
```

## 2.1 initialize的使用场景
initialize方法主要用来对一些不方便在编译期初始化的对象进行赋值。比如NSMutableArray这种类型的实例化依赖于runtime的消息发送，所以显然无法在编译器初始化.

```
// In Parent.m
static int someNumber = 0;     // int类型可以在编译期赋值  
static NSMutableArray *someObjects;

+ (void)initialize {
    if (self == [Parent class]) {
        // 不方便编译期复制的对象在这里赋值
        someObjects = [[NSMutableArray alloc] init];
    }
}
```

# 3 load vs initialize

* load和initialize方法都会在实例化对象之前调用，以main函数为分水岭，前者在main函数之前调用，后者在之后调用。这两个方法会被自动调用，不能手动调用它们.
* load和initialize方法都不用显示的调用父类的方法而是自动调用，即使子类没有initialize方法也会调用父类的方法，而load方法则不会调用父类.
* load方法通常用来进行Method Swizzle，initialize方法一般用于初始化全局变量或静态变量.
* load和initialize方法内部使用了锁，因此它们是线程安全的。实现时要尽可能保持简单，避免阻塞线程，不要再使用锁.


|  | +load | +initialize |
| --- | --- | --- |
|调用时机|被添加到 runtime 时|收到第一条消息前，可能永远不调用|
|调用顺序|父类->子类->分类|父类->子类|
|调用次数|1次	|多次|
|是否需要显式调用父类实现|	否|	否|
|是否沿用父类的实现	|否	|是|
|分类中的实现|	类和分类都执行|	覆盖类中的方法，只执行分类的实现|


------
参考资料:
1.[细说OC中的load和initialize方法](https://bestswifter.com/load-and-initialize/#initialize)
2.[Notification Once](http://blog.sunnyxx.com/2015/03/09/notification-once/)
3.[Objective-C 中的+initialize 和+load](https://zhangbuhuai.com/post/initialize-and-load-in-objective-c.html)
4.[Notification once](http://blog.sunnyxx.com/2015/03/09/notification-once/)
5.[iOS初探+load和+initialize](http://liumh.com/2015/07/29/ios-load-and-initialize/)
6.[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)
7.[你真的了解 load 方法么？](https://draveness.me/load)

