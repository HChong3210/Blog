---
title: iOS开发基础-Category & Extension
date: 2018-10-24 09:42:14
tags:
	- 基础知识
categories:
	- iOS开发-基础
---

Category和Extension是iOS开发中常见的两种语言特性. 下面就介绍一下.

# 1 Category
Category是Objective-C 2.0之后添加的语言特性, Category的主要使用场景有下面几种:

* 为已经存在的类添加方法, 属性, 协议等
* 可以把类的实现分开在几个不同的文件里面. 这样做有几个显而易见的好处，a)可以减少单个文件的体积 b)可以把不同的功能组织到不同的Category里 c)可以由多个开发者共同完成一个类 d)可以按需加载想要的Category 等等
* 声明私有方法(只在需要的地方引入分类，即可调用分类中的方法，不引用该分类的就无法调用（重写原类方法的除外))
* 模拟多继承(分类中声明别的类中的方法，不实现，通过消息转发实现调用)
* 把framework的私有方法公开(只要知道Framework中私有方法的声明，那么可以在分类中声明Framework中的私有方法，这样可以在需要的地方引入这个分类，就可以调用到Framework中的私有方法了)

## 1.1 category特点

* category只能给某个已有的类扩充方法，不能扩充成员变量。
* category中也可以添加属性，只不过@property只会生成setter和getter的声明，不会生成setter和getter的实现以及成员变量。
* 如果category中的方法和类中原有方法同名，运行时会优先调用category中的方法。也就是，category中的方法会覆盖掉类中原有的方法。所以开发中尽量保证不要让分类中的方法和原有类中的方法名相同。避免出现这种情况的解决方案是给分类的方法名统一添加前缀。比如category_。
* 如果多个category中存在同名的方法，运行时到底调用哪个方法由编译器决定，最后一个参与编译的方法会被调用

## 1.2 调用优先级
分类(category) > 本类 > 父类。即，优先调用cateory中的方法，然后调用本类方法，最后调用父类方法. 

注意：category是在运行时加载的，不是在编译时.

