---
title: Aspects源码分析
date: 2018-05-02 22:35:35
tags:
    - 源码解析
categories: 
    - 源码解析
---

Aspects 是一个轻量型的提供面向切面编程方案的框架, 我们项目中的统计埋点就是使用这个框架, 减少了对工程结构的入侵. Aspects 提供的API十分简单, Aspects 是基于 Method Swizzling 来实现的, 他的大致实现思路如下.

OC 中, 每个类是一个结构体, 里面存储了类的一些基本信息, 其中 methodList 方法链表里面存储的是 Method 类型. Method 保存了一个方法的全部信息, 包括 SEL 方法名, method_types 各参数的返回值类型, IMP 该方法具体实现的函数指针. Method Swizzling 的核心实际上只做了两件事情: 

1. `class_addMethod` 添加一个新的方法, 可能是把其它类中实现的方法添加到目标类中, 也可能是把父类实现的方法添加一份在子类中, 可能是添加的实例方法, 也可能是添加的类方法, 总之就是添加了方法.
2. 交换IMP 交换方法的实现IMP, 完成这个步骤除了使用`method_exchangeImplementations`这个方法外, 也可以是调用了`method_setImplementation`方法来单独修改某个方法的IMP, 或者是采用在调用`class_addMethod`方法中设定了IMP而直接就完成了IMP的交换, 总之就是对IMP的交换.

OC 是一门动态语言, 我们执行一个函数时, 其实是在发一条消息: `[receiver message]`. 我们在 receiver 中找到 selector -> message 的 IMP执行, 如果没找到IMP就执行 `objc_msgForward`进行消息转发. 那么这个查找的过程我们就可以动态的改变 selector 和 IMP 的对应关系, 从而使原来的消息转发到新的函数实现. 

在消息转发的时候, 如果 selector 有对应的IMP, 则直接执行. 如果没有, 还有三次补救机会, 依次是: 

1. `resolveInstanceMethod`, 适合给类/对象动态添加一个相应的实现.
2. `forwardingTargetForSelector`, 适合将消息转发给其他对象处理.
3. `forwardInvocation`, 消息的重定向, 在此可以获得该次调用的所有参数和调用对象, 并随意重新组装, 十分灵活.

Aspects 的方案就是, 对于待Hook的 selector, 将其指向 `objc_msgForward / _objc_msgForward_stret`, 同时生成一个新的 `aliasSelector` 指向原来的 IMP. 并且 hook 住 `forwardInvocation` 函数, 使他指向自己的实现. 当被 hook 的 selector 被执行的时候, 原 selector 的IMP则会统一指向 `objc_msgForward / _objc_msgForward_stret`, 启动消息转发流程, 从而进入 forwardInvocation. 同时由于 forwardInvocation 的指向也被修改了, 因此会转入新的 forwardInvocation 函数, 在里面执行需要嵌入的附加代码, 完成之后, 再转回原来的 IMP. 下面是相应的代码:  

## 添加一个aspect

``` OC
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        //判断能否被Hook
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            //记录相关的数据结构
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception. Hook的核心方法
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```
所有的核心方法都被封装到这个核心的static方法里面, 这里面主要做了三个操作: 

1. 判断能否Hook. 这里是有一个类似于黑名单和是否已经被Hook过的判断, 例如`retain`, `dealloc`等. 对于类对象还要确保同一个类继承关系层级中, 只能被 hook 一次, 因此这里需要判断子类, 父类有没有被 hook, 之所以做这样的实现, 主要是为了避免出现死循环的出现.
2. 记录数据结构.
3. swizzling的核心. 一个是对对象的 forwardInvocation 进行 swizzling, 另一个是对传入的 selector 进行 swizzling. swizzling forwardInvocation主要在 `aspect_hookClass`方法中进行. swizzling selector主要在 `aspect_getMsgForwardIMP`方法中进行.

## swizzling forwardInvocation

```
static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

    // Already subclassed
	if ([className hasSuffix:AspectsSubclassSuffix]) {
		return baseClass;

        // We swizzle a class object, not a single object.
	}else if (class_isMetaClass(baseClass)) {
        return aspect_swizzleClassInPlace((Class)self);
        // Probably a KVO'ed class. Swizzle in place. Also swizzle meta classes in place.
    }else if (statedClass != baseClass) {
        return aspect_swizzleClassInPlace(baseClass);
    }

    // Default case. Create dynamic subclass.
	const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);

	if (subclass == nil) {
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }

		aspect_swizzleForwardInvocation(subclass);
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
		objc_registerClassPair(subclass);
	}

	object_setClass(self, subclass);
	return subclass;
}
```
源代码中并没有直接 swizzling 对象的 forwardInvocation 方法, 而是动态生成一个当前对象的子类, 并将当前对象与子类关联, 然后替换子类的 forwardInvocation 方法(这里具体方法就是调用了 object_setClass(self, subclass), 将当前对象 isa 指针指向了 subclass, 同时修改了 subclass 以及其 subclass metaclass 的 class 方法, 使他返回当前对象的 class, 类似于 kvo 的实现. 

将当前对象变成一个 subclass 的实例, 同时对于外部使用者而言, 又能把它继续当成原对象在使用, 而且所有的 swizzling 操作都发生在子类, 这样做的好处是你不需要去更改对象本身的类, 也就是, 当你在 remove aspects 的时候, 如果发现当前对象的 aspect 都被移除了, 那么, 你可以将 isa 指针重新指回对象本身的类, 从而消除了该对象的 swizzling, 同时也不会影响到其他该类的不同对象). 对于每一个对象而言, 这样的动态对象只会生成一次, 这里 aspect_swizzlingForwardInvocation 将使得 forwardInvocation 方法指向 aspects 自己的实现逻辑.

## swizzling selector

``` OC
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    //swizzling forwardInvocation
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // swizzling selector
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```
当 forwradInvocation 被 hook 之后，接下来, 将对传入的 selector 进行 hook, 这里的做法是, 将 selector 指向了转发 IMP, 同时生成一个 aliasSelector, 指向了原来的 IMP, 同时为了放在重复 hook, 做了一个判断, 如果发现 selector 已经指向了转发 IMP, 那就就不需要进行交换了.

## 被Hook的ForwardInvocation的处理
在`aspect_swizzleForwardInvocation` -> `aspect_swizzleForwardInvocation`可以发现, 最终 ForwardInvocation 被指向 `__ASPECTS_ARE_BEING_CALLED__`. 

``` OC
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

------
参考资料: 
1.[Method Swizzling的各种姿势](http://www.tanhao.me/code/160723.html/)

2.[面向切面编程之 Aspects 源码解析及应用](http://wereadteam.github.io/2016/06/30/Aspects/)


