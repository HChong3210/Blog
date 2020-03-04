---
title: iOS开发总结系列-性能优化
date: 2018-04-04 10:20:38
tags:
    - 基础知识
categories:
    - 基础知识
---

这里主要记录一下APP开发中, 经常遇到的一些性能问题, 以及优化的建议.

# 卡顿优化
通常来说, 计算机系统中 CPU、GPU、显示器是以上面这种方式协同工作的. CPU 计算好显示内容提交到 GPU, GPU 渲染完成后将渲染结果放入帧缓冲区, 随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据, 经过可能的数模转换传递给显示器显示.
## CPU主要责任

* 对象创建: 对象的创建会分配内存, 调整属性, 甚至还有读取文件等操作, 比较消耗 CPU 资源. 尽量用轻量的对象代替重量的对象, 可以对性能有所优化, 例如不需要触摸事件时我们可以使用CALayer代替UIView. 尽量推迟对象创建的时间, 并把对象的创建分散到多个任务中去, 如果对象可以复用, 并且复用的代价比释放, 创建新对象要小, 那么这类对象应当尽量放到一个缓存池里复用.
* 对象调整: UIView 的关于显示相关的属性(比如 frame/bounds/transform)等实际上都是 CALayer 属性映射来的, 所以对 UIView 的这些属性进行调整时, 消耗的资源要远大于一般的属性. 对此你在应用中, 应该尽量减少不必要的属性修改. 当视图层次调整时, UIView、CALayer 之间会出现很多方法调用与通知, 所以在优化性能时, 应该尽量避免调整视图层次、添加和移除视图.
* 对象销毁: 对象的销毁虽然消耗资源不多, 但累积起来也是不容忽视. 如果对象可以放到后台线程去释放, 那就挪到后台线程去. 把对象捕获到 block 中, 然后扔到后台队列去随便发送个消息以避免编译器警告, 就可以让对象在后台线程销毁了.
* 布局计算: 不论通过何种技术对视图进行布局, 其最终都会落到对 UIView.frame/bounds/center 等属性的调整上. 上面也说过, 对这些属性的调整非常消耗资源, 所以尽量提前计算好布局, 在需要时一次性调整好对应属性, 而不要多次、频繁的计算和调整这些属性.
* AutoLayout: 苹果本身提倡Autolayout, 但是 Autolayout 对于复杂视图来说常常会产生严重的性能问题. 随着视图数量的增长, Autolayout 带来的 CPU 消耗会呈指数级上升.
* 文本计算: 如果一个界面中包含大量文本(比如微博微信朋友圈等), 文本的宽高计算会占用很大一部分资源, 并且不可避免. 
    可以参考下 UILabel 内部的实现方式: 用 `-[NSAttributedString boundingRectWithSize:options:context:]` 来计算文本宽高, 用 `-[NSAttributedString drawWithRect:options:context:]` 来绘制文本. 尽管这两个方法性能不错, 但仍旧需要放到后台线程进行以避免阻塞主线程.
    如果你用 CoreText 绘制文本，那就可以先生成 CoreText 排版对象, 然后自己计算了, 并且 CoreText 对象还能保留以供稍后绘制使用.
* 文本渲染: 屏幕上能看到的所有文本内容控件, 包括 UIWebView, 在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的. 当显示大量文本时, CPU 的压力会非常大. 我们可以自定义文本控件, 用 TextKit 或最底层的 CoreText 对文本异步绘制.
    CoreText 对象创建好后, 能直接获取文本的宽高等信息, 避免了多次计算(调整 UILabel 大小时算一遍、UILabel 绘制时内部再算一遍);CoreText 对象占用内存较少, 可以缓存下来以备稍后多次渲染.
* 图片的解码: 当你用 UIImage 或 CGImageSource 的那几个方法创建图片时, 图片数据并不会立刻解码. 图片设置到 UIImageView 或者 CALayer.contents 中去, 并且 CALayer 被提交到 GPU 前, CGImage 中的数据才会得到解码. 这一步是发生在主线程的, 并且不可避免. 如果想要绕开这个机制, 常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中, 然后从 Bitmap 直接创建图片.
* 图像的绘制: 图像的绘制通常是指用那些以 CG 开头的方法把图像绘制到画布中, 然后从画布创建图片并显示这样一个过程. 这个最常见的地方就是 `[UIView drawRect:]` 里面. 由于 CoreGraphic 方法通常都是线程安全的, 所以图像的绘制可以很容易的放到后台线程进行.

