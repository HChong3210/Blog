---
title: 自己动手制作Xcode插件
date: 2017-06-25 21:02:48
tags:
    - 基础知识
    - Xcode

categories:
    - 插件
---

# 自己动手制作Xcode插件

Xcode8以后, 以前的插件都不能用了. 网上虽然也有方法来解决这个问题, 但是稍显复杂, 这里我们采用曲线救国的方式, 使用Apple官方推荐的方式*Xcode Source Editor Extension*来自己制作插件. Extension的方式开发的插件, 可以独立上架AppStore, 并且是独立于Xcode工程独立运行的. 但是没有UI交互, 不能在后台运行并且只能在开发者调用的时候直接修改代码.

在平常的开发过程中, 有些操作是经常遇到的. 下面我们以*根据属性自动生成Getter方法*和*根据选中内容导入头文件*两种经常碰到的场景来制作Xcode插件.

## 新建插件制作工程

1. 打开Xcode, `command + shift + N` 选择macOS -> Cocoa Application, 点击Next新建一个工程. 此处我新建的工程名为*AutoImportPlugin*.

2. 在Xcode工具栏中选择Editor -> Add Target -> MacOS -> Xcode Source Editor Extension来新建一个Extension来编写插件的主要代码. 新建的Target名称不能与工程的名字一样. 接下来会有一个弹窗, 提示你是否*Activate xxx Scheme*, 选择*Activate*即可, 这一步骤会帮你创建一个和你新建Target对应的Scheme. 完成后你的页面应该是这个样子的.

![主工程页面](https://ws1.sinaimg.cn/large/006tKfTcgy1fgyotdiqdbj312w0q03zl.jpg)

## 工程环境和参数介绍

这里我们会主要用到AutoImport文件夹下面的两个类`SourceEditorExtension` 和 `SourceEditorCommand`. 

`SourceEditorExtension.m`中我们可以看到两个方法:

* `-(void)extensionDidFinishLaunching{}`, 注释内容为`If your extension needs to do any work at launch, implement this optional method.`   指的是指刚刚加载好插件但还未点击插件按钮时，可以执行某些准备工作.
* `- (NSArray <NSDictionary <XCSourceEditorCommandDefinitionKey, id> *> *)commandDefinitions{}`. 返回字典类型的数组, 可以为每个插件重写名字、标识符和自定义类名等信息，和`Info.plist`文件中对应的`XCSourceEditorCommandName`、`XCSourceEditorCommandIdentifier`和`XCSourceEditorCommandClassName`等信息一致.

`SourceEditorCommand.m`中只有一个方法:

````
- (void)performCommandWithInvocation:(XCSourceEditorCommandInvocation *)invocation completionHandler:(void (^)(NSError * _Nullable nilOrError))completionHandler
{
    // Implement your command here, invoking the completion handler when done. Pass it nil on success, and an NSError on failure.
    
    completionHandler(nil);
}
````

关于插件的核心逻辑代码, 就是在这个方法里面实现. `XCSourceEditorCommandInvocation`类包含了所有的信息.可以通过他的各种属性来拿到原工程中的各种数据, 主要用到的有:

* `invocation.buffer.lines`是一个数组, 包含了使用插件的工程的页面的每一行代码, 艺术组的形式存在. 
* `invocation.buffer.selections`是一个数组, 数组的内容是`XCSourceTextRange`. `XCSourceTextRange`包含了两个结构体, 用于标记原工程中选中部分的代码的位置信息.
## 核心代码

此处以自动导入头文件的插件为例. 主要思路是遍历类的所有行, 拿到选中的内容, 判断是否被import过, 没有的话, 在合适位置插入`#import`;
```
- (void)performCommandWithInvocation:(XCSourceEditorCommandInvocation *)invocation completionHandler:(void (^)(NSError * _Nullable nilOrError))completionHandler
{
    XCSourceTextRange *range = [invocation.buffer.selections firstObject];
    
    NSString *selectedLines = [invocation.buffer.lines objectAtIndex:range.start.line];
    NSString *selection = [selectedLines substringWithRange:NSMakeRange(range.start.column, range.end.column - range.start.column)];
    
    for (NSInteger i = 0; i < invocation.buffer.lines.count - 1; i++) {
        NSString *importString = [invocation.buffer.lines objectAtIndex:i];
        if ([importString containsString:@"@interface"]) {
            self.startLineNumber = i;
            break;
        }
    }
    
    BOOL isImported = NO;
    for (NSInteger i = 0; i <= self.startLineNumber ; i++) {
        NSString *importString = [invocation.buffer.lines objectAtIndex:i];
        if ([importString containsString:[NSString stringWithFormat:@"%@.h", selection]]) {
            isImported = YES;
            break;
        }
    }
    if (isImported == NO) {
        [invocation.buffer.lines insertObject:[NSString stringWithFormat:@"#import \"%@.h\"", selection] atIndex:self.startLineNumber - 1];
    }
    completionHandler(nil);
}
```


## 插件使用

### 基础用法

我们在插件工程中选中AutoImport的Scheme(新建的那个), `command + R`运行,这时会出现一个弹窗, 让你选择要运行的工程. 选中要运行的工程后, 正常打开. 在Editor的最下面可以看到我们运行的插件工程, 点击就可以运行插件了. 打断点和调试什么的和正常的Xcode项目一样.

![插件运行调试](https://ws4.sinaimg.cn/large/006tNc79gy1fgz0hh71w6j30ht0ccdgm.jpg)

需要注意的是, 我们在插件工程中两个Target的Singing一定要保持一致, 并且不能为空. 否则在Xcode中看不到插件.

### 进阶用法

以上我们已经可以在自己电脑上调试和使用Xcode插件了, 那么怎么才能在其他电脑上也使用我们开发的插件呢?

1. 我们可以在插件工程中, 找到Products文件夹下生成的的.app文件

2. 右键点击此文件 -> “在Finder中显示” -> 将这个.app文件拷贝到你或者小伙伴电脑上的"应用程序"里

3. 在“应用程序”中双击.app文件运行。然后，打开“系统偏好设置” -> "扩展" -> "Xcode Source Editor" -> 确认插件名字前已打钩 -> 此时Xcode中菜单栏Editor下的插件虽然显示，但是为灰色，无法点按，所以要 -> 重启Xcode -> 大功告成！

4. 添加快捷键: Xcode -> "Preferences" -> "Key Bindings" -> 搜索插件名字 -> 添加对应的快捷键.

-------
下载地址: [AutoPlugin](https://github.com/HChong3210/AutoGetterPlugin)

-------
参考文章:
1.[详解一步步实现Xcode 8 插件——Source Editor Extensions](http://www.jianshu.com/p/9c9d0fcc62cc)

2.[使用 Xcode Source Editor Extension开发Xcode 8 插件](http://www.code4app.com/blog-822721-394.html)

