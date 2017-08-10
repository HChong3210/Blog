---
title: Cocoapods实践
date: 2017-05-24 22:39:38
tags:
    - 基础知识
---

# Cocoapods实践

Cocoapods是一个基于Ruby的包管理工具, 类似的还有Carthage. Cocoapods的安装在这里不在详述, 请自行百度, 在这里着重讲一下如何使用Cocoapods制作私有包, 以及Cocoapods的实现原理. 

## Cocoapods的实现原理

cocoapods安装成功后, 我们怎么来使用它呢. 这里就要用到cocoapods的核心文件之一`Podfile`. Podfile 是一个文件，用于定义项目所需要使用的第三方库。该文件支持高度定制，你可以根据个人喜好对其做出定制。[查看更多官方介绍](https://guides.cocoapods.org/syntax/podfile.html). 

下面是一个🌰, 我们来挨个分析下他们背后都代表着什么.

``` Ruby
source 'http://source.git'
platform :ios, '8.0'

target 'Demo' do
    pod 'AFNetworking'
    pod 'SDWebImage'
    pod 'Masonry'
end
``` 


```
  MyLib
  ├── .travis.yml
  ├── _Pods.xcproject
  ├── Example
  │   ├── MyLib
  │   ├── MyLib.xcodeproj
  │   ├── MyLib.xcworkspace
  │   ├── Podfile
  │   ├── Podfile.lock
  │   ├── Pods
  │   └── Tests
  ├── LICENSE
  ├── MyLib.podspec
  ├── Pod
  │   ├── Assets
  │   └── Classes
  │     └── RemoveMe.[swift/m]
  └── README.md
```

## Cocoapods创建私有Pod



-------
参考文章:
1.[Carthage 包管理工具，另一种敏捷轻快的 iOS & MAC 开发体验](https://swiftcafe.io/2015/10/25/swift-daily-carthage-package/).

2.[细聊Cocoapods与Xcode工程配置](https://bestswifter.com/cocoapods/).

3.[Cocoapods官方文档](http://guides.cocoapods.org/making/using-pod-lib-create.html).

4.[使用Cocoapods创建私有podspec](http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/)

5.[CocoaPods 都做了什么？](http://draveness.me/cocoapods.html)

6.[使用私有Cocoapods仓库 中高级用法](http://www.jianshu.com/p/d6a592d6fced)

7.[深入理解CocoaPods](https://objccn.io/issue-6-4/)

8.[podspec官方文档](https://guides.cocoapods.org/syntax/podspec.html)

9.[podfile官方文档](https://guides.cocoapods.org/syntax/podfile.html)