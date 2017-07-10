---
title: 统计打点的AOP实现
date: 2017-02-13 19:35:28
tags: 
- AOP
- 统计打点
- 解决方案
categories:
- 统计打点
---
# 统计打点的AOP实现
每一个App, 必然会有大量的分析数据来统计用户行为. 而这些统计对应在客户端就是, 统计打点, 又称埋点.
关于埋点的本质, 我理解的就是用户出发一个行为后, 调用一个特定的接口. 服务端拿到我们的请求后, 根据客户端传的参数也就是事件ID来区分是什么操作(注意, 这里的事件ID是具有唯一性的, 不同的ID对应用户不同的操作). 有时也可能会需要其他的信息, 比如操作人的ID等. 服务端拿到这些信息之后, 再整理筛选, 通过视图, 报表的形式展现出来.
因为埋点是和业务紧密相连的, 所以一般我们的埋点代码(就是调用特定网络接口的代码)分散在整个项目的各个地方. 当业务越来越复杂, 工程越来越大时, 我们的埋点代码就会变得很难维护, 埋点事件分散在各个地方, 很难有个清晰的逻辑. 并且把埋点事件和业务代码高度耦合在一起, 也不是一个明智的选择.
这个时候, 就会想到, 要是能用AOP的方式来解决埋点的实现, 把埋点事件和业务代码解耦开来, 那我们维护起来就会方便好多.
> AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP与OOP是面向不同领域的两种设计思想。
> OOP（面向对象编程）针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分。
> AOP则是针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。
> AOP可以用在日志记录，性能统计，安全控制，事务处理，异常处理等等，本篇文章主要讲的是埋点也就是日志记录

