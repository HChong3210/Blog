---
title: iOS开发总结系列-RunLoop
date: 2018-03-11 22:30:35
tags:
    - 基础知识
categories: 
    - 基础知识
---

这篇主要是RunLoop相关的知识, 具体关于RunLoop可以看[RunLoop用法与分析](http://hchong.net/2017/12/11/RunLoop%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/)
## RunLoop和线程是什么关系
普通线程执行的任务是一条直线, 任务执行完就释放掉. RunLoop可以理解为特殊的线程, 这个现成不断的循环. 每个线程, 包括程序的主线程（main thread）都有与之相应的run loop对象.

他们之间的关系(CFDictionarySetValue(loopsDic, thread, loop);), 线程和 RunLoop 之间是一一对应的, 他们之间的关系保存在一个全局的字典中(key 是 pthread_t, value 是 CFRunLoopRef), 线程刚创建时是没有RunLoop的, 如果不主动获取就一直不会有, RunLoop的创建是发生在第一次获取时, RunLoop的销毁发生自线程结束时. 只能在一个线程的内部获取其RunLoop.

## runloop的mode作用是什么
主要是用来指定事件在运行循环中的优先级, 系统默认注册了5个Mode: 

* NSDefaultRunLoopMode:(kCFRunLoopDefaultMode, 公开) App的默认Mode, 通常主线程是在这个Mode下运行, App 平时就是处在这个状态.
* UITrackingRunLoopMode: 界面跟踪 Mode, 用于 ScrollView 追踪触摸滑动, 保证界面滑动时不受其他 Mode 影响.
* UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode, 启动完成后就不再使用.
* GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode, 通常用不到.
* NSRunLoopCommonModes:(kCFRunLoopCommonModes, 公开) 这是一个占位用的Mode, 不是一种真正的Mode

一个runloop可以包含多个model, 每个model都是独立的, 而且runloop只能选择一个model运行, 也就是currentModel. 如果需要切换 Mode, 只能退出 Loop, 再重新指定一个 Mode 进入. 这样做主要是为了分隔开不同组的 Source/Timer/Observer, 让其互不影响. 

这里有个概念叫 “CommonModes”: 一个 Mode 可以将自己标记为”Common”属性(通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中). 每当 RunLoop 的内容发生变化时, RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 "Common"标记的所有Mode里.

## 以+ scheduledTimerWithTimeInterval的方式触发的timer, 在滑动页面上的列表时, timer会暂定回调，为什么？如何解决
RunLoop只能运行在一种mode下, 如果要换mode, 当前的loop也需要停下重启成新的. 利用这个机制, ScrollView滚动过程中NSDefaultRunLoopMode(kCFRunLoopDefaultMode)的mode会切换到UITrackingRunLoopMode来保证ScrollView的流畅滑动: 只能在NSDefaultRunLoopMode模式下处理的事件会影响ScrollView的滑动. 如果我们把一个NSTimer对象以NSDefaultRunLoopMode(kCFRunLoopDefaultMode)添加到主运行循环中的时候, ScrollView滚动过程中会因为mode的切换, 而导致NSTimer将不再被调度.
正常情况下, 我们可能这样使用NSTimer:

```
    [NSTimer scheduledTimerWithTimeInterval:1.0
                                     target:self
                                   selector:@selector(timerTick)
                                   userInfo:nil
                                    repeats:YES];
```

这个问题有两个解决方案:

1. 我们使用创建一个NSTimer对象, 我们把它添加到当前的RunLoop中去, 指定他的Mode为NSRunLoopCommonModes, 或者分别加入到NSDefaultRunLoopMode和UITrackingRunLoopMode中.

    ```
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0
         target:self
         selector:@selector(timerTick:)
         userInfo:nil
         repeats:YES];
    //分别加入两种Mode中
    [[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    [[NSRunLoop mainRunLoop] addTimer:timer forMode: UITrackingRunLoopMode];
    //加入CommonMode中
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    ```
2. 使用GCD定时器

    ```
    @property (nonatomic, strong) dispatch_source_t timer;
    
    // 获得队列
    dispatch_queue_t queue = dispatch_get_main_queue();
    // 创建一个定时器(dispatch_source_t本质还是个OC对象)
    self.timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    // 设置定时器的各种属性（几时开始任务，每隔多长时间执行一次）
    // GCD的时间参数，一般是纳秒（1秒 == 10的9次方纳秒）
    // 比当前时间晚1秒开始执行
    dispatch_time_t start = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC));
    //每隔一秒执行一次
    uint64_t interval = (uint64_t)(1.0 * NSEC_PER_SEC);
    dispatch_source_set_timer(self.timer, start, interval, 0);
    // 设置回调
    dispatch_source_set_event_handler(self.timer, ^{
        NSLog(@"------------%@", [NSThread currentThread]);
    });
    // 启动定时器
    dispatch_resume(self.timer);
    ```
## 猜想runloop内部是如何实现的
![RunLoop内部逻辑](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)

1. 通知观察者, RunLoop已经启动.
2. 通知观察者, 将要处理定时器.
3. 通知观察者, 将要处理Source0(即将启动的非基于端口的源).
4. 启动任何准备好的Source0(非基于端口的源).
5. 如果有任何Source1(基于端口的源)准备好并且处于等待状态, 立即启动, 并进入步骤9.
6. 通知所有观察者, 线程即将进入休眠.
7. 线程处于休眠状态, 直到遇到下列事件中的任意一个:
    * 某一事件到达Source0(非基于端口的源)
    * NSTimer定时器启动
    * RunLoop设置的时间已经超时
    * RunLoop被外部手动显示唤醒.
8. 通知观察者, 线程刚被唤醒
9. 处理唤醒时收到的事件:
    * 如果是定时器启动, 处理定时器并重启RunLoop, 进入步骤2
    * 如果是输入源启动, 传递相应的消息
    * 如果是RunLoop被显式唤醒(Source1), 重启RunLoop, 进入步骤2
10. 通知观察者RunLoop即将结束.
详细内容参考[RunLoop用法与分析](http://hchong.net/2017/12/11/RunLoop%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/).
## autoreleasepool如何实现, 一个autoreleasepool对象何时释放
可以看[这里](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/), [这里](https://draveness.me/autoreleasepool), [这里](http://hchong.net/2018/03/23/Autoreleasepool%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)

autoreleasepool实际上被两个函数`objc_autoreleasePoolPush()`和`objc_autoreleasePoolPop()`包括, 这两个函数的实质又是AutoreleasePoolPage. 
AutoreleasePoolPage是一个双向链表, 存储了对象的内存地址. AutoreleasePoolPage里面有个next指针, 当向一个对象发送- autorelease消息, 就是将这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向的位置. 
每次调用`objc_autoreleasePoolPush()`, runtime就向链表中插入一个哨兵对象(nil), `objc_autoreleasePoolPush()`的返回值正是这个哨兵对象的地址. `objc_autoreleasePoolPop()`的入参也是这个哨兵对象的地址, 根据传入的哨兵对象地址找到哨兵对象所处的page, 在当前page中(从最新加入的对象一直向前清理, 可以向前跨越若干个page, 直到哨兵所在的page), 将晚于哨兵对象插入的所有autorelease对象都发送一次`- release`消息, 并向回移动next指针到正确位置.

在没有手加Autorelease Pool的情况下, Autorelease对象是在当前的runloop迭代结束时释放的, 而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop.



