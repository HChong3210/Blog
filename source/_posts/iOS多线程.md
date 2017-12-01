---
title: iOS多线程
date: 2017-11-21 15:34:43
tags:
	- 基础知识
categories:
	- 基础知识
---

# iOS开发多线程

首先我们来搞清楚几个概念和他们之间的联系和区别: 

## 多线程开发常用概念

1.  进程和线程

    进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动, 进程是系统进行资源分配和调度的一个独立单位. 
    ​	
    线程是进程的一个实体, 是CPU调度和分派(资源分配)的基本单位, 它是比进程更小的能独立运行的基本单位. 线程自己基本上不拥有系统资源, 只拥有一点在运行中必不可少的资源(如程序计数器, 一组寄存器和栈), 但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源. 

    一个线程可以创建和撤销另一个线程; 同一个进程中的多个线程之间可以并发执行. 

    进程和线程的主要差别在于它们是不同的操作系统资源管理方式. 进程有独立的地址空间, 一个进程崩溃后, 在保护模式下不会对其它进程产生影响. 而线程只是一个进程中的不同执行路径. 线程有自己的堆栈和局部变量, 但线程之间没有单独的地址空间, 一个线程死掉就等于整个进程死掉, 所以多进程的程序要比多线程的程序健壮, 但在进程切换时, 耗费资源较大, 效率要差一些. 但对于一些要求同时进行并且又要共享某些变量的并发操作, 只能用线程, 不能用进程. 

    1. 简而言之, 一个程序至少有一个进程, 一个进程至少有一个线程.
       2. 线程的划分尺度小于进程, 使得多线程程序的并发性高. 
       3. 另外, 进程在执行过程中拥有独立的内存单元, 而多个线程共享内存, 从而极大地提高了程序的运行效率. 
       4. 线程在执行过程中与进程还是有区别的. 每个独立的线程有一个程序运行的入口, 顺序执行序列和程序的出口. 但是线程不能够独立执行, 必须依存在应用程序中, 由应用程序提供多个线程执行控制. 
       5. 从逻辑角度来看, 多线程的意义在于一个应用程序中, 有多个执行部分可以同时执行. 但操作系统并没有将多个线程看做多个独立的应用, 来实现进程的调度和管理以及资源分配. 这就是进程和线程的重要区别.

    打个比方, 多进程就好比同时执行多个程序. 比如同时运行QQ, 音乐播放器, 以及浏览器. 多线程就是同一时刻执行多个线程, 用浏览器一边下载, 一边听歌, 一边看视频, 一边看网页. 浏览器这个进程下有多个线程, 这些线程共享系统分配给浏览器这个进程的资源, 所以各个线程之间可很方便的通信, 一个线程死了, 不会影响其他线程的运行. 如果浏览器这个进程死了, 那么他下面的所有线程就都用不了了. 我们可以通过浏览器打开QQ这里就是进程间通信, 由于不同的进程系统资源不同, 所以进程间的通信不是很容易实现. 并且切换进程会有性能问题.

