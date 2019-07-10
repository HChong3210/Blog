---
title: iOS开发中的锁
date: 2018-03-21 22:31:14
tags:
    - 基础知识
categories: 
    - 基础知识
---

锁是比较常用的同步工具, 用于保证一段代码段在同一个时间只能允许被有限个线程访问, 下面我们来介绍下iOS开发中常见的锁.

## NSLock

> The NSLock class uses POSIX threads to implement its locking behavior. When sending an unlock message to an NSLock object, you must be sure that message is sent from the same thread that sent the initial lock message. Unlocking a lock from a different thread can result in undefined behavior.

NSLock是一个对象锁, 遵循 NSLocking 协议, 加锁和解锁务必在同一线程中完成. 常用方法有以下几个: `-lock` 方法是加锁, `-unlock` 是解锁, `-tryLock` 是尝试加锁, 如果失败的话返回 NO, `-lockBeforeDate:` 是在指定Date之前尝试加锁, 如果在指定时间之前都不能加锁, 则返回NO. 

```
//主线程中
NSLock *lock = [[NSLock alloc] init];
    
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   NSLog(@"进入线程1");
   [lock lock];
   NSLog(@"线程1");
   sleep(5);
   [lock unlock];
   NSLog(@"线程1解锁成功");
});
    
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   NSLog(@"进入线程2");
   sleep(1);
   [lock lock];
   NSLog(@"线程2");
   [lock unlock];
});
```
执行结果是: 

```
2018-03-25 14:20:16.642007+0800 HCLock[9802:336637] 进入线程1
2018-03-25 14:20:16.642007+0800 HCLock[9802:336636] 进入线程2
2018-03-25 14:20:16.642181+0800 HCLock[9802:336637] 线程1
2018-03-25 14:20:21.647601+0800 HCLock[9802:336637] 线程1解锁成功
2018-03-25 14:20:21.647624+0800 HCLock[9802:336636] 线程2
```
这里说一下lock的具体过程: 首先两个异步线程, 分别进入两个线程. 当线程1 lock 时, 这时会阻塞线程. 因为线程2中也有lock, 这时会空转(可以理解为跑一个while循环), 不断去申请加锁(在空转1s后, 线程会进入waiting状态, 此时线程就不占用CPU资源了), 等锁可用的时候, 这个线程会立即被唤醒. 

这里值得注意的是tryLock是不会阻塞线程的. lockBeforeDate: 方法会在所指定 Date 之前尝试加锁, 会阻塞线程, 如果在指定时间之前都不能加锁, 则返回 NO, 指定时间之前能加锁, 则返回 YES.

NSLock实际是在内部封装了pthread_mutex, 属性为 PTHREAD_MUTEX_ERRORCHECK, 它会损失一定性能换来错误提示.
## NSConditionLock
NSConditionLock是一个条件锁, 和 NSLock 类似, 都遵循 NSLocking 协议, 方法都类似, 只是多了一个 condition 属性, 以及每个操作都多了一个关于 condition 属性的方法, 例如 tryLock, tryLockWhenCondition:, NSConditionLock 可以称为条件锁, 只有 condition 参数与初始化时候的 condition 相等, lock 才能正确进行加锁操作. 而 unlockWithCondition: 并不是当 Condition 符合条件时才解锁, 而是解锁之后, 修改 Condition 的值. 

```
@interface NSConditionLock : NSObject <NSLocking> {
@private
    void *_priv;
}

- (instancetype)initWithCondition:(NSInteger)condition NS_DESIGNATED_INITIALIZER;

@property (readonly) NSInteger condition;
- (void)lockWhenCondition:(NSInteger)condition;
- (BOOL)tryLock;
- (BOOL)tryLockWhenCondition:(NSInteger)condition;
- (void)unlockWithCondition:(NSInteger)condition;
- (BOOL)lockBeforeDate:(NSDate *)limit;
- (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end
```
关于上面几个方法有几点要说明: 

* 初始化时候的 condition 参数为 0.
* lockWhenCondition加锁失败会阻塞线程, tryLockWhenCondition则不会.
* NSConditionLock 可以通过Condition来实现任务之间的依赖

