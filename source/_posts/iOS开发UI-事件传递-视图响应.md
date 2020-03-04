---
title: iOS开发UI-事件传递&视图响应
date: 2018-04-08 22:26:45
tags:
    - 基础知识
categories: 
    - iOS开发-UI
---

iOS开发中有大量的人机交互事件, 我们怎么来处理这些人机交互, 就需要代表用户操作事件的传递和视图的响应来完成整个传递链.
iOS中的事件分为3大类型: 触屏事件(手势), 传感器事件(摇一摇, 陀螺仪), 远程控制事件(耳机的线控, 外接手柄). 按照时间顺序, 事件的生命周期概括如下: 

1. 事件的产生和传递
2. 找出最合适的View后事件的处理

下面以触摸事件为例. 在iOS中不是任何对象都能处理事件, 只有继承了UIResponder的对象才能接受并处理事件, 我们称之为"响应者对象". `UIApplication`, `UIViewController`, `UIView`都是继承自UIResponder的, 所以都能接收并处理事件.

# 1 事件的产生
发生触摸事件后, 系统会将该事件加入到一个由UIApplication管理的事件队列中. 因为队列的特点是FIFO, 先进先出, 先产生的事件先处理.

UIApplication会从事件队列中取出最前面的事件, 并将事件分发下去以便处理, 通常先发送事件给应用程序的主窗口(Keywindow).

主窗口(keywindow)会在视图层析结构找到一个最合适的视图来处理触摸事件, 找到合适的视图控件后就会调用视图控件的touches方法来做具体的事件处理.

# 2 事件的传递
触摸事件的传递是从父控件到子控件, 也就是UIApplication->window->寻找处理事件最合适的view. *如果父控件不能接受触摸事件, 那么子控件就不可能接收到触摸事件.*

1. 主窗口接收到应用程序传递过来的事件后, 首先判断自己能否接手触摸事件. 如果能, 那么在判断触摸点在不在窗口自己身上.
2. 如果触摸点在自己身上, 那么窗口会倒序遍历子控件来寻找最合适的View(倒序先遍历最新添加的View, 效率更高).
3. 遍历到每一个子控件, 会重复上面的两个步骤. (传递事件给子控件, 判断子控件能否接受事件, 触摸点是否在子控件上)
4. 循环遍历子控件, 直到找到最合适的View. 如果没有符合条件的子控件, 那么就认为自己最合适处理.

如果UIView的`userInteractionEnabled = NO`, 或者`hidden = YES`, 或者透明度<0.01则不能接受触摸事件. 注意父控件的hidden属性和透明度alpha属性都会影响他的子控件.

## 2.1 寻找最合适View的底层实现
寻找最合适的View用到了两个重要方法: `hitTest:withEvent:` 和 `pointInside:withEvent:`.

## 2.2 hitTest:withEvent:
只要事件一传递给一个控件, 这个控件就会调用他自己的`hitTest:withEvent:`方法. 他的作用就是寻找并返回最合适的view(能够响应事件的那个最合适的view). *不管这个控件能不能处理事件, 也不管触摸点在不在这个控件上, 事件都会先传递给这个控件, 随后再调用 `hitTest:withEvent:`方法.*

```
底层具体实现如下: 
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    // 1.判断当前控件能否接收事件
    if (self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01) return nil;
    // 2. 判断点在不在当前控件
    if ([self pointInside:point withEvent:event] == NO) return nil;
    // 3.从后往前遍历自己的子控件
    NSInteger count = self.subviews.count;
    for (NSInteger i = count - 1; i >= 0; i--) {
        UIView *childView = self.subviews[i];
        // 把当前控件上的坐标系转换成子控件上的坐标系
        CGPoint childP = [self convertPoint:point toView:childView];
        UIView *fitView = [childView hitTest:childP withEvent:event];
        if (fitView) { // 寻找到最合适的view
            return fitView;
        }
    }
    // 循环结束,表示没有比自己更合适的view
    return self;
}
```
事件传递给窗口或者控件后, 就调用`hitTest:withEvent:`寻找最合适的View. 所以是先传递事件, 再根据事件在自己身上找到最合适的View. 不管子控件是不是最合适的View, 系统都会先把事件传递给子控件, 经过子控件的`hitTest:withEvent:`验证后才知道有没有最合适的View. 所以如果确定最终父控件是最合适的view, 那么该父控件的子控件的hitTest:withEvent:方法也是会被调用的.

一般我们在父控件的hitTest:withEvent:中返回子控件作为最合适的view. 例如: 当遍历子控件时, 如果触摸点不在子控件A自己身上而是在子控件B身上, 还要要求返回子控件A作为最合适的view, 采用返回自己的方法可能会导致还没有来得及遍历A自己, 就有可能已经遍历了点真正所在的view, 也就是B. 这就导致了返回的不是自己而是触摸点真正所在的view, 所以还是建议在父控件的hitTest:withEvent:中返回子控件作为最合适的view.

