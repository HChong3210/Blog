---
title: Xcode中的Workspace, Scheme, Project, Target和Build Settings的关系
date: 2017-05-16 20:33:57
tags:
    - 基础知识
    - Target
categories:
    - 基础知识
---

# Xcode中的Workspace, Scheme, Project, Target和Build Settings的关系

## Xcode Workspace

官方文档如下: 

> A workspace is an Xcode document that groups projects and other documents so you can work on them together. A workspace can contain any number of Xcode projects, plus any other files you want to include. In addition to organizing all the files in each Xcode project, a workspace provides implicit and explicit relationships among the included projects and their targets.

workspace是Xcode的一种文件，用来管理工程和里面的文件，一个workspace可以包含若干个工程，甚至可以添加任何你想添加的文件。workspace提供了工程和工程里面的target之间隐式和显式依赖关系，用来管理和组织工程里面的所有文件.

 一个workspace可以管理多个Project, `pod install`的过程就是生成了一个workspace和一个全是Pod组件的Project, 然后我们通过生成的workspace来管理新生成的Project和原本的Project.

## Xcode Project

官方文档如下:

> An Xcode project is a repository for all the files, resources, and information required to build one or more software products. A project contains all the elements used to build your products and maintains the relationships between those elements. It contains one or more targets, which specify how to build products. A project defines default build settings for all the targets in the project (each target can also specify its own build settings, which override the project build settings).

project就是一个个的仓库，里面会包含属于这个项目的所有文件，资源，以及生成一个或者多个软件产品的信息。每一个project会包含一个或者多个 targets，而每一个 target 告诉我们如何生产 products。project 会为所有 targets 定义了默认的 build settings，每一个 target 也能自定义自己的 build settings，且 target 的 build settings 会重写 project 的 build settings。

Xcode中的 project里面包含了所有的源文件，资源文件和构建一个或者多个product的信息。project利用他们去编译我们所需的product，也帮我们组织它们之间的关系。一个project可以包含一个或者多个target。project定义了一些基本的编译设置，每个target都继承了project的默认设置，每个target可以通过重新设置target的编译选项来定义自己的特殊编译选项。

project包含了以下信息：

* 源文件
    * 代码的头文件和实现文件
    * 静态库，动态库，
    * 资源文件(如文本，xml，plist等)
    * 图片资源
    * 界面资源文件(xib， storyboard等)
* 在文件结构的导航中，采用group去组织文件(实际开发中，尽量使用实体文件夹)
* project的编译级别配置文件如(debug， release)
* target
* 运行环境如：debug，test

project可以单独存在，或者存在于一个workspace中.
​    
## Xcode Target

官方文档如下:

> A target specifies a product to build and contains the instructions for building the product from a set of files in a project or workspace. A target defines a single product; it organizes the inputs into the build system—the source files and instructions for processing those source files—required to build that product. Projects can contain one or more targets, each of which produces one product.

target 定义了生成的唯一 product, 它将构建该 product 所需的文件和处理这些文件所需的指令集整合进 build system 中。Projects 会包含一个或者多个 targets,每一个 target 将会产出一个 product.

这些指令以 build setting 和 build phases 的形式存在，你可在 Xcode 的项目编辑器(TARGETS->Build Setting, TARGETS->Build Phases)中进行查看和编辑。target 中的 build setting 参数继承自 project 的 build settings, 但是你可以在 target 中修改任意 settings 来重写 project settings，这样，最终生效的 settings 参数以在 target 中设置的为准. Project 可包含多个 target, 但是在同一时刻，只会有一个 target 生效，可用 Xcode 的 scheme 来指定是哪一个 target 生效.

target 和其生成的 product 可与另一个 target 有关，如果一个 target 的 build 依赖于另一个 target 的输出，那么我们就说前一个 target 依赖于后一个 target .如果这些 target 在同一个 workspace 中，那么 Xcode 能够发现这种依赖关系，从而使其以我们期望的顺序生成 products.这种关系被称为隐式依赖关系。同时，你可以显示指定 targets 之间的依赖关系，并且这种依赖关系会覆盖 Xcode 推测出的隐式依赖关系。

指定 targets 之间的依赖关系的地方在 Project Editor->TRAGETS->Build Phases->Target Dependencies 处设置.

## Scheme

官方文档如下:
> An Xcode scheme defines a collection of targets to build, a configuration to use when building, and a collection of tests to execute.

一个Scheme就包含了一套targets(这些targets之间可能有依赖关系)，一个configuration，一套待执行的tests。指定了编译哪个target，使用哪个build configuration去编译target，提供运行target的执行环境等等。可以通过scheme editor来编辑scheme.

> You can have as many schemes as you want, but only one can be active at a time. You can specify whether a scheme should be stored in a project—in which case it’s available in every workspace that includes that project, or in the workspace—in which case it’s available only in that workspace. When you select an active scheme, you also select a run destination (that is, the architecture of the hardware for which the products are built).

scheme定义了一系列构建的targets，构建时的配置，和一系列执行的测试。可以有很多scheme,但同一时刻只能有一个有效的。选择scheme时，意味着你也选择了一个运行目标（product构建的硬件平台)。可以指定scheme是否存储在project中，以便包含此project的所有workspace都可以使用些scheme,当然也可以指定只存储在某个workspace中。

## Build Settings

官方文档如下:
> A build setting is a variable that contains information about how a particular aspect of a product’s build process should be performed. For example, the information in a build setting can specify which options Xcode passes to the compiler.

一个build setting是一个指示产品某个方面构建方式的变量，比如决定xcode传给编译器的参数选项是怎样。其是一个常量或者一个公式供给xcode在构建的时候计算build setting。

build setting 中包含了 product 生成过程中所需的参数信息。你可以在 project-level 和 target-level 层指定 build settings。project-level 的 build settings 适用于 project 中的所有targets，但是当 target-level 的 build settings 重写了 project-level 的 build settings，以 target-level 中的 build settings 中的值为准

一个 build configaration 指定了一套 build settings 用于生成某一 target 的 product，例如，在 Xcode 创建项目时默认就有两套独立的 build configarations, 分别用于生成 debug 和 release 模式下的 product。

除了创建工程时生成的默认 build settings，你也可以自定义 project-level 或者 target-level 的 build settings.

关于继承关系，[The Unofficial Guide to xcconfig files](https://pewpewthespells.com/blog/xcconfig_guide.html#BuildSettingInheritance) 这里也有详细的说明，强烈建议阅读。

动态环境配置就是使用自定义的 build settings 来实现的.

------
参考资料:

1.[Xcode使用xcconfig文件配置环境](http://liumh.com/2016/05/22/use-xcconfig-config-specific-variable/).   
​    
2.[Apple官方文档](https://developer.apple.com/library/content/featuredarticles/XcodeConcepts/Concept-Targets.html).

3.[Xcode中的Scheme和Build Configuration](https://xiuchundao.me/post/xcode-scheme-and-build-configuration).

4.[Xcode workSpace 多个project联编](http://www.jianshu.com/p/1f312abafeff)

5.[The Unofficial Guide to xcconfig files](https://pewpewthespells.com/blog/xcconfig_guide.html#BuildSettingInheritance)