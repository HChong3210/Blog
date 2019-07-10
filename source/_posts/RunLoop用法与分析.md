---
title: RunLoop用法与分析
date: 2017-12-11 23:26:30
tags:
    - 基础知识
categories: 
    - 基础知识
---

RunLoop是iOS开发中一个常用的概念, 用来保证线程能够随时处理事件, 但是并不退出. 代码逻辑通常是这样的: 

```
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
在iOS开发中RunLoop的主要作用是:
 
1. 保持程序的持续运行(如: 程序一启动就会开启一个主线程(线程中的 runloop 是自动创建并运行), runloop 保证主线程不会被销毁, 也就保证了程序的持续运行). 
2. 处理App中的各种事件(如: touches 触摸事件, NSTimer 定时器事件, Selector事件(选择器 performSelector)).
3. 节省CPU资源, 提高程序性能(有事情就做事情, 没事情就休息(其资源释放)). 
4. 负责渲染屏幕上的所有UI.

[这篇文章](https://blog.ibireme.com/2015/05/18/runloop/)讲的实在太详细了, 这里只是做一个总结和拾遗. 官方文档可以看[这里](https://developer.apple.com/documentation/corefoundation/cfrunloop?language=objc).

## RunLoop的概念
RunLoop实际上是一个对象, 这个对象管理了其需要处理的时间和消息, 并提供了一个入口函数来执行上面的逻辑, 线程执行了这个函数后, 就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中, 直到这个循环结束(比如传入 quit 的消息), 函数返回.

iOS提供了两个这样的对象: NSRunLoop 和 CFRunLoopRef. CFRunLoopRef 是在 CoreFoundation 框架内的, 它提供了纯 C 函数的 API, 所有这些 API 都是线程安全的. 
NSRunLoop 是基于 CFRunLoopRef 的封装, 提供了面向对象的 API, 但是这些 API 不是线程安全的.
## RunLoop与线程的关系
CFRunLoop 是基于 pthread 来管理的. 苹果不允许直接创建 RunLoop, 它只提供了两个自动获取的函数: CFRunLoopGetMain() 和 CFRunLoopGetCurrent(). 

线程与RunLoop之间是一一对应的关系, 他们之间的关系(`CFDictionarySetValue(loopsDic, thread, loop);`)保存在一个全局的字典中(key 是 pthread_t, value 是 CFRunLoopRef), 线程刚创建时是没有RunLoop的, 如果不主动获取就一直不会有, RunLoop的创建是发生在第一次获取时, RunLoop的销毁发生自线程结束时. 只能在一个线程的内部获取其RunLoop. 
## RunLoop对外的接口
在 CoreFoundation 里面关于 RunLoop 有5个类:

* CFRunLoopRef
* CFRunLoopModeRef
* CFRunLoopSourceRef
* CFRunLoopTimerRef
* CFRunLoopObserverRef

其中 CFRunLoopModeRef 类并没有对外暴露, 只是通过 CFRunLoopRef 的接口进行了封装, 他们之间的关系如图所示.
![RunLoop类之间的关系](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)

Source/Timer/Observer 被统称为 mode item, 一个 item 可以被同时加入多个 mode. 但一个 item 被重复加入同一个 mode 时是不会有效果的. 如果一个 mode 中一个 item 都没有, 则 RunLoop 会直接退出, 不进入循环. 
### CFRunLoopSourceRef
CFRunLoopSourceRef是事件源(输入源), 例如外部的触摸, 点击事件和系统内部进程间的通信等. 按照官方文档Source主要分为: 

* Input Sources: Input Sources以异步的方式将事件传递给线程, 通过分为两种:
    * Port-based input sources: Cocoa和Core Foundation提供了内置的支持, 可以使用端口相关的对象和函数创建基于端口的输入源. 我们不必直接创建输入源, 只需创建一个端口对象, 并使用NSPort的方法将该端口添加到运行循环中. 它是来监视应用程序的Mach端口, 消息由内核发出.
    * Custom Input Sources: 我们使用CFRunLoopSourceRef来创建自定义输入源, 监视事件的自定义源, 消息由其他线程手动发出.
* Cocoa Perform Selector Sources: 与基于端口的源不同, 我们可以在任何线程上perform a selector, 并且执行选择器源在执行选择器之后从运行循环中移除自己.
* Timer Sources: 在将来的预定时间内, 计时器源会同步地将事件发送到线程.
更多详细信息可以看[Threading Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1).

按照函数调用栈, Source主要分为两类:
 
* Source0: 非基于Port的. 只包含了一个回调(函数指针), 它并不能主动触发事件. 使用时, 需要先调用 CFRunLoopSourceSignal(source), 将这个 Source 标记为待处理, 然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop, 让其处理这个事件. 
* Source1: 基于Port的, 通过内核和其他线程通信, 接收, 分发系统事件. 这种 Source 能主动唤醒 RunLoop 的线程. 创建常驻线程就是在线程中添加一个NSport来实现的.
### CFRunLoopObserverRef
每个 Observer 都包含了一个回调(函数指针)当 RunLoop 的状态发生变化时, 观察者就能通过回调接受到这个变化. 可以观测的时间点有以下几个: 

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop     （1）
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer   （2）
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source  （4）
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠      （32）
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒     (64)
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop      (128)
    kCFRunLoopAllActivities = 0x0FFFFFFU, // 包含上面所有状态

};
```
### CFRunLoopTimerRef
是基于时间的触发器, 它和 NSTimer 是toll-free bridged 的, 可以混用, 我们基本上可以认为他就是NSTimer,  其包含一个时间长度和一个回调(函数指针), 当其加入到 RunLoop 时, RunLoop会注册对应的时间点, 当时间点到时, RunLoop会被唤醒以执行那个回调. 
它受RunLoop的Mode影响, GCD的定时器不受RunLoop的Mode影响.
注意, 这里这个NSTimer实际是有误差的
### CFRunLoopModeRef
一个runloop可以包含多个model, 每个model都是独立的, 而且runloop只能选择一个model运行, 也就是currentModel. 如果需要切换 Mode, 只能退出 Loop, 再重新指定一个 Mode 进入. 这样做主要是为了分隔开不同组的 Source/Timer/Observer, 让其互不影响.