2.  并发和并行

     并发指能够让多个任务在逻辑上交织执行的程序设计, 但并发事件之间不一定要同一时刻发生. 并行指的是程序运行时的状态是物理上同时执行, 是指同时发生的两个并发事件, 具有并发的含义, 而并发则不一定并行. 

     并发是一种现象, 面对这一现象, 我们首先创建多个线程. 真正加快程序运行速度的, 是并行技术. 也就是让多个CPU同时工作. 而多线程, 是为了让多个CPU同时工作成为可能. 并发设计让并发执行成为可能, 而并行是并发执行的一种模式.

     并发和并行的区别就是一个人同时吃三个馒头和三个人同时吃三个馒头. 一个人同时吃三个馒头但是同一时间只能啃一个馒头, 三个人同时吃三个馒头, 三个馒头同一时间都会被啃.

     下图反映了一个包含8个操作的任务在一个有两核心的CPU中创建四个线程运行的情况. 假设每个核心有两个线程, 那么每个CPU中两个线程会交替并发, 两个CPU之间的操作会并行运算. 就CPU1而言虽然实现了并发但是同一时间只有一个任务在进行, CPU只是快速的在任务之间切换, CPU1和CPU2整体来看是并行元算, 同一时间是有多个任务在进行. 单就一个CPU而言两个线程可以解决线程阻塞造成的不流畅问题, 其本身运行效率并没有提高. 多CPU的并行运算才真正解决了运行效率问题, 这也正是并发和并行的区别. 

     ![并发和并行](http://img.souche.com/f2e/5d7717afde012bbfc557f26bc5eaffea.png)

     注意: 并行和串行是相对应的. 串行在物理层面上同一时刻只会有一个任务在进行.

3.  同步和异步

     同步在发出一个同步调用时, 在没有得到结果之前, 该调用就不返回. 多个任务情况下, 一个任务A执行结束, 才可以执行另一个任务B. 只存在一个线程. 异步在发出一个异步调用后, 调用者不会立刻得到结果, 该调用就返回了. 多个任务情况下, 一个任务A正在执行, 同时可以执行另一个任务B. 任务B不用等待任务A结束才执行. 存在多条线程. 

## 多线程实现方案

在iOS开发中常见的多线程方案一共有4套, 分别是`Pthreads`, `NSThread`, `GCD`, `NSOperation & NSOperationQueue`. 前两种不怎么常用, 这里就简单的介绍一下, 着重介绍后两种. 

### Pthresds

POSIX线程(POSIX threads), 简称Pthreads, 是线程的POSIX标准. 该标准定义了创建和操纵线程的一整套API, 在类Unix操作系统(Unix、Linux、Mac OSX等)中, 都使用Pthreads作为操作系统的线程. 那也就意味着可以跨平台使用, 但是使用起来特别酸爽. 🙂

```
#import <pthread.h>

- (void)pthreadsDoTask {
    /*
     pthread_t：线程指针
     pthread_attr_t：线程属性
     pthread_mutex_t：互斥对象
     pthread_mutexattr_t：互斥属性对象
     pthread_cond_t：条件变量
     pthread_condattr_t：条件属性对象
     pthread_key_t：线程数据键
     pthread_rwlock_t：读写锁
     //
     pthread_create()：创建一个线程
     pthread_exit()：终止当前线程
     pthread_cancel()：中断另外一个线程的运行
     pthread_join()：阻塞当前的线程, 直到另外一个线程运行结束
     pthread_attr_init()：初始化线程的属性
     pthread_attr_setdetachstate()：设置脱离状态的属性（决定这个线程在终止时是否可以被结合）
     pthread_attr_getdetachstate()：获取脱离状态的属性
     pthread_attr_destroy()：删除线程的属性
     pthread_kill()：向线程发送一个信号
     pthread_equal(): 对两个线程的线程标识号进行比较
     pthread_detach(): 分离线程
     pthread_self(): 查询线程自身线程标识号
     //
     *创建线程
     int pthread_create(pthread_t _Nullable * _Nonnull __restrict, //指向新建线程标识符的指针
     const pthread_attr_t * _Nullable __restrict,  //设置线程属性. 默认值NULL. 
     void * _Nullable (* _Nonnull)(void * _Nullable),  //该线程运行函数的地址
     void * _Nullable __restrict);  //运行函数所需的参数
     *返回值：
     *若线程创建成功, 则返回0
     *若线程创建失败, 则返回出错编号
     */
    
    //
    pthread_t thread = NULL;
    NSString *params = @"Hello World";
    int result = pthread_create(&thread, NULL, threadTask, (__bridge void *)(params));
    result == 0 ? NSLog(@"creat thread success") : NSLog(@"creat thread failure");
    //设置子线程的状态设置为detached,则该线程运行结束后会自动释放所有资源
    pthread_detach(thread);
}

void *threadTask(void *params) {
    NSLog(@"%@ - %@", [NSThread currentThread], (__bridge NSString *)(params));
    return NULL;
}

```

看下这些API设计可以说是相当不友好, 并且需要手动管理线程的各个状态的转换和生命周期的管理.

### NSThread

`NSThread`的API设计大致如下: 

```
@interface NSThread : NSObject
//当前线程
@property (class, readonly, strong) NSThread *currentThread;
//使用类方法创建线程执行任务
+ (void)detachNewThreadWithBlock:(void (^)(void))block API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));
+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;
//判断当前是否为多线程
+ (BOOL)isMultiThreaded;
//指定线程的线程参数, 例如设置当前线程的断言处理器. 
@property (readonly, retain) NSMutableDictionary *threadDictionary;
//当前线程暂停到某个时间
+ (void)sleepUntilDate:(NSDate *)date;
//当前线程暂停一段时间
+ (void)sleepForTimeInterval:(NSTimeInterval)ti;
//退出当前线程
+ (void)exit;
//当前线程优先级
+ (double)threadPriority;
//设置当前线程优先级
+ (BOOL)setThreadPriority:(double)p;
//指定线程对象优先级 0.0～1.0, 默认值为0.5
@property double threadPriority NS_AVAILABLE(10_6, 4_0);
//服务质量
@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0);
//线程名称
@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);
//栈区大小
@property NSUInteger stackSize NS_AVAILABLE(10_5, 2_0);
//是否为主线程
@property (class, readonly) BOOL isMainThread NS_AVAILABLE(10_5, 2_0);
//获取主线程
@property (class, readonly, strong) NSThread *mainThread NS_AVAILABLE(10_5, 2_0);
//初始化
- (instancetype)init NS_AVAILABLE(10_5, 2_0) NS_DESIGNATED_INITIALIZER;
//实例方法初始化, 需要再调用start方法
- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument NS_AVAILABLE(10_5, 2_0);
- (instancetype)initWithBlock:(void (^)(void))block API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));
//线程状态, 正在执行
@property (readonly, getter=isExecuting) BOOL executing NS_AVAILABLE(10_5, 2_0);
//线程状态, 正在完成
@property (readonly, getter=isFinished) BOOL finished NS_AVAILABLE(10_5, 2_0);
//线程状态, 已经取消
@property (readonly, getter=isCancelled) BOOL cancelled NS_AVAILABLE(10_5, 2_0);
//取消, 仅仅改变线程状态, 并不能像exist一样真正的终止线程
- (void)cancel NS_AVAILABLE(10_5, 2_0);
//开始
- (void)start NS_AVAILABLE(10_5, 2_0);
//线程需要执行的代码, 一般写子类的时候会用到
- (void)main NS_AVAILABLE(10_5, 2_0);
@end

还有一个NSObject的分类
@interface NSObject (NSThreadPerformAdditions)
//隐式的创建并启动线程, 并在指定的线程（主线程或子线程）上执行方法. 
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array;
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array NS_AVAILABLE(10_5, 2_0);
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait NS_AVAILABLE(10_5, 2_0);
- (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg NS_AVAILABLE(10_5, 2_0);
@end
```

常见的用法有以下几种: 

```
- (void) startThread {

    //创建并且手动启动
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [thread start];

    //类方法创建并且自动启动
    [NSThread detachNewThreadSelector:@selector(run) toTarget:self withObject:nil];

    //通过NSObject的方法自动启动
    [self performSelectorInBackground:@selector(run) withObject:nil];
}    

- (void)run {
    NSLog(@"%@", [NSThread currentThread]);
}
```

### GCD

Grand Central Dispatch, 它是苹果为多核的并行运算提出的解决方案, 会自动合理地利用更多的CPU内核(比如双核、四核), 最重要的是它会自动管理线程的生命周期(创建线程、调度任务、销毁线程), 完全不需要我们管理, 我们只需要告诉干什么就行. 同时它使用的也是C语言, 不过由于使用了Block(Swift里叫做闭包), 使得使用起来更加方便, 而且灵活.

了解GCD我们需要先了解两个概念, **`队列`**和**`任务`**.

1.  任务

     要执行的操作或方法函数. 在GCD中指的就是block块, 在NSThread中指的是`performSelector:`中的方法. 加入任务时有两种形式: `同步任务(dispatch_sync)`和`异步任务(dispatch_async)`.

    1. 同步任务: 不会开启新的线程, 会阻塞当前线程. 完成需要做的任务后才会返回, 进行下一任务. 创建方式为:
        ```
         //把右边的参数(任务)提交给左边的参数(队列)进行执行
         dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
        ```
    2. 异步任务: 不会等待任务完成才返回, 会立即返回. 异步是多线程的代名词, 因为必定会开启新的线程, 线程的申请是由异步负责, 起到开分支的作用. 创建方式为:

        ```
         //把右边的参数(任务)提交给左边的参数(队列)进行执行
         dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
        ```
2.  队列

     存放任务的集合, 我们要做的就是将任务添加到队列然后执行, GCD会自动将队列中的任务按先进先出的方式取出. 队列主要有的有`串行队列(Serial Dispatch Queue)`, `并行队列(Concurrent Dispatch Queue)`, `全局队列(Global Queue)`和`主队列(Main Queue)`.

    1.   串行队列: 任务依次执行, 同一时间队列中只有一个任务在执行, 每个任务只有在前一个任务执行完成后才能开始执行. 你不知道在一个Block(任务)执行结束到下一个Block(任务)开始执行之间的这段时间时间是多长, 这部分是由GCD控制. 创建方式为:

          ```
           //创建一个名为queue的串行队列
           //第一个参数为队列名称
           //第二个参数为队列类型, DISPATCH_QUEUE_SERIAL和NULL表示串行队列
           dispatch_queue_t queue = dispatch_queue_create("com.private.SerialQueue", DISPATCH_QUEUE_SERIAL);
          ```

    2.   并行队列: 任务并发执行, 唯一能保证的是, 这些任务会按照被添加的顺序开始执行. 但是任务可以以任何顺序完成, 你不知道在执行下一个任务是从什么时候开始, 或者说任意时刻有多个Block(任务)运行, 这个完全是取决于GCD. GCD默认已经提供了全局的并发队列, 供整个应用使用, 一般不需要手动创建创建方式为:

          ```
           //创建一个名为queue的并行队列
           //第一个参数为队列名称
           //第二个参数为队列类型, DISPATCH_QUEUE_CONCURRENT表示并行队列
           dispatch_queue_t queue = dispatch_queue_create("com.private.ConcurrentQueue",DISPATCH_QUEUE_CONCURRENT);
          ```

    3.   全局队列: **隶属于并行队列**, 不要与 barrier 栅栏方法搭配使用, barrier 只有与自定义的并行队列一起使用, 才能让 barrier 达到我们所期望的栅栏功能. 与串行队列或者global队列 一起使用, barrier的表现会和dispatch_sync方法一样. 创建方式为:

          ```
           //获取系统全局队列, 并赋值给队列queue
           //第一个参数表示优先级, 这里是新老优先级的对照Map
         *  - DISPATCH_QUEUE_PRIORITY_HIGH:         QOS_CLASS_USER_INITIATED//高
         *  - DISPATCH_QUEUE_PRIORITY_DEFAULT:      QOS_CLASS_DEFAULT//默认
         *  - DISPATCH_QUEUE_PRIORITY_LOW:          QOS_CLASS_UTILITY//低
         *  - DISPATCH_QUEUE_PRIORITY_BACKGROUND:   QOS_CLASS_BACKGROUND//后台

         //第二个参数是预留参数, 传0就好
         dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
          ```

    4.   主队列: **隶属于串行队列**, 不能与sync同步方法搭配使用, 会造成死循环. 主队列是GCD自带的一种特殊的串行队列, 放在主队列中的任务, 都会放到主线程中执行. 创建方式为:

          ```
           //获取系统主队列, 并赋值给队列queue
           dispatch_queue_t queue = dispatch_get_main_queue();
          ```

不管是串行队列(SerialQueue)还是并行队列(ConcurrencyQueue), 都是FIFO队列. 也就意味着, 任务一定是一个一个地, 按照先进先出的顺序来执行.

### NSOperation & NSOperationQueue

`NSOperation` 是苹果公司对`GCD`的封装, 完全面向对象, 所以使用起来更好理解. 大家可以看到 `NSOperation`和`NSOperationQueue`分别对应`GCD`的**任务**和**队列**. 所以他的操作步骤和`GCD`类似: 首先将要执行的任务封装到一个`NSOperation`对象中. 然后将此任务添加到一个`NSOperationQueue`对象中, 然后系统就会自动在执行任务.

1. `NSOperation`主要有一下这些属性和方法. 

         ​```
        @interface NSOperation : NSObject {
        @private
            id _private;
            int32_t _private1;
        #if __LP64__
            int32_t _private1b;
        #endif
        }
        
        - (void)start;//启动任务 默认在当前线程执行
        - (void)main;//自定义NSOperation，写一个子类，重写这个方法，在这个方法里面添加需要执行的操作。
        
        @property (readonly, getter=isCancelled) BOOL cancelled;//是否已经取消，只读
        - (void)cancel;//取消任务
        
        @property (readonly, getter=isExecuting) BOOL executing;//正在执行，只读
        @property (readonly, getter=isFinished) BOOL finished;//执行结束，只读
        @property (readonly, getter=isConcurrent) BOOL concurrent; // To be deprecated; use and override 'asynchronous' below
        @property (readonly, getter=isAsynchronous) BOOL asynchronous NS_AVAILABLE(10_8, 7_0);//是否并发，只读
        @property (readonly, getter=isReady) BOOL ready;//准备执行
        
        - (void)addDependency:(NSOperation *)op;//添加依赖
        - (void)removeDependency:(NSOperation *)op;//移除依赖
        
        @property (readonly, copy) NSArray<NSOperation *> *dependencies;//所有依赖关系，只读
        
        typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
            NSOperationQueuePriorityVeryLow = -8L,
            NSOperationQueuePriorityLow = -4L,
            NSOperationQueuePriorityNormal = 0,
            NSOperationQueuePriorityHigh = 4,
            NSOperationQueuePriorityVeryHigh = 8
        };//系统提供的优先级关系枚举
        
        @property NSOperationQueuePriority queuePriority;//执行优先级
        
        @property (nullable, copy) void (^completionBlock)(void) NS_AVAILABLE(10_6, 4_0);//任务执行完成之后的回调
        
        - (void)waitUntilFinished NS_AVAILABLE(10_6, 4_0);//阻塞当前线程，等到某个operation执行完毕。
        
        @property double threadPriority NS_DEPRECATED(10_6, 10_10, 4_0, 8_0);//已废弃，用qualityOfService替代。
        
        @property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0);//服务质量，一个高质量的服务就意味着更多的资源得以提供来更快的完成操作。
        
        @property (nullable, copy) NSString *name NS_AVAILABLE(10_10, 8_0);//任务名称
        
        @end
        ​```

    由于`NSOperation`是一个抽象基类, 不能直接使用, 在这里我么你一般使用`NSOperation`的两个子类`NSInvocationOperation`和`NSBlockOperation`, 或者`NSOperation的自定义子类`:

        1. `NSInvocationOperation`基于应用的一个target对象和selector来创建operation object. 如果你已经有现有的方法来执行需要的任务, 就可以使用这个类. 使用方式大致如下: 

        ```
        - (void)NSInvocationOperationRun {
            NSInvocationOperation *invocationOper = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(invocationOperSel) object:nil];
            [invocationOper start];
        }
        - (void)invocationOperSel {
            NSLog(@"NSInvocationOperationRun_%@", [NSThread currentThread]);
        }
        ```
    从打印结果可以看出, 该实现方式是同步顺序执行.

        2.  `NSBlockOperation`用来并发地执行一个或多个block对象. operation object使用**组**的语义来执行多个block对象，所有相关的 block 都执行完成之后, operation object才算完成. 使用方法如下
    
             ```
             //系统提供的API
             @interface NSBlockOperation : NSOperation {
             @private
                 id _private2;
                 void *_reserved2;
             }
             
            + (instancetype)blockOperationWithBlock:(void (^)(void))block;//在当前线程执行
        
            - (void)addExecutionBlock:(void (^)(void))block;//新开线程执行
            @property (readonly, copy) NSArray<void (^)(void)> *executionBlocks;
        
            @end
             ```

        使用`NSBlockOperation`类方法创建任务, 从打印结果可以看出来该任务是在主线程执行的

            ```
            - (void)NSBlockOperationRun {
                NSBlockOperation *blockOper = [NSBlockOperation blockOperationWithBlock:^{
                    NSLog(@"NSBlockOperationRun_%@_%@", [NSOperationQueue currentQueue], [NSThread currentThread]);
                }];
                [blockOper start];
            }
            ```

        使用`NBlockOperation`的实例方法创建任务, 从打印结果可以看出来, 第一个任务是在主线程执行, 其他任务均是新开的线程, 所有的任务是以异步并发的形式执行. 

            ```
            - (void)NSBlockOperationRun {
                NSBlockOperation *blockOper = [NSBlockOperation blockOperationWithBlock:^{
                    NSLog(@"NSBlockOperationRun_1_%@", [NSThread currentThread]);
                }];
                [blockOper addExecutionBlock:^{
                    NSLog(@"NSBlockOperationRun_2_%@", [NSThread currentThread]);
                }];
                [blockOper addExecutionBlock:^{
                    NSLog(@"NSBlockOperationRun_3_%@", [NSThread currentThread]);
                }];
                [blockOper addExecutionBlock:^{
                    NSLog(@"NSBlockOperationRun_4_%@", [NSThread currentThread]);
                }];
                [blockOper start];
            }
            ```
            
        3. `NSOperation的自定义子类`. 
        
            ```        
            在子类中重写父类的`-(void)main`函数, 在里面实现主要逻辑
            ```

     `NSBlockOperation`, `NSBlockOperationRun`或者`NSOperation的自定义子类`创建的operation object都可以使用`NSOperation`的所有属性, 意味着他们具有以下主要特性: 

    1. 多个任务之间可以使用有依赖关系
    2. 可以获得Operation object任务执行完成之后的回调
    3. 支持应用使用 KVO 通知来监控 operation 的执行状态
    4. 可以通过operation优先级, 从而影响相对的执行顺序
    5. 可以终止正在执行的任务

2. `NSOperationQueue`

    类似于GCD中的队列, 只需要吧任务加入到`NSOperationQueue`中, 就会自动运行, 由系统通过最大并发数来控制是并行还是串行, 官方提供的API如下:
    
    ```
    static const NSInteger NSOperationQueueDefaultMaxConcurrentOperationCount = -1;
    
    NS_CLASS_AVAILABLE(10_5, 2_0)
    @interface NSOperationQueue : NSObject {
    @private
        id _private;
        void *_reserved;
    }
    
    - (void)addOperation:(NSOperation *)op;//向队列中添加任务
    - (void)addOperations:(NSArray<NSOperation *> *)ops waitUntilFinished:(BOOL)wait API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//添加一组任务
    
    - (void)addOperationWithBlock:(void (^)(void))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//添加一个block形式的任务
    
    @property (readonly, copy) NSArray<__kindof NSOperation *> *operations;//队列中所有任务的数组
    @property (readonly) NSUInteger operationCount API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//队列中所有任务数
    
    @property NSInteger maxConcurrentOperationCount;//最大并发数
    
    @property (getter=isSuspended) BOOL suspended;//暂停
    
    @property (nullable, copy) NSString *name API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//名称
    
    @property NSQualityOfService qualityOfService API_AVAILABLE(macos(10.10), ios(8.0), watchos(2.0), tvos(9.0));//服务质量, 系统会为权重高的服务分配更多的资源
    
    @property (nullable, assign /* actually retain */) dispatch_queue_t underlyingQueue API_AVAILABLE(macos(10.10), ios(8.0), watchos(2.0), tvos(9.0));
    
    - (void)cancelAllOperations;//取消队列中所有的任务
    
    - (void)waitUntilAllOperationsAreFinished;//阻塞当前线程, 等到队列中的全部任务全部执行完毕
    
    @property (class, readonly, strong, nullable) NSOperationQueue *currentQueue API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//获取当前队列
    @property (class, readonly, strong) NSOperationQueue *mainQueue API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));//获取主队列
    
    @end

    ```
    
    使用方式如下:
    
    ```
    //以三种方式为队列添加了三个任务, 
    - (void)NSOperationQueueRun {
        NSOperationQueue *queue = [[NSOperationQueue alloc] init];
        NSInvocationOperation *invocationOper = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(invocationOperSel) object:nil];
        [queue addOperation:invocationOper];
        NSBlockOperation *blockOper = [NSBlockOperation blockOperationWithBlock:^{
            NSLog(@"NSBlockOperationRun_%@", [NSThread currentThread]);
        }];
        [queue addOperation:blockOper];
        [queue addOperationWithBlock:^{
            NSLog(@"QUEUEBlockOperationRun_%@", [NSThread currentThread]);
        }];
    }
    
    - (void)invocationOperSel {
        NSLog(@"NSInvocationOperationRun_%@", [NSThread currentThread]);
    }
    ```
    一般情况下, `NSOperationQueue`会按照任务添加的顺序来执行任务, 但是我们可以通过使用优先级和依赖关系来改变执行顺序. 并发数大于任务数, 没有设置依赖关系, 并且在同一队列.
    
## 常见的使用方式

### GCD向并行队列中添加异步任务

该过程一共分为两步, 第一步获取系统提供的全局并行队列, 第二步向并行队列中添加异步任务. 任务会遵守FIFO原则来执行, 但是每个任务何时执行完我们却并不知道. 但是系统会新开线程来执行异步任务, 三个异步任务会分别在三个线程中执行, 各个任务的执行顺序无法确定.

```
- (void)test1 {
    //获取系统提供的全局队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片1----%@", [NSThread currentThread]);
        sleep(5);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片2----%@", [NSThread currentThread]);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片3----%@", [NSThread currentThread]);
    });
}
```

### GCD向串行队列中添加异步任务

该过程一共分为两步, 第一步创建串行队列, 第二步向并行队列中添加异步任务. 任务会遵守FIFO原则来执行, 每个任务何时执行完我们却并不知道. 但是由于我们是在串行队列中添加的任务, 不会新开线程, 所有的异步任务会按照顺序前一个执行完毕再执行后一个.

 ```
 - (void)test2 {
    //创建串行队列, 第一个参数为串行队列的名称, 第二个参数为队列的属性, NULL或者DISPATCH_QUEUE_SERIAL表示是串行队列
    dispatch_queue_t queue = dispatch_queue_create("test2", NULL);
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片1----%@", [NSThread currentThread]);
        sleep(5);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片2----%@", [NSThread currentThread]);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片3----%@", [NSThread currentThread]);
    });
}
 ```

### GCD向并行队列添加同步任务

该过程一共分为两步, 第一步获取系统提供的全局并行队列, 第二步向并行队列中添加同步任务. 任务会遵守FIFO原则来执行, 每个任务何时执行完我们却并不知道. 虽然是并行队列, 单由于添加的是同步任务, 不会新开线程, 全都在当前线程中执行, 所以三个任务会顺序执行, 并行队列失去了并行的能力. 

```
//并行队列中添加同步任务
- (void)test3 {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    //向队列中添加异步任务
    dispatch_sync(queue, ^{
        NSLog(@"下载图片1----%@", [NSThread currentThread]);
        sleep(5);
    });
    
    //向队列中添加异步任务
    dispatch_sync(queue, ^{
        NSLog(@"下载图片2----%@", [NSThread currentThread]);
    });
    
    //向队列中添加异步任务
    dispatch_sync(queue, ^{
        NSLog(@"下载图片3----%@", [NSThread currentThread]);
    });
}
```

### GCD 向串行队列添加异步任务

该过程一共分为两步, 第一步创建自定义串行队列, 第二步向并行队列中添加异步任务. 虽然异步任务会新开线程, 但是由于是在串行队列中执行, 所以不新开线程, 所有任务前一个执行完毕再执行下一个.

```
- (void)test4 {
    dispatch_queue_t queue = dispatch_queue_create("test4", NULL);
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片1----%@", [NSThread currentThread]);
        sleep(5);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片2----%@", [NSThread currentThread]);
    });
    
    //向队列中添加异步任务
    dispatch_async(queue, ^{
        NSLog(@"下载图片3----%@", [NSThread currentThread]);
    });
}
``` 

### GCD监听并行队列中多个异步任务是否已经完成(Dispatch Group)

分组模式 `dispatch_group_notify`. 可以异步执行多个耗时操作. 等耗时操作都执行完毕之后会回到主线程执行操作, 主要用于监听任务是否完成. 主要有两种用法: 

第一种用法是通过`dispatch_group_async`, 首先创建一个全局并行队列和一个group队列组, 再次创建异步group任务加入到前面创建的group队列中去. 当所有异步任务完成后, 会通过`dispatch_group_notify`回到主线程.

```
- (void)groupTest1 {
    //获取全局队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    //创建一个队列组
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"--- 1 开始--- %@", [NSThread currentThread]);
        //延时5秒 模仿堵塞子线程
        [NSThread sleepForTimeInterval:5];
        NSLog(@"--- 1 --- 完成 %@", [NSThread currentThread]);
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"--- 2 开始--- %@", [NSThread currentThread]);
        //延时5秒 模仿堵塞子线程
        [NSThread sleepForTimeInterval:5];
        NSLog(@"--- 2 --- 完成 %@", [NSThread currentThread]);
    });
    
    //在这个队列组里面，会等group中的全部代码执行完毕再去执行其它的操作
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 等前面的异步操作都执行完毕后，回到主线程...
        NSLog(@"全部完成");
    });
}
```

第二种用法是通过信号量`dispatch_group_enter` 和 `dispatch_group_leave`. 创建一个并行线程和多个异步任务, 将异步任务添加到并行线程中区. 通过`dispatch_group_enter`来告知`group`一个异步任务开始,  未执行完毕任务数加1. 在异步线程任务执行完毕时, 通过`dispatch_group_leave`告知group, 一个任务结束, 未执行完毕任务数减1. 当未执行完毕任务数为0的时候, 这时group才认为组内任务都执行完毕了(这个和GCD的信号量的机制有些相似), 这时候才会回调`dispatch_group_notify`中的block. 此处需要注意, `dispatch_group_enter`和`dispatch_group_leave`的数量一定要一致. 

```
- (void)groupTest2 {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        sleep(2); //这里线程睡眠1秒钟，模拟异步请求
        NSLog(@"%@ one finish", [NSThread currentThread]);
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        sleep(2); //这里线程睡眠1秒钟，模拟异步请求
        NSLog(@"%@ two finish", [NSThread currentThread]);
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, queue, ^{
        NSLog(@"group finished");
    });
}
```

### GCD延时操作(dispatch_after)

使用很简单, 两个核心的对象`dispatch_time_t`和`dispatch_after`. 大家都经常使用, 不用多说. 下面的代码和在3妙手用`dispatch_async`函数追加block到主线程的操作是相同的

```
- (void)afterTest {
    double delayInSeconds = 2.0;
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t) (delayInSeconds * NSEC_PER_SEC));
    //第一个参数代表时间, 第二各参数代表要在哪个线程执行接下来的任务, 第三个参数就是任务block
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
        NSLog(@"%@", [NSThread currentThread]);
    });
}
```

### GCD单例(dispatch_once)

这个更是不用多说, 直接贴上代码.

```
@interface Tool : NSObject
+ (instancetype)sharedTool;
@end

@implementation Tool
static id _instance;
+ (instancetype)sharedTool {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _instance = [[Tool alloc] init];
    });
    return _instance;
}
@end
```

### GCD 线程间的通讯

常用的方式就是在后台线程中执行长时间任务, 处理结束时, 主线程使用该处理结果. 代码如下: 

```
- (void)threadTest {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(queue, ^{
        //在这里执行耗时的操作
        dispatch_async(dispatch_get_main_queue(), ^{
            //在主线程使用上面操作的结果
        });
    });
}
```

### GCD重复执行同一个任务(dispatch_apply)

`disaptch_apply`函数是`dispatch_sync`函数和`Dispatch Group`的关联API, 该函数按照指定的次数将指定的Block追加到指定的Dispatch Queue中, 并等待处理执行结束. 

```
- (void)applyTest {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply(10, queue, ^(size_t index) {
        //并行处理10次任务
        NSLog(@"%zu", index);
    });
}
```

但是由于`dispatch_apply`函数与`dispatch_sync`函数相同, 会等待任务执行结束, 所以我们强烈推荐在`dispatch_async`函数中异步执行. 

```
- (void)applyTest {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    //在全局并行队列中异步执行
    dispatch_async(queue, ^{
        //等待函数中操作全部执行完毕
        dispatch_apply(10, queue, ^(size_t index) {
            //并行处理10次任务
            sleep(2);
            NSLog(@"%zu", index);
        });
        
        //dispatch_apply中的处理全部结束, 在主线程异步执行
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"Done");
        });
    });
}
```

### GCD设置优先级(dispatch_set_target_queue)

将queue的优先级通过`dispatch_set_target_queue`变更的和queue1的优先级一致, 代码如下. 

```
- (void)targetQueeuTest {
    dispatch_queue_t queue = dispatch_queue_create("test", NULL);
    dispatch_queue_t queue1 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    //把第一个参数的执行优先级设置的和第二个参数的优先级一致.
    dispatch_set_target_queue(queue, queue1);
}
```

### GCD栅栏(dispatch_barrier_async)

以下面的代码为例, 假设一个并行队列中添加了五个异步任务, 虽然遵循FIFO的原则, 这五个任务的执行必定是无序的. 当我们在第三个任务之后加入`dispatch_barrier_async`任务, 那么这五个任务就被分割成两部分, 前三次无序执行, 然后执行`dispatch_barrier_async`任务, 然后再无序执行后两次任务. 注意**此处的队列只能使用自定义的并行队列, 系统提供的全局队列不行**

```
- (void)barrierTest {
    
    //此处的队列只能使用自定义的并行队列, 系统提供的全局队列不行
    dispatch_queue_t queue = dispatch_queue_create("barrierTest", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"1");
    });
    dispatch_async(queue, ^{
        NSLog(@"2");
    });
    dispatch_async(queue, ^{
        NSLog(@"3");
    });
    dispatch_barrier_async(queue, ^{
        sleep(3);
        NSLog(@"插入执行");
    });
    dispatch_async(queue, ^{
        NSLog(@"4");
    });
    dispatch_async(queue, ^{
        NSLog(@"5");
    });
}
```

### GCD信号量(dispatch_semaphore_t)

我们以这个使用场景为例, 来说明一下, 不考虑顺序 将所有的数据添加到空数组中去, 我们可能会这么实现:

```
- (void)dispatchSemaphoreDemo {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    NSMutableArray *array = [NSMutableArray array];
    for (int i = 0; i < 100000; i++) {
        dispatch_async(queue, ^{
            NSLog(@"%@", [NSThread currentThread]);
            [array addObject:@(i)];
        });
    }
}
```

因为是并行队列, 异步任务, 所以会开大量的线程, 并且这些线程会保存在内存中,相当耗费资源, 最终运行的结果就是崩溃. 那么我们就需要通过信号量来控制任务的执行. `dispatch_semaphore_t`类似于单个队列的最大并发数控制机制, 提高并行效率的同时, 也可以防止太多线程的开辟对系统造成太大的负担. 改造后应该是这样的: 

```
- (void)dispatchSemaphoreDemo {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    NSMutableArray *array = [NSMutableArray array];
    
    //设置信号量初始值, 当信号量为0时, 所有任务等待, 信号量越大, 允许可并行执行的任务数量越多. 并发的线程由系统调配, 不一定一直是同样的两条, 但是最多只能同时存在两条.
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(2);
    for (int i = 0; i < 100; i++) {
        
        //当信号量大于等于设定的初始值时就继续执行, 否则一直等待
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        dispatch_async(queue, ^{
            //执行到这里就代表信号量大于等于设定的初始值, 所以在这里信号量要减1
            NSLog(@"%d+++++%@", i, [NSThread currentThread]);
            [array addObject:@(i)];
            //到这里的时候, 因为异步任务已经将要结束, 要将信号量加1. 如果前面有等待的线程, 最先等待的线程先执行
            dispatch_semaphore_signal(semaphore);
        });
    }
}
```

### GCD定时器

GCD实现定时器主要用到`dispatch_source_t`函数, 该函数实际有多种Type, `DISPATCH_SOURCE_TYPE_TIMER`是定时器相关的任务, 还有其他类型的处理事件类型. 

```
- (void)timerDemo {
    NSLog(@"%@", [NSThread currentThread]);
    //指定DISPATCH_SOURCE_TYPE_TIMER类型, 作成Dispatch Source
    
    //在定时器经过指定时间, 把任务追加到main queue
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
    
    //定时器的相关设置, 将定时器设置为5s后, 不指定为重复, 允许延迟1s
    dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, 5ull * NSEC_PER_SEC), DISPATCH_TIME_FOREVER, 1ull * NSEC_PER_SEC);
    
    //指定定时器指定时间内执行的处理
    dispatch_source_set_event_handler(timer, ^{
        NSLog(@"wake up");
        dispatch_source_cancel(timer);
    });
    
    //取消定时器时的处理
    dispatch_source_set_cancel_handler(timer, ^{
        NSLog(@"canceled");
    });
    
    //定时器启动
    dispatch_resume(timer);
}
```

### NSOperation的优先级

NSOperationQueue中NSOperation对象的执行方式是按照FIFO的原则顺序执行, 但是如果我们设置了任务的优先级, 那么系统就会给优先级高的优先分配资源. 

优先级一共有这几种, 从高到低依次排列. 
```
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
	NSOperationQueuePriorityVeryLow = -8L,
	NSOperationQueuePriorityLow = -4L,
	NSOperationQueuePriorityNormal = 0,
	NSOperationQueuePriorityHigh = 4,
	NSOperationQueuePriorityVeryHigh = 8
};
```

示例代码如下, 三个任务被添加到队列中去, 这三个任务分别在三个线程, 互不干扰, 类似于GCD的异步并行模式, 如果我们设置`invocationOper.queuePriority = NSOperationQueuePriorityVeryLow`理论上, `invocationOper`任务会在最后执行, 但是我们发现并没有, 那是因为我们没有设置最大并发数的原因, 我猜测系统可能是做了某些限制, 虽然是新开了线程, 但是由于最大并发数小于等于1实际还是按照FIFO顺序执行, **所以注意一定要设置最大并发数**.

```
//把注释打开就可以看到想要的效果
- (void)NSOperationQueueRun {
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    //queue.maxConcurrentOperationCount = 3;
    NSInvocationOperation *invocationOper = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(invocationOperSel) object:nil];
    //invocationOper.queuePriority = NSOperationQueuePriorityVeryLow;
    [queue addOperation:invocationOper];
    NSBlockOperation *blockOper = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"NSBlockOperationRun_%@", [NSThread currentThread]);
    }];
    [queue addOperation:blockOper];
    [queue addOperationWithBlock:^{
        NSLog(@"QUEUEBlockOperationRun_%@", [NSThread currentThread]);
    }];
}

- (void)invocationOperSel {
    NSLog(@"NSInvocationOperationRun_%@", [NSThread currentThread]);
}
```

### NSOperation的依赖关系

首先我们创建一个队列, 分别创建要执行的任务, **在把要执行的任务添加到队列中去之前, 通过`addDependency`来建立任务间的依赖关系**. 如果我们没有设置依赖的话, 如果设置的并发数大于1,  那么`blockOper_1`和``blockOper_2`的执行顺序是随机的. 可是当我们执行`[blockOper_1 addDependency:blockOper_2]`时, 就给两个任务添加了依赖关系, `blockOper_1`永远只会在`blockOper_2`后执行. 即使我们给`blockOper_1`设置了最高的优先级, 因为**依赖的优先级要高于`queuePriority`**. 

```
- (void)NSOperationQueueRun2 {
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
//    queue.maxConcurrentOperationCount = 2;
    NSBlockOperation *blockOper_1 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"blockOper_1_%@_%@",@(1),[NSThread currentThread]);
    }];
    
    NSBlockOperation *blockOper_2 = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"blockOper_2_%@_%@",@(2),[NSThread currentThread]);
    }];
    
