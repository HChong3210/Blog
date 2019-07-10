---
title: iOS开发总结系列-属性关键字
date: 2018-03-01 16:10:53
tags:
    - 基础知识
categories: 
    - 基础知识
---

常见的问题, 在这里做一个汇总. 只讲结论, 内部细节, 慢慢填坑.

## 什么情况使用 weak 关键字，相比 assign 有什么不同
weak常用在有可能出现循环引用的地方, 他表示一种非拥有关系, 为这种属性设置新值时, 设置方法既不保留新值, 也不释放旧值. 与assign类似, 但是不同的是当属性所指的对象被释放时, 属性值会被置为nil, 而assign只执行基础类型的简单赋值操作. 

weak必须用于OC对象类型, assign可以用于基础类型.

## 怎么用 copy 关键字
1. NSString, NSArray, NSDictionary 等等经常使用copy关键字, 是因为他们有对应的可变类型: NSMutableString, NSMutableArray, NSMutableDictionary. 
2. 若想令自己所写的对象具有拷贝功能要实现NSCopying. 如果自定义的对象分为可变版本与不可变版本, 那么就要同时实现 NSCopying 与 NSMutableCopying 协议.
3. block 也经常使用 copy 关键字. 

对可变对象进行copy, 返回的会是一个不可变对象, 并不是一般意义上的copy行为, 因为copy后的对象类型不同了.
对可变对象进行mutableCopy, 返回的是另一个可变对象, 这种才是copy行为, 两个对象内容一样, 类型一样, 地址不同.
对不可变对象进行copy, 系统不会再创建对象, 而是直接返回源对象地址.
对不可变对象进行mutableCopy则返回一个可变对象, 同样不是一般意义上的copy.
    
