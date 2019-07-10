---
title: iOS开发总结系列-多线程
date: 2018-03-24 14:25:41
tags:
    - 基础知识
categories: 
    - 基础知识
---

这一篇主要总结开发中的多线程问题, 之前已经写过一篇多线程总结, 可以看[这里](http://hchong.net/2017/11/21/iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B/).

## GCD里面有哪几种Queue
1. 主队列 dispatch_main_queue(); 串行, 主要用来更新UI 
2. 全局队列 dispatch_global_queue(); 并行, 四个优先级：background, low, default, high 
3. 自定义队列 dispatch_queue_t queue; 可以自定义是并行DISPATCH_QUEUE_CONCURRENT 或者 串行DISPATCH_QUEUE_SERIAL.

## 若干个url异步加载多张图片，然后在都下载完成后合成一张整图
使用dispatch_group, 有两种使用方式

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, ^{ /*加载图片1 */ });
dispatch_group_async(group, queue, ^{ /*加载图片2 */ });
dispatch_group_async(group, queue, ^{ /*加载图片3 */ }); 
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 合并图片
});
```

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
## dispatch_barrier_async的作用
在并行队列中, 为了保持某些任务的顺序, 需要等待一些任务完成后才能继续进行, 使用 barrier 来等待之前任务完成, 避免数据竞争等问题. 假设一个并行队列中添加了五个异步任务, 虽然遵循FIFO的原则, 这五个任务的执行必定是无序的. 当我们在第三个任务之后加入dispatch_barrier_async任务, 那么这五个任务就被分割成两部分, 前三次无序执行, 然后执行dispatch_barrier_async任务, 然后再无序执行后两次任务. 注意此处的队列只能使用自定义的并行队列, 系统提供的全局队列不行.

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

## 为什么要废弃dispatch_get_current_queue
dispatch_get_current_queue可能造成死锁, 详细的原因看[这里](为什么dispatch_get_current_queue被废弃)
## 如何取消一个正在运行的线程
### GCD
1. 使用dispatch_block_cancel(iOS8之后)来取消, 但是必须使用 dispatch_block_create 创建 dispatch_block_t.

    ```
    - (void)gcdBlockCancel{
        dispatch_queue_t queue = dispatch_queue_create("com.gcdtest.www", DISPATCH_QUEUE_CONCURRENT);
        
        dispatch_block_t block1 = dispatch_block_create(0, ^{
            sleep(5);
            NSLog(@"block1 %@",[NSThread currentThread]);
        });
        
        dispatch_block_t block2 = dispatch_block_create(0, ^{
            NSLog(@"block2 %@",[NSThread currentThread]);
        });
        
        dispatch_block_t block3 = dispatch_block_create(0, ^{
            NSLog(@"block3 %@",[NSThread currentThread]);
        });
        
        dispatch_async(queue, block1);
        dispatch_async(queue, block2);
        dispatch_block_cancel(block3);
    }
    ```
    注意: dispatch_block_cancel只能取消尚未执行的任务, 对正在执行的任务不起作用.
2. 模仿NSOperation, 定义外部变量, 标记block是否需要取消

```
@property (nonatomic, assign) BOOL isCancel;

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    [self gcdCancel];
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        self.isCancel = YES;
    });
}

- (void)gcdCancel {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(queue, ^{
        for (int i = 0; i < 10000; i++) {
            NSLog(@"%d", i);
            if (self.isCancel) {
                return;
            }
        }
    });
}
```
不等循环结束, self.isCancel已经被标记为YES, 线程就已经return, 结束.
### NSOperation
NSOperation提供了cancel方法, 但是这也只是把状态标记为cancel, 已经执行的任务是还在继续跑的, 所以我们要要在Operation的声明周期函数中, 频繁的检查 .cancelled 状态, 只要 .cancelled == true我们就把operation的 finished 状态, 设置为true, 并且结束没有完成的任务.

我们要在`- start`, `- main`等任何声明周期函数中检查, 也可以通过KVO观察cancel的变化来做相应的操作. 

> In life cycle of operation, you should frequently check .cancelled whenever .cancelled == true, you should stop operation and set .finished == true
> In macOS 10.6 and later, if you call the cancel method on an operation that is in an operation queue and has unfinished dependent operations, those dependent operations are subsequently ignored. Because the operation is already cancelled, this behavior allows the queue to call the operation’s start method to remove the operation from the queue without calling its main method. If you call the cancel method on an operation that is not in a queue, the operation is immediately marked as being cancelled. In each case, marking the operation as ready or finished results in the generation of the appropriate KVO notifications.

## 死锁
死锁是指两个或两个以上的进程在执行过程中, 因争夺资源而造成的一种互相等待的现象, 若无外力作用, 它们都将无法推进下去.

死锁产生的主要原因:

* 系统资源不足
* 进程运行推进的顺序不合适
* 资源分配不当

产生死锁的四个必要条件:

* 互斥条件: 一个资源每次只能被一个进程使用. 这个无法预防, 恰恰是设备的固有属性, 我们还应该保护.
* 占有且等待: 一个进程因请求资源而阻塞时, 对已获得的资源保持不放. 这个我们可以通过要求进程一次性的请求所有需要的资源, 并且阻塞这个进程直到所有请求都同时满足来预防. 
* 不可强行占有: 进程已获得的资源, 在未使用完成之前, 不能强行剥夺. 如果占有某些资源的一个进程进行进一步资源请求时被拒绝, 则该进程必须释放它最初占有的资源, 或者抢占另外一条进程并要求释放资源这两种方式来预防.
* 循环等待条件: 若干进程之间形成一种头尾相接的循环等待资源关系. 我们可以通过定义资源类型的线性顺序来预防.

