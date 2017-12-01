---
title: JavaScript第一弹
date: 2017-08-18 14:37:04
tags:
	- JavaScript
	- 基础知识
categories:
	- JavaScript
---

# JavaScript第一弹

主要是一些JavaScript的一些基础知识, 和OC的一些区别

## 入门

JavaScript可以嵌在网页的任意地方, 不过通常有两种写法:

```
<html>
<head>
  <script>
		JavaScript代码
  </script>
</head>
<body>
  ...
</body>
</html>
```

或者把JavaScript代码放到一个单独的`.js`文件中, 然后在HTML中引用

```
<html>
<head>
  <script src=".js文件的路径"></script>
</head>
<body>
  ...
</body>
</html>
```


把JavaScript代码放入一个单独的`.js`文件中更利于维护代码，并且多个页面可以各自引用同一份`.js`文件。

可以在同一个页面中引入多个`.js`文件，还可以在页面中多次编写`<script> js代码... </script>`，浏览器按照顺序依次执行。

## 数据类型

JavaScript 拥有动态类型。这意味着相同的变量可用作不同的类型, 这一点和OC是有区别的.

### 字符串

字符串是以`单引号'`或`双引号"`括起来的任意文本, 这和OC的略有不同. 和OC的字符串初始化不同的是, JS只需用引号括起来就代表一个字符串了.

### 比较运算符

JavaScript允许对任意数据类型做比较. 要特别注意相等运算符`==`。JavaScript在设计时，有两种比较运算符：

* 第一种是`==`比较，它会自动转换数据类型再比较，很多时候，会得到非常诡异的结果；
* 第二种是`===`比较，它不会自动转换数据类型，如果数据类型不一致，返回false，如果一致，再比较。

由于JavaScript这个设计缺陷，不要使用==比较，始终坚持使用`===`比较。

这个和`OC`类似, 如果使用`===`比较的是两个基本数据类型, 只要值相等就位`true`. 但是如果是数组, 字典, 对象等其他类型, 那比较的就是内存地址, 而不是元素.

```
var arr1 = [1,2,3];
var arr2 = [1,2,3];
alert(arr1 === arr2)//false

var str1 = '123';
var str2 = '123';
alert(str1 === str2);//true
```

关于数组元素的比较, 如果是OC的话, 可以使用`-(BOOL)isEquleToArray:(NSArray *)array;`来比较数组中的元素, 然而JS没有这种方法, 常见的做法是先排序, 再转成String, 再比较String是否相等. 参考[equalArray.js](https://gist.github.com/smallnewer/6535788)

`NaN`这个值通过`===`来比较大小永远为`false`, 包括与自身比较. 唯一的判断方法就是通过`isNaN()`函数.

### null和undefined

`null`表示一个“空”的值类似于OC的`nil`，它和0以及空字符串''不同，0是一个数值，''表示长度为0的字符串，而null表示“空”.

还有一个和null类似的undefined，它表示“未定义”. JavaScript的设计者希望用null表示一个空的值，而undefined表示值未定义.

* 大多数情况下，我们都应该用null.
* undefined仅仅在判断函数参数是否传递的情况下有用, 未使用值来声明的变量，其值实际上是 undefined.


## 变量

变量和OC的变量是一样的, 但是命名规则有所不同. 变量名是大小写, 数字, `$`, 和`_`组合, 且不能用数字开头. 

和OC一样, 使用`=`对变量进行赋值, 由于JS是一门动态语言, 在初始化变量时可以不指定变量类型, 这与OC略有不同.

仅仅声明但是没有赋值的变量, 其值实际上是`undefined`.
```
var carname;
console.log(carname);
```

----
1.[Underscore - 一个JavaScript 工具库](http://www.bootcss.com/p/underscore/).
2.[JavaScript 新手的踩坑日记](https://halfrost.com/lost_in_javascript/).

