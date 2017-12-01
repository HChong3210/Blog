---
title: UIView系列之---iOS的动态高度
date: 2017-07-24 14:44:28
tags:
    - 面试题
    - 基础知识
    - UIView
categories:
    - 面试题
---

# UIView系列之---iOS的动态高度

[UIView系列之---UIView和CALayer](http://hchong.net/2017/08/30/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E5%92%8CCALayer/)
[UIView系列之---UIView的常见layout方法](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---UIView%E7%9A%84%E5%B8%B8%E8%A7%81layout%E6%96%B9%E6%B3%95/)
[UIView系列之---iOS的动态高度](http://hchong.net/2017/09/24/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---iOS%E7%9A%84%E5%8A%A8%E6%80%81%E9%AB%98%E5%BA%A6/)
[UIView系列之---如何写一个自定义View](http://hchong.net/2017/09/21/UIView%E7%B3%BB%E5%88%97%E4%B9%8B---%E5%A6%82%E4%BD%95%E5%86%99%E4%B8%80%E4%B8%AA%E8%87%AA%E5%AE%9A%E4%B9%89View/)

不论是UITableViewCell的高度也好, 或者是一个输入框也好, 都会用到动态布局. 实际上就是对一些显示文本内容的控件进行高度计算, 然后根据子视图的约束, 布局来得到父视图的高度并且改变他. 

## iOS布局机制大概分这么几种常见的方式: 

* frame layout. frame layout最简单直接, 即通过设置view的frame属性值进而控制view的位置(相对于superview的位置)和大小. 

* autoresizing. autoresizing和frame layout一样, 从一开始存在, 它算是对frame layout的的补充,, 基于autoresizing机制, 能够让subview和superview维持一定的布局关系, 譬如让subview的大小适应superview的大小,, 随着后者的改变而改变. 

    站在代码接口的角度来看, autoresizing主要体现在几个属性上, 包括(但不限于):

    * `translatesAutoresizingMaskIntoConstraints`. 标识view是否愿意被autoresize;

    * `autoresizingMask`. 是一个枚举值, 决定了当superview的size改变时, subview应该做出什么样的调整;

    autoresizing存在的不足是非常显著的, 通过autoresizingMask的可选枚举值可以看出: 基于autoresizing机制, 我们只能让view在superview的大小改变时做些调整; 而无法处理兄弟view之间的关系, 譬如处理与兄弟view的间隔; 更无法反向处理, 譬如让superview依据subview的大小进行调整. 

* Auto layout. Auto Layout是随着iOS 6推出来的, 它是一种基于约束的布局系统, 可以根据你在元素(对象)上设置的约束自动调整元素(对象)的位置和大小对于某个view的布局方式. 

    > Auto Layout is a system that lets you lay out your app’s user interface by creating a mathematical description of the relationships between the elements. You define these relationships in terms of constraints either on individual elements, or between sets of elements. Using Auto Layout, you can create a dynamic and versatile interface that responds appropriately to changes in screen size, device orientation, and localization.

autoresizing和auto layout只能二选一, 简单来说, 若要对某个view采用auto layout布局, 则需要设置其`translatesAutoresizingMaskIntoConstraints`属性值为NO.


## 常见Auto Layout场景下的运用

下面主要就几种常见场景下, Auto Layout的运用来说明一下用法. 在这里先说明一个概念 *Leaf-level views*, Leaf-level views指的是不包含任何subview的view, 譬如UILabel, UIButton等. 但是有些view不包含content, 譬如UIView, 这种view被认为「has no intrinsic size」, 它们的intrinsicContentSize返回的值是(-1, -1). 

###  Leaf-level views高度计算

这类的view往往能够直接计算出content(譬如UILabel的text, UIButton的title, UIImageView的image)的大小. 以UILabel为例:

假设我们已经设置了UILabel的x, y值约束, 没有设置与size有关的约束. 如果我们要根据UILabel的文本内容来计算最合适的size, 我们可以自定义一个Custom Label, 继承于UILabel, 在Custom Label中重写`- (CGSize)intrinsicContentSize`方法. 返回我们希望返回的size. 在需要使用UILabel的地方我们就可以通过使用Custom Label来实现搞得的正确计算. 

关于`intrinsicContentSize`方法的理解是, Auto Layout System在layout时, 不知道该为view分配多大的size, 因此回调view的`intrinsicContentSize`方法, 该方法会给auto layout system一个合适的size, system根据此size对view的大小进行设置; 

```
@interface CustomLabel : UILabel
    
@end
    
@implementation CustomLabel
    
- (CGSize)intrinsicContentSize {
    CGSize size = [super intrinsicContentSize];
    size.width  += 20;
    size.height += 20;
    return size;
}
    
@end
```

以上如果是单行label的话, 实现起来没问题. 但是如果label一行显示不下需要换行的话, 那事情就没这么简单了. 但是怎么计算多行label的高度呢? 有以下几种方法: 

下面几种方法都需要我们首先设置`preferredMaxLayoutWidth`, 也就是UILabel的Width最大值, label会根据这个最大值来换行. 再设置`numberOfLines = 0`, 来实现换行. ==注意:== `preferredMaxLayoutWidth`适用于没有指定UILabel的Width的情况, 如果设置了Width的约束, 又设置了`preferredMaxLayoutWidth`. 那么计算size会以`preferredMaxLayoutWidth`为准, 显示则以Width的约束为准.

1. `boundingRectWithSize:options:attributes:context:`

    `boundingRectWithSize:options:attributes:context:`是NSString的方法. 理解起来也非常简单, 根据一些绘制字符的选项和字符属性(字体, 字号, 字体颜色)等信息返回一个可以容纳字符串内容的CGRect. 它同样也需要一个CGSize来确定绘制区域. 
    size: 通常你可以传一个任意的Size, 它会返回一个它认为最合适的CGSize给你. 不过如果要想把视图的内容显示完全(纵向), 最好是将视图的实际宽度和最大高度`CGFLOAT_MAX`作为参数传递. 这样返回的才是完全显示内容的Size. 
    options: 默认情况下这个方法不会绘制多行, 如果要绘制多行字符, 那么options参数必须为: `NSStringDrawingUsesLineFragmentOrigin`. 
    attributes: 字符属性信息也非常重要. 如果要显示UILabel的全部内容, 必须传递这个参数. 以确保绘制的字体大小和UILabel的字体大小一致, 
    
    最后一个关键点是，这个方法返回的CGRect中Size的width和height都是小数，所以必须使用ceil函数才能确保结果的准确性。
    
2. <span id = 'sizeThatFits'>sizeThatFits:</span>

    > Asks the view to calculate and return the size that best fits the specified size.
 
    > Return Value
    > A new size that fits the receiver’s subviews.
 
    > Discussion
    > The default implementation of this method returns the existing size of the view. Subclasses can override this method to return a custom value based on the desired layout of any subviews. For example, a UISwitch object returns a fixed size value that represents the standard size of a switch view, and a UIImageView object returns the size of the image it is currently displaying.

    sizeThatFits: 方法意味着「根据文本计算最适合的size」, 但是并不改变调用者的size. 它需要传入一个CGSize参数这个参数和`boundingRectWithSize:options:attributes:context:`中size的作用和意义是一样的. 

3. <span id = 'sizeToFit'>sizeToFit:</span>

    > calls sizeThatFits: with current view bounds and changes bounds size. 
    
    `sizeToFit`内部会调用`sizeThatFits:`方法, 然后改变调用者的size. sizeToFit的伪代码大致如下：

    ```
// calls sizeThatFits
CGSize size = [self sizeThatFits:self.bounds.size];
// change bounds size
CGRect bounds = self.bounds;
bounds.size.width = size.width;
bounds.size.height = size.width;
self.bounds = bounds;
    ```

4. <span id = 'systemLayoutSizeFittingSize'>systemLayoutSizeFittingSize</span> 

    `systemLayoutSizeFittingSize`, 它也是UIView的方法, 是AutoLayout诞生后的产物. 所以使用它的前提是需要展示内容的控件(这里指的就是UILabel)必须约束完美. 不然就不会起作用。而且必须要设置UILabel的`preferredMaxLayoutWidth`属性. 
    这个属性非常重要, 它影响着layout. 如果设置了`preferredMaxLayoutWidth`, 当内容超过约束区域, 就会自动换行并且更新约束. 在良好约束的前提下, `systemLayoutSizeFittingSize`同样接受一个CGSize, 不同的是这次不用计算了, 直接使用系统提供的Fitting Size即可:

    ```
const CGSize UILayoutFittingCompressedSize; //在保证适当尺寸的前提下尽量压缩CGSize的大小
const CGSize UILayoutFittingExpandedSize; //在保证适当尺寸的前提下尽量扩充CGSize的大小
    ```

    所以为了刚好将UILabel的内容显示完全，应该使用UILayoutFittingCompressedSize。代码如下：

    ```
- (void)layoutSubviews {
    [super layoutSubviews];
    self.label.preferredMaxLayoutWidth = CGRectGetWidth(self.label.bounds);
    CGSize size = [self.label systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    self.labelConstraintHeight.constant = size.height + (2 * MARGIN);
}
    ```

    需要注意的是: 约束的上下左右一定要写好, 但是不能约束UILabel的高度. 否则可能会导致返回的CGRect不准确. `numberOfLines = 0` 让Label可以显示多行内容. 设置`preferredMaxLayoutWidth`属性, 使UILabel能自适应多行内容. `UILayoutFittingCompressedSize` 使用这个参数会返回符合条件最合适的Size. 最后也要加上边距, 主要是因为这里我们在内部计算UILabel的Size, 而如果在外部对View调用`systemLayoutSizeFittingSize`方法, 就会得到整个View视图的Size. 

`sizeThatFits:`和`boundingRectWithSize:options:attributes:context:`这两个API也可以在传统布局(基于Frame)的情况下使用. 

### 非Leaf-level views高度计算

以UITextView显示文本为例, 让其能够自适应文本, 即根据文本自动调整其大小; 由于`intrinsicContentSize`的特性, 当其内部含有subView时返回值是(-1, -1), 无法向auto layout system传递我们想要传达的值, 我们可以使用[sizeThatFits](#sizeThatFits)或者[sizeToFit](#sizeToFit)来计算或者改变UITextView的大小. 

==这里需要注意的是:== 当调用`sizeThatFits:`的size=(width, height)，当width/height的值为0时，width/height就被认为是无穷大, size就不能被正常的显示. 所以区别于UILabel, 我们的约束一定要设置好Width和height, 否则使用`sizeToFit`也不能正确计算出. 

### Leaf-level views和非Leaf-level views混合使用下高度计算

这里我们需要计算的是根据subView来确定superView的frame, 大致分为一下几种情况: 

1. 多个纯Leaf-level views的组合使用

    这个计算起来比较简单, 我们以一个UIView里面添加两个UILabel为例. 我们只需要从上到下, 把subViews的子约束撑满superView. 需要注意, 如果UILabel需要换行, 那么高度约束就不能写, 并且要设置`numberOfLines = 0`. 虽然我们什么也没做, 但是子控件会通过`intrinsicContentSize`方法将最合适的size告诉superView. 但是对于superView, 因为它本身是UIVIew, 他的`intrinsicContentSize`返回是(-1, -1), 那么它是怎么得出正确的结果呢. auto layout system在处理某个view的size时，<span id = 'size'>参考值</span>包括:
    
    * 自身的`intrinsicContentSize`方法返回值;
    * subviews的`intrinsicContentSize`方法返回值;
    * 自身和subviews的constraints;

    系统会将superView的size和subViews的约束以及`intrinsicContentSize`返回的正确size相加, 然后比较两个值的大小, 然后取最大的一个. 
    ![约束](https://ws1.sinaimg.cn/large/006tKfTcgy1fjvnn0tr2lj30ok0ru0st.jpg)
    
    size1, size2, size3, 分别是label1, label2, superView的`intrinsicContentSize`方法返回的size.
    width = max{91 + size1.width + 91 + size2.width, size3.width}
    height = max{60 + size1.height + 36 + size2.height + 58, size3.height}

2. 多个纯非Leaf-level views的组合使用

    仍旧以UITextView为例, 如果没有设定UItextView的height属性的话, 由于非Leaf-level views的`intrinsicContentSize`返回值值为(-1, -1). 设置某个View的size有三个[参考值](#size), 在`intrinsicContentSize`方法返回值和constraints中取最大值为0, 导致height=0不能正常显示. 解决方案参考[下一个](#next);

3. <span id = 'next'>Leaf-level views和非Leaf-level views混合使用</span>

    因为有UItextView这种非Leaf-level views的存在, 会导致superView的size不能得到正常值. 解决方案有两种:
    
    1. 设置非Leaf-level views的height, width约束, 因为`intrinsicContentSize`方法不能正常的返回size, 但是如果我们设置了width和height约束, constraints就不为0, 那么就可以正常显示了. 

    2. 使用`systemLayoutSizeFittingSize:`, 具体使用方式可以参考[systemLayoutSizeFittingSize:](#systemLayoutSizeFittingSize).

### Cell的动态高度计算

对于使用auto layout机制布局的view, auto layout system会在布局过程中综合各种约束的考虑为之设置一个size, 在布局完成后, 该size的值即为view.frame.size的值; 这包含的另外一层意思, 即在布局完成前, 我们是不能通过view.frame.size准确获取view的size的. 但有时候, 我们需要在auto layout system对view完成布局前就知道它的size, `systemLayoutSizeFittingSize:`方法正是能够满足这种要求的API. `systemLayoutSizeFittingSize:`方法会根据其constraints返回一个合适的size值. 

在这里看一下比较知名的cell高度计算库[UITableView-FDTemplateLayoutCell](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell)的核心高度计算方法, 大体上和我们所使用的方法差不多. Auto layout mode using `-systemLayoutSizeFittingSize:`, Frame layout mode using `-sizeThatFits:`. 

```
- (CGFloat)fd_systemFittingHeightForConfiguratedCell:(UITableViewCell *)cell {
    CGFloat contentViewWidth = CGRectGetWidth(self.frame);
    
    // If a cell has accessory view or system accessory type, its content view's width is smaller
    // than cell's by some fixed values.
    if (cell.accessoryView) {
        contentViewWidth -= 16 + CGRectGetWidth(cell.accessoryView.frame);
    } else {
        static const CGFloat systemAccessoryWidths[] = {
            [UITableViewCellAccessoryNone] = 0,
            [UITableViewCellAccessoryDisclosureIndicator] = 34,
            [UITableViewCellAccessoryDetailDisclosureButton] = 68,
            [UITableViewCellAccessoryCheckmark] = 40,
            [UITableViewCellAccessoryDetailButton] = 48
        };
        contentViewWidth -= systemAccessoryWidths[cell.accessoryType];
    }
    
    // If not using auto layout, you have to override "-sizeThatFits:" to provide a fitting size by yourself.
    // This is the same height calculation passes used in iOS8 self-sizing cell's implementation.
    //
    // 1. Try "- systemLayoutSizeFittingSize:" first. (skip this step if 'fd_enforceFrameLayout' set to YES.)
    // 2. Warning once if step 1 still returns 0 when using AutoLayout
    // 3. Try "- sizeThatFits:" if step 1 returns 0
    // 4. Use a valid height or default row height (44) if not exist one
    
    CGFloat fittingHeight = 0;
    
    if (!cell.fd_enforceFrameLayout && contentViewWidth > 0) {
        // Add a hard width constraint to make dynamic content views (like labels) expand vertically instead
        // of growing horizontally, in a flow-layout manner.
        NSLayoutConstraint *widthFenceConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:contentViewWidth];
        [cell.contentView addConstraint:widthFenceConstraint];
        
        // Auto layout engine does its math
        fittingHeight = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
        [cell.contentView removeConstraint:widthFenceConstraint];
        
        [self fd_debugLog:[NSString stringWithFormat:@"calculate using system fitting size (AutoLayout) - %@", @(fittingHeight)]];
    }
    
    if (fittingHeight == 0) {
#if DEBUG
        // Warn if using AutoLayout but get zero height.
        if (cell.contentView.constraints.count > 0) {
            if (!objc_getAssociatedObject(self, _cmd)) {
                NSLog(@"[FDTemplateLayoutCell] Warning once only: Cannot get a proper cell height (now 0) from '- systemFittingSize:'(AutoLayout). You should check how constraints are built in cell, making it into 'self-sizing' cell.");
                objc_setAssociatedObject(self, _cmd, @YES, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
            }
        }
#endif
        // Try '- sizeThatFits:' for frame layout.
        // Note: fitting height should not include separator view.
        fittingHeight = [cell sizeThatFits:CGSizeMake(contentViewWidth, 0)].height;
        
        [self fd_debugLog:[NSString stringWithFormat:@"calculate using sizeThatFits - %@", @(fittingHeight)]];
    }
    
    // Still zero height after all above.
    if (fittingHeight == 0) {
        // Use default row height.
        fittingHeight = 44;
    }
    
    // Add 1px extra space for separator line if needed, simulating default UITableViewCell.
    if (self.separatorStyle != UITableViewCellSeparatorStyleNone) {
        fittingHeight += 1.0 / [UIScreen mainScreen].scale;
    }
    
    return fittingHeight;
}

```

[这篇文章](http://www.cocoachina.com/industry/20140604/8668.html)也有一些常见cell布局的高度计算方法, 可以参考下.

-------

参考资料:

1.[深入理解Auto Layout](http://zhangbuhuai.com/beginning-auto-layout-part-1/).

2.[动态计算UITableViewCell高度](http://www.cocoachina.com/industry/20140604/8668.html)

3.[优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

4.[iOS动态变高总结](http://sylenthwave.github.io/2016/01/02/iOS%E5%8A%A8%E6%80%81%E5%8F%98%E9%AB%98%E6%80%BB%E7%BB%93/)


