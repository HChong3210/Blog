---
title: 属性修饰符之copy和strong
date: 2016-09-21 17:46:10
tags:
- 基础知识
categories:
- 基础知识
---

# 属性修饰符之copy和strong

在了解属性修饰符的copy和strong的区分之前, 我们先来了解下浅拷贝和深拷贝的区别.

## 深浅拷贝

浅复制并不是拷贝对象本身, 仅仅是拷贝指向对象的指针; 深复制是直接拷贝整个对象内存到另一块内存中. 简单地说:

* **浅复制就是指针拷贝,  不会产生新的对象，指向的是同一个对象; **
* **深复制就是内容拷贝，会产生新的对象, 不同的对象, 内容相同.**

![浅复制和深复制](http://7s1ssm.com1.z0.glb.clouddn.com/image_note50592_1.png)

## copy和mutableCopy

`copy`就是复制了一个imutable对象, `mutableCopy`就是复制了一个mutable对象.

一个`NSObject`对象要想使用这两个函数, 必须实现`NSCopying`协议和`NSMutableCopying`协议, 并且分别实现`- (id)copyWithZone:(nullable NSZone *)zone;`和`- (id)mutableCopyWithZone:(nullable NSZone *)zone;`方法.但是常见的`NSString`, `NSArray`, `NADictionary`等常用的系统提供的结构体都已经实现. 

### 系统的非容器类对象

这里指的是`NSString`, `NSNumber`等等一类的对象.下面以`NSString`为例.

对`NSString`进行copy和mutableCop操作:

```objective-c
NSString *originString = @"origin";
NSString *originStringCopy = [originString copy];
NSMutableString *originStringMutableCopy = [originString mutableCopy];
NSLog(@"%p, %p, %p", originString, originStringCopy, originStringMutableCopy);
```

内存地址分别是:

```objective-c
(lldb) p originString
(__NSCFConstantString *) $0 = 0x0000000108dd8190 @"origin"
(lldb) p originStringCopy
(__NSCFConstantString *) $1 = 0x0000000108dd8190 @"origin"
(lldb) p originStringMutableCopy
(__NSCFString *) $2 = 0x0000608000262800 @"origin"
```

---

对`NSMutableString`进行copy和mutableCopy操作:

```objective-c
NSMutableString *mutableOriginString = [NSMutableString stringWithString:@"mutableOrigin"];
NSString *mutableOriginStringCopy = [mutableOriginString copy];
NSMutableString *mutableOriginStringMutableCopy = [mutableOriginString mutableCopy];
```

内存地址分别是:

```objective-c
(lldb) p mutableOriginString
(__NSCFString *) $0 = 0x000060800026aa40 @"mutableOrigin"
(lldb) p mutableOriginStringCopy
(__NSCFString *) $1 = 0x0000608000222340 @"mutableOrigin"
(lldb) p mutableOriginStringMutableCopy
(__NSCFString *) $2 = 0x000060800026a700 @"mutableOrigin"
```

这里要注意的是:

```objective-c
NSMutableString *mutableOriginStringCopy = [mutableOriginString copy];
[mutableOriginStringCopy appendString:@"123"];
```

这样写是会crash的, 因为copy生成的是imutable对象, 不管声明是什么样的, 依旧是imutable的.

**综上, 对于系统的非容器类对象:**

* **如果对一不可变对象复制, copy是指针复制(浅复制), mutableCopy是内容复制(深复制).**
* **如果对一可变对象复制, 都是深复制, 但是copy返回的对象是不可变的.**

### 系统的容器类对象

容器类对象指的是`NSArray`, `NSDictionary`, `NSSet`等系统提供的结构体, 下面以`NSArray`为例.

对`NSArray`进行copy操作:

```objective-c
NSArray *originArray = [NSArray arrayWithObjects:@"a",@"b",@"c",nil];
NSArray *originArrayCopy = [originArray copy];
NSMutableArray *originArrayMutableCopy = [originArray mutableCopy];
for (NSInteger i = 0; i < originArray.count; i++) {
    NSLog(@"originArray--%p", originArray[i]);
    NSLog(@"originArrayCopy--%p", originArrayCopy[i]);
    NSLog(@"originArrayMutableCopy--%p", originArrayMutableCopy[i]);
}
```

内存地址分别是:

```objective-c
(lldb) p originArray
(__NSArrayI *) $0 = 0x00006080002405a0 @"3 elements"
(lldb) p originArrayCopy
(__NSArrayI *) $1 = 0x00006080002405a0 @"3 elements"
(lldb) p originArrayMutableCopy
(__NSArrayM *) $2 = 0x00006080002403c0 @"3 elements"
```

可以看出, copy是浅复制, mutableCopy是深复制. 需要注意的是,mutableCopy的对象是一个可变对象,  数组内元素全都是浅复制.

---

对`NSMutableArray`进行copy和mutableCopy操作:

```objective-c
NSMutableArray *originArray = [NSMutableArray arrayWithObjects:@"a",@"b",@"c", nil];
NSArray *originArrayCopy = [originArray copy];
NSMutableArray *originArrayMutableCopy = [originArray mutableCopy];
for (NSInteger i = 0; i < originArray.count; i++) {
    NSLog(@"originArray--%p", originArray[i]);
    NSLog(@"originArrayCopy--%p", originArrayCopy[i]);
    NSLog(@"originArrayMutableCopy--%p", originArrayMutableCopy[i]);
}
```

打印内存地址如下:

```objective-c
(lldb) p originArray
(__NSArrayM *) $0 = 0x000060800005eae0 @"3 elements"
(lldb) p originArrayCopy
(__NSArrayI *) $1 = 0x000060800005e9f0 @"3 elements"
(lldb) p originArrayMutableCopy
(__NSArrayM *) $2 = 0x000060800005e900 @"3 elements"
```

可以发现可以看出, copy是浅复制, mutableCopy是深复制. 数组内元素都是浅复制.

需要注意的是, mutable对象copy的对象是imutable对象, 如果当做可变对象来用是会崩溃的.

```objective-c
NSMutableArray *originArray = [NSMutableArray arrayWithObjects:@"a",@"b",@"c", nil];
NSMutableArray *originArrayCopy1 = [originArray copy];
[originArrayCopy1 addObject:@"d"];//crash
```

综上, 对于容器类对象:

- **如果对一不可变对象复制, copy是指针复制(浅复制), mutableCopy是内容复制(深复制).**
- **如果对一可变对象复制, 都是深复制, 但是copy返回的对象是不可变的.**
- **元素对象是浅复制.**

### 系统类对象的完全深复制

对于容器类对象而言, 元素对象始终是浅复制, 要想深复制可通过如下方法:

```objective-c
NSArray *array = [NSArray arrayWithObjects:[NSMutableString stringWithString:@"a"],[NSString stringWithString:@"b"],@"c",nil];
NSArray *deepCopyArray=[[NSArray alloc] initWithArray: array copyItems: YES];
NSArray* trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject: array]];
```

打印元素内存地址可以发现, trueDeepCopyArray的元素都是深复制, 而deepCopyArray由于第一个元素是可变对象, 所以是深复制, 其他的元素都是浅复制.

**综上, 要想实现容器对象所有元素的深复制, 只能通过归档来实现.**

> If you need a true deep copy, such as when you have an array of arrays, you can archive and then unarchive the collection, provided the contents all conform to the `NSCoding` protocol

```objective-c
@protocol NSCoding

- (void)encodeWithCoder:(NSCoder *)aCoder;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder; // NS_DESIGNATED_INITIALIZER

@end
```

### 自定义对象

对于自定义对象, 我们要实现`NSCopying`, `NSMutableCopying`协议, 这样我们就能调用copy和mutableCopy了.假设有一个`Person`类, 继承于`NSObject`.

`person.h`

```objective-c
#import <Foundation/Foundation.h>
@interface Person : NSobject

@property (nonatomic, assign) NSInteger age;
@end
```



`person.m`

```objective-c

#import "Person.h"
@interface Person()<NSCopying>
@end

@implementation Person
  
- (id)copyWithZone:(NSZone *)zone {
	Person *person = [[Person allocWithZone:zone] init];
  	person.age = self.age;
    return person;
}
@end
```

这样当我们在外面调用的时候:

```objective-c
Person *p = [[Person alloc] init];
p.age = 20;

Person *copyP = [p copy];
NSLog(@"p = %p copyP = %p", p, copyP);
NSLog(@"age = %ld", copyP.age);
```

通过打印地址:

```objective-c
(lldb) p p
(Person *) $0 = 0x000060800002a1e0
(lldb) p copyP
(Person *) $1 = 0x000060800002a2e0
(lldb) p p.age
(NSInteger) $2 = 20
(lldb) p copyP.age
(NSInteger) $4 = 20
```

我们可以发现, 自定义对象内部的属性都被浅拷贝, 自定义对象本身被深拷贝.

需要注意的是, 如果我们的自定义对象不实现`NSCopying`协议而直接copy时, 是会crash的.

## 属性修饰之copy与strong

我们新建一个`Person`类, 添加几个属性来看一下copy和strong对属性使用的影响.

```objective-c
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, strong) NSString *strongName;
@property (nonatomic, copy) NSString *copyName;

@end
```

在外面调用`Person`类, 代码如下:

```objective-c
Person *person = [[Person alloc] init];
NSMutableString *testString = [NSMutableString stringWithString:@"test"];
person.nameCopy = testString;
person.nameStrong = testString;
[testString appendString:@"copy&&strong"];
```

打印`testString`改变前后属性

```objective-c
//testString改变前
(lldb) p person.nameCopy
(NSTaggedPointerString *) $1 = 0xa000000747365744 @"test"
(lldb) p person.nameStrong
(__NSCFString *) $2 = 0x0000600000260540 @"test"
//testString改变后
(lldb) p person.nameCopy
(NSTaggedPointerString *) $3 = 0xa000000747365744 @"test"
(lldb) p person.nameStrong
(__NSCFString *) $4 = 0x0000600000260540 @"testcopy&&strong"
```

由以上可以看出:

对源头是`NSMutableString`的字符串，strong仅仅是指针引用，增加了引用计数器，这样源头改变的时候，用这种strong方式声明的变量（无论被赋值的变量是可变的还是不可变的，它也会跟着改变, 相当于浅拷贝; 而copy属性则是对源字符串做了次深拷贝，产生一个新的对象，且copy属性对象指向这个新的对象。另外需要注意的是，这个copy属性对象的类型始终是`NSString`，而不是`NSMutableString`，因此其是不可变的。

当源字符串是`NSString`时，由于字符串是不可变的，所以，不管是strong还是copy属性的对象，都是指向源对象，copy操作只是做了次浅拷贝。

这里还有一个性能问题，即在源字符串是`NSMutableString`，strong是单纯的增加对象的引用计数，而copy操作是执行了一次深拷贝，所以性能上会有所差异。而如果源字符串是`NSString`时，则没有这个问题。

综上可以发现, 如果property是`NSString`或者`NSArray`及其子类的时候，最好选择使用copy属性修饰。为什么呢？这是为了防止赋值给它的是可变的数据，如果可变的数据发生了变化，那么该property也会发生变化。

参考文档:

1.[copy与mutableCopy](http://www.fanliugen.com/?p=278)

2.[Copying Collections](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html#//apple_ref/doc/uid/TP40010162-SW3)

3.[iOS浅谈: 深.浅拷贝与copy.strong](http://www.jianshu.com/p/e6a7cdcc705d)

4.[NSString属性什么时候用copy，什么时候用strong?](http://www.cocoachina.com/ios/20150512/11805.html)