系统默认注册了5个Mode:
NSDefaultRunLoopMode: App的默认Mode, 通常主线程是在这个Mode下运行, App 平时就是处在这个状态.
UITrackingRunLoopMode: 界面跟踪 Mode, 用于 ScrollView 追踪触摸滑动, 保证界面滑动时不受其他 Mode 影响.
UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode, 启动完成后就不再使用.
GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode, 通常用不到.
NSRunLoopCommonModes: 这是一个占位用的Mode, 不是一种真正的Mode.

这里有个概念叫 "CommonModes": 一个 Mode 可以将自己标记为"Common"属性(通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中). 每当 RunLoop 的内容发生变化时, RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 "Common"标记的所有Mode里. 
应用场景举例: 主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为"Common"属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作. 有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到commonMode 中。那么所有被标记为commonMode的mode（defaultMode和TrackingMode）都会执行该timer。这样你在滑动界面的时候也能够调用timer，下面会有实例讲解

CFRunLoop对外暴露的管理 Mode 接口只有下面2个:

```
CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);
CFRunLoopRunInMode(CFStringRef modeName, ...);
```
Mode暴露的管理Mode item的接口有下面几个: 

```
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```
你只能通过 mode name 来操作内部的 mode, 当你传入一个新的 mode name 但 RunLoop 内部没有对应 mode 时, RunLoop会自动帮你创建对应的 CFRunLoopModeRef. 对于一个 RunLoop 来说, 其内部的 mode 只能增加不能删除.

