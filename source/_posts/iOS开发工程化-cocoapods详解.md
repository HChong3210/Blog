---
title: iOS开发工程化-cocoapods详解
date: 2019-07-29 20:37:46
tags:
    - 基础知识
categories:
    - iOS开发-工程化
---

本文主要讲解cocoapods的相关知识, 关于cocoapods的实践, 可以参考[这里](http://hchong.net/2017/05/24/Cocoapods%E5%AE%9E%E8%B7%B5/).

# 1 源码分析
pod的使用主要在 `pod install` 和 `pod update`, 我们可以在cocoapods的1.7.5版本中查看入口的[源码](https://github.com/CocoaPods/CocoaPods/blob/1.7.5/lib/cocoapods/installer.rb).

``` ruby
def install!
  prepare
  resolve_dependencies
  download_dependencies
  validate_targets
  generate_pods_project
  if installation_options.integrate_targets?
    integrate_user_project
  else
    UI.section 'Skipping User Project Integration'
  end
  perform_post_install_actions
end
```
分析源码可以发现, install的执行分为如下几个步骤: *准备阶段, 查找依赖库, 下载依赖文件, 检验target, 生成pod工程, 整合project文件, 执行安装后操作*, 下面会逐步讲解一下.

## 1.1 准备阶段

``` ruby
def prepare
  # Raise if pwd is inside Pods
  if Dir.pwd.start_with?(sandbox.root.to_path)
    message = 'Command should be run from a directory outside Pods directory.'
    message << "\n\n\tCurrent directory is #{UI.path(Pathname.pwd)}\n"
    raise Informative, message
  end
  UI.message 'Preparing' do
    deintegrate_if_different_major_version
    sandbox.prepare
    ensure_plugins_are_installed!
    run_plugins_pre_install_hooks
  end
end
```
我们可以看到, 准备阶段(prepare)主要做了以下事情:

* 检查podfile.lock的写入的cocoapods版本和当前cocoapods版本是否一致，如果不一致将会重塑工程，将除了Podfile、Podfile.lock、Workspace以外的其他关联和依赖全部重置
* 沙盒的准备 - 一些文件以及目录的删除以及创建
* 迁移沙盒中部分文件(区分Pods版本迁移地址不同)
* 确保Podfile指定的插件都已经安装(不然抛错)
* 执行pre_install的Hook

## 1.2 查找依赖库

```ruby
def resolve_dependencies
  plugin_sources = run_source_provider_hooks
  analyzer = create_analyzer(plugin_sources)

  UI.section 'Updating local specs repositories' do
    analyzer.update_repositories
  end if repo_update?

  UI.section 'Analyzing dependencies' do
    analyze(analyzer)
    validate_build_configurations
  end

  UI.section 'Verifying no changes' do
    verify_no_podfile_changes!
    verify_no_lockfile_changes!
  end if deployment?

  analyzer
end
```
我们可以看到, 查找依赖库(resolve_dependencies)主要做了以下事情:

* 遍历注册的所有插件，其中通过HooksManager.register方法注册name为:source_provider的插件
* 执行create_analyzer方法创建安装分析器
* 如果我们在执行pod install时附加了--repo-updateflag, 则刚才创建的analyzer实例将执行update_repositories方法去更新本地repo仓库的所有pod spec文件.
* 验证Build Configurations参数的有效性
* 验证podfile的变化
* 验证lockfile的变化

## 1.3 下载依赖文件

```ruby
def download_dependencies
  UI.section 'Downloading dependencies' do
    install_pod_sources
    run_podfile_pre_install_hooks
    clean_pod_sources
  end
end
```
我们可以看到, 下载依赖文件(download_dependencies)主要做了以下事情:

* 下载安装Pods依赖库源文件
* 执行Podfile中pre_install钩子方法
* 根据Config和Installers参数清理Pods的源文件

## 1.4 检验target

```ruby
def validate_targets
  validator = Xcode::TargetValidator.new(aggregate_targets, pod_targets)
  validator.validate!
end
```
检验target(validate_targets), 主要做了以下事情:

* 检测是否有多重引用 framework 或者 library 的情况(Framework的名字是否冲突, 如果冲突会抛出`frameworks with conflicting names`异常)
* 处理静态库传递依赖问题(静态库的传递依赖如果形成会主动抛出`transitive dependencies that include static binaries`异常)
* 校验不同 target 所引用的代码中, 如果包含 swift, 所使用的 swift 版本是否相同
* 检查是否引用了Switf书写的framework(Podfile中没有指定use framework!。如果验证不通过, 主动抛出异常)

## 1.5 生成Pods工程

```ruby
def generate_pods_project
  stage_sandbox(sandbox, pod_targets)

  cache_analysis_result = analyze_project_cache
  pod_targets_to_generate = cache_analysis_result.pod_targets_to_generate
  aggregate_targets_to_generate = cache_analysis_result.aggregate_targets_to_generate

  clean_sandbox(pod_targets_to_generate)

  create_and_save_projects(pod_targets_to_generate, aggregate_targets_to_generate,
                           cache_analysis_result.build_configurations, cache_analysis_result.project_object_version)
  SandboxDirCleaner.new(sandbox, pod_targets, aggregate_targets).clean!

  update_project_cache(cache_analysis_result, target_installation_results)
  write_lockfiles
end
```
生成Pods工程(generate_pods_project), 主要做了:

* 调用Podfile中post_install钩子方法
* 生成Pods/目录下面的所有工程
* 生成Podfile.lock文件和Manifest.lock文件

## 1.6 整合project文件

```ruby
def integrate_user_project
  UI.section "Integrating client #{'project'.pluralize(aggregate_targets.map(&:user_project_path).uniq.count)}" do
    installation_root = config.installation_root
    integrator = UserProjectIntegrator.new(podfile, sandbox, installation_root, aggregate_targets, generated_aggregate_targets,
                                           :use_input_output_paths => !installation_options.disable_input_output_paths?)
    integrator.integrate!
  end
end
```
整合project文件(integrate_user_project), 主要做了:

* 创建.xcworkspace文件
* 集成Target
* 警告检查
* 保存.xcworkspace文件到目录

## 1.7 执行安装后操作

``` ruby
def perform_post_install_actions
  run_plugins_post_install_hooks
  warn_for_deprecations
  warn_for_installed_script_phases
  print_post_install_message
end
```
执行安装后操作(perform_post_install_actions), 主要做了以下操作:

* unLock Pods库下的文件
* 调用plugin的post_install钩子方法
* 打印所有被废弃的pods警告信息
* 打印所有pods中脚本的警告信息
* 打印install中的所有信息

## 1.8 总结

* 准备阶段
    * 检查podfile.lock的写入的cocoapods版本和当前cocoapods版本是否一致，如果不一致将会重塑工程，将除了Podfile、Podfile.lock、Workspace以外的其他关联和依赖全部重置
    * 沙盒的准备 - 一些文件以及目录的删除以及创建
    * 迁移沙盒中部分文件(区分Pods版本迁移地址不同)
    * 确保Podfile指定的插件都已经安装(不然抛错)
    * 执行pre_install的Hook
* 查找依赖库
    * 遍历注册的所有插件，其中通过HooksManager.register方法注册name为:source_provider的插件
    * 执行create_analyzer方法创建安装分析器
    * 如果我们在执行pod install时附加了--repo-updateflag, 则刚才创建的analyzer实例将执行update_repositories方法去更新本地repo仓库的所有pod spec文件.
    * 验证Build Configurations参数的有效性
    * 验证podfile的变化
    * 验证lockfile的变化   
* 下载依赖文件
    * 下载安装Pods依赖库源文件
    * 执行Podfile中pre_install钩子方法
    * 根据Config和Installers参数清理Pods的源文件* 
* 检验target
    * 检测是否有多重引用 framework 或者 library 的情况(Framework的名字是否冲突, 如果冲突会抛出`frameworks with conflicting names`异常)
    * 处理静态库传递依赖问题(静态库的传递依赖如果形成会主动抛出`transitive dependencies that include static binaries`异常)
    * 校验不同 target 所引用的代码中, 如果包含 swift, 所使用的 swift 版本是否相同
    * 检查是否引用了Switf书写的framework(Podfile中没有指定use framework!。如果验证不通过, 主动抛出异常)
* 生成pod工程
     * 调用Podfile中post_install钩子方法
    * 生成Pods/目录下面的所有工程
    * 生成Podfile.lock文件和Manifest.lock文件
* 整合project文件
    * 创建.xcworkspace文件
    * 集成Target
    * 警告检查
    * 保存.xcworkspace文件到目录 
* 执行安装后操作
    * unLock Pods库下的文件
    * 调用plugin的post_install钩子方法
    * 打印所有被废弃的pods警告信息
    * 打印所有pods中脚本的警告信息
    * 打印install中的所有信息  

# 2 实践 & 私有Pod
可以参考之前的[总结](http://hchong.net/2017/05/24/Cocoapods%E5%AE%9E%E8%B7%B5/)

# 3 二进制方案

[这里](https://dmanager.github.io/ios/2019/01/21/%E5%9F%BA%E4%BA%8ECocoaPods%E7%9A%84%E7%BB%84%E4%BB%B6%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%8C%96%E5%AE%9E%E8%B7%B5/)

[这里](https://www.jianshu.com/p/5338bc626eaf)是业界比较完善的解决方案

* 主要的思路就是在创建pod的时候会同时存在源码和framework两种形态的包(可以通过新建target或者新建工程两种方式) 
* 在podspec中根据传入的参数来指定如何在源码和二进制中切换.

# 4 常见问题


-------

参考资料:
1.[CocoaPods 都做了什么?](https://draveness.me/cocoapods)
2.[Cocoapods源码浅谈](http://silentcat.top/2018/09/04/Cocoapods%E6%BA%90%E7%A0%81%E6%B5%85%E8%B0%88/)
3.[pod install和pod update背后那点事](http://blog.startry.com/2015/09/29/Something-about-Pod-Install-And-Pod-Update/)
4.[cocoapods源码](https://github.com/CocoaPods/CocoaPods/blob/1.7.5/lib/cocoapods/installer.rb)
5.[Cocoapods 二进制](https://juejin.im/post/5cbec5fb5188250aa21919d0)
6.[基于 CocoaPods 的组件二进制化实践](https://dmanager.github.io/ios/2019/01/21/%E5%9F%BA%E4%BA%8ECocoaPods%E7%9A%84%E7%BB%84%E4%BB%B6%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%8C%96%E5%AE%9E%E8%B7%B5/)
7.[iOS CocoaPods组件平滑二进制化解决方案](https://www.jianshu.com/p/5338bc626eaf)
8.[iOS CocoaPods组件平滑二进制化解决方案及详细教程二之subspecs篇](https://www.jianshu.com/p/85c97dc9ab83)




