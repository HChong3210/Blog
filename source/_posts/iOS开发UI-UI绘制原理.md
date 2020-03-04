---
title: iOS开发UI-UI绘制原理
date: 2019-05-11 21:25:09
tags:
    - 基础知识
categories:
    - iOS开发-UI
---

本文将从下面几个方面来阐述iOS上面的UI绘制过程, 以及UI性能优化的技巧.

# 1 渲染过程中硬件相关
这篇文章讲解的非常详细[渲染过程中硬件](http://chuquan.me/2018/08/26/graphics-rending-principle-gpu/), 下面就只大概讲解一下常用的概念.

## 1.1 CPU & GPU
可视化应用程序都是由 CPU 和 GPU 协作执行的. CPU 内部采用的是流水线结构, 使其拥有一定程度的并行计算能力. GPU 是一种可进行绘图运算工作的专用微处理器, 是一个被高度优化设计的用来并发计算的硬件单元. 硬件设计的模式差异就决定了两者在图像显示中的不同定位. CPU擅长分支预测等复杂操作, GPU擅长对大量数据进行简单操作. 一个是复杂的劳动, 一个是大量并行的工作. 其实GPU可以看作是一种专用的CPU, 专为单指令在大块数据上工作而设计, 这些数据都是进行相同的操作.

关于GPU和CPU, 在知乎上看到一个[很形象的比喻](https://www.zhihu.com/question/22219245/answer/365354011):
> CPU的核心是大学生，4核就是4个大学生GPU的核心是小学生，上千个就处理器就是上千个小学生他们一起参加一场考试，试卷是一百万道四则运算和四道高数. 两个小时过后大学生奔溃了，这一百万道四则运算太特么多了再看小学生全都懵逼了，四则运算都做完了，剩下的数学题里面怎么全是字母 ？？

GPU 增加了处理器指令和存储器硬件, 以支持通用编程语言, 并创立了一种编程环境, 从而允许使用熟悉的语言（包括 C/C++）对 GPU 进行编程. GPU 及其相关驱动实现了图形处理中的 OpenGL 和 DirectX 模型, 从而允许开发者能够轻易地操作硬件.

图像想显示到屏幕上使人肉眼可见都需借助像素的力量. 简单地说, 每个像素由红，绿，蓝三种颜色组成, 它们密集的排布在手机屏幕上, 将任何图形通过不同的色值表现出来. 计算机显示的流程大致可以描述为将图像转化为一系列像素点的排列然后打印在屏幕上. 计算机将存储在内存中的形状转换成实际绘制在屏幕上的对应的过程称为渲染. 渲染过程中最常用的技术就是光栅化, 光栅化就是从矢量的点线面的描述, 变成像素的描述.

## 1.2 GPU 图形渲染流水线
GPU 图形渲染流水线的主要工作可以被划分为两个部分: 

1. 把 3D 坐标转换为 2D 坐标
2. 把 2D 坐标转变为实际的有颜色的像素

GPU 图形渲染流水线的具体实现可分为六个阶段，如下图所示

* 顶点着色器（Vertex Shader）
* 形状装配（Shape Assembly），又称 图元装配
* 几何着色器（Geometry Shader）
* 光栅化（Rasterization）
* 片段着色器（Fragment Shader）
* 测试与混合（Tests and Blending）
[OpenGL渲染示意图](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/daily-images/opengl-graphics-pipeline.png)

## 1.3 CPU-GPU 异构系统
实际应用中 CPU 和 GPU 通过下面两种常见的 CPU-GPU 异构架构来协同工作
[常见的 CPU-GPU 异构架构](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/cpu-gpu-architecture.png)

左图是分离式的结构，CPU 和 GPU 拥有各自的存储系统，两者通过 PCI-e 总线进行连接。这种结构的缺点在于 PCI-e 相对于两者具有低带宽和高延迟，数据的传输成了其中的性能瓶颈。目前使用非常广泛，如PC、智能手机等。

右图是耦合式的结构，CPU 和 GPU 共享内存和缓存。AMD 的 APU 采用的就是这种结构，目前主要使用在游戏主机中，如 PS4。

在存储管理方面，分离式结构中 CPU 和 GPU 各自拥有独立的内存，两者共享一套虚拟地址空间，必要时会进行内存拷贝。对于耦合式结构，GPU 没有独立的内存，与 GPU 共享系统内存，由 MMU 进行存储管理。

图形应用程序调用 OpenGL 或 Direct3D API 功能，将 GPU 作为协处理器使用。API 通过面向特殊 GPU 优化的图形设备驱动向 GPU 发送命令、程序、数据。

## 1.4 CPU-GPU 工作流
CPU-GPU 异构系统的工作流, 当 CPU 遇到图像处理的需求时，会调用 GPU 进行处理，主要流程可以分为以下四步: 

1. 将主存的处理数据复制到显存中
2. CPU 指令驱动 GPU
3. GPU 中的每个运算单元并行处理
4. GPU 将显存结果传回主存

# 2 屏幕显示图像原理
## 2.1 屏幕图像显示原理
图像想显示到屏幕上使人肉眼可见都需借助像素的力量. 简单地说, 每个像素由红，绿，蓝三种颜色组成, 它们密集的排布在手机屏幕上, 将任何图形通过不同的色值表现出来. 计算机显示的流程大致可以描述为将图像转化为一系列像素点的排列然后打印在屏幕上.

介绍屏幕图像显示的原理，需要先从 CRT 显示器原理说起，如下图所示。CRT 的电子枪从上到下逐行扫描，扫描完成后显示器就呈现一帧画面。然后电子枪回到初始位置进行下一次扫描。为了同步显示器的显示过程和系统的视频控制器，显示器会用硬件时钟产生一系列的定时信号。当电子枪换行进行扫描时，显示器会发出一个水平同步信号（horizonal synchronization），简称 HSync；而当一帧画面绘制完成后，电子枪回复到原位，准备画下一帧前，显示器会发出一个垂直同步信号（vertical synchronization），简称 VSync。显示器通常以固定频率进行刷新，这个刷新率就是 VSync 信号产生的频率。虽然现在的显示器基本都是液晶显示屏了，但其原理基本一致, 如下图所示:
![扫描图](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-screen-scan.png)

## 2.2 CPU、GPU、显示器工作方式
CPU 计算好显示内容提交至 GPU，GPU 渲染完成后将渲染结果存入帧缓冲区，视频控制器会按照 VSync 信号逐帧读取帧缓冲区的数据，经过数据转换后最终由显示器进行显示. 下图所示为常见的 CPU、GPU、显示器工作方式:
![协同工作图](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-renderIng-gpu-internal-structure.png)

## 2.3 帧缓冲 & 垂直同步
最简单的情况下，帧缓冲区只有一个。此时，帧缓冲区的读取和刷新都都会有比较大的效率问题。为了解决效率问题，GPU 通常会引入两个缓冲区，即 双缓冲机制。在这种情况下，GPU 会预先渲染一帧放入一个缓冲区中，用于视频控制器的读取。当下一帧渲染完毕后，GPU 会直接把视频控制器的指针指向第二个缓冲器。
双缓冲虽然能解决效率问题，但会引入一个新的问题。当视频控制器还未读取完成时，即屏幕内容刚显示一半时，GPU 将新的一帧内容提交到帧缓冲区并把两个缓冲区进行交换后，视频控制器就会把新的一帧数据的下半段显示到屏幕上，造成画面撕裂现象，如下图：
![画面撕裂](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ios-vsync-off.jpg)
为了解决这个问题，GPU 通常有一个机制叫做垂直同步（简写也是 V-Sync），当开启垂直同步后，GPU 会等待显示器的 VSync 信号发出后，才进行新的一帧渲染和缓冲区更新。这样能解决画面撕裂现象，也增加了画面流畅度，但需要消费更多的计算资源，也会带来部分延迟。

iOS 设备会始终使用双缓存，并开启垂直同步。而安卓设备直到 4.1 版本，Google 才开始引入这种机制，目前安卓系统是三缓存+垂直同步。

# 3 iOS图像渲染原理
这里主要介绍 iOS 开发过程中图形渲染原理. 首先介绍几个概念.
> UIKit Framework 正如Apple官方文档对UIKit Framework的介绍，它主要提供了：界面呈现能力、事件响应能力、驱动RunLoop运行和与系统内核通信的数据。简单来说就是：主要负责界面展示、事件响应以及是RunLoop的需求方。UIView当然是属于UIKit Framework。

> QuartzCore Framework 与 CoreAnimation 正如Apple官方文档对Quarz Core Framework的介绍，它提供了图形处理和视频图像处理的能力。简单来说就是：QuartzCore Framework负责把图形图像最终显示到屏幕上，这里我觉得应该是特指CoreAnimation。不要从字面上去理解CoreAnimation的职责，因为它的核心工作不单是负责动画的创建和执行，它还负责把我们用代码构建的界面显示到屏幕上，实际上是CoreAnimation通过OpenGLES做的。（别急，虽然这里面的机理很复杂，但是在后面会提到）。CALayer是属于QuarzCore Framework下的CoreAnimation

> CoreGraphics Framework 如Apple官方文档对Core Graphics Framework的介绍。CoreGraphics Framework一个基于C库函数的高级绘画引擎，它提供了非常强大的轻量级2D渲染能力。可以使用CoreGraphics处理基于path的绘图工作(如，CGPath)、变形操作(如，CGAffineTransform)、颜色管理(如，CGColor)、离屏渲染(如，CGBitmapContextCreateImage)、渲染模式(patterns)、渐变(gradients)、阴影效果、图形数据管理、图形创建、蒙版以及PDF文档的创建、显示和解析。 千万别和QuartzCore混淆，CoreGraphics负责创建显示到屏幕上的数据模型，QuartzCore(CoreAnimation –> OpenGLES)负责把CoreGraphics创建的数据模型真正显示到屏幕上。 CG打头的类都是属于CoreGraphics Framework

> 以上三者的关系 三者的关系是通过界面展示以及动画的创建、执行关联起来的，所以它们之间是协作而不是从属的关系。在接下来的部分我会从几个方面来阐述以上几个Framework是怎样游游走与UIView与CALayer之间的

App 使用 Core Graphics、Core Animation、Core Image 等框架来绘制可视化内容，这些软件框架相互之间也有着依赖关系。这些框架都需要通过 OpenGL 来调用 GPU 进行绘制，最终将内容显示到屏幕之上.
![图形渲染](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ios-rendering-framework-relationship.png)

## 3.1 UIView & CALayer
二者的区别和联系在[这里](https://www.jianshu.com/p/079e5cf0f014)介绍的很详细, 摘要说一下. 

> UIView是继承自UIResponder的，UIView是可以响应事件的，而CALayer不能
> UIView负责处理用户交互，负责绘制内容的则是它持有的那个CALayer，我们访问和设置UIView的这些负责显示的属性实际上访问和设置的都是这个CALayer对应的属性，UIView只是将这些操作封装起来. 一个 Layer 的 frame 是由它的 anchorPoint,position,bounds,和 transform 共同决定的，而一个 View 的 frame 只是简单的返回 Layer的 frame，同样 View 的 center和 bounds 也是返回 Layer 的一些属性.
> 一个UIView只有一个相关联的CALayer（自动创建），同时它也可以支持添加无数多个子CALayer
> UIView主要是对显示内容的管理而 CALayer 主要侧重显示内容的绘制. 在 View显示的时候，UIView 做为 Layer 的 CALayerDelegate,View 的显示内容由内部的 CALayer 的 display
> CALayer 是默认修改属性支持隐式动画的，在给 UIView 的 Layer 做动画的时候，View 作为 Layer 的代理，Layer 通过 actionForLayer:forKey:向 View请求相应的 action(动画行为)
> layer 内部维护着三分 layer tree,分别是 presentLayer Tree(动画树),modeLayer Tree(模型树), Render Tree (渲染树),在做 iOS动画的时候，我们修改动画的属性，在动画的其实是 Layer 的 presentLayer的属性值,而最终展示在界面上的其实是提供 View的modelLayer


## 3.2 UIView与CALayer的界面渲染
关于界面的渲染流程, [这里](http://blog.handy.wang/blog/2015/10/03/uiviewyu-calayerxie-zuo-xuan-ran-jie-mian-de-guo-cheng/)讲解的很详细.

1. 通过在loadView过程中debug子view的drawRect:方法得知：RunLoop处于kCFRunLoopBeforeWaiting状态时会回调CoreAnimation中监听kCFRunLoopBeforeWaiting状态的RunLoopObserver，从而通过RunLoopObserver来进一步调用CoreAnimation内部的CA::Transaction::commit() ();方法，进而一步一步地调用到drawRect方法
2. 通过在VC里给一个按钮添加点击事件，并在事件对应的selector中修改子view的背景色，debug子view的drawRect:方法得知：RunLoop被iOS系统传递来的点击事件唤醒并由source1处理(__IOHIDEventSystemClientQueueCallback)，并且在下一个runloop里由source0转发给UIApplication(_UIApplicationHandleEventQueue)，从而能过source0里的事件队列来调用CoreAnimation内部的CA::Transaction::commit() ();方法，进而一步一步的调用drawRect方法
3. 上面两种情况都是触发CoreAnimation的CA::Transaction::commit() ();方法来达到触发CALayer/UIView的渲染，所以这个CA::Transaction机制很关键
4. CA::Transaction已经进入到了Quarz Core的内部(Core Animation)，即调用CA::Transaction::commit() ();来创建CATrasaction，然后进一步调用`-[CALayer drawInContext:] ()`
5. 在drawRect:方法里可以通过CoreGraphics函数或UIKit中对CoreGraphics封装的方法进行画图操作，这些画图的操作内容都是以Off-Screen离屏(广义的离屏，因为没有在GPU中进行)方式进行画图。在这里可以了解离屏绘图及CPU/GPU的工作
6. 无论是有UIView参与的或是直接采用CALayer渲染的操作都会体现在CALayer上(在没有CoreGraphics参与的情况下，UIView或CALayer本身也有一些在业务层面需要显示的内容，所以这里说的“体现在CALayer上”，是泛指UIViewr的子视图或CALaye的子图层以及CoreGraphics参与的内容)。
7. CoreAnimation(CALayer)把它的内容转成位图(纹理)，然后通过OpenGLES把位图内容传送到GPU的帧缓冲区
8.  等到由iOS显示屏时钟信号驱动的VSync信号来临时，则把GPU帧缓冲区里的内容显示到iOS显示屏上。在这里的iOS 渲染过程一节可以了解得更详细。

## 3.3 Core Animation 流水线
app 本身并不负责渲染，渲染则是由一个独立的进程负责，即 Render Server 进程. App 通过 IPC 将渲染任务及相关数据提交给 Render Server。Render Server 处理完数据后，再传递至 GPU。最后由 GPU 调用 iOS 的图像设备进行显示。

Core Animation 流水线的详细过程如下：

* 首先，由 app 处理事件（Handle Events），如：用户的点击操作，在此过程中 app 可能需要更新 视图树，相应地，图层树 也会被更新。
* 其次，app 通过 CPU 完成对显示内容的计算，如：视图的创建、布局计算、图片解码、文本绘制等。在完成对显示内容的计算之后，app 对图层进行打包，并在下一次 RunLoop 时将其发送至 Render Server，即完成了一次 Commit Transaction 操作。
* Render Server 主要执行 Open GL、Core Graphics 相关程序，并调用 GPU
* GPU 则在物理层上完成了对图像的渲染。
* 最终，GPU 通过 Frame Buffer、视频控制器等相关部件，将图像显示在屏幕上

在 Core Animation 流水线中，app 调用 Render Server 前的最后一步 Commit Transaction 其实可以细分为 4 个步骤：

* Layout: Layout 阶段主要进行视图构建，包括：LayoutSubviews 方法的重载，addSubview: 方法填充子视图等
* Display: Display 阶段主要进行视图绘制，这里仅仅是设置最要成像的图元数据。重载视图的 drawRect: 方法可以自定义 UIView 的显示，其原理是在 drawRect: 方法内部绘制寄宿图，该过程使用 CPU 和内存。
* Prepare: Prepare 阶段属于附加步骤，一般处理图像的解码和转换等操作。
* Commit: Commit 阶段主要将图层进行打包，并将它们发送至 Render Server。该过程会递归执行，因为图层和视图都是以树形结构存在.

iOS 动画的渲染也是基于上述 Core Animation 流水线完成的。这里我们重点关注 app 与 Render Server 的执行流程。日常开发中，如果不是特别复杂的动画，一般使用 UIView Animation 实现，iOS 将其处理过程分为如下三部阶段：

* 调用 animationWithDuration:animations: 方法
* 在 Animation Block 中进行 Layout，Display，Prepare，Commit 等步骤。
* Render Server 根据 Animation 逐帧进行渲染

# 4 绘制流程总结

1. 计算机系统中 CPU、GPU、显示器是以上面这种方式协同工作的。CPU 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 VSync 信号，逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示.
2. 在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。从上图中可以看到，CPU 和 GPU 不论哪个阻碍了显示流程，都会造成掉帧现象。所以开发时，也需要分别对 CPU 和 GPU 压力进行评估和优化。
3. iOS 的显示系统是由 VSync 信号驱动的，VSync 信号由硬件时钟生成，每秒钟发出 60 次（这个值取决设备硬件，比如 iPhone 真机上通常是 59.97）。iOS 图形服务接收到 VSync 信号后，会通过 IPC 通知到 App 内。App 的 Runloop 在启动后会注册对应的 CFRunLoopSource 通过 mach_port 接收传过来的时钟信号通知，随后 Source 的回调会驱动整个 App 的动画与显示.
4. Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 标记，并通过 CATransaction 提交到一个中间状态去。当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 Core Animation 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，通过 DisplayLink 稳定的刷新机制会不断的唤醒runloop，使得不断的有机会触发observer回调，从而根据时间来不断更新这个动画的属性值并绘制出来。为了不阻塞主线程，Core Animation 的核心是 OpenGL ES 的一个抽象物，所以大部分的渲染是直接提交给GPU来处理。 而Core Graphics/Quartz 2D的大部分绘制操作都是在主线程和CPU上同步完成的，比如自定义UIView的drawRect里用CGContext来画图
5. Core Animation 在 RunLoop 中注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件 。当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。当Oberver监听的事件到来时，回调执行函数中会遍历所有待处理的UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

# 5 CPU 和 GPU渲染
## 5.1 GPU渲染
OpenGL中，GPU屏幕渲染有以下两种方式:

* On-Screen Rendering 
意为当前屏幕渲染，指的是GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行。
* Off-Screen Rendering 
意为离屏渲染，指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。 
按照这样的说法，如果将不在GPU的当前屏幕缓冲区中进行的渲染都称为离屏渲染，那么就还有另一种特殊的“离屏渲染”方式-CPU渲染.

## 5.2 CPU渲染
CPU渲染。如果我们重写了drawRect方法，并且使用任何Core Graphics的技术进行了绘制操作，就涉及到了CPU渲染。整个渲染过程由CPU在App内同步地完成，渲染得到的bitmap最后再交由GPU用于显示。

## 5.3 离屏渲染
相比于当前屏幕渲染，离屏渲染的代价是很高的，[这里](http://sonnewilling.com/blog/2016/10/19/iostu-xing-yuan-li-yu-chi-ping-xuan-ran/#anchor1.4.2)和[这里](https://imlifengfeng.github.io/article/593/)介绍的很详细, 主要体现在两个方面: 

* 创建新缓冲区 
要想进行离屏渲染，首先要创建一个新的缓冲区。
* 上下文切换 
离屏渲染的整个过程，需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。而上下文环境的切换是要付出很大代价的。

设置了以下属性时，都会触发离屏绘制：

* shouldRasterize（光栅化）
* masks（遮罩）
* shadows（阴影）
* edge antialiasing（抗锯齿）
* group opacity（不透明） 

需要注意的是，如果shouldRasterize被设置成YES，在触发离屏绘制的同时，会将光栅化后的内容缓存起来，如果对应的layer及其sublayers没有发生改变，在下一帧的时候可以直接复用。这将在很大程度上提升渲染性能。而其它属性如果是开启的，就不会有缓存，离屏绘制会在每一帧都发生。

在开发时需要根据实际情况来选择最优的实现方式，尽量使用On-Screen Rendering。简单的Off-Screen Rendering可以考虑使用Core Graphics让CPU来渲染。

# 6 保持界面流畅的技巧
上面介绍了绘制的流程, 这里结合上面的内容介绍下如果保持界面的流畅.

关于卡顿的原因和分析这里讲解的十分详细[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/).

在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

从上面的图中可以看到，CPU 和 GPU 不论哪个阻碍了显示流程，都会造成掉帧现象。所以开发时，也需要分别对 CPU 和 GPU 压力进行评估和优化。

## 6.1 CPU 资源消耗原因和解决方案
CPU 中主要用来计算显示内容, CPU 将计算好的内容提交到 GPU 去.

### 6.1.1 对象创建
对象的创建会分配内存、调整属性、甚至还有读取文件等操作，比较消耗 CPU 资源。尽量用轻量的对象代替重量的对象，可以对性能有所优化。比如 CALayer 比 UIView 要轻量许多，那么不需要响应触摸事件的控件，用 CALayer 显示会更加合适。如果对象不涉及 UI 操作，则尽量放到后台线程去创建，但可惜的是包含有 CALayer 的控件，都只能在主线程创建和操作。通过 Storyboard 创建视图对象时，其资源消耗会比直接通过代码创建对象要大非常多，在性能敏感的界面里，Storyboard 并不是一个好的技术选择。

尽量推迟对象创建的时间，并把对象的创建分散到多个任务中去。尽管这实现起来比较麻烦，并且带来的优势并不多，但如果有能力做，还是要尽量尝试一下。如果对象可以复用，并且复用的代价比释放、创建新对象要小，那么这类对象应当尽量放到一个缓存池里复用。

### 6.1.2 对象调整
对象的调整也经常是消耗 CPU 资源的地方。这里特别说一下 CALayer：CALayer 内部并没有属性，当调用属性方法时，它内部是通过运行时 resolveInstanceMethod 为对象临时添加一个方法，并把对应属性值保存到内部的一个 Dictionary 里，同时还会通知 delegate、创建动画等等，非常消耗资源。UIView 的关于显示相关的属性（比如 frame/bounds/transform）等实际上都是 CALayer 属性映射来的，所以对 UIView 的这些属性进行调整时，消耗的资源要远大于一般的属性。对此你在应用中，应该尽量减少不必要的属性修改。

当视图层次调整时，UIView、CALayer 之间会出现很多方法调用与通知，所以在优化性能时，应该尽量避免调整视图层次、添加和移除视图。

### 6.1.3 对象销毁
对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。这里有个小 Tip：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。

```
NSArray *tmp = self.array;
self.array = nil;
dispatch_async(queue, ^{
    [tmp class];
});
```

### 6.1.4 布局计算
视图布局的计算是 App 中最为常见的消耗 CPU 资源的地方。如果能在后台线程提前计算好视图布局、并且对视图布局进行缓存，那么这个地方基本就不会产生性能问题了。

不论通过何种技术对视图进行布局，其最终都会落到对 UIView.frame/bounds/center 等属性的调整上。上面也说过，对这些属性的调整非常消耗资源，所以尽量提前计算好布局，在需要时一次性调整好对应属性，而不要多次、频繁的计算和调整这些属性。

### 6.1.5 Autolayout
Autolayout 是苹果本身提倡的技术，在大部分情况下也能很好的提升开发效率，但是 Autolayout 对于复杂视图来说常常会产生严重的性能问题。随着视图数量的增长，Autolayout 带来的 CPU 消耗会呈指数级上升。具体数据可以看这个文章：http://pilky.me/36/。 如果你不想手动调整 frame 等属性，你可以用一些工具方法替代（比如常见的 left/right/top/bottom/width/height 快捷属性），或者使用 ComponentKit、AsyncDisplayKit 等框架。

### 6.1.6 文本计算
如果一个界面中包含大量文本（比如微博微信朋友圈等），文本的宽高计算会占用很大一部分资源，并且不可避免。如果你对文本显示没有特殊要求，可以参考下 UILabel 内部的实现方式：用 [NSAttributedString boundingRectWithSize:options:context:] 来计算文本宽高，用 -[NSAttributedString drawWithRect:options:context:] 来绘制文本。尽管这两个方法性能不错，但仍旧需要放到后台线程进行以避免阻塞主线程。

如果你用 CoreText 绘制文本，那就可以先生成 CoreText 排版对象，然后自己计算了，并且 CoreText 对象还能保留以供稍后绘制使用。

### 6.1.7 文本渲染
屏幕上能看到的所有文本内容控件，包括 UIWebView，在底层都是通过 CoreText 排版、绘制为 Bitmap 显示的。常见的文本控件 （UILabel、UITextView 等），其排版和绘制都是在主线程进行的，当显示大量文本时，CPU 的压力会非常大。对此解决方案只有一个，那就是自定义文本控件，用 TextKit 或最底层的 CoreText 对文本异步绘制。尽管这实现起来非常麻烦，但其带来的优势也非常大，CoreText 对象创建好后，能直接获取文本的宽高等信息，避免了多次计算（调整 UILabel 大小时算一遍、UILabel 绘制时内部再算一遍）；CoreText 对象占用内存较少，可以缓存下来以备稍后多次渲染。

### 6.1.8 图片的解码
当你用 UIImage 或 CGImageSource 的那几个方法创建图片时，图片数据并不会立刻解码。图片设置到 UIImageView 或者 CALayer.contents 中去，并且 CALayer 被提交到 GPU 前，CGImage 中的数据才会得到解码。这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能。

### 6.1.9 图像的绘制
图像的绘制通常是指用那些以 CG 开头的方法把图像绘制到画布中，然后从画布创建图片并显示这样一个过程。这个最常见的地方就是 [UIView drawRect:] 里面了。由于 CoreGraphic 方法通常都是线程安全的，所以图像的绘制可以很容易的放到后台线程进行。一个简单异步绘制的过程大致如下（实际情况会比这个复杂得多，但原理基本一致）：

```
- (void)display {
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

## 6.2 GPU 资源消耗原因和解决方案
相对于 CPU 来说，GPU 能干的事情比较单一：接收提交的纹理（Texture）和顶点描述（三角形），应用变换（transform）、混合并渲染，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理（图片）和形状（三角模拟的矢量图形）两类

### 6.2.1 纹理的渲染
所有的 Bitmap，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，GPU 占用非常高，界面仍然会掉帧。避免这种情况的方法只能是尽量减少在短时间内大量图片的显示，尽可能将多张图片合成为一张进行显示。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU 都会带来额外的资源消耗。目前来说，iPhone 4S 以上机型，纹理尺寸上限都是 4096×4096，更详细的资料可以看这里：iosres.com。所以，尽量不要让图片和视图的大小超过这个值。

### 6.2.2 视图的混合 (Composing)
当多个视图（或者说 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，把多个视图预先渲染为一张图片来显示。

### 6.2.3 图形的生成。
CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，但这会把原本离屏渲染的操作转嫁到 CPU 上去。对于只需要圆角的某些场合，也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性。

---------
参考资料:
1.[iOS UI绘制原理](https://juejin.im/post/5adf0e7af265da0b814b32da)
2.[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
3.[谈谈 iOS 中图片的解压缩](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/#jtss-tsina)
4.[UI绘制原理](https://leoliuyt.github.io/2018/05/26/UI%E7%BB%98%E5%88%B6%E5%8E%9F%E7%90%86/)
5.[iOS 图像渲染原理](http://www.chuquan.me/2018/09/25/ios-graphics-render-principle/)
6.[iOS图形原理与离屏渲染](http://sonnewilling.com/blog/2016/10/19/iostu-xing-yuan-li-yu-chi-ping-xuan-ran/)
7.[iOS 开发：绘制像素到屏幕](https://segmentfault.com/a/1190000000390012)
8.[iOS界面渲染流程](https://www.zybuluo.com/qidiandasheng/note/494700)
9.[界面渲染的整体流程](http://blog.handy.wang/blog/2015/10/03/uiviewyu-calayerxie-zuo-xuan-ran-jie-mian-de-guo-cheng/)
10.[iOS开发-视图渲染与性能优化](https://www.jianshu.com/p/748f9abafff8)
11.[iOS 事件处理机制与图像渲染过程](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=400417748&idx=1&sn=0c5f6747dd192c5a0eea32bb4650c160&scene=4#wechat_redirect)
12.[iOS图形原理与离屏渲染](http://sonnewilling.com/blog/2016/10/19/iostu-xing-yuan-li-yu-chi-ping-xuan-ran/#anchor1.4.2)