//    blockOper_1.queuePriority = NSOperationQueuePriorityVeryHigh;
    [blockOper_1 addDependency:blockOper_2];
    [queue addOperation:blockOper_1];
    [queue addOperation:blockOper_2];
}
```

## 常见使用案列分析

### 案例一

```
- (void)case1 {
    NSLog(@"任务一 - %@", [NSThread currentThread]);
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"同步任务 - %@",[NSThread currentThread]);
    });
    NSLog(@"任务二 - %@", [NSThread currentThread]);
}

```

以上`- (void)case1`默认在当前线程(主线程)执行,  首先执行"任务一". 然后`dispatch_sync`阻塞当前线程(主线程), 由于主队列是串行队列, 任务不能并发执行, 同时只能有一个任务在执行. 又因为**"同步任务"后入列, 按照FIFO原则, 必须等到`- (void)case1`执行完毕才能执行**, 但是`- (void)case1`中的"任务二"又在等着"同步任务"执行完毕才能执行, 这样就造成同一队列中的两个任务相互等待, 造成死锁. 

![案例1](http://img.souche.com/f2e/5b1729532683ad164c061958d5248a66.png)

有两种解决思路, 一是为当前队列扩容, 让它变成并行队列, 就不会造成阻塞. 第二种解决思路是把同步任务添加到一个新的自定义串行队列中去, 两个队列之间不存在阻塞就避免了死锁问题. 

### 案例二

```
- (void)case2 {
    NSLog(@"1");
    //3会等2，因为2在全局并行队列里，不需要等待3，这样2执行完回到主队列，3就开始执行
    dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

打印顺序一定为1, 2, 3. 2新开队列, 不会造成死锁. `case2`和新开的block任务之间没有队列的约束, 但是由于是同步任务,阻塞线程, 所以block先执行, 执行完毕后再回到`case2`, 接着执行.

![案例2](http://img.souche.com/f2e/27086c860f921209238d3f60c2f8722c.png)

### 案例三

```
- (void)case3 {
    dispatch_queue_t queue = dispatch_queue_create("com.demo.serialQueue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"1"); // 任务1
    dispatch_async(queue, ^{
        NSLog(@"2"); // 任务2
        dispatch_sync(queue, ^{
            NSLog(@"3"); // 任务3
        });
        NSLog(@"4"); // 任务4
    });
    NSLog(@"5"); // 任务5
}
```

控制台输出1, 5, 2. (5, 2的顺序不一定). 可以肯定, 使用的是自定义串行队列. 首先执行任务1, 接下来有一个异步任务, 新开辟线程(将任务2, 同步线程, 任务4加入新线程). 因为是异步线程, 主线程中的任务5不用等待, 所以 2 和 5 的输出顺序不定. 任务2执行完后, 遇到同步任务, 因为是在串行队列, 所以会阻塞当前线程, 任务3执行完后才能执行任务4. 但是任务4比任务3早加入队列, 根据FIFO原则, 任务3要等待任务4执行完后才能执行. 任务3和任务4相互等待造成死锁. 

![案例三图片](http://img.souche.com/f2e/9c13eaeac28f7fe7105af2feb878d5ef.png)

### 案例四

```
- (void)case4 {
    NSLog(@"1"); // 任务1
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2"); // 任务2
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"3"); // 任务3
        });
        NSLog(@"4"); // 任务4
    });
    NSLog(@"5"); // 任务5
}
```

打印结果是: 1, 5, 2, 3, 4.(2, 5的顺序不定). 首先在主线程中有三个任务, 分别是任务1, 并行队列的异步任务, 任务5. 并行队列的异步任务中又有三个任务, 分别是任务2, 异步任务, 任务4.  

所以先打印1, 然后把异步任务加入全部并行队列, 由于是异步任务新开辟线程, 不阻塞主线程, 5不用等待. 所以2, 5顺序不定. 然后再分析并行队列中的异步任务, 任务2执行后遇到同步任务, 同步任务被加入到主线程(因为前面已经被加过3个任务, 所以该任务在任务5后面). 因为是同步, 阻塞主线程, 所以4一定在3后面. 

![案例四](http://img.souche.com/f2e/43a48f9de1c0bb336215543157ad875b.png
)

### 案例五

```
- (void)case5 {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"1"); // 任务1
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"2"); // 任务2
        });
        NSLog(@"3"); // 任务3
    });
    NSLog(@"4"); // 任务4
    while (1) {
    }
    NSLog(@"5"); // 任务5
}

```

打印结果是: 1, 4(顺序不定). 首先主线程中有四个任务, 分别是全局并行队列的异步任务, 任务4, 死循环, 任务5. 全局并行队列的异步任务中又有三个任务, 分别是任务1, 主线程的同步任务(任务2), 任务3. 

由于是异步任务, 不阻塞主线程, 所以全局并行队列的异步任务执行顺序和任务4不定, 全局并行队列的异步任务又是先执行任务1, 所以任务1和任务4的顺序不定. 当任务4执行完后, 进入死循环, 主线程被阻塞. 由于同步任务(任务2)是被加在主线程中, 但是此时主线程已被阻塞, 所以2不会被执行. 任务3又是在任务2后被加入队列, 任务2是同步任务, 所以任务3要等待任务2完成才执行. 所以任务2, 任务3都不会执行. 任务5在死循环后执行, 所以任务5永远不会被执行. 如果没有死循环, 任务2肯定在任务后后面, 任务3肯定在任务2后面. 

![案例五](http://img.souche.com/f2e/0d108cba7a737b62e2f5e1be402bc5d0.png)

-------
参考资料:

1.[关于iOS多线程, 你看我就够了](http://www.cocoachina.com/ios/20150731/12819.html)

2.[iOS多线程编程总结](https://bestswifter.com/multithreadconclusion/)

3.[关于iOS多线程, 我说, 你听, 没准你就懂了](http://www.jianshu.com/p/51fd1362249e)

4.[深入理解 GCD](http://www.jianshu.com/p/06a18323d9d2)

5.[关于 iOS 多线程, 都在这里了](http://www.jianshu.com/p/6a6722f12fe3)

6.[进程与线程的区别](http://blog.csdn.net/mxsgoden/article/details/8821936)

7.[并发和并行](https://laike9m.com/blog/huan-zai-yi-huo-bing-fa-he-bing-xing,61/)



















