---
title: Autoreleasepool原理分析
date: 2018-03-12 21:46:37
tags:
    - 基础知识
categories:
    - 基础知识
---

Autoreleasepool是用来管理内存的好工具, 官方推荐了几个我们要使用的地方:

* If you are writing a program that is not based on a UI framework, such as a command-line tool.
* If you write a loop that creates many temporary objects. You may use an autorelease pool block inside the loop to dispose of those objects before the next iteration. Using an autorelease pool block in the loop helps to reduce the maximum memory footprint of the application.
* If you spawn a secondary thread. You must create your own autorelease pool block as soon as the thread begins executing; otherwise, your application will leak objects. 

下面我们来分析下他其中的实现. 本文是对大神博客的理解和拾遗. 
## AutoreleasePoolPage
ARC下我们使用`@autoreleasepool{}`来使用一个AutoreleasePool, 随后编译器将其改写成下面的样子: 

```
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```
而这两个函数都是对AutoreleasePoolPage的简单封装, 所以核心在AutoreleasePoolPage. AutoreleasePoolPage是一个C++类. 

```
magic_t const magic;
id *next;
pthread_t const thread;
AutoreleasePoolPage * const parent;
AutoreleasePoolPage *child;
uint32_t const depth;
uint32_t hiwat;
```
他并没有单独的节后是由若干个AutoreleasePoolPage以双向链表的形式组合而成. AutoreleasePool是按线程一一对应的. AutoreleasePoolPage每个对象会开辟4096字节内存(也就是虚拟内存一页的大小), 除了上面的实例变量所占空间, 剩下的空间全部用来储存autorelease对象的地址. `id *next`指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置. 一个AutoreleasePoolPage的空间被占满时, 会新建一个AutoreleasePoolPage对象, 连接链表, 后来的autorelease对象在新的page加入.

下面是AutoreleasePoolPage中存储数据的示例, 
![AutoreleasePoolPage存储示例](http://ww2.sinaimg.cn/mw690/51530583gw1elj5gvphtqj20dy0cx756.jpg)
如果这一页满了的话(next指针马上指向栈顶), 这是就要建立下一页page对象, 与这一页链表链接完成后, 新page的next指针被初始化在栈底, 然后继续向栈中添加对象.
所以, 向一个对象发送- autorelease消息, 就是将这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向的位置.
## 释放时刻
AutoreleasePool 是在两次 RunLoop 之间释放, 系统会把需要 release 的对象集中释放掉.

每次调用`objc_autoreleasePoolPush`时, runtime就向AutoreleasePoolPage中add一个哨兵对象, 值为nil. objc_autoreleasePoolPush的返回值正是这个哨兵对象的地址, 被objc_autoreleasePoolPop(哨兵对象)作为入参.

根据传入的哨兵对象地址找到哨兵对象所处的page, 在当前page中, 将晚于哨兵对象插入的所有autorelease对象都发送一次release, 从最新加入的对象一直向前清理, 可以向前跨越若干个page, 直到哨兵所在的page. 然后将next指针指向正确的位置.

------
参考资料: 
1.[黑幕背后的Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

2.[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)

3.[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)