## 1.3 为什么category不能添加成员变量
Objective-C类是由Class类型来表示的，它实际上是一个指向objc_class结构体的指针。它的定义如下：
```
typedef struct objc_class *Class;
```
objc_class结构体的定义如下：

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    #if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类
    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                      OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
    #endif
} OBJC2_UNAVAILABLE;
```

在上面的objc_class结构体中，ivars是objc_ivar_list（成员变量列表）指针；methodLists是指向objc_method_list指针的指针。在Runtime中，objc_class结构体大小是固定的，不可能往这个结构体中添加数据，只能修改。所以ivars指向的是一个固定区域，只能修改成员变量值，不能增加成员变量个数。methodList是一个二维数组，所以可以修改*methodLists的值来增加成员方法，虽没办法扩展methodLists指向的内存区域，却可以改变这个内存区域的值（存储的是指针）。因此，可以动态添加方法，不能添加成员变量。

## 1.4 category中能添加属性吗?
Category不能添加成员变量（instance variables），那到底能不能添加属性（property）呢? 这个我们要从Category的结构体开始分析: 

```
typedef struct category_t {
    const char *name;  //类的名字
    classref_t cls;  //类
    struct method_list_t *instanceMethods;  //category中所有给类添加的实例方法的列表
    struct method_list_t *classMethods;  //category中所有添加的类方法的列表
    struct protocol_list_t *protocols;  //category实现的所有协议的列表
    struct property_list_t *instanceProperties;  //category中添加的所有属性
} category_t;
```

从Category的定义也可以看出Category的可为（可以添加实例方法，类方法，甚至可以实现协议，添加属性）和不可为（无法添加实例变量）。

但是为什么网上很多人都说Category不能添加属性呢? 实际上，Category实际上允许添加属性的，同样可以使用@property，但是不会生成_变量（带下划线的成员变量），也不会生成添加属性的getter和setter方法的实现，所以，尽管添加了属性，也无法使用点语法调用getter和setter方法(实际上，点语法是可以写的，只不过在运行时调用到这个方法时候会报方法找不到的错误). 但实际上可以使用runtime去实现Category为已有的类添加新的属性并生成getter和setter方法

## 1.5 category的加载
Objective-C的运行是依赖OC的runtime的, 而OC的runtime和其他系统库一样, 是OS X和iOS通过dyld动态加载的.

category被附加到类上面是在map_images的时候发生的，在new-ABI的标准下，_objc_init里面的调用的map_images最终会调用objc-runtime-new.mm里面的_read_images方法. 在_read_images方法的结尾:

* 会把category的实例方法、协议以及属性添加到类上(category的各种列表是怎么最终添加到类上的, 以实例方法列表来说: 在上述的代码片段里，addUnattachedCategoryForClass只是把类和category做一个关联映射，而remethodizeClass才是真正去处理添加事宜的方法) 
* 把category的类方法和协议添加到类的metaclass上(对于添加类的实例方法而言，又会去调用attachCategoryMethods这个方法. attachCategoryMethods做的工作相对比较简单，它只是把所有category的实例方法列表拼成了一个大的实例方法列表，然后转交给了attachMethodLists方法)

综上我们可以发现: 

* category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA.
* category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休，殊不知后面可能还有一样名字的方法

## 1.5 同名category优先级

* 如果同名方法只存在在category中, 后编译的先被访问.
* 原类的方法和category中的方法同名, 原类的方法会被category的直接”覆盖”, 如果存在于多个category中则参考第一条.

## 1.6 load和initialize的调用顺序

关于load和initialize的相关信息, 可以看[这里](http://hchong.net/2019/07/24/iOS%E5%BC%80%E5%8F%91%E5%9F%BA%E7%A1%80-load-initialize/). 在MyClass及其分类中都添加这两个方法，只不过打印不同. 

```
+(void)load {
    NSLog(@"MyClass:%s",__func__);
}
+(void)initialize {
    NSLog(@"MyClass:%s",__func__);
}
```

![load](http://img.souche.com/f2e/22b0bc484d5bb5354c7dd580b3e5a0ab.png)
load方法，文件镜像被读取时调用，直观的表现是程序刚运行就会看到打印结果. 从上图可以看到, load的执行顺序是先类，后category, 而category的+load执行顺序是根据编译顺序决定的.

附加category到类的工作会先于+load方法的执行. 因为category被附加到类上面是在map_images的时候发生的.

![initialize](http://img.souche.com/f2e/f8ad0648bbbc7ef5c35d4a684544c1d5.png)
从上图可以看到, initialize函数后编译的先被访问, 原类的方法会被分类的”覆盖”.(因为initialize只有在第一次被使用是会调用一次且仅一次，所以其他与普通方法相同).

## 1.7 category和方法覆盖
category其实并不是完全替换掉原来类的同名方法，只是category在方法列表的前面而已，所以我们只要顺着方法列表找到最后一个对应名字的方法，就可以调用原来类的方法:

```
Class currentClass = [MyClass class];
MyClass *my = [[MyClass alloc] init];

if (currentClass) {
    unsigned int methodCount;
    Method *methodList = class_copyMethodList(currentClass, &methodCount);
    IMP lastImp = NULL;
    SEL lastSel = NULL;
    for (NSInteger i = 0; i < methodCount; i++) {
        Method method = methodList[i];
        NSString *methodName = [NSString stringWithCString:sel_getName(method_getName(method)) 
        								encoding:NSUTF8StringEncoding];
        if ([@"printName" isEqualToString:methodName]) {
            lastImp = method_getImplementation(method);
            lastSel = method_getName(method);
        }
    }
    typedef void (*fn)(id,SEL);
    
    if (lastImp != NULL) {
        fn f = (fn)lastImp;
        f(my,lastSel);
    }
    free(methodList);
}
```

## 1.7 category和关联对象
category里面是无法为category添加实例变量的. 但是我们很多时候需要在category中添加和对象关联的值，这个时候可以求助关联对象来实现. 因为在分类中 @property 并不会自动生成实例变量以及存取方法，所以一般使用关联对象为已经存在的类添加属性, 这样我们通过category新增的属性就会存在与之对应的实例变量和存取方法.

对象关联详解可以看[这里](https://draveness.me/ao);

MyClass+Category1.h:

```
#import "MyClass.h"

