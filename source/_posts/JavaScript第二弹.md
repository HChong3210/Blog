---
title: JavaScript第二弹
date: 2017-08-22 15:13:34
tags:
  - JavaScript
  - 基础知识
categories:
  - JavaScript
---

# JavaScript第二弹

第一弹主要是一些基本语法, 数据类型和变量. 这一弹来说一下字符串, 数组和对象. 

> 在 `JavaScript`中，对象的定义是拥有属性和方法的数据.
> 严格来讲, `JavaScript`中的所有事物都是对象：字符串, 数字, 数组, 日期，等等.

## 字符串

和OC的没什么太大的区别, 这里只说几点比较特殊的.

* 可以使用反引号\` `...` \`来标记一串带符号的文本, 例如

```
alert(`你好, 
哈哈`);
```
中间的所有符号都会被保留下来, 

* 还可以在在\` `你好 + ${string}` \`来拼接字符串, 其中`${String}`代表一个叫做`String`的变量


## 数组

区别于OC的数组, `JavaScript`的`Array`可以包含任意的数据类型, 通过索引来访问每个元素.

Array属性:

> constructor 返回对创建此对象的数组函数的引用.
> length 设置或返回数组中元素的数目.
> prototype 使您有能力向对象添加属性和方法.

Array 对象方法:

> indexOf() 搜索一个指定的元素的位置.
> concat() 连接两个或更多的数组，并返回结果.
> join() 把数组的所有元素放入一个字符串.元素通过指定的分隔符进行分隔.
> pop() 删除并返回数组的最后一个元素
> push() 向数组的末尾添加一个或更多元素，并返回新的长度.
> shift() 删除并返回数组的第一个元素.
> unshift() 向数组的开头添加一个或更多元素，并返回新的长度.
> reverse() 颠倒数组中元素的顺序.
> slice() 从某个已有的数组返回选定的元素.
> sort() 对数组的元素进行排序.
> splice(loc, len, newElement, ...) 删除元素，并向数组添加新元素.
> toSource() 返回该对象的源代码.
> toString() 把数组转换为字符串，并返回结果.
> toLocaleString() 把数组转换为本地数组，并返回结果.
> valueOf() 返回数组对象的原始值.

在这里需要注意, 直接使用`arr.length`是会改变原`arr`的长度的, 没有被赋值的元素就会变为`undefined`, 直接通过索引赋值, 也会出现这个问题. 例如:

```
var arr1 = [1, 2, 3];
arr1.length; // 3
arr1.length = 6;
arr1; // arr1变为[1, 2, 3, undefined, undefined, undefined]
arr1.length = 2;
arr1; // arr1变为[1, 2]

var arr2 = [1, 2, 3];
arr2[5] = 'x';
arr2; // arr2变为[1, 2, 3, undefined, undefined, 'x']
```

大多数其他编程语言不允许直接改变数组的大小，越界访问索引会报错。然而，JavaScript的Array却不会有任何错误。在编写代码时，不建议直接修改Array的大小，访问索引时要确保索引不会越界.


