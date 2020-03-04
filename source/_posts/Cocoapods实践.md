---
title: Cocoapods实践
date: 2017-05-24 22:39:38
tags:
    - 基础知识
    - 模块化
categories: 
    - 模块化
---

Cocoapods是一个基于Ruby的包管理工具, 类似的还有Carthage. Cocoapods的安装在这里不在详述, 请自行百度, 在这里着重讲一下如何使用Cocoapods制作私有包, 以及Cocoapods的实现原理. 

# 1 Cocoapods的实现原理

cocoapods安装成功后, 我们怎么来使用它呢. 这里就要用到cocoapods的核心文件之一`Podfile`. Podfile 是一个文件，用于定义项目所需要使用的第三方库。该文件支持高度自定义，你可以根据个人喜好对其做出定制。[查看更多官方介绍](https://guides.cocoapods.org/syntax/Podfile.html). 

## 1.1 Podfile
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

`source`: Specifies the location of specs. spec的地址.

`platform`: Specifies the platform for which a static library should be built. 指定构建静态库的平台.

`target`: Defines a CocoaPods target and scopes dependencies defined within the given block. A target should correspond to an Xcode target. By default the target includes the dependencies defined outside of the block, unless instructed not to inherit them. 定义了CocoaPods在指定target的依赖, 此处的Target应该与Xcode目标相对应。默认情况下，除非表明不继承它们, 否则Target包括在块外部定义的依赖项.

`pod`: A dependency requirement is defined by the name of the Pod and optionally a list of version requirements. 一个依赖项需求是由Pod的名称和可选的版本需求列表所定义的.

当你写完`Podfile`之后, 就需要执行Pod的命令`pod install`, 来按照`Podfile`中的配置来配置我么你的工程

## 1.2 pod install

当运行 `pod install` 命令时会引发许多操作。要想深入了解这个命令执行的详细内容，可以在这个命令后面加上 `--verbose`来查看详细内容。现在运行这个命令 `pod install --verbose`，可以看到类似如下的内容:

```
  Preparing

Analyzing dependencies

Inspecting targets to integrate
  Using `ARCHS` setting to build architectures of target `Pods-XXX`: (``)

Resolving dependencies of `Podfile`

Comparing resolved specification to the sandbox manifest
  A AFNetworking
  A Masonry
  A SDWebImage

Downloading dependencies

-> Installing AFNetworking (3.1.0)
 > Git download
 > Git download
     $ /usr/bin/git clone https://github.com/AFNetworking/AFNetworking.git
     /var/folders/qy/tmvltypx4w954cx9hsfz0vn40000gn/T/d20170813-81556-1sx4ds8
     --template= --single-branch --depth 1 --branch 3.1.0
     Cloning into '/var/folders/qy/tmvltypx4w954cx9hsfz0vn40000gn/T/d20170813-81556-1sx4ds8'...
     Note: checking out '88f13053b1d1f20bf657f5c36459b87a5d317ad7'.
     
     You are in 'detached HEAD' state. You can look around, make experimental
     changes and commit them, and you can discard any commits you make in this
     state without impacting any branches by performing another checkout.
     
     If you want to create a new branch to retain commits you create, you may
     do so (now or later) by using -b with the checkout command again. Example:
     
       git checkout -b <new-branch-name>
     
     $ /usr/bin/git -C
     /var/folders/qy/tmvltypx4w954cx9hsfz0vn40000gn/T/d20170813-81556-1sx4ds8
     submodule update --init --recursive
  > Copying AFNetworking from
  `/Users/hc/Library/Caches/CocoaPods/Pods/Release/AFNetworking/3.1.0-5e0e1` to
  `Pods/AFNetworking`

-> Installing Masonry (1.0.2)
  > Copying Masonry from
  `/Users/hc/Library/Caches/CocoaPods/Pods/Release/Masonry/1.0.2-7c429` to
  `Pods/Masonry`

-> Installing SDWebImage (4.1.0)
  > Copying SDWebImage from
  `/Users/hc/Library/Caches/CocoaPods/Pods/Release/SDWebImage/4.1.0-0e435` to
  `Pods/SDWebImage`
  - Running pre install hooks

Generating Pods project
  - Creating Pods project
  - Adding source files to Pods project
  - Adding frameworks to Pods project
  - Adding libraries to Pods project
  - Adding resources to Pods project
  - Linking headers
  - Installing targets
    - Installing target `AFNetworking` iOS 8.0
      - Generating Info.plist file at `Pods/Target Support
      Files/AFNetworking/Info.plist`
      - Generating module map file at `Pods/Target Support
      Files/AFNetworking/AFNetworking.modulemap`
      - Generating umbrella header at `Pods/Target Support
      Files/AFNetworking/AFNetworking-umbrella.h`
    - Installing target `Masonry` iOS 8.0
      - Generating Info.plist file at `Pods/Target Support
      Files/Masonry/Info.plist`
      - Generating module map file at `Pods/Target Support
      Files/Masonry/Masonry.modulemap`
      - Generating umbrella header at `Pods/Target Support
      Files/Masonry/Masonry-umbrella.h`
    - Installing target `SDWebImage` iOS 8.0
      - Generating Info.plist file at `Pods/Target Support
      Files/SDWebImage/Info.plist`
      - Generating module map file at `Pods/Target Support
      Files/SDWebImage/SDWebImage.modulemap`
      - Generating umbrella header at `Pods/Target Support
      Files/SDWebImage/SDWebImage-umbrella.h`
    - Installing target `Pods-CarMall` iOS 8.0
      - Generating Info.plist file at `Pods/Target Support
      Files/Pods-CarMall/Info.plist`
      - Generating module map file at `Pods/Target Support
      Files/Pods-CarMall/Pods-CarMall.modulemap`
      - Generating umbrella header at `Pods/Target Support
      Files/Pods-CarMall/Pods-CarMall-umbrella.h`
  - Running post install hooks
  - Writing Xcode project file to `Pods/Pods.xcodeproj`
    - Generating deterministic UUIDs
  - Writing Lockfile in `Podfile.lock`
  - Writing Manifest in `Pods/Manifest.lock`

Integrating client project

[!] Please close any current Xcode sessions and use `CarMall.xcworkspace` for this project from now on.

Integrating target `Pods-CarMall` (`CarMall.xcodeproj` project)
  - Running post install hooks
    - cocoapods-stats from
    `/Users/hc/.rvm/gems/ruby-2.4.1/gems/cocoapods-stats-1.0.0/lib/cocoapods_plugin.rb`

Sending stats
      - AFNetworking, 3.1.0
      - Masonry, 1.0.2
      - SDWebImage, 4.1.0

-> Pod installation complete! There are 3 dependencies from the Podfile and 3 total pods installed.
```

先看我们工程的变化, 可以发现工程里面多了三个文件, 一个`XXX.xcworkspace`文件, 一个`Podfile.lock`文件, 还有一个`Pods`文件夹. 我们在通过终端输出的命令来分析, 为什么会生成这几个文件, 以及他们的作用.

1. Analyzing dependencies. 弄清楚声明了哪些第三方库.在加载 `podspecs` 过程中，`CocoaPods` 就建立了包括版本信息在内的所有的第三方库的列表。`Podspecs` 被存储在本地路径 `~/.cocoapods` 中.
2. Inspecting targets to integrate. 检查目标集成.
3. Resolving dependencies of `Podfile` 和 Comparing resolved specification to the sandbox manifest. 分析`Podfile`文件的依赖和将已经解析的Pod与缓存过的Pod进行比对, 是添加还是删除, 还是更新.
4. Downloading dependencies. 根据第三部的分析结果来下载依赖到`Pods`文件夹下面.
5. Generating Pods project. 生成Pods的工程. 这一步还包含了许多其他的步骤.
	* Creating Pods project
	* Adding source files to Pods project
	* Adding frameworks to Pods project
	* Adding libraries to Pods project
	* Adding resources to Pods project
	* Linking headers
	* Installing targets
	* Running post install hooks
	* Writing Xcode project file to `Pods/Pods.xcodeproj` 如果检测到改动时，CocoaPods 会利用 Xcodeproj gem 组件对 Pods.xcodeproj 进行更新。如果该文件不存在，则用默认配置生成。否则，会将已有的配置项加载至内存中.
	* Writing Lockfile in `Podfile.lock`. 记录各个Pod的版本号和之家你的依赖关系.
	* Writing Manifest in `Pods/Manifest.lock`
  

## 1.3 pod install vs pod update

引用官方的文档[https://guides.cocoapods.org/using/pod-install-vs-update.html](https://guides.cocoapods.org/using/pod-install-vs-update.html)来说明一下二者的区别, 以及使用场景.

You will only use pod update when you want to update the version of a specific pod (or all the pods).

* `pod install`主要用在第一次安装pods时, 如果后面你新增, 修改, 删除你的`Podfile`文件时也可以使用该命令. 每次执行`pod install`命令会把每一个安装的pod的版本写进`Podfile.lock`文件中, 来记录和lock这些已经安装Pod的版本. 当执行`Pod install`. 如果是新增Pod, 那么会搜索与Podfile中描述的匹配的版本; 如果已经存在, 他会下载`Podfile.lock`文件中明确的版本, 但不会去检查有没有可用的最新版本.
* `pod update`会不关注`Podfile.lock`中的版本而直接更新到符合`Podfile`中定义的最新版本.

# 2 使用CocoaPods创建私有Pod

上面我们已经介绍过如何使用CocoaPods了, 下面要讲解的就是如何创建Pod来供别人使用. 在创建私有Pod之前我们需要两个git地址:

* 用来保存Spec Repo的内容的Git地址
* 用来保存具体Pod内容的Git地址

## 2.1 创建一个Spec Repo

在这一步, 我们主要创建第一个Git地址, 并且关联到本地.

`Spec Repo`是所有的Pods的一个索引，就是一个容器，所有公开的Pods都在这个里面，他实际是一个Git仓库remote端. 在GitHub上，但是当你使用了Cocoapods后他会被clone到本地的~/.cocoapods/repos目录下，可以进入到这个目录看到master文件夹就是这个官方的Spec Repo了。这个master目录的结构是这个样子的:

```
.
├── Specs
    └── [SPEC_NAME]
        └── [VERSION]
            └── [SPEC_NAME].podspec
```

如果你要创建私有Pod, 那么你的`Spec Repo`的远端地址就必须是私有的. 反之如果你要创建一个公有的Pod, 那么就可以使用GitHub来托管你的代码. 当你创建好远端的仓库之后, 执行`pod repo add [Spec Repo的仓库名] [Spec Repo的git地址]`来把远端的仓库clone到本地. 注意, 这里`[Spec Repo的仓库名]`不一定是远端Git仓库的名字, 而是clone到本地后, 本地文件加的名字, 但是这个名字会在后面提交`PodSpec`文件时用到.

## 2.2 创建Pod工程文件

我们在你需要创建Pod的目录下使用`pod lib create [Pod名称]`来创建对应的Pod模板.  实际上该命令行隐藏了默认参数, 参数补全后应该是`pod lib create ProjectName --template-url=https://github.com/CocoaPods/pod-template.git`. 

接下来会问你四个问题:
1. What language do you want to use?? [ Swift / ObjC ]. 使用什么语言
2. Would you like to include a demo application with your library?. 是否需要一个例子工程, 一般选择YES
3. Which testing frameworks will you use? [ Specta / Kiwi / None ]. 选择一个测试框架
4. Would you like to do view based testing? [ Yes / No ]. 是否基于View测试
5. What is your class prefix?. 类的前缀

根据自己的实际需要来选择后, 就会自动执行`Pod install`命令来创建项目并且生成依赖. 这是这个Pod的没目录结构应该是这样的:

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

接下来, 我们需要创建第二个Git地址, 用来保存Pod的实现代码. 我们进入到Pod文件夹的根目录下, 使用如下代码来关联Pod到远端仓库:

```
$ git add .
$ git commit -s -m "Initial Commit of Library"
$ git remote add origin [Pod的远端地址]           #添加远端仓库
$ git push origin master     #提交到远端仓库
```

## 2.3 编辑Pod文件

Pod文件就是这个Pod要实现功能的具体逻辑, 在主工程根目录下面有一个和Pod同名的文件夹, 里面有两个子文件夹. 一个是`Assets`, 一个是`Classes`.

* Assets文件主要用来存放资源文件, 例如图片资源和XIB文件.
* Classes则存放主要的功能代码, 类.

在这里需要注意两个地方:
1. 当我们要使用`Pod`中的资源时, 以图片为例, 我们通过`[UIImage imageWithName:@"xxx.png"]`是取不到Pod中的图片的, 因为`imageWithName:`方法默认是从`mainBundle`中来取的, 而Pod不属于`mainBundle`的范畴, 我们需要先根据`class`来拿到当前类所在的`bundle`, 再取该`Bundle`中的资源.
2. 每次在Pod文件夹中添加新的文件或者资源时, 都需要在根目录的Example目录下执行`pod update`命令来重新建立索引.

## 2.4 编辑Podspec文件

关于`Podspec`文件[官方](http://guides.cocoapods.org/syntax/podspec.html)是这样描述的:

> A specification describes a version of Pod library. It includes details about where the source should be fetched from, what files to use, the build settings to apply, and other general metadata such as its name, version, and description.

`pod lib create XXX`创建出来的Pod, 初始时的`Podspec`文件包含了各种信息, 详细的说明我们可以看[官方文档](http://guides.cocoapods.org/syntax/podspec.html), 这里贴上最基础的用法代码:

```
Pod::Spec.new do |s|
  s.name             = 'HCPods'
  #Pod的版本
  s.version          = '0.1.0'
  s.summary          = '你在搜索时会呈现'

  s.description      = <<-DESC
    这里是关于你Pod功能的描述
                       DESC

  s.homepage         = 'https://github.com/HChong3210/HCPods'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'HChong3210' => 'hchong7557@gmail.com' }
  #Pod的远端仓库地址
  s.source           = { :git => 'https://github.com/HChong3210/HCPods.git', :tag => s.version.to_s }
	
	#Pod支持的最低版本
  s.ios.deployment_target = '8.0'
	#Pod源文件的位置
  s.source_files = 'HCPods/Classes/**/*'
  #Pod中资源文件的位置
  s.resource_bundles = {
    'DFCForms' => ['HCPods/Assets/*.{png,xib,plist}']
  }
  #对外公开的类
  s.public_header_files = 'DFCForms/Classes/**/*.h'
  
  #Pod中用到的第三方库
  s.frameworks = 'UIKit'
  s.dependency 'AFNetworking', '~> 2.3'
  s.dependency 'SDWebImage'

end
```


## 2.5 提交Pod文件

Pod文件编辑好后, 我们要把代码提交到远端服务器, 我们就使用正常的方式来提交代码, 并且给代码打上Tag, * 注意, 这里的Tag必须和`Podspec`文件中的Pod版本号一致 *, 因为Podspec会根据Tag从远端来找相应的代码, 否则会出现版本和代码不匹配的现象.

如果不使用Sourcetree这样的GUI工具, 可以参考下面的Git代码:

```
# 在根目录下
git status
git add .
git tag -m '备注' 版本号
git commit -s -m '备注' 
git push origin master —tags
```
## 2.6 提交Podspec文件

提交完Pod文件后, 我们只用把`Podspec`文件也提交上去, 这样就可以在Cocoapods中简历起来索引, 找到自己的Pod了. 

在提交之前我们可以在根目录下使用`pod lib lint`命令来验证是否编译通过. 也可以直接提交`pod repo push [你clone到本地的Spec Repo的仓库名] [Pod名称].podspec      --use-libraries --allow-warnings --sources='[Podspec远端地址],https://github.com/CocoaPods/Specs' --verbose`

## 2.7 subspec的使用

有时一个Pod太大了, 而我们又用不到全部的内容, 这时我们就可以使用subspec来解决这个问题. 我们可以在Pod文件夹中, 使用文件夹来分割各个子Pod, 然后在`Podspec`文件中这样设置:

```
  s.subspec '[子Pod名称]' do |pod1|
      pod1.source_files = 'SCCQRCode/Classes/[子文件夹名]/**/*'
  end

  s.subspec '[子Pod名称]' do |pod2|
			pod2.source_files = 'SCCQRCode/Classes/[子文件夹名]/**/*'
  end
```

我们也可以在各个子Pod中分别设置他们的资源路径, 对外暴露的header路径, 以及dependency.

我们在外面引用该Pod的时候就可以使用`pod [Pod/子Pod]`的方式来只引用一个子Pod. 
# 3 cocoapods的相关知识

这里是CocoaPods的其他相关知识, 做一个备忘.
## 3.1 Pod的版本说明
CocoaPods 使用[语义版本控制 - Semantic Versioning](http://semver.org/lang/zh-CN/)命名约定来解决对版本的依赖. 常见的版本说明符号有以下这些.

*	= 0.1 Version 0.1.
*	0.1 Any version higher than 0.1.
*	>= 0.1 Version 0.1 and any higher version.
*	< 0.1 Any version lower than 0.1.
*	<= 0.1 Version 0.1 and any lower version.
*	~> 0.1.2 Version 0.1.2 and the versions up to 0.2, not including 0.2. 

## 3.2 常见Pod依赖的几种写法

* pod 'AFNetworking', :configurations => ['Debug', ‘Beta']
* pod 'QueryKit/Attribute'
* pod 'QueryKit', :subspecs => ['Attribute', 'QuerySet']
* pod 'AFNetworking', :path => '~/Documents/AFNetworking'
* pod 'AFNetworking', :git => 'https://github.com/gowalla/AFNetworking.git'
* pod 'JSONKit', :podspec => 'https://example.com/JSONKit.podspec'

-------
参考文章:
1.[Carthage 包管理工具，另一种敏捷轻快的 iOS & MAC 开发体验](https://swiftcafe.io/2015/10/25/swift-daily-carthage-package/).

2.[细聊Cocoapods与Xcode工程配置](https://bestswifter.com/cocoapods/).

3.[Cocoapods官方文档](http://guides.cocoapods.org/making/using-pod-lib-create.html).

4.[使用Cocoapods创建私有podspec](http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/)

5.[CocoaPods 都做了什么？](http://draveness.me/cocoapods.html)

6.[使用私有Cocoapods仓库中引用.a库](http://www.jianshu.com/p/d6a592d6fced)

7.[深入理解CocoaPods](https://objccn.io/issue-6-4/)

8.[podspec官方文档](https://guides.cocoapods.org/syntax/podspec.html)

9.[Podfile官方文档](https://guides.cocoapods.org/syntax/Podfile.html)

