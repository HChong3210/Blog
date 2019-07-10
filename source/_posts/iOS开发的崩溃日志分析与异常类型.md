---
title: iOS开发的崩溃日志分析与异常类型
date: 2018-04-05 00:13:08
tags:
    - 基础知识
categories:
    - 基础知识
---

# 崩溃分析
## 崩溃日志
### 如何得到crash report

1. 当一个iOS应用程序崩溃时, 系统会创建一份crash日志保存在设备上. 这份crash日志记录着应用程序崩溃时的信息, 通常包含着每个执行线程的栈调用信息(低内存闪退日志例外). 如果设备就在身边, 可以连接设备, 打开Xcode - Window - Organizer, 在左侧面板中选择Device Logs(可以选择具体设备的Device Logs或者Library下所有设备的Device Logs), 然后根据时间排序查看设备上的crash日志. 这是开发, 测试阶段最经常采用的方式.
2. 如果应用程序已经提交到App Store发布, 用户已经安装使用了, 那么开发者可以通过iTunes Connect（Manage Your Applications - View Details - Crash Reports）获取用户的crash日志. 不过这并不是100%有效的, 因为这需要用户设备同意上传相关信息.
3. 线上app的崩溃日志会被app store收集并符号化分组. 类似的崩溃报告的集合被称为崩溃点, (如果用户选择了与苹果共享诊断数据, 这些崩溃日志才会被收集并被符号化). 打开Xcode - Window - Organizer, 在点击相应应用后, 会显示此应用的崩溃集合. 可以看到每一个集合中都会有很多个设备, 如果右键进去查看的话, 会看到很多文件. 右键显示包内容, 会看到最终的详细日志, 当选中了一个崩溃集合后, 如果选择在项目中打开, 会在项目代码中找到具体出问题的代码. 选中Open in Project的话, 会直接在工程中打开.
4. 如果用户反馈应用曾亏, 也可以通过让用户设备与iTunes同步, 设备与电脑上的iTunes Store同步后, 会将崩溃日志保存在电脑上(路径：Mac OS X:~/Library/Logs/CrashReporter/MobileDevice/)到上述位置把崩溃日志下载下来, 然后通过电子邮件发送给你.
5. 通过第三方工具来获取崩溃信息.

### 如何得到.dSYM
我们在Archive的时候会生成.xcarchive文件, 显示包内容就能够在里面找到.dSYM文件和.app文件. .dSYM文件位于 /Users/<用户名>/Library/Developer/Xcode/Archives 目录下, 在目录中包含了一个16进制的保存函数地址映射信息的中转文件, 所有Debug文件的symbols都在这个文件中(包含文件名, 函数名, 行号等), 也称之为调试符号信息文件.

当我们软件 release 模式打包或上线后, 不会像我们在 Xcode 中那样直观的看到用崩溃的错误, 这个时候我们就需要分析 crash report 文件了, iOS 设备中会有日志文件保存我们每个应用出错的函数内存地址, 通过 Xcode 的 Organizer 可以将 iOS 设备中的 DeviceLog 导出成 crash 文件, 这个时候我们就可以通过出错的函数地址去查询 dSYM 文件中程序对应的函数名和文件名. 大前提是我们需要有软件版本对应的 dSYM 文件, 这也是为什么我们很有必要保存每个发布版本的 Archives 文件了.

每一个 xx.app 和 xx.app.dSYM 文件都有对应的 UUID, crash 文件也有自己的 UUID, 只要这三个文件的 UUID 一致, 我们就可以通过他们解析出正确的错误函数信息了.

1. 通过`dwarfdump --uuid xx.app/xx (xx代表你的项目名)`查看xx.app文件的UUID
2. 通过`dwarfdump --uuid xx.app.dSYM`查看xx.dSYM的UUID
3. crash 文件内第一行 Incident Identifier 就是该 crash 文件的 UUID.

### 结合分析crash文件
根据crash report, .dSYM分析崩溃函数.

