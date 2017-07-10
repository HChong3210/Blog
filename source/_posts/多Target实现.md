---
title: 多渠道包和多环境包的自动化实现
date: 2017-03-12 14:45:08
tags:
    - 自动化打包
    - Target
categories:
    - 自动化打包
---

# 多渠道包和多环境包的自动化实现
在App的开发过程中, 肯定会有很多后台环境的区分. 例如开发时, 我们用的是开发环境. 测试时使用的是测试环境. 在预发布之前肯定要有个缓冲的环境, 也就是预发布环境. 最后还有一个线上的环境, 也称为生产环境. 那么这么多环境, 问题就来了, 该怎么在各个环境之间无缝切换呢?  

首先我们会想到在代码中建一个单独的类, 把用到的所有服务按照环境列一个遍, 每次需要运行什么环境就按照什么环境从类里面去取相关环境代码. 于是有了方案一.

## 方案一
方案一的伪代码, 大概是这个样子的:

```objective-c
+ (void)getHttpHostWithStatus:(NSInteger)status {
    switch (status) {
        case 0: {
            Host1 = @"开发环境";
            Host2 = @"开发环境";
        }
            break;
        case 1: {
            Host1 = @"测试环境";
            Host2 = @"测试环境";
        }
            break;
        case 2: {
            Host1 = @"预发布环境";
            Host2 = @"预发布环境";
        }
            break;
        case 3: {
            Host1 = @"线上环境";
            Host2 = @"线上环境";
        }
            break;
        default:
            break;
    }
}
```

在App每次启动时调用这个方法, 来获取各个Host的地址, 完美解决. 

测试每次想要个新的包来测试, 都会有如下对话, "帮我打个包吧", "你要什么环境的", "预发环境", "好的". 于是 ,我们一同操作, 传入预发的status, 调用动态取Host的方法, 然后Product->Archive, 等个十几二十分钟, 打包完毕, 生成一个IPA文件, 自己拿ITunes安装. 嗯, 虽然辛苦点, 但是目前看来, 一切都很完美嘛, 打包的时候还能喝个茶.

然而, 随着我司测试流程的规范化, 逐渐推进使用[Jenkins持续集成]()来打包, 那么问题就来了. 你让测试也装个Xcode, 每次切换一下环境, 手动Archive吗, 呵呵. 于是我们有了方案二.

## 方案二

我们先来理一下方案二主要解决的问题: *测试要能够在不改代码的前提下使用Jenkins打不同环境的包*. 听起来有点复杂呢, 我们来逐个分析下.
1. 使用Jenkins来打包, 那就是跑脚本嘛, xcodebuild已经寂寞难耐.
2. 既然搞Jenkins了, 不能再用IPA自己安装的方式了, Low, 蒲公英和Fir简直不要太方便.
3. 既然要用脚本来打包了, 那不能每次跑脚本之前, 先改一下代码吧, 于是想到我们应该有一个参数来在工程中区分不同的环境, 

以上就是我们方案二要解决的主要问题, 下面我们来逐个分析和解决.
### 打包脚本

