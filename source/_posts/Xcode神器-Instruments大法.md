---
title: Xcode神器-Instruments大法
date: 2017-04-13 18:13:59
tags:
    - 基础知识
    - 调试
categories:
    - 基础知识
---

# Xcode调试神器-Instruments大法
用户体验, 是每个App开发很重要的点, 是每个程序猿都应该时刻被想到的点. 好的用户体验就必然要有好的性能. Xcode给我们提供了强大的性能分析工具Instruments.

打开Xcode, 选择Xcode -> Open Developer Tool打开如下界面
![Instruments首界面](http://ww1.sinaimg.cn/large/006tNc79gy1ffevnub9j1j316s0q2wfd.jpg)
这里我们主要介绍三个常用的工具Core Animation(检测帧率), Leaks(内存泄漏), Time Profiler(检查耗时操作)
## Core Animation
Core Animation主要用来评估屏幕渲染时的帧率, 帧率一般来说越接近60就越流畅, 当低于40时, 就会感觉到明显的卡顿.
![帧率](http://ww1.sinaimg.cn/large/006tNbRwgy1ffffmifexvj310w0nxwfn.jpg)
在屏幕的下方有一个Debug Options按钮, 点击展开菜单中可以选择一些更加具体的UI性能的检测, 
![选项](http://ww3.sinaimg.cn/large/006tNbRwgy1ffffp2z37cj30xk0i474v.jpg)
我们可以使用这些选项，来监测更加具体的图形性能, 具体参考如下:

* `Color Blended Layers`，这个选项选项基于渲染程度对屏幕中的混合区域进行绿到红的高亮显示，越红表示性能越差，会对帧率等指标造成较大的影响。红色通常是由于多个半透明图层叠加引起。
* `Color Hits Green and Misses Red`，当 UIView.layer.shouldRasterize = YES 时，耗时的图片绘制会被缓存，并当做一个简单的扁平图片来呈现。这时候，如果页面的其他区块(比如 UITableViewCell 的复用)使用缓存直接命中，就显示绿色，反之，如果不命中，这时就显示红色。红色越多，性能越差。因为栅格化生成缓存的过程是有开销的，如果缓存能被大量命中和有效使用，则总体上会降低开销，反之则意味着要频繁生成新的缓存，这会让性能问题雪上加霜。
* `Color Copied Images`，对于 GPU 不支持的色彩格式的图片只能由 CPU 来处理，把这样的图片标为蓝色。蓝色越多，性能越差。因为，我们不希望在滚动视图的时候，由 CPU 来处理图片，这样可能会对主线程造成阻塞。
* `Color Non-Standard Surface Formats`, 不标准的表面颜色格式.
* `Color Immediately`，通常 Core Animation Instruments 以每毫秒 10 次的频率更新图层调试颜色。对某些效果来说，这显然太慢了。这个选项就可以用来设置每帧都更新（可能会影响到渲染性能，而且会导致帧率测量不准，所以不要一直都设置它）。
* `Color Misaligned Images`，这个选项检查了图片是否被缩放，以及像素是否对齐。被放缩的图片会被标记为黄色，像素不对齐则会标注为紫色。黄色、紫色越多，性能越差。
* `Color Offscreen-Rendered Yellow`，这个选项会把那些离屏渲染的图层显示为黄色。黄色越多，性能越差。这些显示为黄色的图层很可能需要用 shadowPath 或者 shouldRasterize 来优化。
* `Color Compositing Fast Path Blue`，这个选项会把任何直接使用 OpenGL 绘制的图层显示为蓝色。蓝色越多，性能越好。如果仅仅使用 UIKit 或者 Core Animation 的 API，那么不会有任何效果。如果使用 GLKView 或者 CAEAGLLayer，那如果不显示蓝色块的话就意味着你正在强制 CPU 渲染额外的纹理，而不是绘制到屏幕。
* `Flash Updated Regions`，这个选项会把重绘的内容显示为黄色。不该出现的黄色越多，性能越差。通常我们希望只是更新的部分被标记完黄色。

使用时要注意Xcode和手机系统的版本号匹配, 否则会出现设备off line 无法被选中的情况.
## Leaks
关于内存方面的监控, 有`Leaks`用来检测内存泄漏, `Zombies`用来检测僵尸对象. 关于内存泄漏常见的几种情况, [这里](http://www.jianshu.com/p/d465831aebbf)讲的特别清晰和全面.
## Time Profiler
这个主要是用来统计各个方法消耗的时间.
![Time Profiler](http://ww4.sinaimg.cn/large/006tNbRwgy1fffhjy18vij31kw0zi44c.jpg)
如图所示, 展示各个方法占比和消耗的时间, 以ms为单位. 右侧菜单栏一般会显示最耗时的一些操作, 如果一般前面的小图片是黑色的话, 那就说明这部分代码, 占用了大量的系统时间, 是需要迫切优化的.

在下方的call Tree中有一些选项菜单, 选择不同的菜单, 可以查看不同的状态, 具体如下:

* `Separate byt Thread`（建议选择）：通过线程分类来查看那些纯种占用CPU最多。
* `Invert Call Tree`（不建议选择）：调用树倒返过来，将习惯性的从根向下一级一级的显示，如选上就会返过来从最底层调用向一级一级的显示。如果想要查看那个方法调用为最深时使用会更方便些。
* `Hide Missing Symbols`（建议选择）：隐藏丢失的符号，比如应用或者系统的dSYM文件找不到的话，在详情面板上是看不到方法名的，只能看一些读不明的十六进值，所以对我们来说是没有意义的，去掉了会使阅读更清楚些。
* `Hide System Libraries`（建议选择）：选上它只会展示与应用有关的符号信息，一般情况下我们只关心自己写的代码所需的耗时，而不关心系统库的CPU耗时。
* `Flatten Recursion`（一般不选）：选上它会将调用栈里递归函数作为一个入口。
* `Top Functions`（可选）：选上它会将最耗时的函数降序排列，而这种耗时是累加的，比如A调用了B，那么A的耗时数是会包含B的耗时数。

 
## 其他问题
在调试过程中也遇到一些问题, 记录下来.

### 设备灰色不可选
解决方案：重启
我最终的解决步骤：
1.拔掉iPhone的USB线，重启iPhone
2.关闭Xcode和Instruments
3.重新连接iPhone到Mac上
4.重启Xcode并启动Profile
5.成功

参考这个[帖子](http://stackoverflow.com/questions/32878283/unable-to-profile-app-on-device-with-ios-9-0-1-using-xcode-7-7-0-1-or-7-1-beta).
### Time Profiler无法定位到代码
Time Profliter 都是地址符号，往深里也一直是地址符号，根本没法判断是哪些代码的执行时间, 无法定位代码

解决方法:
1. Project->Build Settings->Debug Information Format 选择DWARF with dSYM File
2. Profile要在debug模式下运行, BuildConfiguration要选择debug.

-------

参考资料:

1.[iOS 性能优化：Instruments 工具的救命三招](https://blog.leancloud.cn/2835/)

2.[Instruments性能优化-Core Animation](http://www.jianshu.com/p/439e158b44de)

3.[使用 Instruments 做 iOS 程序性能调试](http://www.samirchen.com/use-instruments/)

4.[关于内存泄漏，还有哪些是你不知道的？](http://www.jianshu.com/p/d465831aebbf)

5.[使用Instruments定位iOS应用的Memory Leaks](http://www.jianshu.com/p/c0aa12d91f05)

6.[instrument Time Profiler总结](http://www.jianshu.com/p/21d29be26479)

7.[Unable to profile app on device with iOS 9.0.1 using Xcode 7, 7.0.1 or 7.1 beta](http://stackoverflow.com/questions/32878283/unable-to-profile-app-on-device-with-ios-9-0-1-using-xcode-7-7-0-1-or-7-1-beta)