具体使用的案例可以看[这里](https://www.jianshu.com/p/ddbe44064ca4)
## NSRecursiveLock
NSRecursiveLock是一个递归锁, 他和 NSLock 的区别在于, NSRecursiveLock 可以在一个线程中重复加锁(单线程内任务是按顺序执行的, 不会出现资源竞争问题), NSRecursiveLock 会记录上锁和解锁的次数, 当二者平衡的时候, 才会释放锁, 其它线程才可以上锁成功.

```
@interface NSRecursiveLock : NSObject <NSLocking> {
@private
    void *_priv;
}

- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate *)limit;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end
```
递归锁也是通过 pthread_mutex_lock 函数来实现, 在函数内部会判断锁的类型, 如果显示是递归锁, 就允许递归调用, 仅仅将一个计数器加一. 锁的释放过程也是同理.
## NSCondition
NSCondition是一个条件锁, 是封装了一个互斥锁和条件变量, 锁上之后其它线程也能上锁, 而之后可以根据条件决定是否继续运行线程, 即线程是否要进入 waiting 状态. NSCondition 并不会像上文的那些锁一样, 先轮询, 而是直接进入 waiting 状态, 当其它线程中的该锁执行 signal 或者 broadcast 方法时, 线程被唤醒, 继续运行之后的方法.

```
@interface NSCondition : NSObject <NSLocking> {
@private
    void *_priv;
}

- (void)wait;//让当前线程处于等待状态
- (BOOL)waitUntilDate:(NSDate *)limit;
- (void)signal;
- (void)broadcast;//CPU发信号告诉线程不用在等待, 可以继续执行

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end
```
一般用于多线程同时访问, 修改同一个数据源, 保证在同一时间内数据源只被访问, 修改一次, 其他线程的命令需要在lock 外等待, 只到unlock, 才可访问.

NSCondition 的底层是通过条件变量(condition variable) pthread_cond_t 来实现的. 条件变量有点像信号量, 提供了线程阻塞与信号机制, 因此可以用来阻塞某个线程, 并等待某个数据就绪, 随后唤醒线程.
## @synchronized
这其实是一个 OC 层面的锁, 是一个互斥锁. 主要是通过牺牲性能换来语法上的简洁与可读. @synchronized 后面需要紧跟一个 OC 对象, 它实际上是把这个对象当做锁来使用. 这是通过一个哈希表来实现的, OC 在底层使用了一个互斥锁的数组(可以理解为锁池), 通过对对象去哈希值来得到对应的互斥锁.

```
//主线程中
NSObject *obj = [[NSObject alloc] init];
//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   @synchronized(obj){
       NSLog(@"线程1");
       sleep(10);
   }
});
//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);
   NSLog(@"进入线程2");
   @synchronized(obj){
       NSLog(@"线程2");
   }
});
```
执行结果如下:

```
2018-03-25 15:41:26.477502+0800 HCLock[10997:451231] 线程1
2018-03-25 15:41:27.477745+0800 HCLock[10997:451230] 进入线程2
2018-03-25 15:41:36.478718+0800 HCLock[10997:451230] 线程2
```
@synchronized指令使用的obj为该锁的唯一标识, 只有当标识相同时, 才为满足互斥. @synchronized也是会轮询, 需要加锁的对象是否可以使用.
@synchronized块会隐式的添加一个异常处理例程来保护代码, 该处理例程会在异常抛出的时候自动的释放互斥锁.
@sychronized(object){} 内部 object 被释放或被设为 nil 没有问题, 但如果 object 一开始就是 nil, 则失去了锁的功能. 不过虽然 nil 不行, 但 @synchronized([NSNull null]) 是完全可以的.
调用 @sychronized 的每个对象, Objective-C runtime 都会为其分配一个递归锁并存储在哈希表中. 
## dispatch_semaphore
dispatch_semaphore 是 GCD 用来同步的一种方式, 与他相关的只有三个函数, 一个是创建信号量, 一个是等待信号, 一个是发送信号.

```
dispatch_semaphore_create(long value);

dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout);

dispatch_semaphore_signal(dispatch_semaphore_t dsema);
```
dispatch_semaphore 和 NSCondition 类似，都是一种基于信号的同步方式，但 NSCondition 信号只能发送，不能保存（如果没有线程在等待，则发送的信号会失效）。而 dispatch_semaphore 能保存发送的信号。dispatch_semaphore 的核心是 dispatch_semaphore_t 类型的信号量。

dispatch_semaphore_create(1) 方法可以创建一个 dispatch_semaphore_t 类型的信号量，设定信号量的初始值为 1。注意，这里的传入的参数必须大于或等于 0，否则 dispatch_semaphore_create 会返回 NULL。

dispatch_semaphore_wait(signal, overTime); 方法会判断 signal 的信号值是否大于 0。大于 0 不会阻塞线程，消耗掉一个信号，执行后续任务。如果信号值为 0，该线程会和 NSCondition 一样直接进入 waiting 状态，等待其他线程发送信号唤醒线程去执行后续任务，或者当 overTime  时限到了，也会执行后续任务。

dispatch_semaphore_signal(signal); 发送信号，如果没有等待的线程接受信号，则使 signal 信号值加一（做到对信号的保存）。

从上面的实例代码可以看到，一个 dispatch_semaphore_wait(signal, overTime); 方法会去对应一个 dispatch_semaphore_signal(signal); 看起来像 NSLock 的 lock 和 unlock，其实可以这样理解，区别只在于有信号量这个参数，lock unlock 只能同一时间，一个线程访问被保护的临界区，而如果 dispatch_semaphore 的信号量初始值为 x ，则可以有 x 个线程同时访问被保护的临界区.
## OSSpinLock

```
#import <libkern/OSAtomic.h>

__block OSSpinLock theLock = OS_SPINLOCK_INIT;
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   OSSpinLockLock(&theLock);
   NSLog(@"线程1");
   sleep(10);
   OSSpinLockUnlock(&theLock);
   NSLog(@"线程1解锁成功");
});
    
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
   sleep(1);
   OSSpinLockLock(&theLock);
   NSLog(@"线程2");
   OSSpinLockUnlock(&theLock);
});
```
OSSpinLock 是一种自旋锁, 只有加锁, 解锁, 尝试加锁三个方法. 和 NSLock 不同的是 NSLock 请求加锁失败的话, 会先轮询, 但一秒过后便会使线程进入 waiting 状态, 等待唤醒. 而 OSSpinLock 会一直轮询, 等待时会消耗大量 CPU 资源, 不适用于较长时间的任务.

如果一个低优先级的线程获得锁并访问共享资源, 这时一个高优先级的线程也尝试获得这个锁, 它会处于 spin lock 的忙等状态从而占用大量 CPU. 此时低优先级线程无法与高优先级线程争夺 CPU 时间, 从而导致任务迟迟完不成、无法释放 lock. 因为上面的特性, 导致这个锁已经被废弃, 

具体看[这里](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
## pthread_mutex
pthread_mutex 表示互斥锁. 互斥锁的实现原理与信号量非常相似, 不是使用忙等, 而是阻塞线程并睡眠, 需要进行上下文切换. 常见的API有:

```
pthread_mutexattr_t attr;  
pthread_mutexattr_init(&attr);  
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);  // 定义锁的属性

pthread_mutex_t mutex;  
pthread_mutex_init(&mutex, &attr) // 创建锁

pthread_mutex_lock(&mutex); // 申请锁
pthread_mutex_unlock(&mutex); // 释放锁
```
一般情况下, 一个线程只能申请一次锁, 也只能在获得锁的情况下才能释放锁, 多次申请锁或释放未获得的锁都会导致崩溃. 假设在已经获得锁的情况下再次申请锁, 线程会因为等待锁的释放而进入睡眠状态, 因此就不可能再释放锁, 从而导致死锁.

```
- (void)example5 {
    pthread_mutex_init(&theLock, NULL);//初始化一个锁&theLock
    
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    pthread_mutex_init(&theLock, &attr);
    pthread_mutexattr_destroy(&attr);
    
    pthread_t thread;
    pthread_create(&thread, NULL, threadMethord, 5);
}

void *threadMethord(int value) {
    pthread_mutex_lock(&theLock);
    
    if (value > 0) {
        printf("Value:%i\n", value);
        sleep(1);
        threadMethord(value - 1);
    }
    pthread_mutex_unlock(&theLock);
    return 0;
}
```
`int pthread_mutex_init(pthread_mutex_t * __restrict, const pthread_mutexattr_t * __restrict);
`表示初始化一个锁, pthread_mutexattr_t表示互斥锁的类型, 传NULL表示默认类型, 一共有四种类型.

```
PTHREAD_MUTEX_NORMAL 缺省类型，也就是普通锁。当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后先进先出原则获得锁。

PTHREAD_MUTEX_ERRORCHECK 检错锁，如果同一个线程请求同一个锁，则返回 EDEADLK，否则与普通锁类型动作相同。这样就保证当不允许多次加锁时不会出现嵌套情况下的死锁。

PTHREAD_MUTEX_RECURSIVE 递归锁，允许同一个线程对同一个锁成功获得多次，并通过多次 unlock 解锁。

PTHREAD_MUTEX_DEFAULT 适应锁，动作最简单的锁类型，仅等待解锁后重新竞争，没有等待队列。
```
## 互斥锁与信号量的区别
信号量用在多线程多任务同步的, 一个线程完成了某一个动作就通过信号量告诉别的线程, 别的线程再进行某些动作. 信号量的作用域是进程间或线程间.
而互斥锁是用在多线程多任务互斥的, 一个线程占用了某一个资源, 那么别的线程就无法访问, 直到这个线程unlock, 其他的线程才开始可以利用这个资源. 比如对全局变量的访问, 有时要加锁, 操作完了再解锁. 有的时候锁和信号量会同时使用. 互斥锁的作用域是线程间.
信号量不一定是锁定某一个资源, 而是流程上的概念, 比如: 有A, B两个线程, B线程要等A线程完成某一任务以后再进行自己下面的步骤, 这个任务并不一定是锁定某一资源, 还可以是进行一些计算或者数据处理之类. 而线程互斥量则是"锁住某一资源"的概念, 在锁定期间内, 其他线程无法对被保护的数据进 行操作. 在有些情况下两者可以互换.
具体看[这里](https://blog.csdn.net/jenny8080/article/details/52094140)

------
参考资料:

1.[正确使用多线程同步锁@synchronized()](http://mrpeak.cn/blog/synchronized/)

2.[iOS 中几种常用的锁总结](https://www.jianshu.com/p/1e59f0970bf5)

3.[线程同步(互斥锁与信号量的作用与区别)](https://blog.csdn.net/jenny8080/article/details/52094140)

4.[iOS的线程安全与锁](http://www.cocoachina.com/ios/20171218/21570.html)

5.[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

6.[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

7.[iOS 常见知识点（三）：Lock](https://www.jianshu.com/p/ddbe44064ca4)

8.[深入理解 iOS 开发中的锁](https://bestswifter.com/ios-lock/)

