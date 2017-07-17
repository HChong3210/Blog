---
title: 使用Jenkins实现持续集成
date: 2017-03-23 21:23:12
tags:
    - 自动化打包
    - Jenkins
    - 持续集成
categories:
    - 自动化打包
    - 持续集成
---

# 使用Jenkins实现持续集成

什么是持续集成, 持续集成是一种软件开发实践，即团队开发成员经常集成它们的工作，通过每个成员每天至少集成一次，也就意味着每天可能会发生多次集成。每次集成都通过自动化的构建（包括编译，发布，自动化测试）来验证，从而尽早地发现集成错误。

常见的持续集成的工具有[Jenkins](http://jenkins-ci.org/)  [Travis](https://travis-ci.com/)  [Hudson](http://hudson-ci.org/)  [Circle](https://circleci.com/). 然而好多我并没有实践过, 😂. [这里](http://www.infoq.com/cn/articles/ios-code-server-jenkins-travis-fastlane)有一篇文章, 对比讲解了主流iOS持续化集成方案, 包括了Xcode Server.

Code review, 单元测试, 打包与分发, 这些构成了一个APP开发生命周期. 所以一个完整持续化集成应该包含以上的而全部信息, 这里我们主要讲解一下打包与分发的持续化集成方案. 关于我们为什么选择Jenkins, 而不使用其他的持续化方案, [这篇文章](http://www.infoq.com/cn/articles/ios-code-server-jenkins-travis-fastlane)已经讲解的很清楚了. 下面就跟随我一步步的搭建Jenkins, 实现一个简单但强大的持续化集成方案. 本文均以Jenkins2.35为例.

## 安装Jenkins

首先, 我们要安装Jenkins. 安装Jenkins一般有两种方式. 

第一种我们可以从[官网](https://jenkins.io/)上下载最新的pkg安装包。

![安装步骤1](https://ws2.sinaimg.cn/large/006tKfTcgy1fh5vhnmh9bj30h80c7dgr.jpg)

一路点击继续, 直到安装成功为止.

![安装完成](https://ws2.sinaimg.cn/large/006tKfTcgy1fh5vikplwxj30hb0c5q35.jpg)

第二种方式也可以下载`brew install jenkins`, 切换到切换到 `cd /usr/local/Cellar/jenkins/版本号/libexec/jenkins.war`,  然后运行Java -jar jenkins.war，进行安装.
注意: 如果brew无效要先安装homebrew, `ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
`

安装完成后, 重启电脑, 你会发现多了一个Jenkins的用户, 但是他的登录密码我们并不知道. 我们应该打开[http://localhost:8080](http://localhost:8080), 此时可以看到Jenkins的初始界面. 

![Jenkins初始界面](https://ws2.sinaimg.cn/large/006tKfTcgy1fh5vod9epvj30yg0jet93.jpg)
注意, 由于Jenkins是由Java开发的, 所以Jenkins的运行必须是在Java环境下, 如果你打开界面为空白的话, 那就说明你需要安装Java环境了.

按照提示，找到/Users/Shared/Jenkins/Home/ 这个目录, 由于非Jenkins用户没有查看权限, 所以需要我们右键 -> 显示简介, 给这个文件夹赋予读写权限. 按照提示找到*initialAdminPassword*文件, 复制出密码，就可以填到网页上去重置密码了.

![重置密码](https://ws3.sinaimg.cn/large/006tKfTcgy1fh5vsrxh73j30yg0je0t5.jpg)

![开始安装](https://ws4.sinaimg.cn/large/006tKfTcgy1fh5vt9bk89j30yg0jemxk.jpg)

然后一路安装下去, 输入用户名，密码，邮件这些，就算安装完成了。

## Jenkins插件配置&系统设置

打开[http://localhost:8080](http://localhost:8080)页面, 我们需要安装一些辅助插件, 选择*系统管理* -> *管理插件*. 下面我列一下我安装到的一些常用的插件: 

* GitLab Plugin, Gitlab Hook Plugin. 用于连接到GitLab, 如果你的源码在Gitlab上托管, 这连个插件必须要安装.

* Xcode integration. 这个看名字就知道必须安装了.

* Environment Injector Plugin. 自定义全局变量的插件, 必须安装.

* Keychains and Provisioning Profiles Management. 用于管理钥匙串和描述文件, 必须安装.

* build timeout plugin. 如果构建超时, 自动停止Jenkins当前的构建, 可选.

* Email Extension Plugin. 用于发邮件, 可选.
## Jenkins打包配置准备工作

首先, 我们需要明确在这一步我们的目的: 把指定代码的指定分支打包为签名过的IPA包, 并且生成一个可供下载的链接. 那么明确了我们的目的, 接下来就一步一步的来实现它. 打开[http://localhost:8080](http://localhost:8080)页面, 接下来我们开始新建一个打包项目. 点击左侧工具栏*新建*, 选择*构建一个自由风格的软件项目*, 注意这里最好使用英文, 不要出现中文和特殊字符.

进入项目配置页面, 接下来我将从*General*, *源码管理*, *构建触发器*, *构建环境*, *构建*, 和*构建后操作*几个方面来配置打包项目.

在项目配置之前, 我们需要先配置一些其他全局的变量, 这些变量在项目的配置中会使用到.
### gitlab代码地址配置
Jenkins系统管理 -> 系统设置 -> Gitlab选项, 因为我们的代码托管在gitlab上面, 所以在这里需要配置相关信息. 点击*增加*. 填写相关信息.
![gitlab地址配置](https://ws2.sinaimg.cn/large/006tNc79gy1fh93xkbelyj31ei0qk75f.jpg)

注: 这里会报一个*API Token for Gitlab access required*的错误, 可以暂时忽略, 是因为gitlab一般都是私有的仓库, Jenkins没有连接的权限导致的.

### 添加与gitlab连接的证书
Jenkins -> Credentials -> Add Credentials, [http://localhost:8080/credentials/store/system/domain/_/newCredentials](http://localhost:8080/credentials/store/system/domain/_/newCredentials)增加一个新的证书, 用来连接gitlab. 

![连接gitlab的证书](https://ws2.sinaimg.cn/large/006tNc79gy1fh94ryjjgvj31kw0qngml.jpg)

*kind* , *Scope* 和 *Private Key* 就按照如图所示选择好. *UserName* 表示这个证书的名字, *ID* 自动生成, 用来标记这个证书. *Description*是关于这个证书的描述. *key* 那里填写SSH的Private Key, *Passphrase* 那里填写生成SSH时填写的密码.  [关于SSH的申请](https://git-scm.com/book/zh/v1/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E7%94%9F%E6%88%90-SSH-%E5%85%AC%E9%92%A5).

注意: 这里记得要在你的gitlab的Profile Settings -> SSH Keys中添加上一步生成的Public Key, 

### 添加钥匙串和描述文件
在Jenkins -> 系统管理 -> KeyChain and Provisioning Profiles Management 中添加 keychains 和 Provisioning Profile文件. 
![添加钥匙串和描述文件](https://ws2.sinaimg.cn/large/006tNc79gy1fha7tvmt2uj30yg08ljrq.jpg)

分别上传keychains 和 Provisioning Profile.

* keychains的路径在/Users/管理员用户名/Library/keychains/login.keychain,当把这个Keychain设置好了之后，把这个Keychain拷贝到/Users/Shared/Jenkins/Library/keychains这里，(Library是隐藏文件)。

* Provisioning Profiles文件在~/Library/MobileDevice/Provisioning Profiles 目录下, 这里面可能会有许多描述文件, 找到我们需要的, 上传并且拷贝到/Users/Shared/Jenkins/Library/MobileDevice文件目录下.

注: 如果 Jenkins 下保存 keychain 和 Provision profile 的目录不存在可以手动创建. 我们可以邮件查看要打包工程*.xcodeproj的包内容, 查看project.pbxproj文件, 通过搜索找到描述文件的ID, 再在~/Library/MobileDevice/Provisioning Profiles文件夹中根据ID来找到具体的描述文件.

### Xcode Builder
在Jenkins -> 系统设置 -> Xcode Builder -> 新增, 如下图所示.
![Xcode Builder](https://ws1.sinaimg.cn/large/006tNc79gy1fha9gi02fyj30nu0fz74v.jpg)

Keychain Name是要打包的证书的名字, 在钥匙串中找到该证书, 点击右键 显示简介, 复制过来就好.

Keychain password是开机密码.
## Jenkins打包配置

![Jenkins打包工程的配置](https://ws4.sinaimg.cn/large/006tNc79gy1fha9o70e7wj31kw0t4758.jpg)
### General
配置一些通用的选项. 项目名称和项目描述如其字面意思一样. `GitLab connection`是源码的地址, 在外面配置好的选项, 在这里可选.

* 节流构建，通过设置时间段内允许并发的次数来实现构建的控制

* 丢弃旧的构建, 按需选择.

* 参数化构建过程, 在Jenkins开始打包之前, 填写需要你传入的参数.

* 关闭构建, 在必要的时候并发构建. 可选

## 源码管理

根据自己的实际需求, 不管是SVN或者Git. 因为我使用的是Git, 在一开始安装了Git管理的插件, 所以这里选择Git.
![](https://ws4.sinaimg.cn/large/006tNc79gy1fhab3vad6rj30tm0f3q3d.jpg)

1. 这里的 *Credentials* 如果使用的是SSH模式, 那上面的 *Repository URL*就要使用SSH类型的地址. 如果是账号密码模式, 那上面的源码就要使用HTTP形式的地址. 两者必须保持一致.

2. 指定代码, 我们可以拉取指定git地址的代码. 因为分支是个变量, 它不像git地址那样一成不变, 所以最好是外部传入, 那就用到了全局环境变量. 

## 构建触发器

构建触发器是设置自动化构建的地方, 如果想设置自动化构建, 例如监测到git有更新就打包, 或者每隔固定时间就打一次包.

* Poll SCM (poll source code management) 轮询源码管理. 需要设置源码的路径才能起到轮询的效果。一般设置为类似结果： 0/5 每5分钟轮询一次

* Build periodically (定时build) 一般设置为类似： 00 20 * 每天 20点执行定时build 。当然两者的设置都是一样可以通用的.

![构建触发器](https://ws3.sinaimg.cn/large/006tNc79gy1fhfrpjn2twj31kw0cegma.jpg)

## 构建环境

构建环境, 主要是对构建时一些环境变量的配置. 
![构建环境](https://ws4.sinaimg.cn/large/006tKfTcgy1fhgvjboomwj31a60eujrv.jpg)

在该模块中 主要设置 xcode build 打包时需要的 keychains 和 Provision Profiles 配置文件。
如果不配置 就会使用 xcode 自动的配置，来去系统中查找相应的配置，不过有一点需要注意,就是钥匙串中，登陆钥匙串中的证书 要复制到 系统钥匙串中，因为jenkins 访问的是系统中的钥匙串 这样在第一次打包的时候，会提示 是否授权访问钥匙串，点击始终允许就可以了。

![Keychains and Code Signing Identities](https://ws1.sinaimg.cn/large/006tNc79gy1fhjnx3dnwvj31gg0jo755.jpg)
*Keychains and Code Signing Identities* 中的选项, 因为在前面已经选择过了 * Keychain * 和 *Code Signing Identity*在前面已经填写过, 此处只用根据当前工程来选择正确的选项.

![Mobile Provisioning Profiles](https://ws3.sinaimg.cn/large/006tNc79gy1fhjo37irdyj31g20hojs1.jpg)

*Mobile Provisioning Profiles*由于前面已经填写过描述文件, 此处只需要选择与当前打包工程相匹配的描述文件.

## 构建

在这里才进入了自动化集成的关键步骤, 前面的都是一些准备工作. 我们选择*增加构建步骤*, 按照自己的实际需求, 选择打包脚本的类型. 因为Jenkins自带Shell脚本集成, 所以此处我选择*Execute shell*, 使用shell脚本来进行构建.

```
export PATH
#######执行脚本命令#######
rm -rf Users/Shared/Jenkins/Library/ipa/JenkinsTest/JenkinsTest.ipa
security unlock-keychain  -p "HChong" ~/Library/Keychains/login.keychain
sh /Users/Shared/Jenkins/Library/BuildScript/build.sh JenkinsTest 4c86cf7c8b4d00013c59b30b0c8d5e77
fir p /Users/Shared/Jenkins/Library/ipa/JenkinsTest/JenkinsTest.ipa -T 4c86cf7c8b4d00013c59b30b0c8d5e77
```


shell脚本的具体内容如下
```
#!/bin/bash

#=================项目路径配置===============
PROJECT_PATH='/Users/Shared/Jenkins/Home/workspace/iOSPackage'
WORKSPACE_NAME='JenkinsTest.xcworkspace'
Archive_PATH='/Users/Shared/Jenkins/Library/archive'
IPA_PATH='/Users/Shared/Jenkins/Library/ipa'
PLIST_PATH='/Users/Shared/Jenkins/Home/workspace/iOSPackage/exportArchive.plist'


#===================================脚本开始=================================================
#使用帮助
if [ $# == 0 ];then
echo "===========================如何使用============================="
echo " eg: ./build [scheme] [token] '版本描述中间不要留空格', 不传token默认用当前已经登录的fir token"
echo " scheme list:"
echo " JenkinsTest"
echo " token list:"
echo "================================================================"
exit
fi

#update code from gitlab
cd $PROJECT_PATH
#git pull

#update pod
#pod install --repo-update
#pod update

#删除旧的编译目录
APP_BUILD_LOCATION=${PROJECT_PATH}/Build/
rm -rf ${APP_BUILD_LOCATION}
#创建dfc目录

#key auth
security unlock-keychain "-p" "钥匙串密码" "/Users/hc/Library/Keychains/login.keychain"

#创建ARCHIVE目录
mkdir -p IPA_PATH
#Archive_NAME = $1.xcarchive

#开始打包
cd ${PROJECT_PATH}
pwd

XCCONFIG_PATH=${PROJECT_PATH}/dfc_v2/appconfig

#xcodebuild -workspace ${WORKSPACE_NAME} -scheme Enterprise -xcconfig ${XCCONFIG_PATH}/$1.xcconfig -archivePath ${Archive_PATH}/$1.xcarchive archive
xcodebuild -workspace ${WORKSPACE_NAME} -scheme $1 -config $1 -archivePath ${Archive_PATH}/$1.xcarchive archive
echo "--------------------------------------------"${Archive_PATH}/$1.xcarchive

#创建ipa
IPA_LOCATION=${IPA_PATH}/$1

#删除旧的ipa
rm -rf IPA_LOCATION
mkdir -p ${IPA_PATH}/$1
xcodebuild -exportArchive -exportOptionsPlist ${PLIST_PATH} -archivePath ${Archive_PATH}/$1.xcarchive -exportPath ${IPA_LOCATION}


IPA_FILE_LOCATION=${IPA_PATH}/$1/$1.ipa

#检查ipa是否创建成功
if [ -f $IPA_FILE_LOCATION ]; then
echo "ipa已经创建:"${IPA_FILE_LOCATION}
else
echo "打包失败"
exit 0
fi

```
## 构建后的操作

顾名思义, 就是构建之后进行的操作. 在这里我们可以选择*Execute a set of scripts*, 通过脚本来实现构建后的操作. 常见的是将构建的结果, 上传到内测平台或者内部下载地址. 但是如果构建操作的脚本中包含有这一项, 这一个步骤也就可以忽略掉了.

## 遇到的问题
### 找不到Fir命令

![找不到Fir命令](https://ws2.sinaimg.cn/large/006tNc79gy1fhjsalg3fej31b80cewfp.jpg)
出现如图所示的Error, 一般是由于没有安装fit-cli命令导致, 通过安装`gem install fir-cli -V --no-ri --no-rdoc`来解决. 如果提示安装成功, 但是还是报错的话, 那就在安装命令前加上`sudo`, 覆盖安装即可.

### fir-cli jenkins fir:command not found

这个错误是由于没有导入fir-cli安装目录导致的, 可以先在终端输入`echo $PATH`, 然后把输出结果复制下来, 在Jenkins -> 系统管理 -> 添加全局变量. 然后在工程构建步骤, 先引入全局变量`export ${PATH}`. 如果没有设置全局变量的话, 也可以直接使用`export PATH=`echo $PATH 的输出结果``

![添加Path全局变量](https://ws4.sinaimg.cn/large/006tNc79gy1fhk9pyt7nsj31dg0cmwen.jpg)

### 证书配置文件没有找到

No iOS profile matching 'xxxxxx/xxxxxxx' found:Xcode couldn't find a profile
matching 'xxxxxx/xxxxxxx'.
Install the profile (by dragging and dropping it onto Xcode's dock item)
or select a different one in the General tab of the target editor.
Code signing is required for product type 'Application' in SDK 'iOS 10.3' 

是因为Archive时没有找到profile导致的. 解决办法是, 把系统的`/Users/用户名/Library/MobileDevice/Provisioning Profiles`整个Provisioning Profiles文件夹复制到`/Users/Shared/Jenkins/Library/MobileDevice`目录下, 并且在Jenkins -> Keychains and Provisioning Profiles Management 的Provisioning Profiles Directory Path中, 设置好profile的存放目录.

### Command/usr/bin/codesign failed with exit code 1

这个是由于没有给钥匙串开锁权限导致, 编译之前添加 `security unlock-keychain -p "你的密码" "path to keychain/login.keychain"`解决.
-----

## 待解决

使用Tomcat, 把Jenkins发布出去, 这个是下一步要解决的问题.
参考文章:
 
1.[iOS持续集成：jenkins+gitlab+蒲公英+邮件通知(Part 2)](https://runningyoung.github.io/2016/04/01/2016-04-05-jenkins2/)

2.[一步一步构建iOS持续集成:Jenkins+GitLab+蒲公英+FTP](http://www.jianshu.com/p/c69deb29720d)

3.[Jenkins/git/KeyChains & Provisioning, 记录CI中的一些坑](http://www.jianshu.com/p/e19c8327b167)

4.[使用 Jenkins 持续集成 iOS 项目时碰到的一些问题](http://www.jianshu.com/p/e19c8327b167)

5.[手把手教你利用Jenkins持续集成iOS项目](http://www.jianshu.com/p/41ecb06ae95f)