## 2.3 pointInside:withEvent:
pointInside:withEvent:方法判断点在不在当前view上(方法调用者的坐标系上)如果返回YES, 代表点在方法调用者的坐标系上; 返回NO代表点不在方法调用者的坐标系上, 那么方法调用者也就不能处理事件.

# 3 事件传递的总结
由上面可知: 事件的传递顺序应该如下:
产生触摸事件 -> `UIApplication`事件队列 -> `[UIWindow hitTest:withEvent:]` -> 返回更合适的view -> 子控件`[hitTest:withEvent:]` -> 返回最合适的view

# 4. 事件的响应
上面介绍了事件的传递, 下面介绍下事件传递到最合适的处理view后, 如果响应.

## 4.1 响应者链条
1. 用户点击屏幕后产生的一个触摸事件, 经过一系列的传递过程后, 会找到最合适的视图控件来处理这个事件. 
2. 找到最合适的视图控件后，就会调用控件的touches方法来作具体的事件处理`touchesBegan:withEvent:`, `touchesMoved:withEvent:`, `touchesEnded:withEvent:`, `touchesCancelled:withEvent:`  
3. touch方法默认不处理事件, 只传递事件, 将事件(顺着响应者链条)交给上一个响应者进行处理. 如果找到合适的View之后就会调用该view的touches方法要进行响应处理具体的事件, 找不到就不会调用. 

## 4.2 响应者链条
响应者链条其实就是很多响应者对象(继承自UIResponder的对象)一起组合起来的链条称之为响应者链条. 响应者链的事件传递过程如下:

1. 如果当前view是控制器的view, 那么控制器就是上一个响应者, 事件就传递给控制器; 如果当前view不是控制器的view, 那么父视图就是当前view的上一个响应者, 事件就传递给它的父视图.
2. 在视图层级结构的最顶级视图, 如果也不能处理接收到的事件或者消息, 则将其事件或消息传递给window对象进行处理.
3. 如果window对象也不处理, 则将其事件或消息传递给UIApplication对象.
4. 如果UIApplication也不能处理该事件或者消息则将其丢弃.

注意: 在事件的响应中, 如果某个控件实现了touches的一系列方法. 则这个事件将由该控件来接受. 如果调用了[super touches...], 就会将事件顺着响应者链条往上传递, 传递给上一个响应者, 接着就会调用上一个响应者的touches...方法.

# 5. 人机交互处理的完整流程

1. 触摸屏产生触摸事件后, 触摸事件会被添加到由UIApplication管理的事件队列中去.
2. UIApplication会从事件队列中取出最前面的事件, 把事件传递给应用程序的主窗口(keywindow).
3. 主窗口会在视图层级结构中找到一个最合适的View来处理触摸事件.
4. 最合适的View会调用自己的touches方法处理事件.
5. 把事件沿着响应者链条向上抛, 一直到找到能够处理的视图或者被UIApplication抛弃.

事件的传递是从上到下(父控件到子控件), 事件的响应是从下到上(顺着响应者链条向上传递: 子控件到父控件).

# 6. 常见用法
下面是事件传递和视图响应中的一些常见用法
## 6.1 一个事件多个对象处理
事件的响应是顺着响应者链条向上传递的, 即从子控件传递给父控件, touch方法默认不处理事件, 而是把事件顺着响应者链条传递给上一个响应者. 这样我们就可以依托这个原理, 让一个事件多个控件响应.

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
  NSLog(@"-- dosomething");
  [super touchesBegan:touches withEvent:event];
}
```

## 6.2 扩大view的点击区域

```
底层具体实现如下: 
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01) return nil;
    CGRect touchRect = CGRectInset(self.bounds, -10, -10);
    if (CGRectContainsPoint(touchRect, point)) {
        NSInteger count = self.subviews.count;
        for (NSInteger i = count - 1; i >= 0; i--) {
            UIView *childView = self.subviews[i];
            CGPoint childP = [self convertPoint:point toView:childView];
            UIView *fitView = [childView hitTest:childP withEvent:event];
            if (fitView) {
                return fitView;
            }
        }
        return self;
    }
    return nil;
}
```

----
参考资料:
1.[iOS事件传递及响应](http://blog.flight.dev.qunar.com/2016/10/28/ios-event-mechanism-summary/)

2.[深入浅出iOS事件机制](https://zhoon.github.io/ios/2015/04/12/ios-event.html)

3.[史上最详细的iOS之事件的传递](https://www.jianshu.com/p/2e074db792ba)

