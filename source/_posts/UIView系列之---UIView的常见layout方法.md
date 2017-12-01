---
title: UIView系列之---UIView的常见layout方法
date: 2017-07-10 18:55:48
tags:
    - 面试题
    - 基础知识
    - UIView
categories:
    - 面试题
---

## UIView系列之---UIView的常见layout方法

* `init` & `initWithFrame`

    `init` 和 `initWithFrame`方法, 实际上都会调用`initWithFrame`方法来完成初始化, 不同的是在`init`方法内部获取到的`self.frame`是`CGRectZero`, 在`initWithFrame`中获取到的`self.frame`就是初始化时, 传入的`frame`的大小. 不过一般不建议在初始化方法中设置子控件的`frame`, 因为这时`self.frame`时机还不是固定的.
    
* layoutSubviews

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622482-layoutsubviews?language=objc)
> Lays out subviews.
The default implementation of this method does nothing on iOS 5.1 and earlier. Otherwise, the default implementation uses any constraints you have set to determine the size and position of any subviews.

> Subclasses can override this method as needed to perform more precise layout of their subviews. You should override this method only if the autoresizing and constraint-based behaviors of the subviews do not offer the behavior you want. You can use your implementation to set the frame rectangles of your subviews directly.

> You should not call this method directly. If you want to force a layout update, call the setNeedsLayout method instead to do so prior to the next drawing update. If you want to update the layout of your views immediately, call the layoutIfNeeded method.

翻译成人话大概就是: `layoutSubviews`方法用来布局子视图. 在`layoutSubviews`方法内部直接给需要改变布局的子视图赋值新frame来改变其`frame`. 该方法主要应用在封装的自定义`view`中, 我们通过重写这个方法完成自定义`view`中子视图的布局. 但是我们不能直接调用他来更新子视图的`frame`. 只能通过[layoutIfNeeded](#layoutIfNeeded)或者[setNeedsLayout](#setNeedsLayout)来调用, 或者等待系统触发. 系统触发`layoutSubviews`的[条件](#layoutSubviews).

* <span id = 'setNeedsLayout'>setNeedsLayout</span>

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622601-setneedslayout?language=objc)
> Invalidates the current layout of the receiver and triggers a layout update during the next update cycle.

> Call this method on your application’s main thread when you want to adjust the layout of a view’s subviews. This method makes a note of the request and returns immediately. Because this method does not force an immediate update, but instead waits for the next update cycle, you can use it to invalidate the layout of multiple views before any of those views are updated. This behavior allows you to consolidate all of your layout updates to one update cycle, which is usually better for performance.

翻译成人话大概就是: 如果一个layer的sublayer布局发生了改变需要更新布局, 我们通过调用`setNeedsLayout`方法来标记这个layer. 或者当layer的bounds发生变化或者layer上进行了add或者remove sublayer操作, 系统会自动调用`setNeedsLayout`方法. 这些被标记需要更新布局的layer会在下一个视图绘制周期(iOS屏幕刷新频率为60HZ, 因此下一个视图绘制周期是1/60s后)触发layoutSubviews完成子视图布局更新.

如果你想要更新一个视图的子视图布局, 那么可以在主线程中调用这个方法来标记当前视图为需要更新子视图布局的视图. 可以使用这个方法给多个不同的view标记需要更新子视图布局, 然后在下一个视图绘制周期中这些view的subviews就会被一块更新布局, 这样做是可以很大地提高性能和效率的. 

* <span id = 'layoutIfNeeded'>layoutIfNeeded</span>

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622507-layoutifneeded?language=objc)
> Lays out the subviews immediately.
Use this method to force the layout of subviews before drawing. Using the view that receives the message as the root view, this method lays out the view subtree starting at the root.

翻译成人话大概就是: 在下一次绘图周期开始之前使用此方法强制进行子视图的布局更新. 此方法会将receiver作为根视图, 然后从根视图开始遍历根视图的subview链, 判断super layer是否被标记需要更新布局, 直到找到一个super layer没有标记更新布局为止, 然后系统会向所有这些被标记需要更新布局的layer发送`layoutSublayers`消息. `layoutSublayers`是`setNeedsLayout`的一个辅助方法, 调用该方法就意味着不会等到下个绘制周期, 而是立马触发`layoutSubviews`方法, 完成子视图的布局. 

但是需要注意只有当系统检测到某个view被`setNeedsLayout`标记之后才会立即触发`layoutSubviews`, 如果没有检测到`setNeedsLayout`标记就不会触发`layoutSubviews`, 所以如果想要立即刷新某个视图的子视图布局, 需要先让该视图调用`setNeedsLayout`方法标记一下, 然后再调用`layoutIfNeeded`. 另外所有的视图在第一次显示之前都是默认有`setNeedsLayout`标记的, 所以视图第一次显示的时候就可以直接调用`layoutIfNeeded`. 但是当第一次显示完成后, 如果还想要调用`layoutIfNeeded`就必须先使用`setNeedsLayout`标记一下. 

