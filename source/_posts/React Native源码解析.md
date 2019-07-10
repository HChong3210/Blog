---
title: React Native源码解析
date: 2018-04-17 21:56:38
tags:
    - RN
categories: 
    - RN
---

# 背景介绍
前端大致的开发流程就是:

* 用 HTML 创建 DOM, 构建整个网页的布局, 结构
* 用 CSS 控制 DOM 的样式, 比如字体, 字号, 颜色, 居中等
* 用 JavaScript 接受用户事件, 动态的操控 DOM

在React中, 我们使用JSX语法, 它是一种 JavaScript 语法拓展. JSX 允许我们写 HTML 标签或 React 标签, 它们终将被转换成原生的 JavaScript 并创建 DOM. 我们也可以在里面写CSS.

React 独创了 Virtual DOM 机制. Virtual DOM 是一个存在于内存中的 JavaScript 对象, 它与 DOM 是一一对应的关系, 也就是说只要有 Virtual DOM, 我们就能渲染出 DOM. 当界面发生变化时, 得益于高效的 DOM Diff 算法, 我们能够知道 Virtual DOM 的变化, 从而高效的改动 DOM, 避免了重新绘制 DOM.

React Native我们可以理解为Native 版本的 React. 即使使用了 React Native, 我们依然需要 UIKit 等框架, 调用的是 Objective-C 代码. JavaScript 只是辅助, 它只是提供了配置信息和逻辑的处理结果. 它只是以 JavaScript 的形式告诉 Objective-C 该执行什么代码. 

