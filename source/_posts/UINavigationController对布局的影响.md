---
title: UINavigationController对布局的影响
date: 2019-08-13 09:59:24
tags:   
    - 基础知识
categories:
    - 基础知识
---

iOS7 之后，所有的 UINavigationBar 默认都是透明的了，同时 View Controller 全部都使用全屏的 layout。为了提供更多调整 view 的选项，苹果又引入了 edgesForExtendedLayout， extendedLayoutIncludesOpaqueBars，automaticallyAdjustsScrollViewInsets 这几个属性用于控制 VC 的 view layout. 下面就来详细说一下他们对页面布局的影响.

# 1. 新建一个空工程
创建一个 Single View 的工程，设置 Navigation Controller 和 rootViewController, 运行结果可以看到:

* 导航栏式透明的, 这就是iOS 7之后的毛玻璃效果
* View 是在全屏幕范围内进行布局的.

这就导致: 

* 我们可以透过导航栏看到下层的背景色
* 如果我们添加一个控件的frame为`CGRectMake(0, 0, 100, 40)`就会导致完全被导航栏挡住. 想解决这个问题，最简单的办法就是调整 Label 的 frame. 但是Apple已经提供了解决方案.

# 2. Apple的解决方案
针对这种情况, Apple提供了下面几个属性来解决这种问题.

## 2.1 edgesForExtendedLayout
edgesForExtendedLayout属性指示了下层 view 扩展 layout 的方向, 系统默认值为`UIRectEdgeAll`, 就是会向各个方向扩展(其实在这里能扩展的方向也只有上方). 我们可以通过更改这个属性解决上面的问题.

* 在navigationBar透明的情况下, 控制器的根View会向四周延伸, 我们可以通过设置`edgesForExtendedLayout`来控制延伸情况.
* 在navigationBar不透明的情况下, 控制器的根View不会向四周延伸.

## 2.2 navigationBar.translucent
translucent属性值会决定导航栏是否有半透明效果. translucent为NO, 意味着导航栏为非透明. 我们可以通过更改这个属性解决问题.

translucent会受navigationBar的backgroudImage属性的影响。也就是说当你使用了一张自定义图片作为navigationBar的背景图时，translucent的值将由系统根据该图片是否颜色值透明，来推断translucent是YES还是NO.

```OC
UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:vc];
nav.navigationBar.translucent = NO;
```

当navigationBar.translucent = NO时，两种方式给NavigationBar设置颜色:

```OC
self.navigationController.navigationBar.tintColor = [UIColor redColor];

[self.navigationController.navigationBar setBackgroundImage:[UIImage imageNamed:@"1.png"] forBarMetrics:(UIBarMetricsDefault)];
```

## 2.3 extendedLayoutIncludesOpaqueBars

该属性的意思为在bar不透明的情况下根View是否进行延伸, 该属性只能使用在bar不透明的情况下, 在navgationBar透明的情况下, 设置无效.

在navgationBar不透明的情况下edgesForExtendedLayout与extendedLayoutIncludesOpaqueBars需要配合使用才可以达到想要的效果.

* UIRectEdgeNone + YES, view不会向四周延伸
* UIRectEdgeAll + NO, view不会向四周延伸 
* UIRectEdgeAll + YES, view会向四周延伸 
* UIRectEdgeNone + NO, view不会向四周延伸

## 2.4 automaticallyAdjustsScrollViewInsets
scrollView和其子类是否有系统自动调整子控件位置(即如果scrollView被bar遮挡时, 子控件自动下移一定距离, 保证内容不会被覆盖).

* scrollView与其子类必须放在控制器的根View上并且automaticallyAdjustsScrollViewInsets设置为YES时, 系统才会为我们自动调整
* 在iOS11中automaticallyAdjustsScrollViewInsets已经失效了, 被替换成需scrollView中的contentInsetAdjustmentBehavior参数
* 以tableView为例，当你使用默认的创建方式，也就是UIRectEdgeAll+导航栏半透明的情况下，首行cell的位置将处于导航栏下方，也就是屏幕坐标系的(0, 64)位置处，但是此时上滑将会形成穿透效果导航栏的效果.

## 2.5 坑
对于导航控制器中的各个childViewController，是共用同一个的navigationBar。当你在一个childViewController中自定义了navigationBar的背景图片，或是直接改变了translucent属性，此时再push或pop到另一个childViewController时，更改导航栏的半透明效果可能会影响到页面的布局起始位置，从而发生视图发生跳动，出现“意外”的上下偏移.

例如: 从一个设置了导航栏不透明的控制器A, pop回到一个原本设置了导航栏透明的控制器B时, B页面发生了下移.

为避免该情况, 应该将控制器B的extendedLayoutIncludesOpaqueBars设置为YES; 或是当B页面viewWillAppear:时, 再度将导航栏设置为半透明效果.

# 3. 总结

```OC
self.navigationController.navigationBar.translucent = NO;
self.extendedLayoutIncludesOpaqueBars = NO;
self.edgesForExtendedLayout = UIRectEdgeNone;
```

* NO + NO + UIRectEdgeNone, 从导航栏下开始计算
* NO + NO + UIRectEdgeAll, 从导航栏下开始计算
* NO + YES + UIRectEdgeNone, 从导航栏下开始计算
* NO + YES + UIRectEdgeAll, 从屏幕顶端开始计算, 导航栏不透明
* YES + NO + UIRectEdgeNone, 从导航栏下开始计算, 但是导航栏是透明色, 会显示底色(黑色)
* YES + NO + UIRectEdgeAll, 从屏幕顶端开始计算, 导航栏是透明色, 显示页面背景色
* YES + YES + UIRectEdgeNone, 从导航栏下开始计算, 导航栏是透明色, 会显示底色(黑色)
* YES + YES + UIRectEdgeAll, 从屏幕顶端开始计算, 导航栏是透明色, 显示页面背景色

总结效果如下:

1. `navigationBar.translucent = NO`时, 导航栏不透明, `extendedLayoutIncludesOpaqueBars`属性生效, 当`extendedLayoutIncludesOpaqueBars = YES && UIRectEdgeAll`时从屏幕下开始计算, 其他均是从导航栏下开始计算
2. `navigationBar.translucent = YES`时, `extendedLayoutIncludesOpaqueBars`属性不生效, 只有当`UIRectEdgeAll`时从屏幕顶端开始计算, 其他均从导航栏下开始计算.

-------
参考资料:
1.[屏幕适配](https://www.cnblogs.com/Zp3sss/p/9108613.html)
2.[UINavigationBar 透明设置以及对 frame 的影响](https://skyline75489.github.io/post/2015-11-27_uinavigation_bar_frame_affect.html)
3.[影响导航控制器中页面布局的几个属性](https://youshaoduo.com/2017/12/13/26/)

