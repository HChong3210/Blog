---
title: iOS开发基础-生命周期
date: 2019-07-27 11:39:24
tags:
    - 基础知识
categories:
    - iOS开发-基础
---

这里主要总结一下iOS开发中一些声明周期相关的知识

# 1 UIView的生命周期
当创建View的时: 

1. `initWithFrame:`, initWithFrame进行初始化时, 当rect的值不为CGRectZero时会触发layoutSubviews. init初始化不会触发layoutSubviews.
2. `willMoveToSuperview:`
3. `didMoveToSuperview`
4. `willMoveToWindow:`
5. `didMoveToWindow`
6. `layoutSubviews`, 在子视图布局变动时会多次调用.

当View销毁时:

1. `willMoveToWindow:`
2. `didMoveToWindow`
3. `willMoveToSuperview:`
4. ``didMoveToSuperview`
5. `removeFromSuperview`
6. `dealloc`

如果View中有子View的话, 创建时除了上面的顺序之外, 还会调用:

1. `layoutSubviews`, 这是因为子视图的布局变动, 所以会触发.
2. `didAddSubview:`
3. `drawRect:`.

移除时, 除了上面的顺序之外, 还会调用:

1. `willRemoveSubview:`, 是在dealloc后面执行的. 如果有多个子视图, willRemoveSubview会循环执行, 直到移除所有子视图.

# 2 UIViewController声明周期

1. `initWithCoder: 或 initWithNibName:bundle:`, 首先从归档文件中加载UIViewController对象. 非StoryBoard创建调用`initWithNibName:bundle:`, 如果使用StoryBoard进行视图管理, 从nib中加载对象实例时, 程序不会直接初始化一个UIViewController, StoryBoard会自动初始化或在segue被触发时自动初始化`initWithCoder:`.
2. `awakeFromNib`, 从xib或者storyboard加载完毕就会调用.
3. `loadView`, 每次访问UIViewController的view(比如controller.view, self.view)而且view为nil, loadView方法就会被调用. loadView方法是用来负责创建UIViewController的view. 如果在初始化UIViewController指定了xib文件名, 就会根据传入的xib文件名加载对应的xib文件, 如果没有明显地传xib文件名, 就会加载跟UIViewController同名的xib文件. 如果没有找到相关联的xib文件, 就会创建一个空白的UIView, 然后赋值给UIViewController的view属性. 苹果设计这个方法就是给我们自定义UIViewController的view用的, 我们直接在该方法中指定UIViewController的View.
4. `viewDidLoad`, 无论你是通过xib文件还是重写loadView方法创建UIViewController的view, 在view创建完毕后, 最终都会调用viewDidLoad方法. 在这里视图层次已经加载到内存中. 通常, 我们对于各种初始化数据的载入, 初始设定, 修改约束, 移除视图等很多操作都可以这个方法中实现.
5. `viewWillAppear:`, 系统在载入所有的数据后, 将会在屏幕上显示视图, 这时会先调用这个方法, 通常我们会在这个方法对即将显示的视图做进一步的设置.
6. `viewWillLayoutSubviews`, view 即将布局其Subviews. 比如view的bounds改变了(例如:状态栏从不显示到显示,视图方向变化), 要调整Subviews的位置, 在调整之前要做的工作可以放在该方法中实现.
7. `viewDidLayoutSubviews`, view已经布局其Subviews, 这里可以放置调整完成之后需要做的工作.
8. `viewDidAppear:`, 在view被添加到视图层级中以及多视图上下级视图切换时调用这个方法, 可以对正在显示的视图做进一步的设置.
9. `viewWillDisappear:`, 当前视图在即将被移除, 或被覆盖时, 会调用该方法. 此时还没有调用removeFromSuperview.
10. `viewDidDisappear:`, view已经消失或被覆盖,视图已经被从屏幕上移除, 用户看不到这个视图了, 此时已经调用removeFromSuperView.
11. `viewWillUnload`, 如果当前有能被释放的view，系统会调用viewWillUnload方法来释放view
12. `dealloc`, UIViewController被销毁时调用, 此次需要对你在init和viewDidLoad中创建的对象进行释放.
13. `didReceiveMemoryWarning`, 内存不够时, 系统会自动调用这个方法. 默认实现是如果当前UIViewController的view不在应用程序的视图层次结构(View Hierarchy)中, 即view的superview为nil的时候, 就会将view释放, 并且调用viewDidUnload方法.
14. `viewDidUnload`, 收到内存警告时, 如果当前UIViewController的View的superView为nil时, 就会将View释放, 并且调用viewDidUnload. 从 iOS 6 开始, viewDidUnload 方法被废弃掉了

## 2.1 push pop
假设现在有一个 AViewController(简称 Avc) 和 BViewController (简称 Bvc)，通过 navigationController 的 push 实现 Avc 到 Bvc 的跳转，下面是各个方法的执行执行顺序：

1. A loadView
2. A viewDidLoad
3. A viewWillAppear
4. A viewWillLayoutSubviews
5. A viewDidLayoutSubviews
6. A viewDidAppear
7. B loadView(开始跳转)
8. B viewDidLoad
9. A viewWillDisappear
10. B viewWillAppear
11. B viewWillLayoutSubviews
12. B viewDidLayoutSubviews
13. A viewDidDisappear
14. B viewDidAppear

如果再从 Bvc pop到 Avc，会产生下面的执行顺序：

1. B viewWillDisappear  
2. A viewWillAppear  
3. B viewDidDisappear  
4. A viewDidAppear

## 2.2 present dismiss
假设现在有一个 AViewController(简称 Avc) 和 BViewController (简称 Bvc)，通过 present 实现 Avc 到 Bvc 的跳转，下面是各个方法的执行执行顺序：

1. A loadView
2. A viewDidLoad
3. A viewWillAppear
4. A viewWillLayoutSubviews
5. A viewDidLayoutSubviews
6. A viewDidAppear
7. B loadView(开始跳转)
8. B viewDidLoad
9. A viewWillDisappear
10. B viewWillAppear
11. B viewWillLayoutSubviews
12. B viewDidLayoutSubviews
13. B viewDidAppear
14. A viewDidDisappear 

如果再从 Bvc dismiss Avc，会产生下面的执行顺序：

1. B viewWillDisappear
2. A viewWillAppear
3. A viewDidAppear
4. B viewDidDisappear

总结: 
加载的顺序是一样的, push和present的不同点:

* push时总是A先消失, B后出现. pop时总是B先消失, A再出现.
* present时总是B先出现, A再消失. dismiss时总是A先出现, B再消失.

# 3 AppDelegate生命周期

1. 进入main函数, 设置AppDelegate称为函数的代理.
2. 程序加载完成, -[AppDelegate application:didFinishLaunchingWithOptions:].
3. 创建window窗口
4. `applicationWillResignActive:`, 将进入后台. 当应用程序从活动状态(active)变到非活动状态(inactive)时被触发调用, 这可能发生在一些临时中断下(例如: 来电话, 来短信)又或者程序退出时, 他会先过渡到后台. 然后使用这方法去暂停正在进行的任务, 禁用计时器, 节流OpenGL ES 帧率. 在游戏中应该在这个方法里面暂停游戏.
5. `applicationDidEnterBackground:`, 已经进入后台. 使用这种方法来释放共享资源, 保存用户数据, 无效计时器, 存储足够多的应用程序状态信息来恢复您的应用程序的当前状态, 以防它终止丢失数据. 如果你的程序支持后台运行, 那么当用户退出时不会调用.
6. `applicationWillEnterForeground`, 将进入前台. 先从后台切换到非活动状态, 然后进入活动状态.
7. `applicationDidBecomeActive`, 已经进入前台. 重启所有的任务, 不管是从非活动状态还是刚启动程序, 还是后台状态.
8. `applicationWillTerminate`, 程序即将退出. 当应用将要终止时, 调用这个方法, 在这里我们可以做一些数据的保存.

# 4 APP启动生命周期

1. 系统先读取APP的可执行文件(Mach-O)文件, 从里面获取dyld的路径, 然后加载dyld, dyld去初始化运行环境. 
2. 开启缓存策略, 加载程序相关依赖库(其中也包含我们的可执行文件)到内存中, 调用每个依赖库的初始化方法, 动态链接依赖库. 在这一步Runtime会被初始化.
4. 当所有依赖库的初始化后, 轮到最后一位(程序可执行文件)进行初始化, 在这时runtime会对项目中所有类进行类结构初始化, 然后调用所有的load方法.
5. 最后dyld返回main函数地址, main函数被调用, 我们就进入了程序的入口.

Mach-O文件格式是 OS X 与 iOS 系统上的可执行文件格式, 像我们编译过程产生的.O文件, 以及程序的可执行文件, 动态库等都是Mach-O文件. 有如下几个部分组成:

* Header: 保存了一些基本信息, 包括了该文件运行的平台, 文件类型, LoadCommands的个数等等.
* LoadCommands: 可以理解为加载命令, 在加载Mach-O文件时会使用这里的数据来确定内存的分布以及相关的加载命令. 比如我们的main函数的加载地址, 程序所需的dyld的文件路径, 以及相关依赖库的文件路径.
* Data: 这里包含了具体的代码, 数据等等.

dyld: (the dynamic link editor)动态链接器, 系统 kernel 做好启动程序的初始准备后，交给 dyld 负责.
ImageLoader: ImageLoader 作用是将这些文件加载进内存, 且每一个文件对应一个ImageLoader实例来负责加载. 在程序运行时它先将动态链接的 image(二进制文件) 递归加载, 
再从可执行文件 image 递归加载所有符号.

dyld 担当了 runtime 和 ImageLoader 中间的协调者, 当新 image 加载进来后交由 runtime 去解析这个二进制文件的符号表和代码.
整个调用栈顺序是这样的:

1. dyld 开始将程序二进制文件初始化.
2. 交由 ImageLoader 读取 image, 其中包含了我们的类, 方法等各种符号.
3. 由于 runtime 向 dyld 绑定了回调, 当 image 加载到内存后, dyld 会通知 runtime 进行处理.
4. runtime 接手后调用 map_images 做解析和处理. 接下来 load_images 中调用 call_load_methods 方法, 遍历所有加载进来的 Class, 按继承层级依次调用 Class 的 +load 方法和其 Category 的 +load 方法.
5. 至此, 可执行文件中和动态库所有的符号 (Class, Protocol, Selector, IMP, …) 都已经按格式成功加载到内存中, 被 runtime 所管理, 再这之后, runtime 的那些方法（动态添加 Class, swizzle 等等才能生效.

