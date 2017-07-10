---
title: hexo常见问题
date: 2016-12-21 21:05:16
tags:
    - hexo
    - 个人博客

categories:
    - hexo
---


# **hexo常见问题**
在使用hexo的过程中遇到了一些问题, 在这里列出来, 做一个记录.
## **hexo的常见发布流程**
* hexo新建一篇文章使用`hexo new type name`, `type`有三种, 最常使用的是`post`, `name`是新建文档的名字.
* hexo新建完成, 编辑之后发布会经常使用到下面几个命令:

    * `hexo clean`清除之前缓存的一些信息, 例如主题之类的, 不是每次都必须执行.
    * `hexo g`相当于编译.
    * `hexo s`发布到本地服务器, 把*.md*文件生成讲台html用于展示, 也可做调试用.
    * `hexo d`推本地的文件到服务器, 这里指的是github上面, 如果绑定的有域名, 就直接发布到Internet. 每次推送前, 要确保`hexo g`和`hexo s`没有问题, 否则可造成Internet上面无法正常显示.

## **hexo目录结构分析**
### **根目录**
* _config.yml: 位于本地博客的根目录下, 在这里面对整个博客的内容进行一些设置.
* source文件夹: 里面存储一些博客使用的文件资源, 例如*category(分类)*, *tag(标签)*, *link(链接)*, *about(关于)*, *project(工程)*, *search(搜索)*, *_posts(使用post格式新建的文章.md文件存储在这里).需要说明一下的是, 这些文件夹的名称和数量不固定, 要看你使用的主题里面的模块大概有几个 ,我使用的是[fexo](http://forsigner.com/2016/03/10/fexo-doc-zh-cn/).还有一些坑, 后面再详述.
* public文件夹: 里面存储的是之前发布过得一些归档数据, 如果要删除之前的测试数据的话, 记得清理里面响应的内容.
* scaffolds文件夹: 存储.md文档的类型.
* themes文件夹: 里面是你下载的主题内容, 如果有多个主题, 就会有多个文件夹, 但只能同时使用一种样式的主题.这个后面会着重分析一下.


### **themes文件夹**
这里面主要会进行一些主题相关的设置.

* _config.yml: 位于主题目录下, 在这里面对当前只用主题的内容进行一些配置, 不同主题的配置可能不太一样, 我是用的是[fexo](http://forsigner.com/2016/03/10/fexo-doc-zh-cn/)


* source文件夹: 该文件夹下面是该主题相关的一些资源, 例如一些静态的图片之类的.
* layout文件夹: 该文件夹下面是静态页面显示的相关配置. 代码高亮的设置也是在该文件夹下面. 其他的例如静态页面的展示, 可以修改相关的js文件.

## **修改代码高亮**
代码高亮的展示, 不同的主题有不同的使用方式, 但是代码高亮的theme可以参考这里, 我使用的是[HighLight](https://highlightjs.org/static/demo/), 它提供了多种Theme, 基本上能满足各种需求.

修改步骤如下:

1.修改博客根目录下的*_config.yml*文件, 关闭hexo自带的代码高亮.

```js
highlight:
  enable: false
  line_number: false
  auto_detect: false
  tab_replace:
```

2.`cd 博客根目录/themes/fexo/layout/_partial`打开*head.ejs*文件, 最好是在`<head></head>`之间开头处插入代码

```html5
  <link rel="stylesheet" href="//cdn.bootcss.com/highlight.js/9.2.0/styles/rainbow.min.css">
  <script src="//cdn.bootcss.com/highlight.js/9.2.0/highlight.min.js"></script>
  <script>hljs.initHighlightingOnLoad();</script>
```

也可以使用下面的写法:

```h5
<link rel="stylesheet" href="/path/to/styles/default.css">
<script src="/path/to/highlight.pack.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
```

二者的区别在于, 第一种写法使用的是CDN创建的[在线文档地址](http://www.bootcdn.cn/?), 该地址还保存了其他一些常见的文档, 非常强大.而第二种写法则是把文件下载到本地, 从本地读取代码高亮的配置.

---
修改过程中, 我参考了以下两篇博文, 还趟过不少坑, 贴上博文的地址:

1. [地址一](http://www.ieclipse.cn/en/2016/07/18/Web/Hexo-dev-highlight/)
2. [地址二](http://jumpbyte.cn/2016/07/02/use-and-install-prettify/)









