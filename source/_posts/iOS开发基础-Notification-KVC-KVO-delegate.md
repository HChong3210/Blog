---
title: iOS开发基础-Notification & KVC & KVO & delegate
date: 2019-07-22 19:24:58
tags:
    - 基础知识
categories: 
    - iOS开发-基础
---

KVC是Key-Value Coding的简称，它是一种可以直接通过字符串的名字(key)来访问类属性的机制。而不是通过调用Setter、Getter方法访问.
KVO是Key-Value Observing的简称. 它是一种键值观察的机制，在更改一个对象的指定属性时, 另外的指定对象能够得到通知.
KVC/KVO是观察者模式的一种实现, 而delegate是代理模式和通知模式在iOS中的具体实现, NSNotification则是发布-订阅模式的一种实现. 网上有很多人说，观察者模式和发布-订阅模式是两种不同的设计模式，它们压根就是两码事，不能混为一谈。也有很多人说，两者其实都是观察者模式，只是实现手段有点不一样罢了，本质是一样的。我认为发布-订阅模式只是观察者模式的一种实现手段, 它本质还是观察者模式. 

# 1 kVC
全称是Key-value coding, 翻译成键值编码。它提供了一种使用字符串而不是访问器方法去访问一个对象实例变量的机制. KVC 机制是由 NSKeyValueCoding 协议定义的.

KVC中以字符串的形式向对象发送消息, 这个字符串是我们关注的属性的关键. 常见的的基本调用如下:

```
**设置值**
// value的值为OC对象，如果是基本数据类型要包装成NSNumber
- (void)setValue:(id)value forKey:(NSString *)key;
// keyPath键路径，类型为xx.xx
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
// 它的默认实现是抛出异常，可以重写这个函数做错误处理。
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;

**获取值**
- (id)valueForKey:(NSString *)key;
- (id)valueForKeyPath:(NSString *)keyPath;
// 如果Key不存在，且没有KVC无法搜索到任何和Key有关的字段或者属性，则会调用这个方法，默认是抛出异常
- (id)valueForUndefinedKey:(NSString *)key;

**其他**
// 允许直接访问实例变量，默认返回YES。如果某个类重写了这个方法，且返回NO，则KVC不可以访问该类。
+ (BOOL)accessInstanceVariablesDirectly;
// 这是集合操作的API，里面还有一系列这样的API，如果属性是一个NSMutableArray，那么可以用这个方法来返回
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;
// 如果你在setValue方法时面给Value传nil，则会调用这个方法
- (void)setNilValueForKey:(NSString *)key;
// 输入一组key，返回该组key对应的Value，再转成字典返回，用于将Model转到字典。
- (NSDictionary *)dictionaryWithValuesForKeys:(NSArray *)keys;
// KVC提供属性值确认的API，它可以用来检查set的值是否正确、为不正确的值做一个替换值或者拒绝设置新值并返回错误原因。
- (BOOL)validateValue:(id)ioValue forKey:(NSString *)inKey error:(NSError)outError;
```

keyPath方法是集成了key的所有功能，也就是说对一个对象的一般属性进行赋值、取值，两个方法是通用的，都可以实现。但是对对象中的对象进的属性行赋值，只有keyPath能够实现.

## 1.1 KVC内部原理
比如说如下的一行KVC的代码：
```
[site setValue:@"sitename" forKey:@"name"];
```

就会被编译器处理成：
```
SEL sel = sel_get_uid ("setValue:forKey:");
IMP method = objc_msg_lookup (site->isa,sel);
method(site, sel, @"sitename", @"name");
```
这下KVC内部的实现就很清楚的清楚了, 一个对象在调用setValue的时候:

1. 首先根据方法名找到运行方法的时候所需要的环境参数.
2. 他会从自己isa指针结合环境参数, 找到具体的方法实现的接口.
3. 再直接查找得来的具体的方法实现.

下面我们再看一下KVC中设置值和获取值的流程:

