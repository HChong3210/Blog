---
title: iOS的路由协议实践
date: 2017-05-23 19:29:29
tags:
    - 模块化
    - 路由
categories:
    - 模块化
---

# iOS的路由协议实践

路由协议, 是组件化的核心所在. 组件化, 实际就会把代码拆分为一个一个的模块, 无论采用Pod的方式,  文件夹分割的方式, 还是静态库的方式, 实质都是把代码分为一个个的模块. 如何在模块之间和应用之间通信, 就是路由协议需要考虑的问题.

关于路由协议, 冰霜大神的[这篇博客](https://halfrost.com/ios_router/)写的实在是太详细了, 看得是男默女泪, 简直是业界良心. 

## 应用间路由协议


## 应用内路由协议设计思路

*页面间跳转* 和 *组件间调用*, 是应用内路由协议要解决的两大问题, 举个例子来说明一下:

* *页面跳转*, 如果以传统的Push来说, 我们怎么才能从任一界面push到另外任一界面, 各个界面之间的跳转必然要相互`Import`, 这些怎么解决.
* *组件见调用*, 随着业务的模块化拆分, 各个模块之间有业务调用怎么办, 本身独立的模块, 为了相互调用必然又增加接口供外部调用, 本身相对独立的业务模块, 瞬间又变得相互耦合了.

而路由协议正是为了解决这一类问题, 现在比较流行的路由协议有如下几种:

* [JLRouts](https://github.com/joeldev/JLRoutes)
* [Routable](https://github.com/clayallsopp/routable-ios)
* [HHRouter](https://github.com/lightory/HHRouter)
* [MGJRouter](https://github.com/meili/MGJRouter)
* [CTMediator](https://github.com/casatwy/CTMediator)

![URL资源](https://ob6mci30g.qnssl.com/Blog/ArticleImage/40_15.png)

通过上面应用间的跳转, 我们可以发现iOS 系统里面使用的是URL Scheme方式. 对于一个资源的访问，苹果也是用URL的方式来访问的, 那么我们就可以想办法通过URL来统一三端的跳转. 一段标准的URL的格式, 每一部分都代表不同的含义. 我们可以按照规则来解析接受到的URL, 从而获得有用的信息. 

下面我们以实现MGJRouter和CTMediator的伪代码来分析目前最为流行的两种路由解决方案.

### MGJRouter

MGJRouter, 最早是由蘑菇街技术团队在解决组件化是提出的路由解决方案. 我司的组件化方案, 在蘑菇街的基础上改造, 但是又略微有些区别, 下面我就一些具体的实现来分析一下.

以A -> B页面为例:
* 首先跳转协议会是一段格式的URL, 
* 会有一个


-------
参考文章:
1.[iOS 组件化 —— 路由设计思路分析](https://halfrost.com/ios_router/)

2.[路由跳转的思考](http://awhisper.github.io/2016/06/12/%E8%B7%AF%E7%94%B1%E8%B7%B3%E8%BD%AC%E7%9A%84%E6%80%9D%E8%80%83/)

3.[iOS组件化方案探讨](http://wereadteam.github.io/2016/03/19/iOS-Component/)

4.[iOS应用架构谈-组件化方案](https://casatwy.com/iOS-Modulization.html)