关于打包的脚本, 版本很多, 有Python流, 有Shell流, 可以[参考这里](). 但说到底, 无非用的是xcodebuild来实现的, 在终端中输入`xcodebuild -help`可以查看xcodebuild的相关操作[这篇文章](http://www.jianshu.com/p/4f4d16326152)将的也比较全面, 可以参考下.

在这里, 我们主要用到的命令有两个.

```objective-c
//编译
xcodebuild -workspace <workspacename> -scheme <schemeName> [-destination <destinationspecifier>]... [-configuration <configurationname>] [-arch <architecture>]... [-sdk [<sdkname>|<sdkpath>]] [-showBuildSettings] [<buildsetting>=<value>]... [<buildaction>]...

//生成IPA
xcodebuild -exportArchive -archivePath <xcarchivepath> -exportPath <destinationpath> -exportOptionsPlist <plistpath>
```

到这一步, 我们的打包脚本仅仅完成了第一步, 生成了IPA文件. 
### 内测分发平台

生成IPA之后, 我们要提供一个可供下载安装的链接. 此处我们使用的是[Fir](https://fir.im/), 它提供的也有[命令行操作](https://github.com/FIRHQ/fir-cli/blob/master/README.md), 可以直接上传IPA文件, 生成可供下载的二维码.👍

当然有条件的公司也可以自己搭建一个下载地址, 制作一个管理后台, 用来管理自己的内测.

### APP配置多个环境变量

接下来这个, 才是本次解决方案的重点, 明确了我们的目的, 就是配置多环境变量, 那么我们就开始吧. 关于多环境变量的配置, 冰霜大神的[这篇文章](http://www.jianshu.com/p/83b6e781eb51)讲解的比较详细. 但是我们采用的方案和他的又略有区别, 一开始我们使用的是多Target的方式, 后来感觉不够轻量化, 并且每个Target的Seeting也基本你没有大的不同, 于是就采用了多Scheme的方式来解决.

具体步骤如下:
1.新建Scheme. 注意这里一定要把Scheme的名字和编译方式区分开，选择了一个Scheme，只是相当于选择了一个环境，并不是代表这Debug还是Release, 此处以`PrePublish`为例.
![新建Scheme](http://ww3.sinaimg.cn/large/006tNbRwgy1ffl8ovbi6rj318o0aq0t6.jpg)

2.新建xcconfig文件, 名字与Scheme对应. 在新建时, 一定要在Target那里勾选上(默认是非勾选状态).
![新建xcconfig](http://ww3.sinaimg.cn/large/006tNbRwgy1ffl8qe5nx2j313k13umy6.jpg)

3.Project -> Info -> Configurations 新建Configurations文件. 注意, 如果有使用Pod的话, 此处需要立马执行`pod install`, 生成对应的Pod的xcconfig文件
![新建Configurations](http://ww3.sinaimg.cn/large/006tNbRwgy1ffl8s76f92j318u0h2aap.jpg)

4.修改Configurations文件, 使之与对应的Scheme相关联.
![修改Configurations](http://ww4.sinaimg.cn/large/006tNbRwgy1ffl8vld5eyj31c00g4mxp.jpg)

5.Edit Scheme
![Edit Scheme](http://ww2.sinaimg.cn/large/006tNbRwgy1ffl92zbnlhj31ds0s0q3p.jpg)

6.编辑PrePublish.xcconfig文件, 设置自定义字段. 具体如下:

```
#include "Pods/Target Support Files/Pods-MultipleChannel/Pods-MultipleChannel.prepublish.xcconfig"

HOST1 = @"HOST1/PrePublish"

HOST2 = @"HOST2/PrePublish"

GCC_PREPROCESSOR_DEFINITIONS = $(inherited) HOST2='$(HOST2)' HOST1='$(HOST1)'
```
注意, 此处必须要引入`pod install`生成的对应的Pod的xcconfig, 否则会报错.
![报错](http://ww4.sinaimg.cn/large/006tNbRwgy1ffl9102hzij30ji0aoglq.jpg)
7.Manage Schemes

因为一般是多人合作开发, 所以此处的Scheme需要设置为Share状态
![](http://ww1.sinaimg.cn/large/006tNbRwgy1ffl969kivwj316o0o0jrt.jpg)

至此, 已经可以在工程中使用我们在xcconfig中自定义的字段了. 因为我们使用了`GCC_PREPROCESSOR_DEFINITIONS`, 他会在编译时生成一个宏定义, 所以我们可以直接使用宏定义
```
NSLog(@"----------%@", HOST1);
NSLog(@"----------%@", HOST2);
```


注: 此外, 还有另外一种使用方式. 我们可以每个Scheme新建一个plist(也可使用Info.plist)文件与之对应, 在plist文件中新增`key-value`来与xcconfig中的自定义字段对应.

使用方法如下:

1.在plist中新增与xcconfig中自定义字段对应的`key-value`
![plist文件新增](http://ww3.sinaimg.cn/large/006tNbRwgy1ffl9dzmseoj310y0nqjsm.jpg)

2.使用方式如下:

```
    NSString *bundlePath = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
    NSMutableDictionary *dict = [NSMutableDictionary dictionaryWithContentsOfFile:bundlePath];
    NSLog(@"----------%@", [dict objectForKey:@"HOST1"]);
    NSLog(@"----------%@", [dict objectForKey:@"HOST2"]);
```
至此, 我们已经实现不同Scheme下, 参数的动态化配置的两种方案, 下面我们分析下原理:

首先, 我们先来区分一下Xcode Workspace、Xcode Scheme、Xcode Project、Xcode Target、Build Settings 这5者的关系, 由于篇幅较长, [另开一文](http://hchong.net/2017/05/16/Settings%E5%85%B3%E7%B3%BB/). 
这5者的关系在苹果官方文档上其实都已经说明的很清楚了。详情见文档[Xcode Concepts](https://developer.apple.com/library/content/featuredarticles/XcodeConcepts/Concept-Targets.html)。 

总结起来, 一个Workspace可以包含多个Project,一个Project可以包含多个Target，Scheme包含了所有的Target集合，这个集合指定了，编译哪个target，使用哪个build configuration去编译target，提供运行target的执行环境等等.

xcconfig文件是一个用来保存Build Settings键值对的纯文本文件, 这些键值对会覆盖Build Settings中的值. xcconifg支持可以根据不同的Configuration选项配置不同的文件. 不同的xcconfig可以指定不同的Build Settings里的属性值.

## 方案三

我们的初衷是用这个来动态配置App的环境. 既然已经可以通过xcconfig和Scheme来配合修改Build Seetings的值, 那么就会有一些更加高级的玩法. 例如我们可以在`Images.xcassets`中新建新的icon和launch Image的分类, 通过`ASSETCATALOG_COMPILER_APPICON_NAME`和`ASSETCATALOG_COMPILER_LAUNCHIMAGE_NAME`中的设置, 以及`Display Name`,`Bundle Identifier`, `Provisioning Profile`等一些配置(此处Build Settings中的值可以直接`command + C`来copy)来管理大量的相似APP.

Info.plist中的值, 需要在Value中使用${Value}来动态配置, 参考域名的配置.

xcconfig还能动态配置Build Settings里面的很多参数。这其实类似于cocopods的做法。但是有一个大神的做法很优雅。值得大家感兴趣的人去学习学习。iOS大神Justin Spahr-Summers的开源库提供了一个类权威的[模板](https://github.com/jspahrsummers/xcconfigs).

那么伴随着业务的发展, 就有有问题了. 如果我想管理大量相似的APP, 每次发布新版本, 我就要打N多个新包. 每次跑脚本, 数量多了时间也很长, 那这就不是一杯茶的功夫了, 甚至要一两个小时. 并且我还要管理N多个Scheme, 这些Scheme明明只有很少的差别, 真的好烦啊. 有没有更加优雅的解决方案呢, 来看下方案四.

## 方案四

首先, 还是先来明确下我们这个方案要解决的问题:

1. Scheme太多, 每个渠道一个Scheme, 管理起来费劲.
2. 新版本发布时, 各个渠道的包, 跑脚本跑到手软.




-----------

附件下载地址:

1.[打包脚本]()    

2.[Demo](https://github.com/HChong3210/MultipleChannel)    


-------------

参考文章:

1.[xcodebuild命令说明](http://www.jianshu.com/p/4f4d16326152)

2.[Fir.im](https://fir.im/)

3.[手把手教你给一个iOS app配置多个环境变量](http://www.jianshu.com/p/83b6e781eb51)

4.[Xcode 配置文件 xcconfig](http://www.jianshu.com/p/9a6f3019d81f)

5.[Xcode使用xcconfig文件配置环境](http://liumh.com/2016/05/22/use-xcconfig-config-specific-variable/)

6.[iOS Xcode使用xcconfig配置环境参数(Debug&Release)](http://blog.csdn.net/vbirdbest/article/details/53454014)

7.[The Unofficial Guide to xcconfig files](https://pewpewthespells.com/blog/xcconfig_guide.html)