在Objective-C中使用AOP主要指的是使用Objective-C的Runtime特性, 给指定的方法添加自定义代码. 有很多方式来实现AOP, MethodSwizzling只是其中之一.而又有一些第三方库, 将Runtime进行了很好地封装, 让我们不用了解Runtime的知识, 就能很好地使用AOP.
我们主要使用的是[Aspects](https://github.com/steipete/Aspects)这个第三方库, 关于Aspects的内部实现, 可以参考这篇博文[iOS 如何实现Aspect Oriented Programming](http://www.jianshu.com/p/dc9dca24d5de).
由以上的解释, 可以基本了解, 我们主要是通过Aspect来Hook对应事件的方法, 传递事件唯一的ID给服务端来标记此事件响应过一次.所以, 我们的代码大致应该是这样.
首先有一个类用来记录埋点事件ID和需要Hook的类和方法, 并且将他们一一对应.

根据记录埋点事件的复杂程度, 我们大致可以将埋点分为简单埋点和复杂埋点两种:
1. 简单埋点: 只用记录某个操作事件响应次数
2. 复杂埋点:
    * 需要传除了事件ID外的参数
    * 需要根据服务端返回的数据来响应不同的事件
    * 埋点事件ID放在服务端返回的字段中

我们新建两个类, 一个用来记录Hook的类`HYZTrackList`, Hook的方法和与之对应的事件ID. 另外一个类`HYZTrackManager`, 用来实现埋点事件的具体操作方法.
## 简单埋点的实现
我们在`HYZTrackList`中实现`trackList`方法, 返回一个数组. 数组的元素是一个个的字典, 用对记录每一个事件的相关信息.

```
+ (NSArray *)trackList {
    NSArray *trackList = @[
  //=======================================简单埋点==========================================
                           //HYZViewController1简单埋点点击事件
                           @{kClassName:@"HYZViewController1",
                             kHookFunction:@"simpleTrack:para2:",
                             kEventType:HYZViewController1SimpleButtonClick,
                             kIsLightEvent:@(YES)},
                           //HYZViewController1复杂埋点点击事件
                           @{kClassName:@"HYZViewController1",
                             kHookFunction:@"blockButtonAction:",
                             kEventType:HYZViewController1BlockButtonClick,
                             kIsLightEvent:@(YES)},
                           //HYZViewController1block埋点点击事件
                           @{kClassName:@"HYZViewController1",
                             kHookFunction:@"complexButtonAction:",
                             kEventType:HYZViewController1ComplexButtonClick,
                             kIsLightEvent:@(YES)},
                           
//=======================================复杂埋点==========================================
                           @{kClassName:@"HYZViewController3",
                             kHookFunction:@"trackWithTag:",
                             kHandlerBlock:@"HYZViewController3TrackHandleBlock",
                             kIsLightEvent:@(NO)}];
    return trackList;
}
```

如上所示, 我们是要记录`HYZViewController1`类里面的`simpleTrack:para2:`方法的点击事件, 事件ID是`HYZViewController1SimpleButtonClick`.
我们在`APPDelegate`的`application:didFinishLaunchingWithOptions:`方法中, 来hook所有的在`trackList`中记录的方法.
在``中, 实现如下相关代码.

```
+ (void)setup {
    //实现和替换hook的block方法
    NSMutableDictionary *blockDict = [[NSMutableDictionary alloc] init];
    [HYZTrackManager weightEventEntry:blockDict];
    
    [[HYZTrackList trackList] enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        BOOL isLightEvent = [obj[kIsLightEvent] boolValue];
        NSString *className = obj[kClassName];
        NSString *functionName = obj[kHookFunction];
        NSString *eventName = obj[kEventType];
        Class class = NSClassFromString(className);
        SEL selector = NSSelectorFromString(functionName);
        if (isLightEvent == YES) {
            if (!functionName) {
                return;
            }
            [HYZTrackManager lightTrackTarget:class selector:selector functionName:functionName trackId:eventName];
        } else {
            NSString *blockName = obj[kHandlerBlock];
            id handleBlock = [blockDict objectForKey:blockName];
            if (!handleBlock) {
                return;
            }
            [HYZTrackManager complexTrackTarget:class selector:selector usingBlock:handleBlock];
        }
    }];
}
```
针对简单埋点, 我们直接使用

```
//简单埋点虽然可以拿到对应方法的参数, 但是如果需要把该参数传到埋点请求的网络事件中的话, 必须使用复杂埋点来处理
+ (void)lightTrackTarget:(Class)target selector:(SEL)selector functionName:(NSString *)functionName trackId:(NSString *)trackId {
    NSError *error;
    NSInteger functionParamCount = [[functionName componentsSeparatedByString:@":"] count] - 1;
    switch (functionParamCount) {
        case 0: {
            [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo){
                [HYZTrackManager trackRequestWithTrackId:trackId, nil];
            }error:&error];
        }
            break;
        case 1: {
            [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, id p1){
                [HYZTrackManager trackRequestWithTrackId:trackId, p1, nil];
            }error:&error];
        }
            break;
        case 2: {
            [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, id p1, id p2){
                [HYZTrackManager trackRequestWithTrackId:trackId, p1, p2, nil];
            }error:&error];
        }
            break;
        case 3: {
            [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, id p1, id p2, id p3){
                [HYZTrackManager trackRequestWithTrackId:trackId, p1, p2, p3, nil];
            }error:&error];
        }
            break;
        case 4: {
            [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, id p1, id p2, id p3, id p4){
                [HYZTrackManager trackRequestWithTrackId:trackId, p1, p2, p3, p4, nil];
            }error:&error];
        }
            break;
        default:
            break;
    }
}

```
简单埋点虽然也可以拿到Hook的方法的参数, 但是由于通用性, 所以不能用来传递可变的参数.

真正的埋点请求是这样的

```
+ (void)trackRequestWithTrackId:(NSString *)trackId, ... NS_REQUIRES_NIL_TERMINATION{
    // 定义一个指向个数可变的参数列表指针；
    va_list args;
    // 用于存放取出的参数
    NSString *arg;
    // 初始化变量刚定义的va_list变量，这个宏的第二个参数是第一个可变参数的前一个参数，是一个固定的参数
    va_start(args, trackId);
    // 遍历全部参数 va_arg返回可变的参数(a_arg的第二个参数是你要返回的参数的类型)
    while ((arg = va_arg(args, NSString *))) {
        NSLog(@"%@", arg);
    }
    // 清空参数列表，并置参数指针args无效
    va_end(args);
    NSLog(@"此处用来实现埋点事件记录%@的网络请求", trackId);
}
```
至此, 实现了简单埋点的方法Hook.

## 复杂埋点
复杂埋点, 因为要传参数进来, 所以我们利用`+ (id<AspectToken>)aspect_hookSelector:(SEL)selector withOptions:(AspectOptions)options usingBlock:(id)block error:(NSError **)error;`可以传block参数来实现.
首先, 我们在`HYZTrackList`中把类名, 方法名和需要定义的Block关联起来,如下所示.

```
{kClassName:@"HYZViewController3",
kHookFunction:@"trackWithTag:",
kHandlerBlock:@"HYZViewController3TrackHandleBlock",
kIsLightEvent:@(NO)}
``` 
在`APPDelegate`中初始化时, 需要在`HYZTrackManager`中实现Block的定义, 

```
//hook的block在这里定义和实现
+ (void)weightEventEntry:(NSMutableDictionary*)blockDict{
    [HYZTrackManager trackButtonAction:blockDict];
}

//block的内部实现
+ (void)trackButtonAction:(NSMutableDictionary *)blockDict {
    void(^HYZViewController3TrackHandleBlock)(id, NSInteger tag) = ^(id <AspectInfo>aspectInfo, NSInteger tag) {
        switch (tag) {
                case 1: {
                    [HYZTrackManager trackRequestWithTrackId:[NSString stringWithFormat:@"%ld", tag], nil];
                }
                break;
                case 2: {
                    [HYZTrackManager trackRequestWithTrackId:[NSString stringWithFormat:@"%ld", tag], nil];
                }
                break;
                case 3: {
                    [HYZTrackManager trackRequestWithTrackId:[NSString stringWithFormat:@"%ld", tag], nil];
                }
                break;
                case 4: {
                    [HYZTrackManager trackRequestWithTrackId:[NSString stringWithFormat:@"%ld", tag], nil];
                }
                break;
            default:
                break;
        }
    };
    [blockDict setObject:[HYZViewController3TrackHandleBlock copy] forKey:@"HYZViewController3TrackHandleBlock"];
}
```

然后Hook原方法;

```
//复杂的埋点,
+ (void)complexTrackTarget:(Class)target selector:(SEL)selector usingBlock:(id)block {
    NSError *error;
    [target aspect_hookSelector:selector withOptions:AspectPositionAfter usingBlock:block error:&error];
}
```

[Demo](https://github.com/HChong3210/AOPTrack)
参考资料: 
1. [iOS 如何实现Aspect Oriented Programming](http://www.jianshu.com/p/dc9dca24d5de)
2. [AOP在iOS中的实践——统计埋点](http://www.vienta.me/2016/09/21/AOP%E5%9C%A8iOS%E4%B8%AD%E7%9A%84%E5%AE%9E%E8%B7%B5%E4%B8%80%E7%BB%9F%E8%AE%A1%E5%9F%8B%E7%82%B9/)
3. [可复用且高度解耦的iOS用户统计实现](http://www.cocoachina.com/ios/20160421/15912.html)
4. [Method Swizzling和AOP(面向切面编程)实践](http://www.cocoachina.com/ios/20150120/10959.html)

