---
title: UIView系列之---如何写一个自定义View
date: 2017-07-15 18:56:20
tags:
    - 面试题
    - 基础知识
    - UIView
categories:
    - 面试题
---

# UIView系列之---如何写一个自定义View

[UIView系列之---UIView和CALayer](http://hchong.net/2017/08/30/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E5%92%8CCALayer/)
[UIView系列之---UIView的常见layout方法](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E7%9A%84%E5%B8%B8%E8%A7%81layout%E6%96%B9%E6%B3%95/)
[UIView系列之---iOS的动态高度](http://hchong.net/2017/09/24/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---iOS%E7%9A%84%E5%8A%A8%E6%80%81%E9%AB%98%E5%BA%A6/)
[UIView系列之---如何写一个自定义View](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---%E5%A6%82%E4%BD%95%E5%86%99%E4%B8%80%E4%B8%AA%E8%87%AA%E5%AE%9A%E4%B9%89View/)

这一篇和前面的实际是一个系列, 但是有不太一样, 也稍微偏架构和规范一些. 说一下在实际编程中的写法. 

## 通用的自定义类的function归纳

一个自定义类中的代码首先是有序的, 不管是`UIVIewControl`, `UIVIew`, 或者`NSObject`, 都应该把具有相同作用的function归纳为一个类, 我一般按照下面这样来分割代码, 这样看上去会比较有条理: 

```
#pragma mark - LifeCycle
这里面是一些类的生命周期的方法, 以及overWrite的父类的方法
#pragma mark - UIConfig
这里面是和当前类的UI相关的设置信息
#pragma mark - HttpRequest
当前类的网络请求部分
#pragma mark - XXXDelegate
当前类响应的代理事件
#pragma mark - Action
当前类的EventResponse事件
#pragma mark - Private
当前类中所用到的一些工具function, 不过一般不建议写在这里面, 我们应该按照模块新建一个专门的工具类来管理这些function
#pragma mark - Getter, Setter
当前类中用到的所有属性的Getter和Setter, 强烈推荐这么些, 这样可以把当前类的子控件的初始化放到Getter中去, 维护了代码的整洁度.
```

除此之外, 上面这个顺序一般是按照使用者对这个类中所有function的关心程度来排序的, 例如对Lifecycle和UIConfig的关心程度就比Getter和Setter的程度高. 如果这个类还是特别长的话, 那就建议把代码在拆分为各个独立功能的category, 或者把尽量多的Private方法拆分为独立的工具类, 这样不管是对当前类的代码量, 或者后面可能会使用到这些PrivateFunction的人来说都是比较合理的做法. 

## 组合代替继承

除此之外, 对于复杂类来说我们要尽量*使用组合来代替继承*. 例如当前有一个类实现了A功能, 业务发展我们要使用A+B功能, 我们可能会想到写一个A的子类A', 在里面再加上B功能, 完美解决. 那么后面我们要再使用A+B+C功能, 那么我们可能会写一个A'的子类A'', 在里面实现C的功能, 又完美实现. 那如果突然有一天, 产品经理突然说要实现A+C功能呢, 傻眼了, 一大坨代码怎么拆分. 所以正确的做法应该是, 我们新建三个类, 分别独立实现A, B, C的功能, 业务方要用哪个就自由组合, 假设后面再来了D, E, F我们也不怕, 我们只用像插件一样, 用到哪个组合哪个就好. 


## UIView的写法

UIView作为直接展示给用户看的层面, 是最重要的部分. 我强烈推荐使用纯代码布局, 使用Frame或者Autolayout都可以. 使用Frame的话, 推荐使用[这个项目](https://github.com/casatwy/HandyAutoLayout). 如果使用AutoLayout的话, `masonry`则是很不错的选择. 

objc构建一个对象使用的是两段式, 首先分配内存`alloc`然后`init`, 这样的好处就是将内存操作和初始化操作解耦合, 让我们能够在初始化的时候对对象做一些必要的操作. 这是个很好的思路, 我们在做很多事情的时候都可以使用这种两段式的思路. 比如布局一个UIView, 我们可以分成两部, [初始化必要的子view和变量](#alloc), [然后在合适的时机进行布局](#init). 一般来说我们的自定义类继承自UIView, 首先在initWithFrame:方法中将需要的子控件加入view中. 注意, 这里只是加入到view中, 并没有设置各个子控件的尺寸.

### <span id = 'alloc'>初始化必要的子view和变量</span>

所以第一步应该是: `- (id)initWithFrame:(CGRect)aRect`, 那我们为什么不适用`- (id)init`来完成初始化呢? 如果是这种情况, 那么在init方法中frame是不确定的, 此时如果在initWithFrame:方法中设置尺寸, 那么各个子控件的尺寸都会是0, 因为这个view的frame还没有设置.所以我们应该保证view的frame设置完才会设置它的子控件的尺寸. 

还有就是这个函数是无论你用什么初始化函数都会被调用的一个, 比如你用`[UIView new]`或者`[[UIView alloc] init]`都会调用initWithFrame这个函数(有些UIView的子类有特殊情况，比如UITableViewCell), 所以你要是对一个view的变量有初始化的操作尽量往initWithFrame里面放还是非常合适的.  并且这样也能够保证, 以后在使用的时候所有的变量都被正确的初始化过. 而我们一般会在initWithFrame中做些什么呢.

* 添加子View
* 初始化属性变量
* 其他一些共用操作

### <span id = 'init'>在合适的时机进行布局</span>

初始化函数中有一个名称withFrame, 大家可能就会以为这个函数使用布局用的. 然而在代码逻辑比较清晰的工程中，几乎很少看到在这个函数中进行界面布局的工作, 因为UIKit给你提供了一个专门的函数`layoutSubViews`来干这个事情. 而且, 在这个函数中做的界面布局的工作, 是一次性编码, 界面布局没有任何复用性, 如果父View的大小变了之后, 这个View还是傻傻的保持原来的模样. 同时也会造成, 初始化函数臃肿, 导致维护上的困难. 所以在`layoutSubViews`中对子视图进行布局才是最合理的地方. 

## 其他

一个控件有2种创建方式: 

* 通过代码创建

初始化时一定会调用`initWithFrame:`方法. 

* 通过xib\storyboard创建

初始化时不会调用`initWithFrame:`方法, 只会调用`initWithCoder:`方法, 初始化完毕后会调用`awakeFromNib`方法注意要在在awakeFromNib中初始化子控件


-----

参考资料:

1.[iOS应用架构谈](https://casatwy.com/iosying-yong-jia-gou-tan-viewceng-de-zu-zhi-he-diao-yong-fang-an.html)

2.[视图类, 如何布局](https://yishuiliunian.gitbooks.io/implementate-tableview-to-understand-ios/content/uikit/1124.html)

