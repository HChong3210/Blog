---
title: Charles抓包Https请求
date: 2017-02-28 14:18:40
tags:
- 逆向
- 安全
- Charles
categories:
- iOS逆向
---
# Charles抓包HTTPS请求
有时我们会对手机里面的App进行一些分析, 产看App的请求包, 这时我们就需要祭出神器[Charles](https://www.charlesproxy.com/)了.
下面跟着我来教你从零开始抓取App的请求包.
## 安装Charles
这个纯属占坑(废话).
## 抓取HTTP请求
我们先拿一个简单的HTTP请求来练一下手.
### 查看本地IP地址和Charles端口号
查看IP地址的方法比较多, 这里只介绍两种比较常用的:
1. 可以在网络偏好设置->高级->TCP/IP下查看对应网络的IP.
    ![网络偏好设置中查看](https://ww2.sinaimg.cn/large/006tKfTcgy1fd5z8oku1vj311a0lwgpb.jpg)
2. option+左键单击屏幕上方的WiFi标志也可以查看.
    ![option+左键查看](https://ww3.sinaimg.cn/large/006tKfTcgy1fd65kx0mjhj308c0eft9q.jpg)

### 查看Charles的端口号
打开Charles->Proxy->Proxy Settings查看端口号, 默认一般是`8888`. 注意不要随意修改端口号, 以避免被占用.
### 在iPhone中设置
打开手机, 连上WiFi, 注意此处的WiFi最好和Mac连接的WiFi是同一个, 避免出现不在一个网段的情况.
打开设置->无线局域网->点击对应WiFi后的更多.
![WiFi列表](https://ww4.sinaimg.cn/large/006tKfTcgy1fd65mobyrjj30b40jrq3k.jpg)
点击手动, 设置HTTP代理地址(与Mac的IP地址一致)和端口号(与Charles中代理的端口号一致)
![设置HTTP代理地址和端口号](https://ww3.sinaimg.cn/large/006tKfTcgy1fd65mo3y0aj30b40jrmxm.jpg)

至此, 就可以轻松的抓取App中的HTTP请求啦.
## 抓取HTTPS请求
HTTPS的请求抓取, 略微复杂, 他的原理大致是这样的.
> Charles能进行https协议抓包分析，是使用了中间人代理的方法（man-in-the-middle，也常作为一个黑客攻击手段）。Charles代替你的app接受server的证书，然后使用这个证书通过SSL和server通信；同时，Charles会动态的生成一个对应的证书（用Charles的CA证书签名），然后使用这个证书和你的app通信，这样就完成了一个中间人代理，从而可以把app和server的https包给抓到和解码出来。

从里面我们可以看到一些关键字: `证书`, `代理`, 等. 那么我们就通过这些手段来抓取HTTPS的请求.
### 设置代理地址和端口号
这一步骤, 和抓取HTTP的方法一样, 这里不再赘述, 需要设置端口号和IP地址.
### Mac安装证书
安装Charles的证书到Mac, 信任该证书, 以便Charles能够通过该证书进行通信. 具体步骤如下: 
1. 打开Charles, 点击Help->SSL Proxying->Install Charles Root Certificate
    ![安装根证书](https://ww1.sinaimg.cn/large/006tKfTcgy1fd607vw0k6j30pk094gmj.jpg)
2. 信任该证书
![证书不被信任时](http://upload-images.jianshu.io/upload_images/1552510-461959c476a2d9d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![信任Charles证书](http://upload-images.jianshu.io/upload_images/1552510-8c43e95be0a35b5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### iPhone安装证书
安装Charles证书到手机, 信任该证书, 以便Charles能够截获网络请求. 具体步骤如下:
1. 在Charles中, 点击Help->SSL Proxying->Install Charles Root Certificate on a Mobile Device or Remote Browser, 来查看当前IP地址下, 手机下载证书的网址.
    ![步骤一:选择生成手机下载证书的网址](https://ww2.sinaimg.cn/large/006tKfTcgy1fd60jpjcgvj30nx094my0.jpg)
2. 在手机中用Safari打开网址(如图所示的网址)
    ![步骤二:输入生成的网址](https://ww4.sinaimg.cn/large/006tKfTcgy1fd60l2cz81j30nz06t3yl.jpg)
3. 打开网址后自动跳转到证书安装界面, 安装并信任该证书.
    ![步骤三:安装并信任该证书](https://ww1.sinaimg.cn/large/006tKfTcgy1fd65rnpokjj30b40jrgly.jpg)
4. 打开Charles选择Proxy->SSL Proxying Settings->SSL Proxying(因为Charles默认是不会抓取任何域名下的HTTPS, 所以需要我们添加域名到Location下)勾选Enable SSL Proxying, 单击Add, 添加域名和端口(用*来表示所有).
    ![步骤四:设置抓取的域名和端口](https://ww1.sinaimg.cn/large/006tKfTcgy1fd60zndgejj30gj0cb0st.jpg)

至此, 我们再打开手机App就会发现, 之前`Unknown`的HTTPS接口, 就都可以正常现实啦.

参考资料:
1.[HTTP/HTTPS抓包工具Charles](http://www.jianshu.com/p/a0215dd2047f)
2.[使用Charles进行https抓包](http://www.jianshu.com/p/a83b19a36a8b)
3.[Charles 从入门到精通](http://blog.devtang.com/2015/11/14/charles-introduction/)


