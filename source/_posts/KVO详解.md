---
title: KVO用法与自定义实现
date: 2018-01-24 20:47:38
tags:
    - 基础知识
    - Runtime
categories: 
    - 基础知识
---

# KVO用法与自定义实现

## KVO用法

KVO 是 Objective-C 对观察者模式（Observer Pattern）的实现。也是 Cocoa Binding 的基础。当被观察对象的某个属性发生更改时，观察者对象会获得通知. KVO的用法在这里不做叙述, 十分简单. 首先注册, 添加一个观察者:

```
- (void)addObserver:(NSObject *)observer
         forKeyPath:(NSString *)keyPath
            options:(NSKeyValueObservingOptions)options
            context:(void *)context
```

> * observer: 观察者，负责处理监听事件的对象, 注册 KVO 通知的对象。观察者必须实现 key-value observing 方法 observeValueForKeyPath:ofObject:change:context:

> * keyPath: 要监听的属性, 观察者的属性的 keypath，相对于接受者，值不能是 nil。

> * options: 观察的选项(观察新、旧值，也可以都观察), NSKeyValueObservingOptions 的组合，它指定了观察通知中包含了什么，可以查看 "NSKeyValueObservingOptions"。

> * context: 上下文，用于传递数据，可以利用上下文区分不同的监听, 在 observeValueForKeyPath:ofObject:change:context: 传给 observer 参数的随机数据


当 keyPath 的值改变的时候这个方法会被调用:

```
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context
```

> * keyPath 监听的属性名

> * object  属性所属的对象

> * change  属性的修改情况（属性原来的值`oldValue`、属性最新的值`newValue`）

> * context 传递的上下文数据，与监听的时候传递的一致，可以利用上下文区分不同的监听

当一个观察者完成了监听一个对象的改变, 经常在 `-observeValueForKeyPath:ofObject:change:context:`，或者 `-dealloc` 中调用注销监听的方法:

```
- (void)removeObserver:(NSObject *)anObserver
            forKeyPath:(NSString *)keyPath

```

-------

这里有几个特殊的方法需要着重说明一下, 

```
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key;
```

`willChangeValueForKey:`和`didChangeValueForKey:`默认是在setter方法中实现的, 用KVO做键值观察后, 系统会在运行时重写属性的set方法, 并且在赋值前后分别调用.

`automaticallyNotifiesObserversForKey:`控制是否自动发送通知, 如果返回NO, KVO无法自动运作, 需手动触发.

## KVO的原理

KVO 的实现也依赖于 Objective-C 强大的 Runtime. Apple 的文档有简单提到过 KVO 的实现: 

> Automatic key-value observing is implemented using a technique called isa-swizzling.

> The isa pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.

> When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.

> You should never rely on the isa pointer to determine class membership. Instead, you should use the class method to determine the class of an object instance.

概述下KVO的实现就是: 

KVO 是通过 isa-swizzling 实现的. 当你观察一个对象时, 会动态创建一个新的类. 这个类继承自该对象的原本的类, 如果用户注册了对某此目标对象的某一个属性的观察，那么此派生类会重写被观察属性的 setter 方法，并在其中添加进行通知的代码. 自然, 重写的 setter 方法会负责在调用原 setter 方法之前和之后, 通知所有观察对象值的更改. 最后把这个对象的 isa 指针 ( isa 指针告诉 Runtime 系统这个对象的类是什么 ) 指向这个新创建的子类, 对象就神奇的变成了新创建的子类的实例. 这个中间类, 继承自原本的那个类. 不仅如此, Apple 还重写了 -class 方法, 企图欺骗我们这个类没有变, 就是原本那个类.