* setNeedsDisplay

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622437-setneedsdisplay?language=objc)
> Marks the receiver’s entire bounds rectangle as needing to be redrawn. 
> You can use this method or the `setNeedsDisplayInRect:` to notify the system that your view’s contents need to be redrawn. This method makes a note of the request and returns immediately. The view is not actually redrawn until the next drawing cycle, at which point all invalidated views are updated.

>其他说明:
> You should only be calling setNeedsDisplay if you override drawRect in a subclass of UIView which is basically a custom view drawing something on the screen, like lines, images, or shapes like a rectangle.

如果我们在自定义的UIView中, 重写了`drawRect:`方法在屏幕上绘制一些东西, 那就需要在合适的地方调用`setNeedsDisplay`来标记当前视图, 系统会在下一个绘制周期时触发`drawRect:`方法. [系统触发drawRect:的条件](#drawRect).

也可以使用`setNeedsDisplayInRect:`方法来标记视图的某个区域需要重新绘制.
    
* drawRect

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622529-drawrect)
> Draws the receiver’s image within the passed-in rectangle

> The default implementation of this method does nothing. Subclasses that use technologies such as Core Graphics and UIKit to draw their view’s content should override this method and implement their drawing code there. You do not need to override this method if your view sets its content in other ways. For example, you do not need to override this method if your view just displays a background color or if your view sets its content directly using the underlying layer object.

> This method is called when a view is first displayed or when an event occurs that invalidates a visible part of the view. You should never call this method directly yourself. To invalidate part of your view, and thus cause that portion to be redrawn, call the `setNeedsDisplay` or `setNeedsDisplayInRect:` method instead.
 
drawRect使用:

该方法默认没有做任何操作, 如果视图中包含我们用`UIKit`或者`Core Graphics`绘制的内容, 我们需要重写该方法. 当视图第一次出现, 或者是改变约束条件让视图的全部或者一部分在屏幕上发生变化时, 系统都会调用`UIView`类的`drawRect`方法. 然后我们在此方法中能过获取到当前图形上下文, 实现我们的绘制内容, 最后系统会在合适的时机自动调用此方法. 

`drawRect`一般调用是在`UIView`的`layoutSubviews`方法执行后. 但是, 在我们的视图全部初始化后,如果视图又发生了改变, 此时视图就需要重绘, 但是系统不会再帮我们自动调用`drewRect`方法. 这个时候就需要我们手动调用`UIView`类的 `setNeedsDisplay`或`setNeedsDisplayInRect`方法. 这两个方法是用来告诉系统, 我们的视图有了更新需要去重绘. 相当于是给系统做了标记, 在系统 runloop 的下一个周期自动调用`drawRect`方法.

使用中要注意的地方:

1. 不要直接调用 drawRect 方法,如果强行调用此方法也是无效果的.苹果要求我们调用 UIView 类的 setNeedsDisplay 方法,则程序会自动调用 drawRect 方法进行重绘.

2. 因为在绘制时要拿到图形上下文,如果在 UIView 初始化时没有设置 rect 大小, drawRect 方法不会被调用.

3. 调用 sizeThatFits 后, 控件 frame 改变, UIView 的 layoutSubviews 被调用, 然后再调用 drawRect 方法. 所以可以先调用 sizeToFit 计算出size. 然后系统自动调用 drawRect 方法.

4. 通过设置 contentMode 属性值为 UIViewContentModeRedraw.那么将在每次设置或更改 bounds 的时候自动调用 drawRect.

5. 若要实时画图, 如果使用 gestureRecognizer 来刷新屏幕, 需要判断并转化 point 的坐标; 使用 touchbegan 等方法, 只需调用 setNeedsDisplay 实时刷新屏幕.

这里有一篇[使用drawRect:实现绘制画板功能的内存优化](http://bihongbo.com/2016/01/03/memoryGhostdrawRect/)的文章.

* sizeToFit

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622630-sizetofit?language=objc)
> Resizes and moves the receiver view so it just encloses its subviews.

> Call this method when you want to resize the current view so that it uses the most appropriate amount of space. Specific UIKit views resize themselves according to their own internal needs. In some cases, if a view does not have a superview, it may size itself to the screen bounds. Thus, if you want a given view to size itself to its parent view, you should add it to the parent view before calling this method.
> You should not override this method. If you want to change the default sizing information for your view, override the sizeThatFits: instead. That method performs any needed calculations and returns them to this method, which then makes the change.