1. 如果使用的是友盟的话, 友盟自带的有一个分析工具. 但是要注意, 使用的时候要确保你的.xcarchive在 ~/Library/Developer/Xcode/或该路径的子目录下. .xcarchive里的.dsYM文件和.app文件是有对应的UUID的. 然后你的crash report里也是有UUID, 只有当UUID相等时才能分析对. 如果是在别人电脑上archive, 那你需要把.dSYM文件copy过来.
2. symbolicatecrash是xcode的一个符号化crash log的命令行工具. 使用方法也就是导出.crash文件（crash log）和找到.dsYM文件, 然后进行分析. [查看这里](http://www.cnblogs.com/ningxu-ios/p/4141783.html)
3. 如果你有多个“.ipa”文件, 多个".dSYMB"文件, 你并不太确定到底“dSYM”文件对应哪个".ipa"文件, 可以使用命令行工具atos. [查看这里](http://blog.sina.com.cn/s/blog_76a1980f0102wjcf.html)

### 崩溃日志分析
Xcode->Window->Organizer->Crashes, [这里](http://www.cocoachina.com/industry/20130725/6677.html)有关于崩溃日志的详细分析.

盗图一张, 关于崩溃日志的详细信息
![崩溃日志](http://devma.cn/images/2016/11/ios_crash_analysis_2.png)

## 野指针分析
因为野指针的原因发生崩溃是常常出现的事, 而且比较随机. 所以我们要提高野指针的崩溃率好来帮我们快速找到有问题的代码. 对象释放后只有出现被随机填入的数据是不可访问的时候才会必现Crash. 

这个地方我们可以做一下手脚, 把这一随机的过程变成不随机的过程. 对象释放后在内存上填上不可访问的数据, 其实这种技术其实一直都有, Edit Scheme -> Diagnostics ->Enable Malloc Scribble 选中就可以实现这个功能

## 僵尸模式分析
启用了NSZombieEnabled, 它会用一个僵尸来替换默认的dealloc实现, 也就是在引用计数降到0时, 该僵尸实现会将该对象转换成僵尸对象. 僵尸对象的作用是在你向它发送消息时, 它会显示一段日志并自动跳入调试器. 当启用NSZombieEnabled时, 一个错误的内存访问就会变成一条无法识别的消息发送给僵尸对象. 僵尸对象会显示接受到得信息, 然后跳入调试器, 这样你就可以查看到底是哪里出了问题. 
一般这时崩溃的原因就是因为调用了已经释放的内存空间，或者说重复释放了某个地址空间.

1. 打开NSZombieEnabled之后, 如果遇到对应的崩溃类型既调用了已经释放的内存空间, 或者说重复释放了某个地址空间, 那么就能在GDB中看到对应的输出信息.
2. 如果崩溃是发生在当前调用栈, 通过上面的做法, 系统就会把崩溃原因定位到具体代码中. 但是, 如果崩溃不在当前调用栈, 系统就仅仅只能把崩溃地址告诉我们, 而没办法定位到具体代码, 这样我们也没法去修改错误. 这时就可以修改scheme, 让xcode记录每个地址alloc的历史, 这样我们就可以用命令把这个地址还原出来. Edit Scheme -> Environment Variables -> 加入MallocStackLoggingNoCompact, 并且设置为YES.

## Enable Address Sanitizer
Edit Scheme -> Diagnostics ->Enable Address Sanitizer选中就可以实现这个功能, 设置这个参数后, 我们可以看到一些更详细的错误信息提示, 设置还有内存使用情况的展示. 

这类工具的理论依据是: 访问内存时, 通过比较访问的内存和程序实际分配的内存, 验证内存访问的有效性, 从而在bug发生时就检测到它们, 而不会等到副作用产生时才有所察觉.

## 静态分析
Static Analyzer是一个非常好的工具去发现编译器警告不会提示的问题和一些个人的内错泄露和死存储(不会用到的赋了值的变量)错误. 这个方法可能大大的提高内存使用和性能, 以及提升应用的整体稳定性和代码质量.

打开方式: Xcode->Product-Analyze 然后我们就能看到如下蓝色箭头所示的一些有问题的代码.

## unrecognized selector send to instancd 快速定位

1. 在debug navigator的断点栏里添加Create Symbolic Breakpoint
2. 在Symbolic中填写如下方法签名： `-[NSObject(NSObject) doesNotRecognizeSelector:]`

# 常见崩溃信息类型

## Exception Type异常类型
### SIGABRT类型
在iOS中就是未被捕获的Objective-C异常(NSException)导致程序向自身发送了SIGABRT信号而崩溃, 常见的如下:

* SIGSEGV: 段错误信息（SIGSEGV）是操作系统产生的一个更严重的问题。当硬件出现错误、访问不可读的内存地址或向受保护的内存地址写入数据时，就会发生这个错误。硬件错误这一情况并不常见。当要读取保存在RAM中的数据，而该位置的RAM硬件有问题时，你会收到SIGSEGV。SIGSEGV更多是出现在后两种情况。默认情况下，代码页不允许进行写操作，而数据而不允许进行执行操作。当应用中的某个指针指向代码页并试图修改指向位置的值时，你会收到SIGSEGV。当要读取一个指针的值，而它被初始化成指向无效内存地址的垃圾值时，你也会收到SIGSEGV SIGSEGV错误调试起来更困难，而导致SIGSEGV的最常见原因是不正确的类型转换。要避免过度使用指针或尝试手动修改指针来读取私有数据结构。如果你那样做了，而在修改指针时没有注意内存对齐和填充问题，就会收到SIGSEGV。
* SIGBUS: 总线错误信号（SIGBUG）代表无效内存访问，即访问的内存是一个无效的内存地址。也就是说，那个地址指向的位置根本不是物理内存地址（它可能是某个硬件芯片的地址）。SIGSEGV和SIGBUS都羽毛球EXC_BAD_ACCESS的子类型。
* SIGTRAP: SIGTRAP代表陷阱信号。它并不是一个真正的崩溃信号。它会在处理器执行trap指令发送。LLDB调试器通常会处理此信号，并在指定的断点处停止运行。如果你收到了原因不明的SIGTRAP，先清除上次的输出，然后重新进行构建通常能解决这个问题。
* EXC_ARITHETIC: 当要除零时，应用会收到EXC_ARITHMETIC信号。这个错误应该很容易解决.
* SIGILL: SIGILL代表signal illegal instruction(非法指令信号)。当在处理器上执行非法指令时，它就会发生。执行非法指令是指，将函数指针会给另外一个函数时，该函数指针由于某种原因是坏的，指向了一段已经释放的内存或是一个数据段。有时你收到的是EXC_BAD_INSTRUCTION而不是SIGILL，虽然它们是一回事，不过EXC_*等同于此信号不依赖体系结构。
* SIGABRT: SIGABRT代表SIGNAL ABORT（中止信号）。当操作系统发现不安全的情况时，它能够对这种情况进行更多的控制；必要的话，它能要求进程进行清理工作。在调试造成此信号的底层错误时，并没有什么妙招。Cocos2d或UIKit等框架通常会在特定的前提条件没有满足或一些糟糕的情况出现时调用C函数abort（由它来发送此信号）。当SIGABRT出现时，控制台通常会输出大量的信息，说明具体哪里出错了。由于它是可控制的崩溃，所以可以在LLDB控制台上键入bt命令打印出回溯信息

更多的SIGABRT信号类型看[这里](http://www.iosxxx.com/blog/2015-08-29-iosyi-chang-bu-huo.html).

### EXC_BAD_ACCESS
EXC_BAD_ACCESS是一个比较难处理的crash, 当一个app进入一种毁坏的状态, 通常是由于内存管理问题而引起的时, 就会出现出现这样的crash. 通常Signal信号错误都会提醒EXC_BAD_ACCESS, 我们可以通过僵尸模式来捕获这种异常.

### 看门狗超时
这种崩溃通常比较容易分辨, 因为错误码是固定的0x8badf00d. 在iOS上, 它经常出现在执行一个同步网络调用而阻塞主线程的情况. 因此, 永远不要进行同步网络调用.

## Exception Codes异常编码
0x8badf00d: 读做 “ate bad food”! (把数字换成字母，是不是很像 :p)该编码表示应用是因为发生watchdog超时而被iOS终止的。 通常是应用花费太多时间而无法启动、终止或响应用系统事件。

0xbad22222: 该编码表示 VoIP 应用因为过于频繁重启而被终止。

0xdead10cc: 读做 “dead lock”!该代码表明应用因为在后台运行时占用系统资源，如通讯录数据库不释放而被终止 。

0xdeadfa11: 读做 “dead fall”! 该代码表示应用是被用户强制退出的。根据苹果文档, 强制退出发生在用户长按开关按钮直到出现 “滑动来关机”, 然后长按 Home按钮。强制退出将产生 包含0xdeadfa11 异常编码的崩溃日志, 因为大多数是强制退出是因为应用阻塞了界面

#常见的崩溃类型收集
## 新老操作系统兼容
在新 iOS 上正常的应用, 到了老版本 iOS 上秒退最常见原因是系统动态链接库或Framework无法找到. 这种情况通常是由于 App 引用了一个新版操作系统里的动态库(或者某动态库的新版本)或只有新 iOS 支持的 Framework, 而又没有对老系统进行测试, 于是当 App 运行在老系统上时便由于找不到而秒退.
还有就是有些方法是新版操作系统才支持的, 而又没有对该方法是否存在于老系统中做出判断.

## 本地存储的数据结构变化
程序在升级时, 修改了本地存储的数据结构, 但是对用户既存的旧数据没有做好升级, 结果导致初始化时因为无法正确读取用户数据而秒退.

## 访问的数据为空或者数据类型不对
这类情况是比较常见的, 后端传回了空数据, 客户端没有做对应的判断继续执行下去了, 这样就产生了crash. 或者自己本地的某个数据为空数据而去使用了. 还有就是访问的数据类型不是期望的数据类型而产生崩溃.

## 操作了不该操作的对象
这里有可能是空指针或者野指针. 
空指针是没有存储任何内存地址的指针, 一般指nil, 但是在iOS中向nil发消息是不会崩溃的.

野指针是指指向一个已删除的对象("垃圾"内存既不可用内存)或未申请访问受限内存区域的指针. 野指针是比较危险的, 因为野指针指向的对象已经被释放了, 不能用了, 你再给被释放的对象发送消息就是违法的, 所以会崩溃.

## 内存处理不当
内存管理是软件开发中一个重要的课题, iOS自从引入ARC机制后, 对于内存的管理开发者好像轻松了很多, 但是还会发生一些内存泄露之类的问题. Facebook工程师们开源了一些自动化工具来解决监测内存泄露问题:FBRetainCycleDetector、FBAllocationTracker、FBMemoryProfiler.

## 主线程UI长时间卡死, 被系统杀掉
主线程被卡住是非常常见的场景, 具体表现就是程序不响应任何的UI交互. 这时按下调试的暂停按钮, 查看堆栈, 就可以看到是到底是死锁, 死循环等, 导致UI线程被卡住.

## 多线程之间切换访问引起的crash
多线程引起的崩溃大部分是因为使用数据库的时候多线程同时读写数据库而造成了crash. [这里](https://blog.csdn.net/lixing333/article/details/42149893)关于多线程crash的调试.

-----
参考资料:
1.[iOS崩溃捕捉与分析](https://www.jianshu.com/p/09b6084bcd01)

2.[全面的理解和分析iOS的崩溃日志](http://www.cocoachina.com/ios/20171026/20921.html)

3.[浅谈iOS Crash](https://www.jianshu.com/p/3261493e6d9e)

4.[iOS中的崩溃类型](https://blog.csdn.net/womendeaiwoming/article/details/44243571)

5.[iOS 崩溃Crash解析](http://devma.cn/blog/2016/11/10/ios-beng-kui-crash-jie-xi/)

6.[iOS异常捕获](http://www.iosxxx.com/blog/2015-08-29-iosyi-chang-bu-huo.html)