## 如何实现带block回调的KVO

 根据Apple的官方文档, 我们不难发现自定义KVO需要的几个步骤:
 
 1. 创建注册子类, 重写子类的class方法
    
     ```
    //1.创建注册子类
    //1.1获取被监听对象的类名称
    Class class = object_getClass(self);
    NSString *className = NSStringFromClass(class);
    //1.2检查被检测对象的class的前缀是否被替换过(通过检查前缀来判断), 如果被替换过就说明正在被观测
    if (![className hasPrefix:kHCKVOClassPrefix]) {
        class = [self makeKvoClassWithOriginalClassName:className];
        //为观测的对象设置一个指定的类
        object_setClass(self, class);
    }
 ```

 2. 为新的子类添加set方法
    
     ```
     //2.为新的子类添加set方法
    //2.1得到Setter方法
    SEL setterSelector = NSSelectorFromString(setterForGetter(key));
    //2.2得到指定类的实例方法
    Method setterMethod = class_getInstanceMethod([self class], setterSelector);
    if (!setterMethod) {
        @throw @"没有对应的Setter方法";
        return;
    }
    //2.3为新类添加set方法
    if (![self hasSelector:setterSelector]) {
        const char *types = method_getTypeEncoding(setterMethod);
        class_addMethod(class, setterSelector, (IMP)kvo_setter, types);
    }
 ```

 3. 改变isa指针, 指向新的子类

    ```
    //3改变isa指针，指向子类
    object_setClass(self, class);
    ```

 4. 保存set, get方法, 保存block

    ```
    //保存set、get方法名
        objc_setAssociatedObject(self, kHCKVO_getter_key, key, OBJC_ASSOCIATION_COPY_NONATOMIC);
        objc_setAssociatedObject(self, kHCKVO_setter_key, setterForGetter(key), OBJC_ASSOCIATION_COPY_NONATOMIC);
        //保存block
        objc_setAssociatedObject(self, kHCKVO_block_key, block, OBJC_ASSOCIATION_COPY_NONATOMIC);
    ```

这里面主要的难点在重写属性的set方法, 代码如下:

```
//新类的set方法
static void kvo_setter(id self, SEL _cmd, id newValue) {
    //包括调用父类的set方法，获取旧值、新值，获取observer并通知observer
    NSString *setterName = NSStringFromSelector(_cmd);
    NSString *getterName = getterForSetter(setterName);
    
    if (!getterName) {
        NSString *reason = [NSString stringWithFormat:@"Object %@ does not have getter %@", self, setterName];
        @throw [NSException exceptionWithName:NSInvalidArgumentException
                                       reason:reason
                                     userInfo:nil];
        return;
    }
    
    /*
     //使用objc_msgSendSuper向父类发消息, 调用父类set方法
    id oldValue = [self valueForKey:getterName];
    
    //superclass
    struct objc_super superclazz = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    // cast our pointer so the compiler won't complain
    void (*objc_msgSendSuperCasted)(void *, SEL, id) = (void *)objc_msgSendSuper;
    // call super's setter, which is original class's setter method
    objc_msgSendSuperCasted(&superclazz, _cmd, newValue);
     */
    
    
    //保存子类类型
    Class class = [self class];
    //isa指向原类
    object_setClass(self, class_getSuperclass(class));
    //调用原类get方法，获取oldValue
    id oldValue = objc_msgSend(self, NSSelectorFromString(getterName));
    //调用原类set方法
    objc_msgSend(self, _cmd, newValue);
    //isa改回子类类型
    object_setClass(self, class);

    
    //取出block
    HCObservingBlock block = objc_getAssociatedObject(self, kHCKVO_block_key);
    block(self, getterName, oldValue, newValue);
}
```

其中, 关于调用父类的set方法有两种方式. 一种是直接向新类的superClass发送消息, 另外一种是先改变isa指向superclass, 调用完set方法后重新再改变isa指向新类.

## 如何实现系统自带的KVO

系统自带KVO的实现方法和自定义带block回调的KVO的实现方法一样, 不同的是我们在重写新类的set方法中不是调用父类的set方法, 而是调用父类的`observeValueForKeyPath: ofObject: change: context:`方法.

-------

完整代码查看[这里](https://github.com/HChong3210/HCKVO)

-------


参考资料:
1.[Key-Value Observing](http://nshipster.cn/key-value-observing/)

2.[如何优雅地使用 KVO](https://draveness.me/kvocontroller)

3.[KVOController](https://github.com/facebook/KVOController)

4.[如何自己动手实现 KVO](http://tech.glowing.com/cn/implement-kvo/)


