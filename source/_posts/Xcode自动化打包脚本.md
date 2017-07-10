---
title: Xcode自动化打包脚本
date: 2017-05-17 22:22:54
tags:
	- 自动化打包
	- 解决方案
	- Target
categories:
- 自动化打包
---

# Xcode自动化打包脚本

自动化打包脚本是配合[多渠道包和多环境包的自动化实现](http://hchong.net/2017/03/12/%E5%A4%9ATarget%E5%AE%9E%E7%8E%B0/)使用的, 实际上脚本语言都可以做到, 我这里选用了Shell和Python两种实现方式. 对比下来发现, Python更好懂一点, 但是Shell更加简洁.

## 打包的基本思路

这里说一下打包脚本的基本实现思路:

1. 需要传入的参数有Scheme(用来指定打哪个环境的包), 如果使用Fir来作为内测分发工具的话, 还需要传入Fir的Token.
2. 在脚本内需要指定工程的路径, 工程名, Archive包的路径, IPA包的路径.
3. 通过Xcodebuild命令行生成Archive包.
4. 根据生成的Archive包导出IPA包.
5. 上传IPA包到Fir.

## 遇到的问题

1. Xcode8.3后, 需要一个打包参数配置的plist文件, 生成Archive包时会用到.
2. shell脚本需要给权限`chomd 777 xx.sh`.
3. Archive导出为IPA有时会报错`Code=14 "No applicable devices found."`, 这个多少是Ruby的路径没有指定导致, 找了一大圈有两个解决方案:
	1. 通过`sudo gem install CFPropertyList`, `rvm list`, `rvm use system`来解决, Python或者Shell都可以使用这种方式.
	2. Shell下还有另外的解决方案, 在shell脚本前加入如下代码, 指定路径.
```
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
rvm use system
xcodebuild "$@"
```


## Xcode8.3后的plist文件


使用`xcodebuild -help` 可以查看Xcodebuild相关介绍

```
Available keys for -exportOptionsPlist:

	compileBitcode : Bool

		For non-App Store exports, should Xcode re-compile the app from bitcode? Defaults to YES.

	embedOnDemandResourcesAssetPacksInBundle : Bool

		For non-App Store exports, if the app uses On Demand Resources and this is YES, asset packs are embedded in the app bundle so that the app can be tested without a server to host asset packs. Defaults to YES unless onDemandResourcesAssetPacksBaseURL is specified.

	iCloudContainerEnvironment

		For non-App Store exports, if the app is using CloudKit, this configures the "com.apple.developer.icloud-container-environment" entitlement. Available options: Development and Production. Defaults to Development.

	manifest : Dictionary

		For non-App Store exports, users can download your app over the web by opening your distribution manifest file in a web browser. To generate a distribution manifest, the value of this key should be a dictionary with three sub-keys: appURL, displayImageURL, fullSizeImageURL. The additional sub-key assetPackManifestURL is required when using on demand resources.

	method : String

		Describes how Xcode should export the archive. Available options: app-store, ad-hoc, package, enterprise, development, and developer-id. The list of options varies based on the type of archive. Defaults to development.

	onDemandResourcesAssetPacksBaseURL : String

		For non-App Store exports, if the app uses On Demand Resources and embedOnDemandResourcesAssetPacksInBundle isn't YES, this should be a base URL specifying where asset packs are going to be hosted. This configures the app to download asset packs from the specified URL.

	teamID : String

		The Developer Portal team to use for this export. Defaults to the team used to build the archive.

	thinning : String

		For non-App Store exports, should Xcode thin the package for one or more device variants? Available options: <none> (Xcode produces a non-thinned universal app), <thin-for-all-variants> (Xcode produces a universal app and all available thinned variants), or a model identifier for a specific device (e.g. "iPhone7,1"). Defaults to <none>.

	uploadBitcode : Bool

		For App Store exports, should the package include bitcode? Defaults to YES.

	uploadSymbols : Bool

		For App Store exports, should the package include symbols? Defaults to YES.

```

我们在工程中新建一个plist文件, 可以看出, 大部分是有默认值的, 所以我们不用每个选项都填, 只写一些必填的就可以. 

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>method</key>
        <string>enterprise</string>
        <key>uploadSymbols</key>
        <true/>
        <key>uploadBitcode</key>
        <false/>
    </dict>
</plist>
```
-------

[附件下载](https://github.com/HChong3210/buildScript.git)

-------

参考文章:

1.[自动化测试-持续集成(7)](https://diaojunxian.github.io/2016/10/21/%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E5%AE%9E%E6%96%BD-%E4%B8%83/)

2.[xcodebuild: “No applicable devices found.” when exporting archive](http://stackoverflow.com/questions/33041109/xcodebuild-no-applicable-devices-found-when-exporting-archive)

3.[]()