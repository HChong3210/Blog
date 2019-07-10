---
title: iOS中常见的内存泄漏
date: 2018-04-04 10:21:18
tags:
    - 基础知识
categories:
    - 基础知识
---

内存泄漏是开发中常见的一种问题, 下面我们就开发中常见的容易出现内存泄漏的场景做一个总结和分析.

## Block下的循环引用
在ARC下基本上不用我们内存管理释放, block中导致的内存泄漏常常就是因为强引用互相之间持有而发生了循环引用无法释放. AFNetWorking上的经典代码, 防止循环引用.

```
//创建__weak弱引用，防止强引用互相持有
__weak __typeof(self)weakSelf = self;
AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
    //创建局部__strong强引用，防止多线程情况下weakSelf被析构
     __strong __typeof(weakSelf)strongSelf = weakSelf;
    strongSelf.networkReachabilityStatus = status;
    if (strongSelf.networkReachabilityStatusBlock) {
         strongSelf.networkReachabilityStatusBlock(status);
    }
};
```
__weak 本身是可以避免循环引用的问题的, 但是其会导致外部对象释放了之后, block 内部也访问不到这个对象的问题, 我们可以通过在 block 内部声明一个 __strong 的变量来指向 weakObj, 使外部对象既能在 block 内部保持住, 又能避免循环引用的问题.

## delegate循环引用问题
delegate循环引用问题比较基础, 只需注意将代理属性修饰为weak即可. 例如:
ViewController的self.view持有了UITableView. 所以UITableView的delegate和dataSource就不能持有ViewController. 所以要使用weak来修饰.

## NSTimer的循环引用
我们一般这么创建NSTimer

```
- (void)viewDidLoad {
    NSTimer *timer = [[NSTimer alloc] initWithFireDate:[NSDate date] interval:1 target:self selector:@selector(timerFire) userInfo:nil repeats:YES];//timer强引用self
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];//RunLoop强引用timer
    self.timer = timer;//self强引用timer
}

- (void)timerFire {
    NSLog(@"timer fire");
}
```
但是当我们把timer添加到RunLoop的时候, 会被RunLoop强引用, timer会对目标对象进行强引用. 这个时候已经出现了循环引用. 这时候我们就不能在 `-dealloc` 里加 `-invalidate` 的方法, 因为他们之间相互引用, 永远不会走`-dealloc`方法. 我们要做的就是打破这个循环.

```
__weak typeof(self) weakSelf = self;
NSTimer *timer = [[NSTimer alloc] initWithFireDate:[NSDate date] interval:1 target:weakSelf selector:@selector(timerFire) userInfo:nil repeats:YES];
```

## 非OC对象内存处理

对于一些非OC对象, 我们在使用完毕后, 一定要记得手动释放内存, 例如下面常用的调节图片亮度的代码:

```
CIImage *beginImage = [[CIImage alloc]initWithImage:[UIImage imageNamed:@"yourname.jpg"]];
CIFilter *filter = [CIFilter filterWithName:@"CIColorControls"];
[filter setValue:beginImage forKey:kCIInputImageKey];
[filter setValue:[NSNumber numberWithFloat:.5] forKey:@"inputBrightness"];//亮度-1~1
CIImage *outputImage = [filter outputImage];
//GPU优化
EAGLContext * eaglContext = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];
eaglContext.multiThreaded = YES;
CIContext *context = [CIContext contextWithEAGLContext:eaglContext];
[EAGLContext setCurrentContext:eaglContext];

CGImageRef ref = [context createCGImage:outputImage fromRect:outputImage.extent];
UIImage *endImg = [UIImage imageWithCGImage:ref];
_imageView.image = endImg;
CGImageRelease(ref);//非OC对象需要手动内存释放
```
其他的对于CoreFoundation框架下的某些对象或变量需要手动释放, C语言代码中的malloc等需要对应free等都需要注意.

## 地图类处理
若项目中使用地图相关类, 一定要检测内存情况, 因为地图是比较耗费App内存的, 因此在根据文档实现某地图相关功能的同时, 我们需要注意内存的正确释放, 大体需要注意的有需在使用完毕时将地图, 代理等滞空为nil, 注意地图中标注(大头针)的复用, 并且在使用完毕时清空标注数组等.

## 大次数循环内存暴涨问题
循环内产生大量的临时对象, 直至循环结束才释放, 可能导致内存泄漏, 解决方法为在循环中创建自己的autoReleasePool, 及时释放占用内存大的临时变量, 减少内存占用峰值.

```
for (int i = 0; i < 100000; i++) {
   @autoreleasepool {
       NSString *string = @"Abc";
       string = [string lowercaseString];
       string = [string stringByAppendingString:@"xyz"];
       NSLog(@"%@", string);
   }
}
```

------
参考资料:
1.[NSTimer定时器](https://www.cnblogs.com/mddblog/p/6517377.html)

2.[iOS中的NSTimer](http://www.cocoachina.com/ios/20150710/12444.html)

