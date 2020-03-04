---
title: iOS进程间通信
date: 2019-01-09 20:13:47
tags:
    - 基础知识
categories:
    - 多线程
---

iOS系统是相对封闭的系统, App各自在各自的沙盒（sandbox）中运行, 每个App都只能读取iPhone上iOS系统为该应用程序程序创建的文件夹AppData下的内容, 不能随意跨越自己的沙盒去访问别的App沙盒中的内容. 所以iOS 的系统中进行App间通信的方式也比较固定, 常见的app间通信方式以及使用场景总结如下.

# 1. URL Scheme
这个是iOS app通信最常用到的通信方式, App1通过openURL的方法跳转到App2, 并且在URL中带上想要的参数, 有点类似http的get请求那样进行参数传递.

典型的使用场景就是各开放平台SDK的分享功能, 如分享到微信朋友圈微博等. 或者是支付场景, 比如从滴滴打车结束行程跳转到微信进行支付.

使用方法也很简单只需要源App1在info.plist中配置`LSApplicationQueriesSchemes`, 指定目标App2的scheme; 然后在目标App2的info.plist中配置好URL types, 表示该app接受何种URL scheme的唤起.

# 2 Keychain
OS系统的Keychain是一个安全的存储容器, 它本质上就是一个sqllite数据库, 它的位置存储在/private/var/Keychains/keychain-2.db，不过它所保存的所有数据都是经过加密的，可以用来为不同的app保存敏感信息，比如用户名，密码等。iOS系统自己也用keychain来保存VPN凭证和Wi-Fi密码。它是独立于每个App的沙盒之外的，所以即使App被删除之后，Keychain里面的信息依然存在. 

基于安全和独立于app沙盒的两个特性，Keychain主要用于给app保存登录和身份凭证等敏感信息，这样只要用户登录过，即使用户删除了app重新安装也不需要重新登录

典型的使用场景就是就是统一账户登录平台. 一般开放平台都会提供登录SDK，在这个SDK内部就可以把登录相关的信息都写到keychain中，这样如果多个app都集成了这个SDK，那么就可以实现统一账户登录了.

Keychain的使用比较简单，使用iOS系统提供的类KeychainItemWrapper，并通过keychain access groups就可以在应用之间共享keychain中的数据的数据了

# 3 UIPasteboard
UIPasteboard是剪切板功能，因为iOS的原生控件UITextView, UITextField, UIWebView, 我们在使用时如果长按，就会出现复制、剪切、选中、全选、粘贴等功能，这个就是利用了系统剪切板功能来实现的。而每一个App都可以去访问系统剪切板，所以就能够通过系统剪贴板进行App间的数据传输了. 

典型的使用场景就是阿里的淘口令.

UIPasteboard的使用很简单, 通过`[UIPasteboard generalPasteboard]`直接写入和读取就可以了.

# 4 UIDocumentInteractionController
UIDocumentInteractionController主要是用来实现同设备上app之间的共享文档，以及文档预览、打印、发邮件和复制等功能.

它的使用非常简单. 首先通过调用它唯一的类方法 `interactionControllerWithURL:`, 并传入一个`URL(NSURL)`为你想要共享的文件来初始化一个实例对象。然后设置`UIDocumentInteractionControllerDelegate`，然后通过`presentPreviewAnimated:`显示菜单和预览窗口.

# 5 local socket
一个App1在本地的端口port1234进行TCP的bind和listen，另外一个App2在同一个端口port1234发起TCP的connect连接，这样就可以建立正常的TCP连接，进行TCP通信了，那么就想传什么数据就可以传什么数据了.

这种方式最大的特点就是灵活, 只要连接保持着, 随时都可以传任何相传的数据，而且带宽足够大。它的缺点就是因为iOS系统在任意时刻只有一个app在前台运行，那么就要通信的另外一方具备在后台运行的权限，像导航或者音乐类app.

常见的使用场景就是某个App1具有特殊的能力，比如能够跟硬件进行通信，在硬件上处理相关数据。而App2则没有这个能力，但是它能给App1提供相关的数据，这样APP2跟App1建立本地socket连接，传输数据到App1，然后App1在把数据传给硬件进行处理。

# 6 AirDrop
通过AirDrop实现不同设备的App之间文档和数据的分享

# 7 UIActivityViewController
iOS SDK中封装好的类在App之间发送数据、分享数据和操作数据.

常见的使用场景就是选中多张图片分享到微信. 不过现在接口貌似被封了.

# 8 App Groups
App Group用于同一个开发团队开发的App之间，包括App和Extension之间共享同一份读写空间，进行数据共享。同一个团队开发的多个应用之间如果能直接数据共享，大大提高用户体验

# 9. CFMessagePort
可以参考[这篇文章](http://foggry.com/blog/2014/06/04/iosjin-cheng-jian-tong-xin-zhi-cfmessageport/), 不过在iOS7及以后系统中, CFMessagePort的通信机制不再可用.

# 10 其他
如果是越狱设备的话, 可以参考[这篇文章](http://nirvan.360.cn/blog/?p=723)的实现思路.


-------
参考资料:
1.[iOS进程间通信之CFMessagePort](http://foggry.com/blog/2014/06/04/iosjin-cheng-jian-tong-xin-zhi-cfmessageport/)
2.[iOS (APP)进程间8中常用通信方式总结](https://blog.csdn.net/kuangdacaikuang/article/details/78891379)
3.[iOS进程通讯安全和利用](http://nirvan.360.cn/blog/?p=723)


