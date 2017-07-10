---
title: iOS逆向-下载历史版本
date: 2017-04-06 16:27:49
tags:
    - iOS逆向
    - Charles
categories:
    - iOS逆向
---

# iOS逆向-下载历史版本
目前App Store默认只能下载最新版, 而我们的目的是要能够下载到App Store中的历史版, 那就要借助一些其他工具来实现. 
原理如下:
> 通过Charles来获取下载链接, 通过分析来得到每个版本在连接中所对应的字段, 再通过Charles来截取和替换下载链接中的参数, 来达到下载特定版本App的目的.
## 准备工作
1. 安装Charles.
2. 为了Charles能够抓取HTTPS类型的链接, 需要安装Charles的证书, 参考这里[Charles抓包HTTPS请求](http://hchong.net/2017/02/28/Charles%E6%8A%93%E5%8C%85Https%E8%AF%B7%E6%B1%82/).

## 开始抓包
安装了Charles之后, 我们就可以获取App的下载链接了, 并且从中分析出我们需要的参数.
### 获取APP的下载链接和版本号对应的参数
下面我们以iOS上一款比较好用的看书软件*追书神器*为例来说明.
1. 首先打开ITunes来下载软件, 点击下载, 通过Charles来确定下载软件的链接.
![点击下载](https://ww3.sinaimg.cn/large/006tKfTcgy1fel4m4udoqj30zo0iwdhx.jpg)
2. 在Charles中观察发现, ITunes使用的是HTTPS链接, 无法直接查看request内容.
![无法直接查看HTTPS](https://ww2.sinaimg.cn/large/006tKfTcgy1fel4n5nyodj30f707e74h.jpg)
3. 我们通过添加SSL Proxying来查看request内容, 并且通过给链接加断点来详细分析请求的参数.

![添加SSL Proxy](https://ww3.sinaimg.cn/large/006tKfTcgy1fel4nt57ykj30b60jr74v.jpg)
![添加断点](https://ww1.sinaimg.cn/large/006tKfTcgy1fel4ogb7edj30bg0lkab0.jpg)
### 推断控制版本的关键字段
观察response, 推断`softwareVersionExternalIdentifiers`展示的是全部的APP对应的版本, `softwareVersionExternalIdentifier`是用来标记当前版本.*追书神器*目前的版本标记是`820420814`
![字段分析](https://ww2.sinaimg.cn/large/006tKfTcgy1fel4pltv74j310w0qjq5r.jpg)
通过在request中搜索`820420814`发现,`appExtVrsId`是来标记要下载哪个版本. 
![推断关键字](https://ww2.sinaimg.cn/large/006tKfTcgy1fel4p2c1jqj30be08wwew.jpg)
### 修改Request参数下载历史版本
我们在`softwareVersionExternalIdentifiers`中任选一个版本号, 此处我以`819147298`为例
1. 删除刚才下载好的软件, 重新下载.
2. 因为我们在下载链接中加的有断点, 再次下载时Charles会停在断点的位置
3. 点击`Edit Request`来编辑`appExtVrsId`字段
![Edit Request](https://ww1.sinaimg.cn/large/006tKfTcgy1fel4qg1avyj30yf0o60v0.jpg)
4. 执行断点, 继续下载.
5. 在ITunes中看到更新的标志说明下载成功
![下载成功](https://ww1.sinaimg.cn/large/006tNbRwgy1fel4komzx2j30p00c8t9i.jpg)

## 软件安装
通过ITunes来安装下载好的软件到手机.

-------
参考文献:
1.[Charles抓包HTTPS请求](http://hchong.net/2017/02/28/Charles%E6%8A%93%E5%8C%85Https%E8%AF%B7%E6%B1%82/).
2.[iOS秘籍-下载历史版本App超详细教程](http://www.jianshu.com/p/edfed1b1822c).



