---
title: Block用法与原理分析
date: 2017-07-04 21:11:18
tags:
    - Block
    - 基础知识
categories: 
    - 基础知识

---

# Block用法及分析

Block的默认格式是这样的: **`返回值类型 (^Block变量名)(形参列表) = ^返回值类型 (形参列表){ 内容 }`**. 后面的返回值类型和形参列表可以省略. 

Block用一句话来形容就是带有自动变量(局部变量)的匿名函数. 他可以嵌套定义, 定义Block方法和定义函数方法类似, Block可以定义在方法内部和外部, 必须调用Block, 才会执行`{}`内部的方法. 本质是对象, 使代码高聚合. 结合`clang`分析可以发现Block的真实面目.

```
struct Block_layout {  
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};

struct Block_descriptor {  
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};
```

![block的数据结构](http://img.souche.com/f2e/d71a93dbbbf11a6055810df74fdf309d.png)

可以看到有`isa`的存在, 由此可以说明, OC处理Block是按照对象来处理的. 在iOS中, isa常见的就是_NSConcreteStackBlock, _NSConcreteMallocBlock, _NSConcreteGlobalBlock这三种类型. 

## 常见的使用方法

常见的Block的使用方式有以下几种:

### 作为变量的用法

官方提供的快捷写法(inlineBlock)的示例是这样的:

```
<#returnType#>(^<#blockName#>)(<#parameterTypes#>) = ^(<#parameters#>) {
   <#statements#>
};
```

我们举个🌰

```
//定义一个block变量sum, 并且赋值(此处省略返回值类型int)
int(^sum)(int, int) = ^(int a, int b) {
   return a + b;
};
//调用block变量
int count = sum(2, 3);
NSLog(@"%d", count);
    
也可以吧定义和赋值分开来写, 类似于这样

//定义一个block变量
int (^sum)(int, int);
//给block变量赋值(此处没有省略返回值类型)
sum = ^int(int a, int b) {
    return a + b;
};
int count = sum(2, 3);
    NSLog(@"%d", count);    
```

### 作为属性的用法

block作为属性, 可以存在在找那个类的声明周期中, 这样就可以全局的使用, 也比较常用.

```
1. 定义一个block属性
@property (nonatomic, copy) int (^sum)(int a, int b);

也可以使用`typedefBlock`的方式来定义(推荐)
typedef int(^Sum)(int, int);

@property (nonatomic, copy) Sum sum;


2. 然后在合适的地方写该属性(block)的实现, 
self.sum = ^int(int a, int b) {
    return a + b;
};

3. 最后是block的调用
int count = self.sum(3, 4);
NSLog(@"%d", count);
```

### 作为函数(方法)参数的用法

在这里C和OC的还略有不同. 

block作为函数参数的用法也是较常见的, 例如网络请求方法中, 会把网络请求结果处理的代码在block中, 等请求到数据的时候直接调用.

block的定义也可以分为两种情况, 一种是使用了`typedefBlock`来定义, 一种是直接定义, 下面例子会给出代码示例. 假设有个下载的网络请求, 会有成功和失败的代码处理被我们封装在block里面, 作为参数在外面被调用.

```

//.h文件
#import <Foundation/Foundation.h>
typedef void(^SuccessBlock)(id obj);
typedef void(^FailBlock)(id obj);

@interface DownLoadManager : NSObject

+ (void)downLoadedSuccess:(SuccessBlock)success fail:(FailBlock)fail;

+ (void)uploadSuccess:(void(^)(id obj))success fail:(void(^)(id obj))fail;
@end


//.m文件
#import "DownLoadManager.h"

@implementation DownLoadManager

+ (void)downLoadedSuccess:(SuccessBlock)success fail:(FailBlock)fail {
    //使用延迟来模拟异步数据请求
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        success(@"我是数据");
    });
}

+ (void)uploadSuccess:(void(^)(id obj))success fail:(void(^)(id obj))fail {
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        success(@"我是数据");
    });
}
@end


//在外面调用: 
[DownLoadManager downLoadedSuccess:^(id obj) {
    NSLog(@"%@", obj);
} fail:^(id obj) {
        
}];

[DownLoadManager uploadSuccess:^(id obj) {
   
} fail:^(id obj) {
   
}];
```

可以看到, 这里的作为参数有两种方式, 一种是使用`typedef`定义, 一种是直接定义, 这两种方式和作为属性的两种用法基本类似.

下面是在C中的使用方法. 对比一下可以发现还是有一些差别的, 在OC中直接使用的话, block的name是没有定义的, C中是有的.

```
// 实现，不使用 typedef
// void (^iblock)()作为函数参数类型，iblock函数形参名
void iprint( void (^iblock)() ){ 
    iblock(); // 调用 block参数}

// 实现，使用 typedef
typedef void (^IBlock)();
void iprint(IBlock iblock){
    iblock(); // 调用 block参数

}

// 调用「不管使用不使用 typedef 调用方式一致」
iprint(^{
    NSLog(@"传入的实参代码块区域");
});
```

### 作为返回值的用法

block作为返回值的用法主要使用场景是函数的链式编程中, 可以参考[这篇文章](http://hchong.net/2017/12/17/%E9%93%BE%E5%BC%8F%E7%BC%96%E7%A8%8B%E5%AE%9E%E8%B7%B5/).

```
typedef NSString *(^NameBlock)(NSString *inputalue);//定义一个返回值是String, 参数是String类型的block, 名字为NameBlock

@property (nonatomic, copy) NameBlock nameBlock;

//实现属性的get方法
- (NSString *(^)(NSString *))nameBlock {
    return ^NSString *(NSString *inputValue) {
        return [inputValue stringByAppendingString:@"test"];
    };
}

//调用方式如下
NSLog(@"%@", self.nameBlock(@"block作为返回值"));
```

### 常见的使用场景

* Enumeration (像我们上面看到的 NSArray 的枚举接口)

* View Animations (animations)

* Sorting (在排序时在 Block 中实现比较逻辑)

* Notification (当某些事件被触发时，执行对应的 Block)

* Error handlers (作为错误事件的 Handler)

* Completion handlers (作为某个任务完成时的 Handler)

* Multithreading (在 Grand Central Dispatch (GCD) API 中使用)

## 捕获外部变量

C语言中的变量一共有五种: 自动变量, 函数参数, 静态局部变量, 静态全局变量, 全局变量. 要研究外部变量的捕获就去除掉函数参数这一项. 下面逐一分析.

```
static int staticGlobalInt = 0;//静态全局变量
int globalInt;//全局变量

//测试Block的变量捕获
- (void)variableTest {
    static int staticInt = 0;//静态局部变量
    __block int intNumber = 0;//局部变量
    globalInt = 0;//全局变量
    staticGlobalInt = 0;//静态全局变量
    
    NSLog(@"初始化时---静态全局变量:%d, 全局变量:%d, 静态局部变量:%d, 局部变量:%d", staticGlobalInt, globalInt, staticInt, intNumber);
    
    void(^addTest)() = ^(){
        staticGlobalInt++;
        globalInt++;
        staticInt++;
        intNumber++;
        NSLog(@"在block中---静态全局变量:%d, 全局变量:%d, 静态局部变量:%d, 局部变量:%d", staticGlobalInt, globalInt, staticInt, intNumber);
    };
    NSLog(@"在block调用结束后---静态全局变量:%d, 全局变量:%d, 静态局部变量:%d, 局部变量:%d", staticGlobalInt, globalInt, staticInt, intNumber);
    addTest();
}
```

在这里, 如果不加`__block`的话, 局部变量在block内部使用是会报错的, 原因后面讲. 这里先来说一下结果, 打印的结果是:

```

初始化时---静态全局变量:0, 全局变量:0, 静态局部变量:0, 局部变量:0
在block调用结束后---静态全局变量:0, 全局变量:0, 静态局部变量:0, 局部变量:0
在block中---静态全局变量:1, 全局变量:1, 静态局部变量:1, 局部变量:1
```
可以发现结果都变了, 但是内部实现实际上不同的.

* 静态全局变量和静态局部变量, 由于他们是存储在全局区(数据区), 作用域的范围是整个程序的生命周期, 只要程序在运行, 就可以访问到. 所以block直接访问了对应的变量, 而没有把他们copy到block中去

* 静态局部变量也是存储在全局区(数据区), 程序在运行就可以访问的到, 但是他的作用范围仅限于定义他的函数中. 系统是把内存地址传递给block, 所以在block也可以直接修改他的的值.

* 局部变量, 必须使用`__block`(存储域类说明符)来修饰, 否则在block内部使用会报错. 对于非对象的变量来说, 自动变量的值, 被copy进了Block, 不带`__block`的自动变量只能在里面被访问, 并不能改变值. 对于对象来说, 在MRC环境下, `__block`根本不会对指针所指向的对象执行copy操作, 而只是把指针进行的复制. 而在ARC环境下, 对于声明为`__block`的外部对象, 在block内部会进行retain, 以至于在block环境内能安全的引用外部对象. 对于没有声明`__block`的外部对象, 在block中也会被retain

*注意*: Block捕获外部变量仅仅只捕获Block闭包里面会用到的值，其他用不到的值，它并不会去捕获.

## Block的存储域相关

通过之前的源码分析可以看出, Block结构体中是有一个isa指针的, 这也就说明Block实际也是一个OC的对象. 在OC中一般Block分为三种: 

### _NSConcreteStackBlock
该类对象的存储域在栈上面. 只用到外部局部变量, 成员属性变量, 且没有强指针引用的block都是StackBlock. StackBlock的生命周期由系统控制的. 由于存储在栈上面, 一旦返回之后所属的变量域一旦结束, 就被系统销毁了. 所以他是不安全的 *该类型的block不持有对象*.

需要注意, 由于`_NSConcreteStackBlock`所属的变量域一旦结束, 那么该Block就会被销毁. 在ARC环境下, 编译器会自动的判断, 把Block自动的从栈copy到堆上. 

### _NSConcreteGlobalBlock
与global变量一样, 该类对象的存储域在数据区(全局区). 一般情况下, 没有用到外界变量或只用到全局变量, 静态变量的block为_NSConcreteGlobalBlock. 由于存储在数据区(全局区), 所以生命周期从创建到应用程序结束. *该类型的block不持有对象*, 因为要么不引用外部变量, 要么使用的是全局变量或者静态变量.

###  _NSConcreteMallocBlock
该类对象设置在由malloc函数分配的内存块(堆)中. 有强指针引用或copy修饰的成员属性引用的block会被复制一份到堆中成为MallocBlock, 没有强指针引用即销毁, 生命周期由程序员控制. *该类型的block会持有外部对象*.

### 总结
Block本身也是一个对象, 那么他自身的存储域, 生命周期和作用域也是我们要了解的.由于`_NSConcreteStackBlock`是在栈上面的, 容易被销毁, 所以我们需要把它copy到堆上面进行操作, 在ARC下, 如果有下面几种方式系统会自动把`block`copy到堆上面去:

	* 手动调用copy 
	* Block 作为函数返回值返回时
	* 将 Block 赋值给类的`__strong` 修饰的 id 类型的成员变量时
	* 将 Block 赋值给类 Block 类型成员变量时
	* 向 GCD 的 API 中或方法名中含有 usingBlock 的 Cocoa 框架方法传递 Block 作为参数时

下面两种情况需要我们自己调用copy方法:

	* 在向一般方法或函数传递 Block 作为参数时，传的时候要调用一下 Block 的 copy 方法。除非在方法或函数体中，对传进来的 Block 参数做了 copy 处理
	* 如果不明确情况, 也推荐手动调用 copy.

Block对象如果内部使用了`__block`修饰的局部变量, 那么当Block从栈上复制到堆上时, `__block`变量也会被copy到堆上面, 并且Block会持有这个变量. 当堆上的 Block 被废弃时, 那么它所使用的 `__block` 变量也会被释放.

## block几种关键字的用法
### __block

如果想在 Block 中读写局部变量, 那么需要在局部变量前加 `__block`. `__block`实际上是提升了变量的作用域. 如果获得对象（在堆上）, 调用变更该对象的方法是没问题的（存储对象的空间在堆上）, 而直接向截获的变量赋值则会产生编译错误（这个对象的指针是在栈上的）.

1. `__block`修饰变量
    ARC环境下, 一旦Block赋值就会触发copy, `__block`就会copy到堆上, Block也是`__NSMallocBlock`. ARC环境下也是存在`__NSStackBlock`的时候, 这种情况下, `__block`就在栈上.
    
    MRC环境下, 只有copy, `__block`才会被复制到堆上, 否则, `__block`一直都在栈上, block也只是__NSStackBlock, 这个时候`__forwarding`指针就只指向自己了.

2. `__block`修饰对象
    对象在OC中, 默认声明自带`__strong`所有权修饰符的
    
    ```
    __block id block_obj = [[NSObject alloc]init];  
    id obj = [[NSObject alloc]init];
    等价于
    Objective-C
    __block id __strong block_obj = [[NSObject alloc]init];  
    id __strong obj = [[NSObject alloc]init];
    ```
    在ARC环境下, 不仅仅是声明了`__block`的外部对象, 没有声明`__block`的对象, 在block内部也会被retain. 因为加了`__block`, 只是对一个自动变量有影响, 它们是指针, 相当于延长了指针变量的声明周期, 只要访问对象的话还是会retain.
    
    在MRC环境下, `__block`根本不会对指针所指向的对象执行copy操作, 而只是把指针进行的复制. 
    
### __weak

`__weak`修饰的对象被Block捕获时是对其进行弱引用持有的, 因为`__NSConcreteMallocBlock`捕获外部对象会在内部持有他, 引用计数会+1. 如果使用`__weak`修饰外部变量, Block捕获的变量就会是弱引用持有. 当Block所有者的作用域结束时, 他指向的对象没有被其他强引用持有, 所以立即被释放, 这是Block内部持有的弱引用也被置为nil.

*注意*: block并不会捕获形参到block内部进行持有. 例如下面这样:

```
Student *student = [[Student alloc]init];
student.name = @"Hello World";

student.study = ^(NSString * name){
    NSLog(@"my name is = %@",name);
};
student.study(student.name);
```

### __strong

`__strong`修饰的对象被Block捕获时是对其进行强引用持有的. 当Block的所有者的作用域结束时, `__strong`修饰的对象依然被Block强引用持有, 所以不会立即释放.

### weakSelf & strongSelf
weakSelf 是为了block不持有self，避免Retain Circle循环引用。在 Block 内如果需要访问 self 的方法、变量，建议使用 weakSelf. weakSelf有下面两种写法.

```
__weak __typeof(self)weakSelf = self;
#define WEAKSELF typeof(self) __weak weakSelf = self;
```

strongSelf的目的是因为一旦进入block执行, 假设不允许self在这个执行过程中释放, 就需要加入strongSelf. block执行完后这个strongSelf 会自动释放, 不会存在循环引用问题。如果在 Block 内需要多次 访问 self，则需要使用 strongSelf. strongSelf的写法如下:

```
__weak __typeof(self)weakSelf = self;
__strong __typeof(weakSelf)strongSelf = weakSelf
```

### 避免Block使用的对象被提前释放
在 Block 中用异步的方式使用了外部对象, 当对象被释放后, 异步方法回调时访问该对象则会为空, 这时就可能造成程序崩溃了. 解决这个问题的方式则是 `__weak`/`__strong`. 例如下面这种样式的: 

```
@implementation TestBlockViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // Init properties.
    self.tag = @"tag is OK.";
    
    // Init TestService's block.
    typeof(self) __weak weakSelf = self;
    self.myBlock = ^{
        typeof(weakSelf) __strong strongSelf = weakSelf;
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSLog(@"strongSelf is OK.");
            NSLog(@"%@", strongSelf.tag);
            //NSLog(@"%@", self.tag); // Retain cycle.
        });
    };
    
}
- (void)backButtonAction {    
    self.myBlock();
    [self.navigationController popViewControllerAnimated:YES];
}
- (void)dealloc {
    NSLog(@"TestBlockViewController dealloc.");
}
@end
```

-----
参考资料: 

1.[深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)

2.[Block技巧与底层解析](http://www.jianshu.com/p/51d04b7639f1)

3.[深入研究Block用weakSelf、strongSelf、@weakify、@strongify解决循环引用](https://halfrost.com/ios_block_retain_circle/)

4.[浅析iOS中Block的用法](http://www.jianshu.com/p/581151493340)

5.[深入浅出Block](http://www.jianshu.com/p/e03292674e60)

6.[Block总结](http://www.samirchen.com/block-in-objc/)


