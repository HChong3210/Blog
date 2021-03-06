---
title: UIView系列之---UIView和CALayer
date: 2017-07-07 21:04:12
tags:
    - 面试题
    - 基础知识
    - UIView
categories:
    - 面试题
---

# UIView系列之---UIView和CALayer

[UIView系列之---UIView和CALayer](http://hchong.net/2017/08/30/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E5%92%8CCALayer/)
[UIView系列之---UIView的常见layout方法](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E7%9A%84%E5%B8%B8%E8%A7%81layout%E6%96%B9%E6%B3%95/)
[UIView系列之---iOS的动态高度](http://hchong.net/2017/09/24/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---iOS%E7%9A%84%E5%8A%A8%E6%80%81%E9%AB%98%E5%BA%A6/)
[UIView系列之---如何写一个自定义View](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---%E5%A6%82%E4%BD%95%E5%86%99%E4%B8%80%E4%B8%AA%E8%87%AA%E5%AE%9A%E4%B9%89View/)

## UIView和CALayer

1. `UIView`继承自`UIResponder`, 可以相应触摸事件. 而`CALayer`继承自`NSObject`, 不能响应触摸.

2. `UIView`主要是对显示内容的管理, 而`CALayer`则主要侧重显示内容的绘制. 访问`UIVIew`的与绘图和跟坐标有关的属性, 例如`frame`, `bounds`等, 实际上内部都是在访问它所包含的CALayer的相关属性. `UIView`的`frame`实际是`layer`的`frame`, `center`, `bounds`实际也是只是内部`layer`相对应属性的`get`和`set`方法, 当你改变一个`view`的`frame`的时候, 你其实改变的是内部`layer`的`frame`. CALayer的frame由`anchorPoint`, `position`, `bounds`, `transform`共同决定. `UIView`的创建, 实际上是一系列`UILayer`创建的过程.

3. `UIView`有个重要属性`layer`. 可以返回它的主`CALayer`实例. 所有从UIView继承来的对象都继承了这个属性. 这意味着你可以在所有的`UIVIew`子类上增加动画, 旋转, 缩放等`CALayer`支持的操作. `UIView`的`layerClass`方法, 可以返回主`layer`所使用的类, `UIView`的子类可以通过重载这个方法, 来让`UIView`使用不同的CALayer来显示. 代码示例：

    ```
    - (class)layerClass {
        return ([CAEAGLLayer class]);
    }
    ```

4. `UIView`的主`CALayer`是类似于`subviews`的树形结构, 我们可以通过给主`layer`添加子`layer`来完成特殊的绘制效果.
    
    在`view`上添加一个黑色透明`layer`层的示例代码:
    
    ```
    grayCover = [[CALayer alloc] init];
    grayCover.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.2] CGColor];
    [self.layer addSubLayer:grayCover];
    ```

5. `UIView`的内部, 有三个`layer tree`: 1.逻辑树, 这里是代码可以操纵的. 2.动画树, 是一个中间层, 系统就在这一层上通过逻辑树来更改属性, 进行各种渲染操作. 3.显示树, 其内容就是当前正被显示在屏幕上得内容. 

6. `UIView`实际上是`CALayer`的`CALayerDelegate`, 通过实现一系列的代理方法来显示`CALayer`绘制的内容.

7. 在做 iOS 动画的时候, 修改非`RootLayer`的属性(譬如位置, 背景色等)会默认产生隐式动画, 而修改`UIView`则不会.

8. `layer`可以设置圆角显示(cornerRadius), 也可以设置阴影(shadowColor). 但是如果`layer`树中某个`layer`设置了圆角, 树种所有`layer`的阴影效果都将不显示了. 因此若是要有圆角又要阴影, 变通方法只能做两个重叠的`UIView`, 一个的`layer`显示圆角, 一个`layer`显示阴影.

9. `UIView` 是`UIKit`框架下的(只能iOS使用). `CALayer` 是`QuartzCore`的(iOS 和macOS通用).

10. `QuartzCore`的渲染能力. 使二维图像可以被自由操纵, 就好像是三维的. 图像可以在一个三维坐标系中以任意角度被旋转, 缩放和倾斜. `CATransform3D`的一套方法提供了一些魔术般的变换效果. 

---------

参考资料:

1.[详解CALayer 和 UIView的区别和联系](http://www.jianshu.com/p/079e5cf0f014)

2.[UIView和CALayer的区别](http://blog.csdn.net/weiwangchao_/article/details/7771538)