## GPU主要责任:
相对于 CPU 来说, GPU 能干的事情比较单一: 接收提交的纹理(Texture)和顶点描述(三角形), 应用变换(transform), 混合并渲染, 然后输出到屏幕上. 通常你所能看到的内容, 主要也就是纹理(图片)和形状(三角模拟的矢量图形)两类.

* 纹理的渲染: 所有的 Bitmap, 包括图片、文本、栅格化的内容, 最终都要由内存提交到显存, 绑定为 GPU Texture. 当在较短时间显示大量图片时(比如 TableView 存在非常多的图片并且快速滑动时), CPU 占用率很低, GPU 占用非常高, 界面仍然会掉帧. 所以应该尽量减少在短时间内大量图片的显示, 尽可能将多张图片合成为一张进行显示.
    当图片过大, 超过 GPU 的最大纹理尺寸时, 图片需要先由 CPU 进行预处理, 这对 CPU 和 GPU 都会带来额外的资源消耗. 所以纹理尺寸都不应超过上限.
* 视图的混合: 当多个视图（或者说 CALayer）重叠在一起显示时, GPU 会首先把他们混合到一起. 如果视图结构过于复杂, 混合的过程也会消耗很多 GPU 资源. 为了减轻这种情况的 GPU 消耗, 应用应当尽量减少视图数量和层次, 并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成. 当然, 这也可以用上面的方法, 把多个视图预先渲染为一张图片来显示.
* 图形的生成: CALayer 的 border、圆角、阴影、遮罩(mask), CASharpLayer 的矢量图形显示, 通常会触发离屏渲染. 快速滑动时, 可以观察到 GPU 资源已经占满, 而 CPU 资源消耗很少. 我们可以尝试开启 CALayer.shouldRasterize 属性, 但这会把原本离屏渲染的操作转嫁到 CPU 上去. 对于只需要圆角的某些场合, 也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果. 最彻底的解决办法, 就是把需要显示的图形在后台线程绘制为图片, 避免使用圆角、阴影、遮罩等属性.

所以, 我们常见的性能优化技巧有以下几种: 预排版, 预渲染, 异步绘制, 全局并发控制, 异步加载图片.

# 编译优化
## 增加XCode执行的线程数
XCode默认使用与CPU核数相同的线程来进行编译, 但由于编译过程中的IO操作往往比CPU运算要多, 因此适当的提升线程数可以在一定程度上加快编译速度. 在终端输入`defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES`开启多线程, 更改线程数设置`defaults write com.apple.Xcode PBXNumberOfParallelBuildSubtasks 5`.

## 将Debug Information Format改为DWARF
将Target->Build Settings中, 找到Debug Information Format这一项, 将Debug时的DWARF with dSYM file改为DWARF.

这一项设置的是是否将调试信息加入到可执行文件中, 改为DWARF后, 如果程序崩溃, 将无法输出崩溃位置对应的函数堆栈, 但由于Debug模式下可以在XCode中查看调试信息, 所以改为DWARF影响并不大. 这一项更改完之后, 可以大幅提升编译速度. 
将Debug Information Format改为DWARF之后, 会导致在Debug窗口无法查看相关类类型的成员变量的值. 当需要查看这些值时, 可以将Debug Information Format改回DWARF with dSYM file, clean(必须)之后重新编译.

## 将Build Active Architecture Only改为Yes
将Target->Build Settings中, 找到Build Active Architecture Only这一项, 将Debug时的 NO 改为 YES.

需要注意的是, 此选项在Release模式下必须为NO, 否则发布的ipa在部分设备上将不能运行. 这一项更改完之后, 可以显著提高编译速度.

## 设计编译优化等级
不要再项目中或者静态库中使用-O4, 因为这会让Clang链接Link Time Optimizations (LTO)使得编译更慢, 通常使用-O3. 在设置编译优化之后, XCode断点和调试信息会不正常, 所以一般静态库或者其他Target这样设置.

## 将常用的代码及文件打包成静态库
我们用Cocoapods来管理第三方包, 我们可以将第三方包打包成静态库, 也可以提升编译速度. 也可以将第三方包打包成二进制文件, 但是这样不方便调试.