### 1.1.1 KVC设置值的流程
1. 首先搜索同名的setter方法(无@synthsize情况下, 因为@synthsize告诉编译器生成set<Key>:格式的setter方法).
2. 上面的setter方法没找到，如果类方法accessInstanceVariablesDirectly返回YES。那么按_<key>，_is<Key>，<key>，is<key>的顺序在类中查找实例变量。（NSKeyValueCodingCatogery中实现的类方法，默认实现为返回YES）
3.  如果找到设置成员的值, 如果没有调用`setValue:forUndefinedKey:`.

注意: 通常情况下，KVC不允许在调用setValue：属性值 forKey：(或者keyPath)时对非对象传递一个nil的值。很简单，因为值类型是不能为nil的。如果你不小心传了，KVC会调用setNilValueForKey:方法。这个方法默认是抛出异常，所以一般而言最好还是重写这个方法。

### 1.1.2 KVC取值的流程
1. 首先按get<Key>、<key>、is<Key>的顺序查找getter方法(无@synthsize情况下, 因为@synthsize告诉编译器自动生成key:格式的getter方法). 如果是bool、int等基础数据类型，会做NSNumber的转换.
2. 上面的getter没有找到，查找countOf<Key>、objectIn<Key>AtIndex:、<Key>AtIndexes格式的方法。
3. 如果countOf<Key>和另外两个方法中的一个找到，那么就会返回一个可以响应NSArray所有方法的代理集合(collection proxy object)。发送给这个代理集合(collection proxy object)的NSArray消息方法，就会以countOf<Key>、objectIn<Key>AtIndex:、<Key>AtIndexes这几个方法组合的形式调用。还有一个可选的get<Key>:range:方法
4. 还没查到，那么查找countOf<Key>、enumeratorOf<Key>、memberOf<Key>:格式的方法
5. 如果这三个方法都找到，那么就返回一个可以响应NSSet所有方法的代理集合(collection proxy object)。发送给这个代理集合(collection proxy object)的NSSet消息方法，就会以countOf<Key>、enumeratorOf<Key>、memberOf<Key>:组合的形式调用
6. 还是没查到，那么如果类方法accessInstanceVariablesDirectly返回YES，那么按_<key>，_is<Key>，<key>，is<key>的顺序直接搜索成员名
7. 再没查到，调用valueForUndefinedKey:。

## 1.2 KVC常见实践
这里是一些常见的KVC的实践.

### 1.2.1 修改和访问私有属性私有变量
对于类里的私有属性，Objective-C是无法直接访问的，但是KVC可以通过key, keypath做到.

### 1.2.2 动态取值和设置值
利用KVC动态的取值和设值是最基本的用途.

### 1.2.3 Model和字典转换
利用runtime来获取成员属性, 通过KVC来赋值. 下面是一个简单的Demo.
```
+ (instancetype)modelWithDict:(NSDictionary *)dict {
    id objc = [[self alloc] init];
    /*
     遍历模型中所有成员属性，去字典中查找
     属性定义在类中，类有个属性列表（数组）
     */
    
    //遍历模型所有成员属性
    /*ivar表示成员属性
     {
     _ID;//这个才叫成员属性，成员属性对应属性
     }
     class_copyIvarList：把成员属性列表复制一份
     Ivar * 表示指向Ivar数组的指针
     */
    unsigned int count;
    Ivar *ivarList = class_copyIvarList([self class], &count);
    for (int i = 0; i < count; i ++) {
        Ivar ivar = ivarList[i];
        NSString *propertyName = [NSString stringWithUTF8String:ivar_getName(ivar)];
        NSString *propertyType = [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)];
        NSString *key = [propertyName substringFromIndex:1];
        id value = dict[key];
        //也有存在类型是NSDictionary但是不想转成模型的情况
        //二级转换
        //值是字典，并且成员属性不是字典的时候需要转模型
        if ([value isKindOfClass:[NSDictionary class]] && ![propertyType containsString:@"NS"]) {
            //转换成哪个类型
            
            //得到的propertyType是@"User"这样的一个字符串，所以需要先做截取
            NSString *typeString = [propertyType substringFromIndex:2];
            typeString = [typeString substringToIndex:typeString.length - 1];
            //获取需要转换类的类对象
            Class modelClass = NSClassFromString(typeString);
            if (modelClass) {
                value = [modelClass modelWithDict:value];
            }
        }
        if (value) {
            [objc setValue:value forKey:key];
        }   
    }
    return objc;
}
```
### 1.2.4 操作集合
Apple对KVC的valueForKey:方法作了一些特殊的实现，比如说NSArray和NSSet这样的容器类就实现了这些方法。所以可以用KVC很方便地操作集合.

