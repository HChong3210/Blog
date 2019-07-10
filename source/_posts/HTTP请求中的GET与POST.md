---
title: HTTP请求中的GET与POST
date: 2017-12-11 23:27:30
tags:
    - 基础知识
categories: 
    - 基础知识
---

# HTTP请求中的GET与POST

GET和POST是网络请求到的两种基本方式, GET把参数包含在URL中, POST通过request body传递参数. 

GET在浏览器回退时是无害, 而POST会再次提交请求. 
GET产生的URL地址可以保存为书签, 而POST不可以.
GET请求会被浏览器主动cache, 而POST不会, 除非手动设置.
GET请求只能进行url编码, 而POST支持多种编码方式. 
GET请求参数会被完整保留在浏览器历史记录里, 而POST中的参数不会被保留.
GET请求在URL中传送的参数是有长度限制的, 而POST没有.
对参数的数据类型, GET只接受ASCII字符, 而POST没有限制.
GET比POST更不安全, 因为参数直接暴露在URL上, 所以不能用来传递敏感信息. 
GET参数通过URL传递, POST放在Request body中.

GET和POST都是HTTP协议中的两种发送请求的方法.HTTP是基于TCP/IP的关于数据如何在万维网中如何通信的协议.

HTTP的底层是TCP/IP, 所以GET和POST的底层也是TCP/IP. 也就是说, GET/POST都是TCP链接. GET和POST能做的事情是一样一样的. 你要给GET加上request body, 给POST带上url参数, 技术上是完全行的通的.

GET产生一个TCP数据包. POST产生两个TCP数据包.对于GET方式的请求, 浏览器会把http header和data一并发送出去, 服务器响应200(返回数据). 而对于POST, 浏览器先发送header, 服务器响应100 continue, 浏览器再发送data, 服务器响应200 ok(返回数据).