# React Native启动流程
![RN整体结构](http://img.souche.com/f2e/bfb53a0a04e1c7a9ca7c99d9440e0972.png)

RN的整体结构如图所示, RCTRootView 是RN的根视图, 它内部持有了 RCTBridge. RCTBridge 内部持有了 RCTBatchBridge 对象, 这个对象里面有着大部分的业务逻辑和核心代码.

RCTBatchBridge 对象会持有一个 RCTDisplayLink对象, 这个对象主要用于一些Timer, Navigator的Module需要按着屏幕渲染频率回调JS用的. 

RCTModuleXXX 表示所有的RN的Module组件都是 RCTModuleData, 无论是RN的核心系统组件还是扩展的UI组件, API组件. 

RCTJSExecutor 是一个特殊的 RCTModuleData. 他是系统组件的核心, 负责单独开一个线程, 执行JS代码, 处理JS回调, 是bridge的核心通道.

RCTEventDispatcher 也是一个特殊的 RCTModuleData. 各个业务模块通过它主动调起JS, 他封装了eventDispatcher得API来方便业务Module使用.

![源码加载顺序](http://img.souche.com/f2e/f0aa996dd91edbb43c65963064be0fae.png)

* 创建RCTRootView -> 设置窗口根控制器的View, 把RN的View添加到窗口上显示.
* 在RCTRootView中创建RCTBridge -> 桥接对象, 管理JS和OC交互, 做中转. 
* 在RCTBridge中创建RCTBatchedBridge -> 批量桥接对象, JS和OC的交互基本都在这里. 
    * 创建一个Displaylink, 以设备屏幕渲染的频率触发一个timer, 判断是否有个别module需要按着timer去回调js, 如果没有module, 这个模块其实就是空跑一个displaylink.
* RCTBatchedBridge对象 -> start
    * 创建一个dispatchQueue, 后面的操作都在这个队列里面, 保证顺序执行.
    * RCTBatchedBridge -> loadSource, 从本地或者远端拉取jsbundle.
    * RCTBatchedBridge -> initModulesWithDispatchGroup, 创建OC的模块表
        * RCTJavaScriptExecutor -> setUp, Asynchronously initialize the JS executor, 异步初始化JS的桥接对象, 通过JSContext设置桥接.
        * RCTBatchedBridge -> moduleConfig, Asynchronously gather the module config, 异步获取moduleConfig.
        * 上面两个完成后, 执行RCTJSCExecutor -> injectJSONText, 向JS中插入OC的模块表
    * 上面几个执行完后, 执行RCTBatchedBridge -> executeSourceCode, 确保在JSExecutor专属的Thread内执行jsbundle代码. 把前面创建的RCTDisplayLink添加在在JSExecutor的Thread所在的runloop上.

# React Native源码加载
源码加载, 也就是JS代码加载的过程, 主要是APP启动流程中RCTBatchedBridge -> loadSource这一步骤. 

* RCTJavaScriptLoader -> loadBundleAtURL, 加载JS代码, 这个URL可以是本地的文件路径也可以是服务器地址.
* RCTJavaScriptLoader -> attemptAsynchronousLoadOfBundleAtURL, 异步加载JS文件, 里面的判断较多, 对本地URL和远端URL有做区分对待.
* 上面下载完成后, 会把源代码通过block回调给外面来处理. RCTBatchedBridge -> executeSourceCode. 在这里会调用RCTJSCExecutor -> executeApplicationScript, 把JS代码交给RCTJSCExecutor对象然后在JS专用线程中来处理JS代码
* 处理结束后在回调中注册RCTDisplayLink到JS专用线程对应的RunLoop中. 然后发送 RCTJavaScriptDidLoadNotification 通知.
* 接收通知的地方有三个, 分别是隐藏self.window, 同步所有的设置, 新建一个RCTRootContentView.
* 在创建RCTRootContentView对象时, 会通过通知传过来的参数获取前面我们创建的RCTBridge中的RCTUIManager(管理UI组件). 并且把RCTRootContentView对象注册为RootView. 并且把它添加到_viewRegistry中去(这是一个字典, 所有注册过的View都在这里面, key是一个reactTag). 
* 在创建RCTRootContentView对象时, 也会创建一个RCTTouchHandler, 用来处理事件. 详见[这里](#React Native事件处理).
* RCTRootContentView对象创建完成后, 执行RCTRootContentView -> runApplication方法. 通知JS运行APP, 来渲染控件. 详见[这里](#React Native页面渲染).
* RCTBridge -> enqueueJSCall, 通过桥接对象让JS调用 APPRegistry. 
* RCTBatchedBridge -> _actuallyInvokeAndProcessModule, 通过批量桥架让JS执行AppRegistry方法.
* RCTJSCExecutor -> _executeJSCall, 让JS执行者调用JS代码.
* 执行完JS代码后, 会返回一个数组, 这个数组里面是OC要执行的操作(包含 RCTBridgeFieldRequestModuleIDs = 0, RCTBridgeFieldMethodIDs,  RCTBridgeFieldParams, RCTBridgeFieldCallID). 然后把数组传递给RCTBatchedBridge -> _processResponse. 
* 在这里面执行RCTBatchedBridge -> handleBuffer方法, 这个方法最终会便利数组依次调用RCTBatchedBridge -> callNativeModule方法.

# <span id = 'React Native页面渲染'>React Native页面渲染</span>
*每一个 js render() 方法中的 component 都对应于 native 中的一个 RCTView. 在渲染前, js 会循环取出各个 component. 然后通过 UIManager (NativeModules 中的一项) 的 createView 方法, 通过 BatchedBridge 调用 native 端的 RCTUIManager 的同名方法, 创建与 component 相对应的类. 对于触摸事件也是类似的. native 端通过 bridge 将触摸事件传给 js, js 在找到对应 component 后, 响应各个触摸事件.*

* RCTRootContentView -> runApplication, 通知JS运行APP.
* 在 native 调用 `-enqueueJSCall:method:args:completion` 后, 我们最终调用JS中 AppRegistry 的 runApplication方法. 下面这些都是JS线程中JS调用OC的方法.
    * RCTUIManager -> createView:viewName:rootTag:props: 
    * 这里使用了 RCT_EXPORT_METHOD 宏标记了该方法, 所以可以在JS中调用. 
    * 每创建一个 UIView 就会 创建一个 RCTShadowView, 与UIView一一对应. 保存对应UIView的布局和子控件, 管理UIView的加载.
    * 把设置好的 view 保存到 _viewRegistry 中. js 和 native 端都会保存好这个 tag 和 view 的映射表. 这里的 _viewRegistry 就是 native 端保存的字典.
    * 最终在 ReactNativeBaseComponent.js 中调用了 UIManager.createView(). 这里实际调用的是原生中 RCTUIManager 暴露出来的 createView 方法.
    * 此时又调用到JS代码中的RCTUIManager -> setChildren:reactTags: 给RCTRootView对应的RCTRootShadowView设置子控件. 继续调用RCTRootShadowView -> insertReactSubview方法, 遍历子控件给RCTRootShadowView插入所有子控件.
* RCTBatchedBridge -> _processResponse:error:, 处理完执行JS代码返回的对象.
* RCTBatchedBridge -> handleBuffer, 其中调用了RCTUIManager -> batchDidComplete方法. 
* 在上面的方法中又调用了RCTUIManager -> _layoutAndMount方法, 布局RCTRootView和增加子控件.
    * 会遍历所有的View, 调用 RCTShadowView -> processUpdatedProperties:处理保存在RCTShadowView中属性, 布局RCTShadowView对应UIView的所有子控件, 给原生View添加子控件
* 完成UI渲染.

# <span id = 'React Native事件处理'>React Native事件处理</span>

* 在创建 RCTRootContentView 时内部会 创建一个 RCTTouchHandler. RCTTouchHandler继承于 UIGestureRecognizer 内部实现了touchBegin等触摸方法, 用来处理触摸事件. RCTTouchHandler被添加到 RCTRootContentView 上面.
* 创建 RCTTouchHandler 时, 内部会创建 RCTEventDispatcher. 
* 当原生的Touchs事件被触发, 会在里面调用 RCTEventDispatcher -> sendEvent. 让事件分发对象调用发送事件对象, 并且内部会把事件保存到_eventQueue.
* 再调用 RCTEventDispatcher -> flushEventsQueue, 遍历获取事件队列中所有事件执行 RCTEventDispatcher -> dispatchEvent.
* 在 RCTEventDispatcher -> dispatchEvent 中我们可以看到又调用了 RCTBatchedBridge -> enqueueJSCall:args:方法, 又进入了桥接对象执行JS的步骤.
* 总的来说, 就是原生响应触摸事件, 通过 RCTEventDispatcher 传递到桥接对象, 最终还是调用JS的代码处理.

# React Native通信机制
React Native用iOS自带的JavaScriptCore作为JS的解析引擎, 并没有用到JavaScript提供的一些可以让JS与OC互调的特性, 而是自己实现了一套机制, 这套机制可以通用于所有JS引擎上.

首先我们要介绍两个宏:

```
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
```
可以看到添加 RCT_EXPORT_MODULE 宏, 相当定义了load, moduleName方法. 正是在load方法中调用了 RCTRegisterModule 方法注册 Module. 注意过多的load方法是会影响APP的启动速度的. RCTRegisterModule 只是简单收集所有需要曝露给 JS 的类. 

```
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)
  
#define RCT_REMAP_METHOD(js_name, method) \
  RCT_EXTERN_REMAP_METHOD(js_name, method) \
  - (void)method

#define RCT_EXTERN_REMAP_METHOD(js_name, method) \
  + (NSArray<NSString *> *)RCT_CONCAT(__rct_export__, \
    RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    return @[@#js_name, @#method]; \
  }  
```
可以看到通过宏添加了一个新方法, 其命名规则: `__rct_export__ + __LINE__ + __COUNTER__`. 同时注意到, 所有曝露给 JS 的方法返回值都是 void, 要返回结果时, 需通过 callback 方式实现.

在RN初始化过程中会调用 RCTBatchedBridge -> -initModulesWithDispatchGroup:方法, 这个方法中会调用如下代码: 

```
  for (Class moduleClass in RCTGetModuleClasses()) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);

    // Check for module name collisions
    RCTModuleData *moduleData = moduleDataByName[moduleName];
    if (moduleData) {
      if (moduleData.hasInstance) {
        // Existing module was preregistered, so it takes precedence
        continue;
      } else if ([moduleClass new] == nil) {
        // The new module returned nil from init, so use the old module
        continue;
      } else if ([moduleData.moduleClass new] != nil) {
        // Both modules were non-nil, so it's unclear which should take precedence
        RCTLogError(@"Attempted to register RCTBridgeModule class %@ for the "
                    "name '%@', but name was already registered by class %@",
                    moduleClass, moduleName, moduleData.moduleClass);
      }
    }

    // Instantiate moduleData (TODO: can we defer this until config generation?)
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass
                                                     bridge:self];
    moduleDataByName[moduleName] = moduleData;
    [moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
  
//查找所有响应 RCTBridgeModule 的class.
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
```
暴露给 JS 的类被 RCT_EXPORT_MODULE 宏注册. 暴露给 JavaScript 的方法被 RCT_EXPORT_METHOD 宏为函数名加上了 `__rct_export__` 前缀, 通过 runtime 获取类的函数列表. 至此, 所有需要曝露给 JS 的 module , Method都已注册完成, 并以RCTModuleData格式存储在 RCTCxxBridge 中. Bridge 持有一个数组, 数组中保存了所有的模块的 RCTModuleData 对象. 只要给定 ModuleId 和 MethodId 就可以唯一确定要调用的方法.

在 React Native 中, Objective-C 和 JavaScript 的交互都是通过传递 ModuleId、MethodId 和 Arguments 进行的.

Objective-C 调用 JavaScript 代码时:

* 在加载完JS代码后会调用 RCTBatchedBridge 的 `-enqueueApplicationScript:url:onComplete`方法.
* 加载完成后会调用 RCTJSCExecutor 的 `-flushedQueue:`方法.
* 最终主要是在 RCTJSCExecutor 的  `-_executeJSCall:arguments:unwrapResult:callback`方法, 这里这个函数名是我们要调用 JavaScript 的中转函数名, 它的作用其实是处理参数, 而非真正要调用的 JavaScript 函数. 这个中转函数接收到的参数包含了 ModuleId、MethodId 和 Arguments, 然后由中转函数查找自己的模块配置表, 找到真正要调用的 JavaScript 函数.

实际使用的时候, 我们可以使用 RCTEventDispatcher 的 `-sendEvent:`方法直接向JS发送事件. 而不用通过模块配置表.

JavaScript 调用 Objective-C 代码时: 

* JS端调用某个OC模块暴露出来的方法. 
* JavaScript 会解析出方法的 ModuleId, MethodId 和 Arguments 并放入到 MessageQueue 中, 等待 Objective-C 主动拿走, 或者超时后主动发送给 Objective-C.
* 把JS的callback函数缓存在MessageQueue的一个成员变量里, 用CallbackID代表callback.
* Objective-C 负责处理调用的方法是 RCTBatchedBridge 的`-handleBuffer:` 在这里解析出ModuleId, MethodId, Params
* 然后调用 RCTBatchedBridge 的 `- (id)callNativeModule:method:params:` 方法.
* RCTModuleMethod对JS传过来的每一个参数进行处理, CallbackID类型被转换为一个 block
* 再实现 RCTModuleMethod 的 `[method invokeWithBridge:self module:moduleData.instance arguments:params]` 方法, OC模块方法调用完, 执行block回调至此方法调用结束.
* RCTModuleMethod 还有一个 `-processMethodSignature`方法, 它会根据 JavaScript 的 CallbackId 创建一个 Block, OC模块方法调用完, 执行上面参数转换的block回调.
* block里带着CallbackID 和block 传过来的参数去调 JS 里 MessageQueue 的方法 invokeCallbackAndReturnFlushedQueue. 
* MessageQueue 通过 CallbackID 找到相应的JS callback方法. 调用callback方法, 并把OC带过来的参数一起传过去, 完成回调.


------

参考资料:
1.[React Native通信原理解析](http://i.dotidea.cn/2016/05/react-native-communication-principle-for-ios/)

2.[ReactNative iOS源码解析](http://awhisper.github.io/2016/07/02/ReactNative%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901/)

3.[React-Native 初始化与通信原理源码分析](https://zhang759740844.github.io/2017/01/17/RN%20Native%E5%88%9D%E5%A7%8B%E5%8C%96%E8%BF%87%E7%A8%8B/)

4.[React Native原理篇整理笔记](https://www.jianshu.com/p/9e4f9f98f82d)

5.[React Native通讯机制—从源码分析](https://blog.csdn.net/u011342466/article/details/57180192)

6.[React Native从源码一步一步解析它的实现原理](https://www.jianshu.com/p/bd7dfd8c9917)

7.[React Native源码分析](http://fighting300.com/2017/03/02/reactnative-code/)

8.[React Native源码分析原理（二）(基于0.48版本)](https://www.jianshu.com/p/0688b24950f4)

9.[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)

10.[React-Native 如何创建原生 View 源码解析](https://zhang759740844.github.io/2017/01/24/RN%E6%98%BE%E7%A4%BA%E5%8E%9F%E7%90%86/)

11.[ReactNative源码解析——通信机制详解](http://zxfcumtcs.github.io/2017/10/08/ReactNativeCommunicationMechanism/)

12.[React Native从入门到原理](https://bestswifter.com/react-native/#javascriptrctjscexecutor)

13.[React Native源码中JavaScriptCore详解](https://www.jianshu.com/p/bd7dfd8c9917)