![集合运算符](https://segmentfault.com/img/remote/1460000013476170)
上面表达式主要分为三部分，left部分是要操作的集合对象，如果调用KVC的对象本来就是集合对象，则left可以为空。中间部分是表达式，表达式一般以@符号开头。后面是进行运算的属性。

集合运算符主要分为三类：

* 集合操作符：处理集合包含的对象，并根据操作符的不同返回不同的类型，返回值以NSNumber为主。
* 数组操作符：根据操作符的条件，将符合条件的对象包含在数组中返回。
* 嵌套操作符：处理集合对象中嵌套其他集合对象的情况，返回结果也是一个集合对象。

[这里](https://segmentfault.com/a/1190000013476163#articleHeader4)有一个很好的操作集合的例子.

### 1.2.5 利用KVC高阶消息传递
当对容器类使用KVC时，valueForKey:将会被传递给容器中的每一个对象，而不是容器本身进行操作。结果会被添加进返回的容器中，这样，开发者可以很方便的操作集合来返回另一个集合.

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
       NSArray* arrStr = @[@"english",@"franch",@"chinese"];
       NSArray* arrCapStrLength = [arrStr valueForKeyPath:@"capitalizedString.length"];
        for (NSNumber* length  in arrCapStrLength) {
            NSLog(@"%ld",(long)length.integerValue);
        }
        
    }
    return 0;
}

//打印结果
7
6
7
```

# 2 KVO 
之前总结过关于KVO的相关, 可以看[这里](http://hchong.net/2018/01/24/KVO%E8%AF%A6%E8%A7%A3/)

通过KVC修改属性会触发KVO, 是因为内部触发了setter方法. 

通过KVC修改成员变量也会触发KVO, 是因为通过KVC修改成员变量的值, 系统内部帮助我们调用了willChangeValueForKey和didChangeValueForKey这两个方法, 并且在didChangeValueForKey 内部调用了 observeValueForKeyPath:方法.

# 3 Notification

* 1对N （多）
* 不关心返回值，单向的传值
* NSNotificationCenter单例统一处理发通知
* 通过不同的唯一的通知标识名NotificationName区分不同通知
* 被观察者主动发出通知
* 使用Notification一定要谨慎，由于1对多的缘故，避免滥用，不好查问题。
* 其次就是在Controller销毁时，一定要注销掉通知.
* 最好保证观察者和被观察者的事件都在同一个线程中来处理, 否则可能会出现一些奇怪的Bug.

# 4 delegate
在 iOS 中，代理(delegate)的本质是 protocol.

* 一般用来处理 一对一 的关系. 
* 支持正向与反向传值(参数, 返回值). 
* 可以用require和optional来修饰的

-------
参考资料:
1.[Objective-C中的KVC和KVO](http://yulingtianxia.com/blog/2014/05/12/objective-czhong-de-kvche-kvo/)
2.[Objective-C KVC机制](https://blog.csdn.net/omegayy/article/details/7381301)
3.[KVC内部执行过程分析](https://www.jianshu.com/p/8b2a98a8808a)
4.[swift中的 KVC 与 KVO](https://swiftcafe.io/2016/01/03/kvc)
5.[iOS KVC实现原理](https://blog.csdn.net/qq_18505715/article/details/80205796)
6.[iOS KVC和KVO详解](https://juejin.im/post/5aef18b76fb9a07aa34a28e6)
7.[KVC原理剖析](https://segmentfault.com/a/1190000013476163)


