---
title: iOS开发基础-hash & isEqual
date: 2018-09-13 17:03:42
tags:
    - 基础知识
categories:
    - iOS开发-基础
---

`+ (NSUInteger)hash;`和`- (BOOL)isEqual:(id)object;`都是NSObject类中的方法, 所有继承自NSObject的类都可以调用这两个方法来实现相应的功能, 下面我们开始一起研究一下他们两个的功能和深层次原理.



# 1 isEqual

## 1.1 ==
对于基本类型, ==运算符比较的是值; 对于对象类型, ==运算符比较的是对象的地址(即是否为同一对象). isEqual只能用来比较对象类型.

==运算符只是简单地判断是否是同一个对象, 而isEqual方法可以判断对象是否相同.

## isEqual
两个 NSObject 如果指向了同一个内存地址，那它们就被认为是相同的.

在 Foundation 框架中，下面这些 NSObject 的子类都有自己的相等性检查实现，分别使用下面这些方法:

* NSAttributedString -isEqualToAttributedString:
* NSData -isEqualToData:
* NSDate -isEqualToDate:
* NSDictionary -isEqualToDictionary:
* NSHashTable -isEqualToHashTable:
* NSIndexSet -isEqualToIndexSet:
* NSNumber -isEqualToNumber:
* NSOrderedSet -isEqualToOrderedSet:
* NSSet -isEqualToSet:
* NSString -isEqualToString:
* NSTimeZone -isEqualToTimeZone:

## NSArray的内部伪实现

``` OC
@implementation NSArray (Approximate)
- (BOOL)isEqualToArray:(NSArray *)array {
  if (!array || [self count] != [array count]) {
    return NO;
  }

  for (NSUInteger idx = 0; idx < [array count]; idx++) {
      if (![self[idx] isEqual:array[idx]]) {
          return NO;
      }
  }

  return YES;
}

- (BOOL)isEqual:(id)object {
  if (self == object) {
    return YES;
  }

  if (![object isKindOfClass:[NSArray class]]) {
    return NO;
  }

  return [self isEqualToArray:(NSArray *)object];
}
@end
```

## 自定义类型的isEqual
Person.h

``` OC
@interface Person : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, strong) NSDate *birthday;

@end
```

Person.m

``` OC
- (BOOL)isEqual:(id)object 
{
    if (self == object) {
        return YES;
    }

    if (![object isKindOfClass:[Person class]]) {
        return NO;
    }

    return [self isEqualToPerson:(Person *)object];
}

- (BOOL)isEqualToPerson:(Person *)person 
{
    if (!person) {
        return NO;
    }

    BOOL haveEqualNames = (!self.name && !person.name) || [self.name isEqualToString:person.name];
    BOOL haveEqualBirthdays = (!self.birthday && !person.birthday) || [self.birthday isEqualToDate:person.birthday];

    return haveEqualNames && haveEqualBirthdays;
}
```

##  注意
``` OC
NSString *a = @"Hello";
NSString *b = @"Hello";
BOOL wtf = (a == b); // YES
```
比较 NSString 对象正确的方法是 -isEqualToString:. 任何情况下都不要直接使用 == 来对 NSString 进行比较. 

上面的结果返回YES是因为一种称为字符串驻留的优化技术，它把一个不可变字符串对象的值拷贝给各个不同的指针。NSString *a 和 *b都指向同样一个驻留字符串值 @"Hello"。 注意所有这些针对的都是静态定义的不可变字符串.

# 2 hash
hash 方法的存在，是因为将对象加到 NSSet 等集合中时，当成员被加入到Hash Table中时, 会给它分配一个hash值, 以标识该成员在集合中的位置, 通过这个位置标识可以将查找的时间复杂度优化到O(1), 当然如果多个成员都是同一个位置标识, 那么查找就不能达到O(1)了. 对于 Hash 值，系统默认是返回该对象的内存地址.

## 2.1 hash Table
分配的这个hash值(即用于查找集合中成员的位置标识), 就是通过hash方法计算得来的, 且hash方法返回的hash值最好唯一.

和数组相比, 基于hash值索引的Hash Table查找某个成员的过程就是: 

* 通过hash值直接找到查找目标的位置
* 如果目标位置上有多个相同hash值得成员, 此时再按照数组方式进行查找

NSSet添加新成员时, 需要根据hash值来快速查找成员, 以保证集合中是否已经存在该成员.
NSDictionary在查找key时, 也利用了key的hash值来提高查找的效率.

## 2.2 如何重写hash
Person类中, 我们重写他的hash方法, 直接返回`[super hash]`是有问题的. 下面我们举例来来说明问题.

``` OC
Person *person1 = [Person personWithName:kName1 birthday:self.date1];
Person *person2 = [Person personWithName:kName1 birthday:self.date1];
NSLog(@"[person1 isEqual:person2] = %@", [person1 isEqual:person2] ? @"YES" : @"NO");

NSMutableSet *set = [NSMutableSet set];
[set addObject:person1];
[set addObject:person2];
NSLog(@"set count = %ld", set.count);
```

打印结果如下:

``` OC
[person1 isEqual:person2] = YES
set count = 2
```

说明isEqual相同的两个对象都被加入到了同一个NSSet中. 所以直接返回[super hash]是不正确的.

> In reality, a simple XOR over the hash values of critical properties is sufficient 99% of the time(对关键属性的hash值进行位或运算作为hash值).

对于上面Person类的hash方法实现如下:

``` OC
- (NSUInteger)hash {
    return [self.name hash] ^ [self.birthday hash];
}
```

# 3 总结
* 对于基本类型, ==运算符比较的是值; 对于对象类型, ==运算符比较的是对象的地址(即是否为同一对象). isEqual只能用来比较对象类型.
* ==运算符只是简单地判断是否是同一个对象, 而isEqual方法可以判断对象是否相同.
* 由于字符串驻留技术的存在, 静态定义的不可变字符串对象, ==的结果是YES.
* 如果两个对象相等, 它们的 hash 值也一定是相等的. 反过来则不然. hash值是对象判等的必要非充分条件.
* 

----
参考资料:
1.[iOS开发 之 不要告诉我你真的懂isEqual与hash!](https://www.jianshu.com/p/915356e280fc)
2.[iOS - isEqual & hash](https://juejin.im/entry/587e0d5e61ff4b00650df3d1)
3.[](https://nshipster.cn/equality/)


