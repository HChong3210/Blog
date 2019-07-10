---
title: Runtime用法与分析
date: 2017-12-11 23:26:11
tags:
    - 基础知识
categories:
    - 基础知识
---

Objective-C 是C的语言的超集, C是一门静态语言而Objective-C却是一门动态语言, 这个动态特性就是由基于[Smalltalk](https://zh.wikipedia.org/wiki/Smalltalk)消息传递特性的[Runtime](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048-CH1-SW1)来提供的. 那么什么是Runtime, 官方的说法是这样的:

> The Objective-C language defers as many decisions as it can from compile time and link time to runtime. Whenever possible, it does things dynamically. This means that the language requires not just a compiler, but also a runtime system to execute the compiled code. The runtime system acts as a kind of operating system for the Objective-C language; it’s what makes the language work.
> 大致翻译一下就是: Objective-C尽可能的提供更多策略把原本需要在编译和链接时做的事尽一切可能放到了运行时来动态的处理. 那也就意味着我们不仅需要一套编译系统, 还需要一套运行时系统来执行编译后的代码. 对于Objective-C来说, Runtime就是这样的一套运行时操作系统; Runtime为Objective-C的的动态特性提供了底层的技术支持.

Runtime其实有两个版本: Modern和Legacy. 我们现在用的 Objective-C 2.0 采用的是现行 (Modern) 版的 Runtime 系统, 只能运行在 iOS 和 macOS 10.5 之后的 64 位程序中. 而 maxOS 较老的32位程序仍采用 Objective-C 1 中的早期(Legacy)版本的 Runtime 系统. 这两个版本最大的区别在于当你更改一个类的实例变量的布局时, 在早期版本中你需要重新编译它的子类, 而现行版就不需要.

## Runtime简介
Objective-C在三种层面上与Runtime系统进行交互: 

1. 通过 Objective-C 源代码
    Runtime 系统自动在幕后把我们写的源代码在编译阶段转换成运行时代码, 在运行时确定对应的数据结构和调用具体哪个方法.
2. 通过 Foundation 框架的 NSObject 类定义的方法
    NSObject类是遵守NSObject协议的, 在这个协议里面有很多方法是和Runtime相关, 或者直接从Runtime中获取信息的, 具体的可以看这里[NSObject Protocol Reference](https://developer.apple.com/documentation/objectivec/1418956-nsobject?language=objc).
3. 通过对 Runtime 库函数的直接调用
    在这里需要注意一下, 系统本身是默认关闭了Runtime的代码提示的, 我们需要在BuildSettings -> Enable Strict Checking objc_msgSend Calls ->设置为NO
    Objective-C 的 Runtime 为我们提供了很多运行时状态下跟类与对象相关的函数, 具体的可以看[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective_c_runtime?language=objc).

## Runtime 基础数据结构

这里我们需要先关注了解几个概念: Object(对象), Class(类), Meta Class(原类), id. 通过objc_class的定义我们大致可以看出他们之间的关系:

```
typedef struct objc_class *Class;  
typedef struct objc_object *id;

@interface Object { 
    Class isa; 
}

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

struct objc_object {  
private:  
    isa_t isa;
}

struct objc_class : objc_object {  
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

union isa_t  
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}

```

用文字总结一下就是: 

1. id 在 Objective-C 中可以代指任意的对象类型, 他是一个指向 objc_object 结构体的指针(这个struct的定义本身就带了一个 *, 所以我们在使用其他NSObject类型的实例时需要在前面加上 *, 而使用 id 时却不用). 

    ```
    /// A pointer to an instance of a class.
    typedef struct objc_object *id;
    ```
2. 那么什么是 objc_object 呢? Objective-C中的Object(objc_object)在最后会被转换成C的结构体, 而在这个struct中有一个 [isa_t](#isa_t) 类型的结构体isa, 通过查看 isa_t 我们发现它里面有一个指向它的类别 Class(定义了对象所属的类). *注意: isa 指针不总是指向实例对象所属的类, 不能依靠它来确定类型, 而是应该用 class 方法来确定实例对象的类.* 

    ```
    struct objc_object {  
    private:  
        isa_t isa;
    public:
        // initIsa() should be used to init the isa of new objects only.
        // If this object already has an isa, use changeIsa() for correctness.
        // initInstanceIsa(): objects with no custom RR/AWZ
        void initIsa(Class cls /*indexed=false*/);
        void initInstanceIsa(Class cls, bool hasCxxDtor);
    private:  
        void initIsa(Class newCls, bool indexed, bool hasCxxDtor);
    }
    ```
3. 那么什么是 Class 呢? Class 其实是一个指向 objc_class 结构体的指针. 

    ```
    /// An opaque type that represents an Objective-C class.
    typedef struct objc_class *Class;
    ```
4. 那么什么是objc_class呢? 我们可以看到 objc_class 继承自 objc_object, 由此可以看出*Objective-C 中类也是一个对象*. 我们调用类方法的时候, 类对象的isa里面是什么呢? 这样就引出了原类的概念. 

    ```
    struct objc_class : objc_object {  
        // Class ISA;
        Class superclass;
        cache_t cache;             // formerly cache pointer and vtable
        class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    }
    ```
    
*总结: Class在设计中本身也是一个对象. 而这个Class对象的对应的类, 我们叫它 Meta Class, 它用来表述类对象本身所具备的元数据, 类方法就定义于此处, 因为这些方法可以理解成类对象的实例方法. 每个类仅有一个类对象, 而每个类对象仅有一个与之相关的元类. 即Class结构体中的 isa 指向的就是它的 Meta Class. 我们可以把Meta Class理解为一个Class对象的Class. 当我们给一个NSObject对象发送消息时(实例方法), 这条消息会在对象所属的类的方法列表里查找. 当我们发送一个消息给一个类时(类方法), 这条消息会在类的Meta Class的方法列表里查找.* 

下面这个图很好地说明了Object, Class, Meta Class之间的关系.
![Object, Class, Meta Class之间的关系](http://7ni3rk.com1.z0.glb.clouddn.com/Runtime/class-diagram.jpg)
概括一下, 上图主要说了以下几点: 

1. 每个实例(Object)的isa指针都指向该实例所属的类. 
2. 每个类(Class)的isa指针都指向为一个该类所属的Meta Class(原类).
3. 每个Meta Class(原类)的isa指针都指向Root Class(Meta)(根原类), 大部分情况是都是NSObject.
4. Root class(meta)的superclass指向Root class(Class), 也就是NSObject, 形成一个回路.
5. Root class(Class)其实就是NSObject, NSObject是没有超类的, 所以Root class(Class)的superclass指向nil.


下面是针对上面出现的一些结构体和对象的详细解析.
### <span id="isa_t">isa_t</span>
objc_object 结构体包含一个 isa 指针, 类型为 isa_t 联合体. 因为 isa_t 使用 union 实现, 所以可能表示多种形态, 既可以当成是指针, 也可以存储标志位. 有关 isa_t 联合体的更多内容可以查看 [Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/#isa-%E6%8C%87%E9%92%88%EF%BC%88NONPOINTER-ISA%EF%BC%89).
```
union isa_t  
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}
```
### cache_t
cache_t 出现在 objc_class 中, cache_t 中存储了一个 bucket_t 的结构体 _buckets ，和两个unsigned int 的变量 _mask 和 _occupied. _mask 分配用来缓存bucket的总数, _occupied 表明目前实际占用的缓存bucket的个数. bucket_t 的结构体中存储了一个 unsigned long 和一个 IMP. IMP是一个函数指针, 指向了一个方法的具体实现. bucket_t *_buckets其实就是一个散列表, 用来存储Method的链表. 

Cache 的作用主要是为了优化方法调用的性能. 当对象receiver调用方法message时, 首先根据对象receiver 的 isa 指针查找到它对应的类, 然后在类的 methodLists 中搜索方法, 如果没有找到, 就使用 super_class 指针到父类中的 methodLists 查找, 一旦找到就调用方法. 如果没有找到, 有可能消息转发, 也可能忽略它. 但这样查找方式效率太低, 所以使用Cache来缓存经常调用的方法, 当调用方法时, 优先在Cache查找, 如果没有找到, 再到methodLists查找.

```
//cache_t结构
struct cache_t {  
   struct bucket_t *_buckets;
   mask_t _mask;
   mask_t _occupied;
}
    
typedef unsigned int uint32_t;  
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
    
typedef unsigned long  uintptr_t;  
typedef uintptr_t cache_key_t;
    
struct bucket_t {  
private:  
   cache_key_t _key;
   IMP _imp;
public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
}
```
### <span id = "class_data_bits_t">class_data_bits_t</span>

```
//class_data_bits_t结构
struct class_data_bits_t {
   // Values are the FAST_ flags above.
   uintptr_t bits;
}
    
struct class_rw_t {  
   uint32_t flags;
   uint32_t version;
    
   const class_ro_t *ro;
    
   method_array_t methods;
   property_array_t properties;
   protocol_array_t protocols;
    
   Class firstSubclass;
   Class nextSiblingClass;
    
   char *demangledName;
}
    
struct class_ro_t {  
   uint32_t flags;
   uint32_t instanceStart;
   uint32_t instanceSize;
#ifdef __LP64__
   uint32_t reserved;
#endif
    
   const uint8_t * ivarLayout;
   
   const char * name;
   method_list_t * baseMethodList;
   protocol_list_t * baseProtocols;
   const ivar_list_t * ivars;
    
   const uint8_t * weakIvarLayout;
   property_list_t *baseProperties;
    
   method_list_t *baseMethods() const {
       return baseMethodList;
   }
};
```

![class_data_bits_t](https://ob6mci30g.qnssl.com/Blog/ArticleImage/23_15.png)

上面这张图很好地说明了 class_data_bits_t 的作用. 详见[深入解析 ObjC 中方法的结构](https://github.com/Draveness/analyze/blob/master/contents/objc/%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90%20ObjC%20%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84.md#%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90-objc-%E4%B8%AD%E6%96%B9%E6%B3%95%E7%9A%84%E7%BB%93%E6%9E%84). 大致说明一下就是, Objc的类的属性, 方法, 以及遵循的协议在obj 2.0的版本之后都放在 class_rw_t 中. class_ro_t 是一个指向常量的指针, 存储来编译器决定了的属性、方法和遵守协议. 在运行时发消息时, 会从 class_data_bits_t 调用 data 方法, 将结果从 class_rw_t 强制转换为 class_ro_t 指针. 最后调用 methodizeClass 方法, 把类里面的属性, 协议, 方法都加载进来.

## 方法与消息
### SEL
SEL 又叫做方法选择器, 是表示一个方法的 selector 的指针, 定义如下: 

```
typedef struct objc_selector *SEL;
```
Objective-C在编译时, 会依据每一个方法的名字, 参数序列, 生成一个唯一的整型标识(Int类型的地址), 这个标识就是SEL. 不同类的不同方法, 只要方法名相同, 哪怕参数类型不同, 这两个方法的SEL就是一样的. 同一个类中就不能存在两个方法, 方法名一致, 参数不一致, 这样是会编译错误的. 但是不同的类就可以, 因为不同的类的实例对象执行方法时, 是从各自类的方法列表中根据selector去寻找自己对应的IMP. 

工程中所有的SEL组合成一个set集合, 因此SEL是唯一的. set中的元素都是唯一的, 所以SEL也是唯一的, 所以我们找一个selector, 通过他对应的SEL是最快的方法. SEL实际就是根据方法名Hash过的一个字符串(这也解释了上面相同方法名编译为什么报错的原因), 字符串的比较只需比较地址就可以了, 速度十分快. 本质上, SEL只是一个指向方法的指针(准确的说, 只是一个根据方法名hash化了的KEY值, 能唯一代表一个方法), 它的存在只是为了加快方法的查询速度.

我们可以在运行时添加和获取selector, 也可以通过下面几种方式来获取SEL:

```
1. sel_registerName函数
2. Objective-C编译器提供的@selector()
3. NSSelectorFromString()方法
```
### IMP
IMP实际上是一个函数指针, 指向方法实现的地址. 定义如下:

```
id (*IMP)(id, SEL,...)
```
该函数的第一个参数是self的指针, 如果是实例方法, 则是类实例的内存地址; 如果是类方法, 则是指向元类的指针. 第二个参数是方法选择器, 后面是方法的参数列表. 

每个方法对应唯一的SEL, 我们通过SEL就是为了查找方法的最终实现IMP, 而IMP这个函数指针指向了最终的这个方法的实现, 取得IMP后, 我们就获得了执行这个方法代码的入口点. 通过取得IMP, 我们可以跳过Runtime的消息传递机制, 直接执行IMP指向的函数实现, 这样省去了Runtime消息传递过程中所做的一系列查找操作, 会比直接向对象发送消息高效一些.

通过一组id和SEL参数就能确定唯一的方法实现地址, 而一个确定的方法也只有唯一的一组id和SEL参数.
### Method
Method用于表示类定义中的方法, 定义如下:

```
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```
我们可以看到, 该结构体包含有 name(方法名, 本质是IMP), types(方法类型, 本质是个char指针, 存储方法的参数类型和返回值), imp(方法指向, 本质是个IMP, 也可以说是函数指针). 该结构体实际上相当于在 SEL 和 IMP 之间做了一个映射, 让我们可以通过 SEL 快速的找到 IMP.
## 成员变量与属性
### Ivar
Ivar用来表示实例变量, 其实际是一个指向 objc_ivar 结构体的指针, 定义如下:

```
typedef struct objc_ivar *Ivar;

struct ivar_t {
    int32_t *offset;//表示基地址偏移字节
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```
关于Ivar我们需要知道以下几点: 

1. 我们对 ivar 的访问就可以通过 对象地址 ＋ Ivar偏移字节的方法来访问, 但是如果增加了父类的Ivar, 那怎么办, Objective-C使用Non Fragile Ivars机制, Runtime会进行检测来调整类中新增的ivar的偏移量, 这样我们就可以通过 对象地址 + 基类大小 + ivar偏移字节的方法来计算出ivar相应的地址, 并访问到相应的ivar, 而不用重新编译子类.
2. 我们无法通过->函数来修改私有属性, 但是我们可以通过对象地址 + Ivar偏移量来访问地址, 获取到指针后直接修改.
3. 属性实际就是Ivar加上系统自动为我们生成的get和set方法.

### objc_property_t
@property 标记了类中的属性, 他是一个指向 objc_property 结构体的指针:

```
typedef struct property_t *objc_property_t;
```
需要注意的是, 与 class_copyIvarList 函数不同, 使用 class_copyPropertyList 函数只能获取类的属性, 而不包含成员变量, 但此时获取的属性名是不带下划线的.

更多姿势可以看[这里](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1).
### protocol_t
这个没啥好讲的, 直接看定义:

```
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;
    ... 省略一些封装的便捷 get 方法
}
```
## Category
美团的[这篇](https://tech.meituan.com/DiveIntoCategory.html)讲的具详细, 这里我们就大概说一下.
Category(分类), 他为现有的类提供了扩展, 它是 category_t 结构体的指针.

```
typedef struct category_t *Category;

typedef struct category_t {
    const char *name;//类的名字
    classref_t cls;//类
    struct method_list_t *instanceMethods;//category中所有给类添加的实例方法的列表
    struct method_list_t *classMethods;//给所有类添加类方法的列表
    struct protocol_list_t *protocols;//所有协议的列表
    struct property_list_t *instanceProperties;//category中添加的所有属性的列表
} category_t;
```
*从category的定义也可以看出我们可以添加实例方法, 类方法, 甚至可以实现协议, 添加属性, 但是无法添加实例变量*.

在APP的启动过程中, 会在 `_read_images` 函数间接调用到 attachCategories 函数, 完成向类中添加 Category 的工作. 向 class_rw_t(上面[class_data_bits_t](#class_data_bits_t)中有介绍) 中的 method_array_t, property_array_t, protocol_array_t 数组中分别添加 method_list_t, property_list_t, protocol_list_t 指针. 把category的实例方法, 协议以及属性添加到类上, 把category的类方法和协议添加到类的metaclass上.

我们需要注意的是: 

1. category不会覆盖原类中的方法, 而是两个都存在, 但是category的在前面, 按照方法列表来找, 找到category的之后, 就不再往下找了, 造成被覆盖的假象. 
2. 附加category的类的工作会先于+load方法的执行. 
3. +load的执行顺序是先类, 后category, 而category的+load执行顺序是根据编译顺序(Compile Sources中的顺序)决定的.

## 消息查找与转发
Objective-C中任何方法的调用, 编译器都会将[receiver message]转化为一个消息函数的调用, 即objc_msgSend, 消息直到运行时才绑定到方法的实现上. objc_msgSend 的定义如下:

```
objc_msgSend(receiver, selector, arg1, arg2, ...)

```
在[Obj-C Optimization: The faster objc_msgSend](http://www.mulle-kybernetik.com/artikel/Optimization/opti-9.html)中有一段 objc_msgSend 方法实现思路的代码:

```
id  c_objc_msgSend( struct objc_class /* ahem */ *self, SEL _cmd, ...)  
{
   struct objc_class    *cls;
   struct objc_cache    *cache;
   unsigned int         hash;
   struct objc_method   *method;   
   unsigned int         index;

   if( self)
   {
      cls   = self->isa;
      cache = cls->cache;
      hash  = cache->mask;
      index = (unsigned int) _cmd & hash;

      do
      {
         method = cache->buckets[ index];
         if( ! method)
            goto recache;
         index = (index + 1) & cache->mask;
      }
      while( method->method_name != _cmd);
      return( (*method->method_imp)( (id) self, _cmd));
   }
   return( (id) self);

recache:  
   /* ... */
   return( 0);
}

```
查了一堆资料, 总结一下消息的查找和转发的过程:

### 消息的查找和动态解析
1. 判断 selector 是不是需要被忽略的垃圾回收用到的方法, 是的话就忽略, 不是的话继续下一步操作.
2. 判断target是不是nil, 如果这里有相应的nil的处理函数, 就跳转到相应的函数中. 如果没有处理nil的函数, 就自动清理现场并返回. 这一点就是为何在OC中给nil发送消息不会崩溃的原因.
3. 查找当前类的缓存, 如果命中缓存获取到了IMP就将IMP返回, 如果没有继续下一步操作.
4. 在当前类的方法列表中查找(根据 selector 查找到 Method 后, 获取 Method 中的 IMP), 对已经排序的列表使用二分法查找, 未排序的列表则是线性遍历. 如果找到把方法加入 cache 并且把IMP返回, 如果没找到继续下一步操作.
5. 在继承层级中递归向父类(一直到 NSObject 为止)中查找, 情况跟上一步类似, 也是先查找缓存, 缓存没中就查找方法列表, 查到后就终止递归查询, 把方法加入 cache 并且把IMP返回, 如果没找到继续下一步操作.
6. 在消息查找阶段, 如果没找到IMP(也就是接收到未知的消息), 会进入动态方法解析阶段, 首先会调用所属类的`+resolveInstanceMethod:`(实例方法)或者`+resolveClassMethod:`(类方法)方法. 前提是我们必须自己实现该方法, 并且添加到类里面.

    ```
    void functionForMethod1(id self, SEL _cmd) {
       NSLog(@"%@, %p", self, _cmd);
    }
    	
    + (BOOL)resolveInstanceMethod:(SEL)sel {
        NSString *selectorString = NSStringFromSelector(sel);
        if ([selectorString isEqualToString:@"method1"]) {
            class_addMethod(self.class, @selector(method1), (IMP)functionForMethod1, "@:");
        }
        return [super resolveInstanceMethod:sel];
    }
    ```
7. 此时如果既没有没查找到 IMP, 动态方法解析也不奏效, 那么就进入了下一个阶段-消息转发阶段.

### <span id = "消息的转发">消息的转发</span>
上面主要是消息的查找阶段主要完成的是通过select()快速查找IMP的过程, 接下来才是消息的转发阶段, 到了转发阶段, 会调用到了转发阶段, 会调用`id _objc_msgForward(id self, SEL _cmd,...)`方法. 在执行`_objc_msgForward`之后会调用 `__objc_forward_handler`函数. 它的实现大致如下: 

```
// Default forward handler halts the process.
__attribute__((noreturn)) void objc_defaultForwardHandler(id self, SEL sel)  
{
    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
                "(no message forward handler is installed)", 
                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
                object_getClassName(self), sel_getName(sel), self);
}

```
当我们给一个对象发送一个没有实现的方法的时候, 如果其父类也没有这个方法, 则会崩溃, 报错信息类似于这样: unrecognized selector sent to instance, 然后接着会跳出一些堆栈信息. 这些信息就是从这里而来. 

1. 如果上面的查找和解析都失败的话, 消息就会无法处理, 这是Runtime会调用以下方法:

    ```
    - (id)forwardingTargetForSelector:(SEL)aSelector
    ```
    我们可以通过重写- (id)forwardingTargetForSelector:(SEL)aSelector方法来把消息的接受者换成一个可以处理该消息的实例对象或者类对象. 示例如下:
    
    ```
    - (id)forwardingTargetForSelector:(SEL)aSelector
    {
        if(aSelector == @selector(Method:)){
            return otherObject;
        }
        return [super forwardingTargetForSelector:aSelector];
    }
    
    + (id)forwardingTargetForSelector:(SEL)aSelector {
        if(aSelector == @selector(xxx)) {
            return NSClassFromString(@"Class name");
        }
        return [super forwardingTargetForSelector:aSelector];
    }
    ```
    如果一个对象实现了这个方法, 并返回一个非nil的结果, 则这个对象会作为消息的新接收者, 且消息会被分发到这个对象. 当然这个对象不能是self自身, 否则就是出现无限循环. 如果我们没有指定相应的对象来处理aSelector, 则应该调用父类的实现来返回结果. 这一步合适于我们只想将消息转发到另一个能处理该消息的对象上, 但这一步无法对消息进行处理, 如操作消息的参数和返回值.
2. 如果在上一步还不能处理未知消息, 则唯一能做的就是启用完整的消息转发机制了. 运行时系统会给消息接收者最后一次机会将消息转发给其它对象. 我们首先要通过, 指定方法签名, 若返回nil, 则表示不处理. 
    
    ```
    - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
       if ([NSStringFromSelector(aSelector) isEqualToString:@"testInstanceMethod"]){
         return [NSMethodSignature signatureWithObjcTypes:"v@:"];
      }  
    return [super methodSignatureForSelector: aSelector];
    }
    ```
    若返回方法签名, 则会进入下一步调用.
    
    ```
    - (void)forwardInvovation:(NSInvocation)anInvocation {
        if ([someOtherObject respondsToSelector:
                [anInvocation selector]]) {
            [anInvocation invokeWithTarget:someOtherObject];
        } else {
            [super forwardInvocation:anInvocation];
        }
    }
    ```
    该方法对象会创建一个表示消息的NSInvocation对象, 把与尚未处理的消息有关的全部细节都封装在anInvocation中, 包括selector, 目标(target)和参数. 我们可以在forwardInvocation方法中选择将消息转发给其它对象. 我们可以通过anInvocation对象做很多处理, 比如修改实现方法, 修改响应对象等.
    
    这个方法的主要作用是定位可以响应封装在anInvocation中的消息的对象(这个对象不需要能处理所有未知消息). 使用anInvocation作为参数, 将消息发送到选中的对象. anInvocation将会保留调用结果, 运行时系统会提取这一结果并将其发送到消息的原始发送者. 在这个方法中我们也可以实现一些更复杂的功能, 我们可以对消息的内容进行修改, 比如追回一个参数等, 然后再去触发消息. 另外, 若发现某个消息不应由本类处理, 则应调用父类的同名方法, 以便继承体系中的每个类都有机会处理此调用请求.
    
3. 上面两个补救措施做完后, 若发现某调用不应由本类处理, 则会调用超类的同名方法. 如此, 继承体系中的每个类都有机会处理该方法调用的请求, 一直到NSObject根类. 如果到NSObject也不能处理该条消息, 那么就是再无挽救措施了, 只能抛出"doesNotRecognizeSelector"异常.

## Runtime经典问题分析
这里有几个网络上常见的Runtime经典问题, 下面我会逐个分析.
### [self class]与[super class]

```
 @implementation Son : Father
- (id)init {
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
return self;
}
@end
```

如题, 先说结果: 

```
2018-03-15 20:02:57.031501+0800 HCRuntime[8115:645357] Son
2018-03-15 20:02:57.034428+0800 HCRuntime[8115:645357] Son
```

首先我们有几个概念要理解: 

```
//[self class]实际是调用objc_msgSend方法
id objc_msgSend(id self, SEL op, ...)
//[super class]实际是调用objc_msgSendSuper方法
id objc_msgSendSuper(struct objc_super *super, SEL op, ...)

struct objc_super {
   __unsafe_unretained id receiver;//类似于objc_msgSend中的self
   __unsafe_unretained Class super_class;//记录当前类的父类是什么
};

- (Class)class {
    return object_getClass(self);
}
```
self 和 super 的唯一区别是, 当使用 self 调用方法时, 会从当前类的方法列表中开始找, 如果没有, 就从父类中再找; 而当使用 super 时, 则从父类的方法列表中开始找, 然后调用父类的这个方法. self 是类的隐藏参数, 指向当前调用方法的这个类的实例. 而 super 是一个 Magic Keyword, 它本质是一个编译器标示符, 和 self 是指向的同一个消息接受者.

结合本题, 当调用`[self class]`时, 先调用的是 `objc_msgSend` 函数, 第一个参数是`Son`这个类的实例, 然后去示例的ISA(Son类)中找 `- (Class)class` 这个方法, 没找到, 然后去Son类的父类(Father类)中找, 没找到, 一直找到NSObject类中找到. 而 `- (Class)class` 的实现就是返回self的类别, 故上述输出结果为 Son.
当调用`[super class]`时, 先调用的是`class_getSuperclass`函数, 该函数第一个参数是结构体 `objc_super`(第一个参数是self, 第二个参数是当前实例变量的super_class, 就是Father类). 然后直接从实例变量所在类的父类(Father)去找`- (Class)class` 这个方法, 没找到, 然后去Son类的父类(Father类)中找, 没找到, 一直找到NSObject类中找到. 最后内部是使用 `objc_msgSend(objc_super->receiver, @selector(class))`去调用, 此时已经和`[self class]`调用相同了, 故上述输出结果仍然返回 Son.

### isKindOfClass与isMemberOfClass

```
@interface Student : NSObject
@end
@implementation Student
@end
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
        BOOL res3 = [(id)[Student class] isKindOfClass:[Student class]];
        BOOL res4 = [(id)[Student class] isMemberOfClass:[Student class]];
        NSLog(@"%d %d %d %d", res1, res2, res3, res4);
    }
    return 0;
}
```
如题, 先说结果: 

```
2018-03-15 21:33:16.047955+0800 HCRuntime[8721:695376] 1 0 0 0
```

再来进行分析, 这里主要的知识点是类, 原类, 实例变量之间的关系, 以及三个函数的内部实现: 

```
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}

Class object_getClass(id obj) {
    if (obj) return obj->getIsa();
    else return Nil;
}

- (BOOL)isKindOf:(Class)cls {
    Class cls;
    for (cls = isa; cls; cls = cls->superclass) 
        if (cls == (Class)aClass)
            return YES;
    return NO;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isMemberOf:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}
```
接下来我们挨个分析: 

1. `[(id)[NSObject class] isKindOfClass:[NSObject class]]`; `[NSObject class]`返回NSObject(Class), 所以走类方法. cls是NSObject(Class). 第一次比较, tcls是object_getClass((id)self)也就是NSObject(Class)的ISA也就是NSObject(Class)的Meta Class, 两者不相等, 第二次比较, cls是NSObject(Class), tcls是NSObject(Class)的Meta Class的superclass, 也就是NSObject(Class), 两者相等, 循环结束, 返回YES.
2. `[(id)[NSObject class] isMemberOfClass:[NSObject class]]`; 因为`[NSObject class]`返回NSObject(Class), 所以走类方法. 只比较一次, cls是NSObject(Class), object_getClass((id)self)也就是NSObject(Class)的ISA也就是NSObject(Class)的Meta Class, 两者不相等, 返回NO;
3. `[(id)[Student class] isKindOfClass:[Student class]]`; 类似于第一题, `[Student class]`返回Student(Class), 所以走类方法. cls是Student(Class). 第一次比较, tcls是Student(Class)的ISA, 也就是Student(Class)的Meta Class, 不相等. 第二次比较, tcls是Student(Class)的Meta Class的superclass, 也就是NSObject的Meta Class, 不相等. 第三次比较, tcls是NSObject的Meta Class的superclass也就是NSObject(Class), 不相等. 第四次比较. tcls是NSObject(Class)的superclass, 是nil, 不相等. 至此不满足循环条件, 退出循环, 返回NO;

    这里需要注意的是如果是`[(id)[[[Student alloc] init] class] isKindOfClass:[Student class]]`的话, `[Student alloc] init]`返回一个实例变量, 所以走实例方法. cls是Student(Class). 第一次比较, tcls是Student(Object)的ISA, 也就是Student(Class), 相等, 循环结束, 返回YES;
4. `[(id)[Student class] isMemberOfClass:[Student class]]`; 类似于第二题, `[Student class]`返回Student(Class), 所以走类方法, 只比较一次. cls是Student(Class), `object_getClass((id)self)`是Student(Class)的ISA, 也就是Student(Class)的Meta Class, 两者不相等, 返回NO;

### Class与内存地址

```
@interface Student : NSObject
@property (nonatomic, copy) NSString *name;
- (void)speak;
@end
@implementation Student
- (void)speak {                            
   NSLog(@"my name's %@", self.name);
}
@end
@implementation ViewController

- (void)viewDidLoad {  
  [super viewDidLoad];
  id cls = [Student class];
  void *obj = &cls;
  [(__bridge id)obj speak];
}
@end
```
如题, 先说结果: 程序可以正常运行, 结果如下:  

```
2018-03-15 22:45:52.486617+0800 HCRuntime[9838:786076] my name's <ViewController: 0x7fc349d0a670>
```
接下来我们分析一下都发生了什么. 首先, obj被转换成了一个指向`[Student class]`的指针, 然后使用`(__bridge id)`转换成了 objc_object 类型, 这时候实际上已经是一个Student类型的实例对象了, 所以可以正常调用speak方法. 那么调用speak方法会输出什么呢? 我们接下来分析.

首先我们要知道, Objective-C中的对象是一个指向ClassObject地址的变量, 即 id obj = &ClassObject, 而对象的实例变量 void *ivar = &obj + offset(N). 在C中局部变量是存储到内存的栈区, 程序运行时栈的地址从高到低. C语言到头来讲是一个顺序运行的语言, 随着程序运行, 栈中的地址越来越低. 我们来看下程序运行到执行到speak方法时, 栈中都放了哪些变量. 

首先, 执行会有两个隐藏参数传进来 self() 和 _cmd这两个参数依次被压入栈中. 然后调用`[super viewDidload]`方法, 这个方法实际是调用的`OBJC_EXPORT id objc_msgSendSuper2(struct objc_super *super, SEL op, ...)`, `objc_msgSendSuper2`方法入参是一个`objc_super *super`, `objc_super`的结构如下.

```
struct objc_super {  
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif
```
在调用`[super viewDidload]`方法时, 又产生了几个局部变量, `super_class`(等同于self.class)和`receiver`(等同于self), 依次被压入栈. 调用完`[super viewDidload]`方法后, 又产生了一个局部变量obj, 被压入栈.

这时候栈中一共有五个变量, 地址从高到低分别是: 第一个是self和第二个是隐藏参数_cmd, 第三个是self.class和第四个self是`[super viewDidLoad]`方法执行时候的参数, 第五个是obj. 当我们调用self.name的时候, 本质上就是self指针在内存向高位地址偏移一个指针. 如果我们打印对象地址就可以看出, obj就是cls的地址, obj就是Student(Object), 所以self.name也就是obj的地址向上偏移一个指针, 指向了第四个参数self, 也就是viewController的地址.

### Category

```
下面的代码会？Compile Error / Runtime Crash / NSLog…?

 @interface NSObject (Student)
 + (void)foo;
 - (void)foo;
 @end

 @implementation NSObject (Student)
 - (void)foo {
    NSLog(@"IMP: -[NSObject(Student) foo]");
 }

@end

#import "NSObject+Student.h"

int main(int argc, char * argv[]) {
    @autoreleasepool {
        [NSObject foo];
        [[NSObject new] foo];

        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
如题, 先说结果, 程序可以正常运行, 结果如下: 

```
2018-03-17 22:47:42.695913+0800 HCRuntime[7043:463076] IMP: -[NSObject(Student) foo]
2018-03-17 22:47:42.700542+0800 HCRuntime[7043:463076] IMP: -[NSObject(Student) foo]
```
category中新增的方法, 如果是实例方法, 协议以及属性是直接添加到当前类上面, 如果是类方法和协议则会添加到当前类的原类上面去. 

所以调用`[NSObject foo]`时, 由于是类方法, 根据objc_msgSend的相关知识, 会先在NSObject(Class)的isa中也就是NSObject的meta-class中, 去查找foo方法的IMP, 没找到, 然后去NSObject的meta-class的superclass(NSObject)中找, 找到了, 执行foo方法. 调用`[[NSObject new] foo]`时, 会先在NSObject(Object)的isa中, 也就是NSObject(Class)中找, 找到了, 直接执行, 输出结果.
## Runtime的常见用法
讲了那么多, 下面看下Runtime在实际应用中主要有用法. 

### 实现多继承Multiple Inheritance
转发和继承相似, 可以用于为Objc编程添加一些多继承的效果, 但是本身是不支持多继承的, 但是我们可以通过消息转发机制来实现多继承的功能. 

消息转发提供了许多类似于多继承的特性, 但是他们之间有一个很大的不同. 多继承合并了不同的行为特征在一个单独的对象中, 会得到一个重量级多层面的对象. 而消息转发则是将各个功能分散到不同的对象中, 得到的一些轻量级的对象, 这些对象通过消息转发联合起来.

我们要重写消息转发函数, 并在其中, 把我们想要转发的类做一个判断, 类似与下面代码:

```
//我们在A类中重写他的消息转发方法, 把他转发给B类的对象
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector {
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
        signature = [objectB methodSignatureForSelector:selector];
    }
    return signature;
}
```
这样, 虽然我们可以正常运行这个方法, 但是如果我们调用`respondsToSelector:`, `isKindOfClass:`和 `instancesRespondToSelector:`方法, 还有如果我们使用了协议, 那么`conformsToProtocol:`和上面三个方法都是要被重写的, 因为这类方法只会考虑继承体系, 不会考虑转发链. 
### Method Swizzling
Method Swizzling本质上就是对IMP和SEL进行交换, 当Method Swilzzling代码执行完毕之后互换才起作用, Method Swizzling也是iOS中AOP(面相切面编程)的一种实现方式. 我们替换ViewController的`viewWillAppear:`方法为例, 使用方式如下: 

```
@implementation ViewController (MethodSwizzling)

+ (void)load {
    static dispatch_once_t once;
    dispatch_once(&once, ^{
        Class class = [self class];
        //这里需要注意, 如果要Swizzling类方法则要获取当前类的原类, 因为根据objc_msgSend, 实例方法我们从对象的isa也就是对象所在的类中开始找, 类方法则是从类的isa也就是类的原类中开始找. object_getClass((id)self) 与 [self class] 返回的结果类型都是 Class, 但前者为元类, 后者为其本身.
        // Class class = object_getClass((id)self);
        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        //我们先把要替换的类添加到category中
        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (didAddMethod) {
            //class_replaceMethod相当于直接调用class_addMethod向类中添加该方法的实现
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            //交换IMP, IMP是函数指针, 直接指向方法的内存地址
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"%@ viewWillAppear", self);
}
@end
```
这里有几点是要注意的: 

1. Objective-C在运行时会自动调用类的两个方法`+load`和`+initialize`. `+load`会在类初始加载时调用, `+initialize`方法是以懒加载的方式被调用的, 只有当你给某个类或它的子类发送消息, 那么这个类的`+initialize`方法才会被调用. 所以Swizzling要写在`+load`方法中, 因为写在`+initialize`方法中, 是有可能永远都不被执行. 
2. Swizzling应该只被执行一次, 如果Swizzling的方法被多次执行, 那么就有可能造成Swizzling失效, 所以我们要用dispatch_once来保证只被执行一次. 
3. Swizzling在`+load`中执行时, 不要调用`[super load]`. 如果是多继承, 并且对同一个方法都进行了Swizzling, 那么调用[super load]以后, 父类的Swizzling就失效了.
4. 在swizzling的过程中, 方法中的`[self xxx_viewWillAppear:animated]`已经被重新指定到UIViewController类的`-viewWillAppear:`中. 这时不会产生无限循环. 如果我们调用的是`[self viewWillAppear:animated]`, 因为`viewWillAppear:`被重定向到`xxx_viewWillAppear:`, 就会产生无限循环.
5. 如果要Swizzling类方法则要获取当前类的原类, 因为根据objc_msgSend, 实例方法我们从对象的isa也就是对象所在的类中开始找, 类方法则是从类的isa也就是类的原类中开始找. `object_getClass((id)self)` 与 `[self class]` 返回的结果类型都是 Class, 但前者为元类, 后者为其本身.
6. 我们也可以使用这个来做一些异常保护, 例如数组的越界问题, 我们可以Swizzling数组的`objectAtIndex:`方法, 在新方法中做一些异常处理, 来抛出一些异常信息, 方便我们定位问题.
### Isa Swizzling
在Objective-C中, 所有的类自身也是一个对象, 这个对象的Class里面也有一个isa指针, 它指向metaClass(元类). 而消息的转发objc_msgSend也是从他的isa开始查找方法列表, 所以如果我们替换了isa, 实际也相当于替换了类. 

可以参考[KVO的实现](http://hchong.net/2018/01/24/KVO%E8%AF%A6%E8%A7%A3/). KVO在调用addObserver方法之后, 苹果的做法是在执行完 `addObserver: forKeyPath: options: context:` 方法之后, 把isa指向到另外一个类去. 
### AOP
面向切面编程, 常用的第三方库有一个[Aspects](https://github.com/steipete/Aspects). 他的实现逻辑我们可以看[这篇文章](). 下面我们来分析一下我们如何为自己实现一个AOP.

AOP的多数操作就是在forwardInvocation([消息转发的第二阶段](#消息的转发))中完成的. 一般会分为2个阶段, 一个是Intercepter注册阶段, 一个是Intercepter执行阶段.

首先会把类里面的某个要切片的方法的IMP加入到Aspect中, 类方法里面如果有forwardingTargetForSelector:的IMP, 也要加入到Aspect中. 然后对类的切片方法和forwardingTargetForSelector:的IMP进行替换, 两者的IMP相应的替换为objc_msgForward()方法和hook过的forwardingTargetForSelector:. 这样主要的Intercepter注册就完成了. 

当执行func()方法的时候, 会去查找它的IMP, 现在它的IMP已经被我们替换为了objc_msgForward()方法, 于是开始查找备援转发对象. 查找备援接受者调用forwardingTargetForSelector:这个方法, 由于这里是被我们hook过的, 所以IMP指向的是hook过的forwardingTargetForSelector:方法. 这里我们会返回Aspect的target, 即选取Aspect作为备援接受者. 有了备援接受者之后, 就会重新objc_msgSend. 从消息发送阶段重头开始. objc_msgSend找不到指定的IMP, 再进行_class_resolveMethod, 这里也没有找到, forwardingTargetForSelector:这里也不做处理, 接着就会methodSignatureForSelector. 在methodSignatureForSelector方法中创建一个NSInvocation对象, 传递给最终的forwardInvocation方法. 

Aspect里面的forwardInvocation方法会干所有切面的事情. 这里转发逻辑就完全由我们自定义了. Intercepter注册的时候我们也加入了原来方法中的method()和forwardingTargetForSelector:方法的IMP, 这里我们可以在forwardInvocation方法中去执行这些IMP. 在执行这些IMP的前后都可以任意的插入任何IMP以达到切面的目的.
### 动态的增加方法
在消息发送阶段, 如果在父类中也没有找到相应的IMP, 就会执行resolveInstanceMethod方法. 在这个方法里面, 我们可以动态的给类对象或者实例对象动态的增加方法. 

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    
    NSString *selectorString = NSStringFromSelector(sel);
    if ([selectorString isEqualToString:@"method1"]) {
        class_addMethod(self.class, @selector(method1), (IMP)functionForMethod1, "@:");
    }
    
    return [super resolveInstanceMethod:sel];
}
```
关于方法操作的函数还有以下这些: 

```
// 调用指定方法的实现
id method_invoke ( id receiver, Method m, ... );  
// 调用返回一个数据结构的方法的实现
void method_invoke_stret ( id receiver, Method m, ... );  
// 获取方法名
SEL method_getName ( Method m );  
// 返回方法的实现
IMP method_getImplementation ( Method m );  
// 获取描述方法参数和返回值类型的字符串
const char * method_getTypeEncoding ( Method m );  
// 获取方法的返回值类型的字符串
char * method_copyReturnType ( Method m );  
// 获取方法的指定位置参数的类型字符串
char * method_copyArgumentType ( Method m, unsigned int index );  
// 通过引用返回方法的返回值类型字符串
void method_getReturnType ( Method m, char *dst, size_t dst_len );  
// 返回方法的参数的个数
unsigned int method_getNumberOfArguments ( Method m );  
// 通过引用返回方法指定位置参数的类型字符串
void method_getArgumentType ( Method m, unsigned int index, char *dst, size_t dst_len );  
// 返回指定方法的方法描述结构体
struct objc_method_description * method_getDescription ( Method m );  
// 设置方法的实现
IMP method_setImplementation ( Method m, IMP imp );  
// 交换两个方法的实现
void method_exchangeImplementations ( Method m1, Method m2 );
```
### 动态关联对象
实现原理[看这里](http://blog.leichunfeng.com/blog/2015/06/26/objective-c-associated-objects-implementation-principle/), [看这里](https://draveness.me/ao). 这个也很常用, 主要涉及到三个函数: 

```
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)  
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);

OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)  
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);

OBJC_EXPORT void objc_removeAssociatedObjects(id object)  
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);
```
上面需要传入的几个参数的含义:

1. id object 设置关联对象的实例对象

2. const void *key 区分不同的关联对象的 key。这里会有3种写法。

    ```
    //使用 &AssociatedObjectKey 作为key值
    static char AssociatedObjectKey = "AssociatedKey";
    
    //使用AssociatedKey 作为key值
    static const void *AssociatedKey = "AssociatedKey";
    
    //使用@selector    
    @selector(associatedKey)
    ```
3. id value 关联的对象
4. objc_AssociationPolicy policy 关联对象的存储策略, 它是一个枚举, 与property的attribute 相对应

    ```
    typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
        //弱引用关联对象
        OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
        //强引用关联对象且为非原子操作
        OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                                *   The association is not made atomically. */
        //复制关联对象, 切位非原子操作
        OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                                *   The association is not made atomically. */
        //强引用关联对象, 且为原子操作
        OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                                *   The association is made atomically. */
        //复制关联对象, 且为原子操作
        OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                                *   The association is made atomically. */
    };
    ```
    
例如我们给NSobjet的category增加一个name属性

```
#import <Foundation/Foundation.h>

@interface NSObject (Student)
- (NSString *)name;

- (void)setName:(NSString *)name;
@end

#import "NSObject+Student.h"
#import <objc/runtime.h>

@implementation NSObject (Student)

- (NSString *)name {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```
### NSCoding的自动归档和自动解档
这个太常用了, 主要思路是获取成员变量列表, 利用KVC读取和赋值来完成`encodeWithCoder`和`initWithCoder`.

```
#import "Student.h"
#import <objc/runtime.h>
#import <objc/message.h>

@implementation Student

- (void)encodeWithCoder:(NSCoder *)aCoder{
    unsigned int outCount = 0;
    Ivar *vars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar var = vars[i];
        const char *name = ivar_getName(var);
        NSString *key = [NSString stringWithUTF8String:name];

        id value = [self valueForKey:key];
        [aCoder encodeObject:value forKey:key];
    }
}

- (nullable __kindof)initWithCoder:(NSCoder *)aDecoder{
    if (self = [super init]) {
        unsigned int outCount = 0;
        Ivar *vars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar var = vars[i];
            const char *name = ivar_getName(var);
            NSString *key = [NSString stringWithUTF8String:name];
            id value = [aDecoder decodeObjectForKey:key];
            [self setValue:value forKey:key];
        }
    }
    return self;
}
@end
```
### 字典和模型互相转换
1. 字典转模型

    1. 调用 class_getProperty 方法获取当前 Model 的所有属性.
    2. 调用 property_copyAttributeList 获取属性列表.
    3. 根据属性名称生成 setter 方法.
    4. 使用 objc_msgSend 调用 setter 方法为 Model 的属性赋值（或者 KVC)
    示例代码如下: 
    
    ```
    +(id)objectWithKeyValues:(NSDictionary *)aDictionary{
        id objc = [[self alloc] init];
        for (NSString *key in aDictionary.allKeys) {
            id value = aDictionary[key];
            
            /*判断当前属性是不是Model*/
            objc_property_t property = class_getProperty(self, key.UTF8String);
            unsigned int outCount = 0;
            objc_property_attribute_t *attributeList = property_copyAttributeList(property, &outCount);
            objc_property_attribute_t attribute = attributeList[0];
            NSString *typeString = [NSString stringWithUTF8String:attribute.value];
            
            //防止model嵌套, 比如说Student里面还有一层Student, 那么这里就需要再次转换一次, 当然这里有几层就需要转换几次.
            if ([typeString isEqualToString:@"@\"Student\""]) {
                value = [self objectWithKeyValues:value];
            }
            
            //生成setter方法，并用objc_msgSend调用
            NSString *methodName = [NSString stringWithFormat:@"set%@%@:",[key substringToIndex:1].uppercaseString,[key substringFromIndex:1]];
            SEL setter = sel_registerName(methodName.UTF8String);
            if ([objc respondsToSelector:setter]) {
                ((void (*) (id,SEL,id)) objc_msgSend) (objc,setter,value);
            }
            free(attributeList);
        }
        return objc;
    }
    ```
    几个出名的开源库JSONModel, MJExtension等都是通过这种方式实现的. 利用runtime的class_copyIvarList获取属性数组, 遍历模型对象的所有成员属性, 根据属性名找到字典中key值进行赋值, 当然这种方法只能解决NSString, NSNumber等. 如果含有NSArray或NSDictionary, 还要进行第二步转换, 如果是字典数组, 需要遍历数组中的字典, 利用objectWithDict方法将字典转化为模型, 在将模型放到数组中, 最后把这个模型数组赋值给之前的字典数组.
2. 模型转字典
    
    1. 调用 class_copyPropertyList 方法获取当前 Model 的所有属性. 
    2. 调用 property_getName 获取属性名称.
    3. 根据属性名称生成 getter 方法.
    4. 使用 objc_msgSend 调用 getter 方法获取属性值（或者 KVC）.
示例代码如下: 

    ```
    //模型转字典
    -(NSDictionary *)keyValuesWithObject{
        unsigned int outCount = 0;
        objc_property_t *propertyList = class_copyPropertyList([self class], &outCount);
        NSMutableDictionary *dict = [NSMutableDictionary dictionary];
        for (int i = 0; i < outCount; i ++) {
            objc_property_t property = propertyList[i];
            
            //生成getter方法，并用objc_msgSend调用
            const char *propertyName = property_getName(property);
            SEL getter = sel_registerName(propertyName);
            if ([self respondsToSelector:getter]) {
                id value = ((id (*) (id,SEL)) objc_msgSend) (self,getter);
                
                /*判断当前属性是不是Model*/
                if ([value isKindOfClass:[self class]] && value) {
                    value = [value keyValuesWithObject];
                }
    
                if (value) {
                    NSString *key = [NSString stringWithUTF8String:propertyName];
                    [dict setObject:value forKey:key];
                }
            }
            
        }
        free(propertyList);
        return dict;
    }
    ```

------
参考资料:

1.[神经病院 Objective-C Runtime 入院系列](https://halfrost.com/objc_runtime_isa_class/)

2.[Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)

3.[Objective-C Runtime 运行时系列](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)

4.[iOS 如何实现 Aspect Oriented Programming](https://halfrost.com/ios_aspect/)

5.[Method Swizzling 和 AOP 实践](http://tech.glowing.com/cn/method-swizzling-aop/)

6.[刨根问底Objective－C Runtime](http://www.cocoachina.com/ios/20141224/10740.html)

7.[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)

8.[Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)