当我们想要resize当前View以便获取他合适的大小时, 我们需要调用该方法. 尤其是`UIKit`的`View`视图是根据内部需要进行尺寸调整时. 在某些情况下, 如果当前View没有父视图, 他会根据屏幕的bounds来resize自身大小. 如果你想让一个View根据父视图来调整大小, 必须将该View添加到父视图中.

一般情况下, 我们不需要重写该方法, 如果你想改变当前View的default size, 我们通过重写[sizeThatFits:](#sizeThatFits)来实现. 在`sizeThatFits:`方法中进行必要的计算, 返回结果, 然后改变他的大小.

* <span id = 'sizeThatFits'>sizeThatFits</span>

> [官方文档](https://developer.apple.com/documentation/uikit/uiview/1622625-sizethatfits?language=objc)
> Asks the view to calculate and return the size that best fits the specified size.

> Parameters
size
The size for which the view should calculate its best-fitting size.

> Return Value
A new size that fits the receiver’s subviews.

> Discussion
The default implementation of this method returns the existing size of the view. Subclasses can override this method to return a custom value based on the desired layout of any subviews. For example, a UISwitch object returns a fixed size value that represents the standard size of a switch view, and a UIImageView object returns the size of the image it is currently displaying.
This method does not resize the receiver.

该方法要求View计算并返回最适合指定大小的大小. 传入的参数就是View需要最合适的大小, 返回值是根据传入的大小, 计算得到的一个最适合receiver子视图的尺寸. 

该方法默认返回视图的现有大小, 子类能够通过重写该方法获得一个基于该子类所有子视图的期望布局的自定义大小. 调用`sizeThatFits:`并不改变view的size, 它只是让view根据已有content和给定size计算出最合适的view.size. 

## sizeToFit vs sizeThatFits:

1. sizeToFit会自动调用sizeThatFits方法；
2. sizeToFit不应该在子类中被重写, 应该重写sizeThatFits
3. sizeThatFits传入的参数是receiver当前的size, 返回一个适合subviews的size
4. sizeToFit可以被手动直接调用, 
5. sizeToFit和sizeThatFits方法都没有递归，对subviews也不负责，只负责自己
6. 调用 sizeToFit() 会去自动调用 sizeThatFits(_ size: CGSize) 方法。
7. sizeThatFits 不会改变 receiver 的 size, 调用 sizeToFit() 会改变 receiver 的 size. 此处的receiver一般是方法的调用者. 

## <span id = 'layoutSubviews'>系统触发layoutSubviews的条件</span>

1. 父视图使用`init`方法完成初始化时不会触发`layoutSubviews`. 
2. 父视图用`initWithFrame`完成初始化并且当frame参数为`CGRectZero`时不会触发`layoutSubviews`. 当frame参数不为`CGRectZero`时则会触发`layoutSubviews`.
3. 父视图`setFrame`的时候会触发`layoutSubviews`, 当然frame前后值得发生变化.
4. 父视图`addSubview`添加子视图时会触发其内部的`layoutSubviews`.
5. 子视图从父视图上`removeFromSuperView`的时候会触发其内部的`layoutSubviews`. 
6. 滚动ScrollView的时候会触发`layoutSubviews`. 
7. 旋转屏幕的时候会触发`layoutSubviews`. 

## <span id = 'drawRect'>系统出发drawRect:的条件</span>

1. 如果在`UIView`初始化时没有设置rect大小, 将直接导致`drawRect`不被自动调用. `drawRect`调用是在Controller->loadView, Controller->viewDidLoad 两方法之后掉用的. 所以不用担心一进入到控制器中, 这些View的drawRect就开始画了. 这样可以在控制器中设置一些值给View(如果这些View draw的时候需要用到某些变量值).
2. 该方法在调用`sizeToFit`后被调用, 所以可以先调用`sizeToFit`计算出size, 然后系统自动调用drawRect:方法. 
3. 通过设置`contentMode`属性值为`UIViewContentModeRedraw`. 那么将在每次设置或更改frame的时候自动调用`drawRect:`.
4. 直接调用`setNeedsDisplay`或者`setNeedsDisplayInRect:`触发`drawRect:`. 但是有个前提条件是rect不能为`CGRectZero`.
以上1, 2推荐; 而3, 4不提倡. 


---------

参考资料:

1.[Core Animation 之 CALayer](http://www.jianshu.com/p/087529c83747)

2.[UIView几个layout方法的理解](http://www.jianshu.com/p/3282c93c1a61)

3.[UIView布局深入理解](http://www.jianshu.com/p/b3bb9b08e3da)

4.[UIViewLayout的几个方法](http://www.jianshu.com/p/eb2c4bb4e3f1)

5.[深入理解Auto Layout 第一弹](http://zhangbuhuai.com/beginning-auto-layout-part-1/)