## 添加预编译文件
使用.pch文件, 把常用的头文件放到预编译文件里面.

# 启动优化
App总启动时间 = t1(main()之前的加载时间) + t2(main()之后的加载时间). 
t1 = 系统dylib(动态链接库)和自身App可执行文件的加载; 
t2 = main方法执行之后到AppDelegate类中的`- (BOOL)Application:(UIApplication *)Application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`方法执行结束前这段时间, 主要是构建第一个界面, 并完成渲染展示.

## main()调用之前的加载过程

1. 系统首先加载可执行文件. 自身App的所有.o文件的集合.
2. 加载动态链接库dyld, dyld是一个专门用来加载动态链接库的库.
3. dyld从可执行文件的依赖开始, 递归加载所有的依赖动态链接库. 动态链接库包括: iOS 中用到的所有系统 framework, 加载OC runtime方法的libobjc, 系统级别的libSystem, 例如libdispatch(GCD)和libsystem_blocks (Block).

所以对于main()调用之前的耗时我们可以优化的点如下:

1. 减少不必要的framework, 因为动态链接比较耗时.
2. check framework应当设为optional和required, 如果该framework在当前App支持的所有iOS系统版本都存在, 那么就设为required, 否则就设为optional, 因为optional会有些额外的检查.
3. 合并或者删减一些OC类, 关于清理项目中没用到的类.
4. 删减一些无用的静态变量.
5. 删减没有被调用到或者已经废弃的方法.
6. 将不必须在+load方法中做的事情延迟到+initialize中.
7. 尽量不要用C++虚函数(创建虚函数表有开销), C++静态对象.

## main()调用之后的加载过程
在main()被调用之后, App的主要工作就是初始化必要的服务, 显示首页内容. 所以主要耗时操作在执行main()函数的耗时, 执行applicationWillFinishLaunching的耗时, rootViewController及其childViewController的加载、view及其subviews的加载.

# 瘦身优化
## 资源瘦身
资源瘦身主要是去掉无用资源和压缩资源, 资源包括图片, 音视频文件, 配置文件以及多语言wording. 资源压缩主要对png进行无损压缩.

## 编译选项优化

* Optimization Level 使用Fastest, Smalllest. 该选项对安装包大小影响几无，但可以提高app的性能
* Strip Linked Product 设置为YES, 需要注意的是Strip Linked Product也受到Deployment Postprocessing设置选项的影响. 在Build Settings中, 我们可以看到， Strip Linked Product是在Deployment这栏中的, 而Deployment Postprocessing相当于是Deployment的总开关. 记得把Deployment Postprocessing也设置为YES.
* Symbols Hidden by Default设置为YES
* Make Strings Read-Only 设置为YES

## 二进制安装包

二进制包是由各种代码文件, 静态库 动态库 经过编译后生成的可执行文件.

* XCode开启编译选项Write Link Map File XCode -> target -> Build Settings -> 搜map -> 把Write Link Map File选项设为yes, 并指定好linkMap的存储位置.
* 编译后到编译目录里找到该txt文件, 文件名和路径就是上述的Path to Link Map File. 
~/Library/Developer/Xcode/DerivedData/XXX-eumsvrzbvgfofvbfsoqokmjprvuh/Build/Intermediates/XXX.build/Debug-iphoneos/XXX.build/. 这个LinkMap里展示了整个可执行文件的全貌, 列出了编译后的每一个.o目标文件的信息(包括静态链接库.a里的), 以及每一个目标文件的代码段, 数据段存储详情.
* 到[https://github.com/huanxsd/LinkMap](https://github.com/huanxsd/LinkMap)下载这个mac工程, 然后运行, 对文件进行分析.
* 通过对上面的文件进行分析, 就知道每个类在最终的可执行文件中占据的大小. 然后有针对性的进行优化就可以了.

## 删除一些无用文件

查找无用selector, 无用OC类, 扫描重复代码.

----
参考资料:
1.[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

2.[今日头条iOS客户端启动速度优化](https://techblog.toutiao.com/2017/01/17/iosspeed/#more)

3.[iOS-Performance-Optimization](https://github.com/skyming/iOS-Performance-Optimization)

4.[Archive、ipa 和 App 包瘦身](https://juejin.im/entry/5d61559e5188253cf525cc2c)


