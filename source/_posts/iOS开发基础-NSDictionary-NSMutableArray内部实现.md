---
title: iOS开发基础-NSDictionary & NSMutableArray内部实现
date: 2019-07-25 14:52:53
tags:
    - 基础知识
categories:
    - iOS开发-基础
---

dictionary(map)和array是常见的数据结构, 下面我们来看下iOS中NSDictionary和NSMutableArray的内部实现. [这里](https://blog.csdn.net/Deft_MKJing/article/details/82732833)讲解的十分详细, 此处我们对主要流程做一个概述.

# 1 NSDictionary

Foundation框架下提供了很多高级数据结构, 这些都是对Core Foundation下的封装, 例如NSDictionary就是对_CFDictionary的封装.

``` C
struct __CFDictionary {
    CFRuntimeBase _base;
    CFIndex _count;
    CFIndex _capacity;
    CFIndex _bucketsNum;
    uintptr_t _marker;
    void *_context;
    CFIndex _deletes;
    CFOptionFlags _xflags;
    const void **_keys;	
    const void **_values;
};
```
根据数据结构可以发现dictionary内部使用了两个指针数组分别来保存keys和values, dictionary采用的是连续存储的方式存储键值对. Apple给的查询复杂度可以快至O(1), 下面我们就来分析下是如何做到的. 

# 1.1 原理分析
创建N个空桶，N为排序数组中最大值加一。然后遍历排序数组，以元素值为下标，将其放入对应的桶中. 如果少量数据，可以根据数组里面的最大值+1创建出那么多空桶，然后遍历，根据索引在空桶的值上累加，最后遍历空桶（已装载），根据值遍历出对应的下标.

虽然这样做效率非常高，但是如果数据过大，内存吃不消，这样就有了哈希排序的介绍。例如一组数据，我们可以根据hash算法后取模的值进行空桶排列，但是如果两个值例如 101和11 % 10都是余1，会被放入同一个桶里面，这样就会有需要二次排列。虽然这个排序效率并不高，因此哈希化就变成了数据存储的一种设计.

从dictionary的结构中可以看到keys大概率是一个数组, 那么当对象完成hash化运算, 这个计算结果要如何和数组实现位置匹配. 由于是两个数组分别存储, 因此, key哈希出来的数组下标地址, 同样这个地址对应到values数组的下标, 就是匹配到的值. 但是hash过程中必定会出现冲突，如何来处理冲突？

## 1.2 hash碰撞
解决hash碰撞的方式是将发生碰撞的多个元素放到一个容器中，这个容器通常使用链表结构，这种解决方案被称作拉链法. 这个方案能够解决hash碰撞的匹配问题。但拉链法会将key和value包装成一个结构存储，而dictionary的结构拥有keys和values这两个数组，说明了这两个数据是被分开存储的，所以使用这个方案的可能性不高。而且拉链法存在一个问题：桶数量不多的情况下，拉链衍生出来的链表会非常庞大，需要二次遍历，匹配损耗一样很大, 官方都说了查找算法接近O(1),因此肯定不是拉链法，下面就有了开放定址法.

使用开放定址法的结构通常允许在通列表的数量达到了某个阈值，通常是通列表长度的80%使用量时，对通列表进行一次扩充grow，然后重新计算数据的keyHash放入新桶中. 开放定址法可以通过动态扩充通列表长度解决了满桶无法插入的问题，也符合O(1)的查询速度，但同样随着数据量的增加，数据会明显的集中在某一段连续区域，称作堆积现象

除了上述提到的拉链和开放定址，还有再哈希以及建立公共溢出区域来解决冲突.

# 1.2 总结
NSDictionary是使用hash表来实现key和value的映射和存储. 可以看到，NSDictionary设置的key和value，key值会根据特定的hash函数算出建立的空桶数组，keys和values同样多，然后存储数据的时候，根据hash函数算出来的值，找到对应的index下标，如果下标已有数据，开放定址法后移动插入，如果空桶数组到达数据阀值，这个时候就会把空桶数组扩容，然后重新哈希插入。这样把一些不连续的key-value值插入到了能建立起关系的hash表中，当我们查找的时候，key根据哈希值算出来，然后根据索引，直接index访问hash表keys和hash表values，这样查询速度就可以和连续线性存储的数据一样接近O(1)了，只是占用空间有点大，性能就很强悍。如果删除的时候，也会根据_maker标记逻辑上的删除，除非NSDictionary（NSDictionary本体的hash值就是count）内存被移除。我们也会根据dictionary之所以采用这种设计，其一出于查询性能的考虑；其二dictionary在使用过程中总是会很快的被释放，不会长期占用内存.

# 2 NSMutableArray
[这里](http://blog.joyingx.me/2015/05/03/NSMutableArray%20%E5%8E%9F%E7%90%86%E6%8F%AD%E9%9C%B2/)讲解的十分详细.

普通c数组，归根接地就是一段能被方便读写的连续内存控件. 使用一段线性内存空间的一个最明显的缺点是，在下标 0 处插入一个元素时，需要移动其它所有的元素. 同样地，假如想要保持相同的内存指针作为首个元素的地址，移除第一个元素需要进行相同的动作. 

当数组非常大时，这样很快会成为问题。显而易见，直接指针存取在数组的世界里必定不是最高级的抽象。C 风格的数组通常很有用，但 Obj-C 程序员每天的主要工作使得它们需要 NSMutableArray 这样一个可变的、可索引的容器

## 2.1 ivars
我们来概括下每个 ivar 的意思：

* _used 是计数的意思
* _list 是缓冲区指针
* _size 是缓冲区的大小
* _offset 是在缓冲区里的数组的第一个元素索引

## 2.2 内存布局
最关键的部分是决定 realOffset 应该等于 fetchOffset（减去 0）还是 fetchOffset 减 _size。看着纯代码不一定能画出完美的图画，我们设想一下两个关于如何获取对象的例子.

_size > fetchOffset这个例子中，偏移量相对较小:
![_size > fetchOffset](http://blog.joyingx.me/images/20150503/4.jpg)
为了获取 0 处的对象，我们计算出 fetchOffset 等于 3 + 0。因为 _size 大于 fetchOffset，realOffset 也等于 3。代码返回 _list[3] 的值。而获取 4 处的对象时，fetchOffset 等于 3 + 4，代码返回 _list[7]。

_size <= fetchOffset, 当偏移量比较大时:
![_size <= fetchOffset](http://blog.joyingx.me/images/20150503/5.jpg)
获取 0 处的对象，使得 fetchOffset 等于 7 + 0，调用方法后如期望的返回 _list[7]。然而，获取 4 处的对象时，fetchOffset 等于 7 + 4 = 11，要大于 _size。获得的 realOffset 要从 fetchOffset 减去 _size，即 11 - 10 = 1，方法返回 list[1].

## 2.3 总结
__NSArrayM 用了环形缓冲区 (circular buffer)。这个数据结构相当简单，只是比常规数组或缓冲区复杂点。环形缓冲区的内容能在到达任意一端时绕向另一端。

环形缓冲区有一些非常酷的属性。尤其是，除非缓冲区满了(每当缓冲区满了，它会重新分配1.625倍大小的空间.)，否则在任意一端插入或删除均不会要求移动任何内存。我们来分析这个类如何充分利用环形缓冲区来使得自身比 C 数组强大得多。在任意一端插入或者删除，只是修改offset参数，不需要移动内存，我们访问的时候只是不和普通的数组一样index多少就是多少，这里会计算加上offset之后处理的值取数据，而不是插入头和尾巴的时候，环形结构会根据最少移动内存指针的方式插入，例如要在A和B之间插入，按照C的数组，我们需要把B到E的元素移动内存，但是环形缓冲区的设计，我们只要把A的值向前移动一个单位内存，即可，同时修改offset偏移量，就能保证最小的移动单元来完成中间插入.

-----
参考资料:
1.[NSMutableArray原理揭露](http://blog.joyingx.me/2015/05/03/NSMutableArray%20%E5%8E%9F%E7%90%86%E6%8F%AD%E9%9C%B2/)
2.[NSDictionary和NSMutableArray底层原理（哈希表和环形缓冲区）](https://blog.csdn.net/Deft_MKJing/article/details/82732833)
3.[如何设计并实现一个线程安全的 Map ？(上篇)](https://juejin.im/post/59bce186f265da065a63ad8d)
4.[如何设计并实现一个线程安全的 Map ？(下篇)](https://juejin.im/post/59d8d7cc6fb9a00a496e93b2)
5.[]()