@interface MyClass (Category1)

@property(nonatomic,copy) NSString *name;

@end
```

MyClass+Category1.m:

```
#import "MyClass+Category1.h"
#import <objc/runtime.h>

@implementation MyClass (Category1)

+ (void)load
{
    NSLog(@"%@",@"load in Category1");
}

- (void)setName:(NSString *)name
{
    objc_setAssociatedObject(self,
                             "name",
                             name,
                             OBJC_ASSOCIATION_COPY);
}

- (NSString*)name
{
    NSString *nameObject = objc_getAssociatedObject(self, "name");
    return nameObject;
}

@end
```

对象销毁时候如何处理关联对象呢? 

所有的关联对象都由AssociationsManager管理. AssociationsManager里面是由一个静态AssociationsHashMap来存储所有的关联对象的。这相当于把所有对象的关联对象都存在一个全局map里面。而map的的key是这个对象的指针地址（任意两个不同对象的指针地址一定是不同的），而这个map的value又是另外一个AssociationsHashMap，里面保存了关联对象的kv对.

runtime的销毁对象函数objc_destructInstance里面会判断这个对象有没有关联对象，如果有，会调用_object_remove_assocations做关联对象的清理工作.

# 2 Extension
Extension被开发者称之为扩展、延展、匿名分类。Extension看起来很像一个匿名的Category，但是Extension和Category几乎完全是两个东西. Extension的主要使用场景有下面几种:

* 声明私有属性
* 声明私有方法
* 声明私有成员变量

# 3 Category vs Extension
extension看起来很像一个匿名的category，但是extension和有名字的category几乎完全是两个东西。 extension在编译期决议，它就是类的一部分，在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡。extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension. 除非创建子类再添加extension。而category不需要有类的源码，我们可以给系统提供的类添加category.

但是category则完全不一样，它是在运行期决议的。 就category和extension的区别来看，我们可以推导出一个明显的事实，extension可以添加实例变量，而category是无法添加实例变量的(因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的).

extension和category都可以添加属性，但是category的属性不能生成成员变量和getter、setter方法的实现.

|扩展的特点|分类的特点|
| --- | --- |
|编译时决议|运行时决议|
|只能以声明的形式存在，多数情况下寄生于宿主类的.m中|分类有声明也有实现|
|不能为系统类添加扩展|可以为系统类添加分类|
|可以添加实例变量|不可以添加实例变量(因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的)|

-----
参考资料:
1.[【iOS】Category VS Extension 原理详解](http://www.cocoachina.com/articles/19163)
2.[巧用 Class Extension 分离接口依赖](http://blog.sunnyxx.com/2016/04/22/objc-class-extension-tips/)
3.[深入理解Objective-C：Category](https://tech.meituan.com/2015/03/03/diveintocategory.html)
4.[运行时之二：分类](https://leoliuyt.github.io/2018/05/20/%E8%BF%90%E8%A1%8C%E6%97%B6%E4%B9%8B%E4%BA%8C%EF%BC%9A%E5%88%86%E7%B1%BB/)
5.[关联对象 AssociatedObject 完全解析](https://draveness.me/ao)
6.[Objective-C Associated Objects 的实现原理](http://blog.leichunfeng.com/blog/2015/06/26/objective-c-associated-objects-implementation-principle/)