## RunLoop的内部逻辑
![官方的RunLoop逻辑](http://upload-images.jianshu.io/upload_images/277755-1184c261c96d3116.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

盗图一张, 大神总结的.
![RunLoop内部逻辑](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)
运行RunLoop具体的流程如下: 

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
9. 处理唤醒时收到的事件
    * 如果是定时器启动, 处理定时器并重启RunLoop, 进入步骤2
    * 如果是输入源启动, 传递相应的消息
    * 如果是RunLoop被显式唤醒(Source1), 重启RunLoop, 进入步骤2
10. 通知观察者RunLoop即将结束.

实际上 RunLoop 内部是一个 do-while 循环函数. 当你调用 CFRunLoopRun() 时, 线程就会一直停留在这个循环里; 直到超时或被手动停止, 该函数才会返回. 

## 系统中RunLoop的常见实现
下面列举几个Apple中常见的RunLoop的使用场景.

### AutoreleasePool
APP启动后, 系统在主线程RunLoop中注册了两个Observer, 其回调都是 _wrapRunLoopWithAutoreleasePoolHandler().

第一个Observer监视的事件是kCFRunLoopEntry(进入RunLoop), 会调用_objc_autoreleasePoolPush() 创建自动释放池, 其 order 是-2147483647, 优先级最高, 保证创建释放池发生在其他所有回调之前. 
第二个Observer监视的事件是kCFRunLoopBeforeWaiting(RunLoop休眠)和kCFRunLoopExit(RunLoop退出). 会在休眠是调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池; 会在退出时调用 _objc_autoreleasePoolPop() 来释放自动释放池. 这个 Observer 的 order 是 2147483647, 优先级最低, 保证其释放池子发生在其他所有回调之后.

在主线程执行的代码, 通常是写在诸如事件回调, Timer回调内的. 这些回调会被 RunLoop 创建好的 AutoreleasePool 包围, 所以不会出现内存泄漏, 开发者也不必显示创建 Pool 了.
### 事件响应
苹果注册了一个 Source1 (基于 mach port) 用来接收系统事件, 其回调函数为 __IOHIDEventSystemClientQueueCallback(). 

当一个硬件事件(触摸/锁屏/摇晃等)发生后,  IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收. SpringBoard 只接收按键(锁屏/静音等), 触摸, 加速, 接近传感器等几种 Event, 随后用 mach port 转发给需要的App进程. 随后苹果注册的那个 Source1 就会触发回调, 并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发. 
_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发, 其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等. 通常事件比如 UIButton 点击, touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的.
### 手势识别
当_UIApplicationHandleEventQueue() 识别了一个手势时, 其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断. 随后系统将对应的 UIGestureRecognizer 标记为待处理. 
苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件, 这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver(), 其内部会获取所有刚被标记为待处理的 GestureRecognizer, 并执行GestureRecognizer的回调.
当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时, 这个回调都会进行相应处理.
### 界面更新
当在操作 UI, 比如改变了 Frame, 更新了 UIView/CALayer 的层次时, 或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后, 这个 UIView/CALayer 就被标记为待处理, 并被提交到一个全局的容器去.
苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件, 回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv(). 这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整, 并更新 UI 界面.
### 定时器
NSTimer 其实就是 CFRunLoopTimerRef, 他们之间是 toll-free bridged 的. 一个 NSTimer 注册到 RunLoop 后, RunLoop 会为其重复的时间点注册好事件. 例如 10:00, 10:10, 10:20 这几个时间点. RunLoop为了节省资源, 并不会在非常准确的时间点回调这个Timer. Timer 有个属性叫做 Tolerance (宽容度), 标示了当时间点到后, 容许有多少最大误差. 
如果某个时间点被错过了, 例如执行了一个很长的任务, 则那个时间点的回调也会跳过去, 不会延后执行.
### performSelecter方法
调用 NSObject 的 performSelecter:afterDelay: 后, 实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中. 所以如果当前线程没有 RunLoop, 则这个方法会失效.
当调用 performSelector:onThread: 时, 实际上其会创建一个 Timer 加到对应的线程去, 同样的, 如果对应线程没有 RunLoop 该方法也会失效.
## 开发中RunLoop的常见使用
### 滚动scrollView导致定时器失效
如果在界面上有一个UIscrollview控件(tableview, collectionview等), 如果此时还有一个定时器在执行一个事件, 你会发现当你滚动scrollview的时候, 定时器会失效. 

原因是当滚动UIscrollview的时候, runloop会进入UITrackingRunLoopMode 模式, 而定时器运行在defaultMode下面, 系统一次只能处理一种模式的runloop, 所以导致defaultMode下的定时器失效. 解决方案有两种:
 
1. 把定时器的runloop的model改为NSRunLoopCommonModes 模式, 这个模式是一种占位mode, 并不是真正可以运行的mode, 它是用来标记一个mode的. 默认情况下default和tracking这两种mode 都会被标记上NSRunLoopCommonModes 标签 改变定时器的mode为commonmodel, 可以让定时器运行在defaultMode和trackingModel两种模式下, 不会出现滚动scrollview导致定时器失效的故障. 

    ```
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    ```
2. 使用GCD创建定时器
    
    ```
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
### 图片下载
由于图片渲染到屏幕需要消耗较多资源, 为了提高用户体验, 当用户滚动UIscrollview的时候, 只在后台下载图片, 但是不显示图片, 当用户停下来的时候才显示图片. 核心代码如下:

```
[self.imageView performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@"placeholder"] afterDelay:3.0 inModes:@[NSDefaultRunLoopMode]];
```
这是因为限定了方法setImage只能在NSDefaultRunLoopMode 模式下使用. 而滚动UIscrollview的时候, 程序运行在tracking模式下面, 所以方法setImage不会执行. 
### 常驻线程
需要创建一个在后台一直存在的程序, 来做一些需要频繁处理的任务. 比如检测网络状态等. 默认情况一个线程创建出来, 运行完要做的事情, 线程就会消亡. 而程序启动的时候, 就创建的主线程已经加入到runloop, 所以主线程不会消亡. AFN里面就有一条通过添加NSPort来实现常驻的线程, 常见的有两种方式: 

1. 添加NSPort

    ```
    - (void)viewDidLoad {
        [super viewDidLoad];
    
        self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [self.thread start];
    }
    
    - (void)run {
        NSLog(@"----------run----%@", [NSThread currentThread]);
        @autoreleasepool{
        /*如果不加这句，会发现runloop创建出来就挂了，因为runloop如果没有CFRunLoopSourceRef事件源输入或者定时器，就会立马消亡。
          下面的方法给runloop添加一个NSport，就是添加一个事件源，也可以添加一个定时器，或者observer，让runloop不会挂掉*/
        [[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
    
        // 方法1 ,2，3实现的效果相同，让runloop无限期运行下去
        [[NSRunLoop currentRunLoop] run];
       }
    
        // 方法2
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    
        // 方法3
        [[NSRunLoop currentRunLoop] runUntilDate:[NSDate distantFuture]];
    
        NSLog(@"---------");
    }
    
    - (void)test {
        NSLog(@"----------test----%@", [NSThread currentThread]);
    }
    
    - (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event {
        [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:NO];
    }
    ```
2. 添加NSTimer

    ```
    - (void)viewDidLoad {
        [super viewDidLoad];
    
        self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [self.thread start];
    }
    - (void)run {
        [NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(test) userInfo:nil repeats:YES];
    
        [[NSRunLoop currentRunLoop] run];
    }
    ```
如果没有实现添加NSPort或者NSTimer, 会发现执行完run方法, 线程就会消亡, 后续再执行touchbegan方法无效. 我们必须保证线程不消亡, 才可以在后台接受时间处理.

RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source, 所以在 `[runLoop run]` 之前先创建了一个新的 NSMachPort 添加进去了. 通常情况下, 调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内; 但此处添加 port 只是为了让 RunLoop 不至于退出, 并没有用于实际的发送消息. 

可以发现执行完了run方法, 这个时候再点击屏幕, 可以不断执行test方法, 因为线程self.thread一直常驻后台, 等待事件加入其中, 然后执行.
### 在所有UI相应操作之前处理任务
主要思路就是我们可以新建一个Observer, 来观察RUnLoop的状态, 因为我们的UI操作都会导致RunLoop状态的变动, 通过日志我们可以发现, 在执行按钮事件之前, 先执行Observer里面的方法, 这样就可以拦截事件, 让我们的代码在UI事件之前执行.

```
- (IBAction)btnClick:(id)sender {
    NSLog(@"btnClick----------");
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self observer];
}

- (void)observer {
    // 创建observer，参数kCFRunLoopAllActivities表示监听所有状态
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        NSLog(@"----监听到RunLoop状态发生改变---%zd", activity);
    });
    
    // 添加观察者：监听RunLoop的状态
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
    
    // 释放Observer
    CFRelease(observer);
}
```

```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),   //1
    kCFRunLoopBeforeTimers = (1UL << 1),    //2
    kCFRunLoopBeforeSources = (1UL << 2),   //4
    kCFRunLoopBeforeWaiting = (1UL << 5),   //32
    kCFRunLoopAfterWaiting = (1UL << 6),    //64
    kCFRunLoopExit = (1UL << 7),    //128
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```
### 其他用法
例如我们实现Cell高度的缓存计算, 我们可以在RunLoop空闲时来计算, 所以我们可以创建一个Observer, 来监听RunLoop的kCFRunLoopBeforeWaiting状态(这一次 RunLoop 迭代处理完成了所有事件, 马上要休眠时), 在其中来计算高度并且缓存, 

```
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
CFStringRef runLoopMode = kCFRunLoopDefaultMode;
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler
(kCFAllocatorDefault, kCFRunLoopBeforeWaiting, true, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity _) {
    // TODO here
});
CFRunLoopAddObserver(runLoop, observer, runLoopMode);
在其中的 TODO 位置，就可以开始任务的收集和分发了，当然，不能忘记适时的移除这个 observer
```

------

参考资料: 

1.[深入理解RunLoop](http://www.bijishequ.com/detail/355655?p=70-67)

2.[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/), 这个是经典之作

3.[Threading Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1), 官方