block使用关键字copy是MRC时代的产物, 因为Block有三种类型, _NSConcreteStackBlock, _NSConcreteGlobalBlock, _NSConcreteMallocBlock. 由于_NSConcreteStackBlock存储在栈上面, 一旦返回之后所属的变量域一旦结束, 就被系统销毁了, 所以他是不安全的, 我们通过copy属性可以把它存储在堆上面, 生命周期我们就可以控制了. 在 MRC 中, 方法内部的 block 是在栈区的, 使用 copy 可以把它放到堆区. 在 ARC 中写不写都行: 对于 block 使用 copy 还是 strong 效果是一样的. 可以参考[这里](http://hchong.net/2017/07/04/Block%E7%94%A8%E6%B3%95%E4%B8%8E%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/).

## 对象的copy与mutableCopy
在集合类对象中, 对不可变对象进行 copy, 是指针复制(浅copy), mutableCopy 是内容复制(深copy); 对可变对象进行 copy 和 mutableCopy 都是内容复制(深copy). 但是集合对象的内容复制仅限于对象本身, 对象元素仍然是指针复制. 

可以参考[这里](http://hchong.net/2016/09/21/%E5%B1%9E%E6%80%A7%E4%BF%AE%E9%A5%B0%E7%AC%A6%E4%B9%8Bcopy%E5%92%8Cstrong/).

## runtime 如何实现 weak 属性
weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship). 为这种属性设置新值时, 设置方法既不保留新值, 也不释放旧值. 此特质同 assign 类似, 然而在属性所指的对象遭到摧毁时, 属性值也会清空(nil out).

Runtime维护了一个weak表, 用于存储指向某个对象的所有weak指针. weak表其实是一个hash表, Key是所指对象的地址, Value是weak指针的地址(这个地址的值是所指对象的地址)数组. 当此对象的引用计数为0的时候会 dealloc, 假如 weak 指向的对象内存地址是a, 那么就会以a为键, 在这个 weak 表中搜索, 找到所有以a为键的 weak 对象, 从而设置为 nil. 

在ARC环境无论是强指针还是弱指针都无需在 dealloc 设置为 nil, ARC 会自动帮我们处理.

详细的可以看[这里](http://www.cocoachina.com/ios/20170328/18962.html).

## @property中有哪些属性关键字？/ @property 后面可以有哪些修饰符？
1. 原子性: nonatomic, atomic(默认值).
2. 读/写权限: readwrite(默认值), readonly.
3. 内存管理: assign, strong, weak, unsafe_unretained, copy.
4. 方法名: getter, setter.
5. 其他: nonnull, null_resettable, nullable.

基本数据类型默认关键字: atomic, assign, readwrite. 对象类型默认关键字: atomic, strong, readwrite.

atomic 会加一个@synchronized()锁, 并且引用计数会 +1, 来向调用者保证这个对象会一直存在, 但是这个锁并不能保证线程安全. 

## @protocol 和 category 中如何使用 @property
在 protocol 中使用 property 只会生成 setter 和 getter 方法声明, 我们使用属性的目的, 是希望遵守我协议的对象能实现该属性.

category 使用 @property 也是只会生成 setter 和 getter 方法的声明, 如果我们真的需要给 category 增加属性的实现, 需要借助于运行时的两个函数`objc_setAssociatedObject`和
`objc_getAssociatedObject`.

## @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
@property的本质是 ivar(实例变量) + getter + setter.

```
typedef struct objc_property *objc_property_t;

struct property_t {
    const char *name;
    const char *attributes;
};

typedef struct {
    const char *name;           /**< The name of the attribute */
    const char *value;          /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;
```
attributes的具体内容包含: 类型, 原子性, 内存语义和对应的实例变量.

我们每次在增加一个属性, 系统都会在 ivar_list 中添加一个成员变量的描述, 在 method_list 中增加 setter 与 getter 方法的描述, 在属性列表中增加一个属性的描述, 然后计算该属性在对象中的偏移量, 然后给出 setter 与 getter 方法对应的实现, 在 setter 方法中从偏移量的位置开始赋值, 在 getter 方法中从偏移量开始取值, 为了能够读取正确字节数, 系统对象偏移量的指针类型进行了类型强转.

详见[Runtime用法与分析](http://hchong.net/2017/12/11/Runtime%E7%94%A8%E6%B3%95%E4%B8%8E%E5%88%86%E6%9E%90/)
## @synthesize和@dynamic分别有什么作用
@property有两个关键词, @synthesize(默认值)和@dynamic. 

@synthesize表示系统会默认添加一个@syntheszie var = _var的实例变量, 并且自动生成setter和getter方法. @synthesize 合成实例变量有以下几点规则:
 
1. 如果指定了成员变量的名称, 会生成一个指定的名称的成员变量.
2. 如果这个成员已经存在就不再生成了.
3. 如果没有指定成员变量的名称会自动生成一个属性同名的成员变量.

@synthesize的使用场景: 

1. 同时重写了setter和getter时
2. 重写了只读属性的getter时
3. 在使用了@dynamic时
4. 在@Protocol和category中定义属性时
5. 重载父类的属性时, 来手动合成Ivar(实例变量/成员变量).


@dynamic表示我们不需要系统自动生成, 由用户自己实现, 如果没有手动生成的话, 在使用过程中是会奔溃的.
## __block vs __weak
__weak主要是用来避免循环引用的. __weak修饰的变量, 在block内部被捕获后, 和外部的实际不是用一个变量. 他是对外部__weak修饰的对象进行了一个弱引用. 当我们在block外部把__weak修饰的变量释放后, block内部夜读不到这个变量. 如果外部被__weak修饰的变量被置为nil, 那么内部的对象实际上也是nil, 就会被释放掉. 

__block实际上是提升了变量的作用域, 我们在block内外, 用__block修饰的变量实际上不是同一个, 在block内部使用相当于被强引用了一份. __block 本身无法避免循环引用的问题, 但是我们可以通过在 block 内部手动把 blockObj 赋值为 nil 的方式来避免循环引用的问题. 
这里需要注意, 我们只能在堆上面block对象使用__block修饰的对象, ARC下一旦block赋值就会触发copy, block就会被复制到堆上因此可以直接使用, 但是MRC下我们则需要手动调用copy, 否则会编译报错.
MRC下__block修饰的变量在block中使用是不會retain的, ARC下是会retain的. 可以使用__weak(ARC下使用, 但只支持iOS5以后)或是__unsafe_unretained来代替.
## nonatomic和atomic的线程安全
atomic: 原子操作, 系统会为setter方法加锁(@synchronized), 需要消耗大量系统资源来为属性加锁. atomic所说的线程安全只是保证了getter和setter存取方法的线程安全, 并不能保证整个对象是线程安全的. 
例如当线程A进行写操作, 这时其他线程的读或者写操作会因为该操作而等待. 当A线程的写操作结束后, B线程进行写操作, 然后当A线程需要读操作时, 却获得了在B线程中的值, 这就破坏了线程安全, 如果有线程C在A线程读操作前release了该属性, 那么还会导致程序崩溃. 所以仅仅使用atomic并不会使得线程安全, 我们还要为线程添加lock来确保线程的安全.

nonatomic: 不会为setter方法加锁, 线程不安全, 适合内存较小的移动设备.


