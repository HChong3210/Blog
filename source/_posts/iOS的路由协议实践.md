---
title: iOS的路由协议实践
date: 2017-05-23 19:29:29
tags:
    - 模块化
    - 路由
categories:
    - 模块化
---


# iOS的路由协议实践

路由协议, 是组件化的核心所在. 组件化, 实际就会把代码拆分为一个一个的模块, 无论采用Pod的方式,  文件夹分割的方式, 还是静态库的方式, 实质都是把代码分为一个个的模块. 如何在模块之间和应用之间通信, 就是路由协议需要考虑的问题.

关于路由协议, 冰霜的[这篇博客](https://halfrost.com/ios_router/)写的实在是太详细了, 看得是男默女泪, 简直是业界良心. 简单说来, 路由协议跳转主要解决这几类问题:

1. 外部跳转到App内部一个很深层次的一个界面.
2. App之间的相互跳转.
3. 解除App组件之间和App页面之间的耦合性.
4. 统一iOS和Android两端的页面跳转逻辑, 统一三端的请求资源的方式.
5. iOS和Android两边只要共用一套动态下发配置文件来配置App的跳转逻辑.
6. 在App任何界面都可以调用任意一个界面或者任意一个组件.

## 应用间路由跳转

应用间路由跳转主要有以下几种常见的使用场景: 

1. 使用第三方用户登录，跳转到需授权的App或跳转到分享app的对应页面.
2. 应用程序推广, 跳转到另一个应用程序(本机已经安装).
3. 跳转到iTunes并显示应用程序下载页面(本机没有安装).
4. 第三方支付, 跳转到第三方支付App, 如支付宝支付, 微信支付.
5. 使用系统内置程序, 如跳转到打电话, 发短信, 发邮件, Safari等.

应用间的跳转主要有两种方式: URL Scheme和Universal Links. 这两种实现方式并不冲突, 可以共存.

### URL Scheme方式跳转

以A->B为例, 来说明下如何跳转. 
首先我们需要分别在两个App的info.plist里面添加对应的URL types - URL Schemes, 如图所示:

![添加URL types](http://img.souche.com/f2e/2b762dfb368b3b50621cd5c39b51a86e.png)

A的URL Schemes是APPA, B的URL Schemes是APPB. 由于iOS9引入了白名单的概念, 
如果使用 canOpenURL:方法, 该方法所涉及到的 URL Schemes 必须在"Info.plist"中将它们列为白名单, 否则`canOpenURL`返回`NO`, 不能正常跳转. 所以要在A中添加B的URL Schemes, B中添加A的Schemes. key叫做`LSApplicationQueriesSchemes`, 键值内容是上一步对应应用程序的URL Schemes. 

```

- jumpToAppB:(id)sender {
   // 1.获取应用程序App-B的URL Scheme
   NSURL *appBUrl = [NSURL URLWithString:@"zacharyB1://"];
   // 2.判断手机中是否安装了对应程序
   if ([[UIApplication sharedApplication] canOpenURL:appBUrl]) {
       // 3. 打开应用程序App-B
       //[[UIApplication sharedApplication] openURL:appBUrl];//iOS 9之后被废弃
       [[UIApplication sharedApplication] openURL:appBUrl options:@{UIApplicationOpenURLOptionUniversalLinksOnly : @YES} completionHandler:^(BOOL success) {
            
        }];
   } else {
       NSLog(@"没有安装");
   }
}
```

options目前可传入参数Key在UIApplication头文件只有一个:UIApplicationOpenURLOptionUniversalLinksOnly, 其对应的Value为布尔值, 默认为False. 如该Key对应的Value为True, 那么打开所传入的Universal Link时, 只允许通过这个Link所代表的iOS应用跳转的方式打开这个链接, 否则就会返回success为false, 也就是说只有安装了Link所对应的App的情况下才能打开这个Universal Link, 而不是通过启动Safari方式打开这个Link的代表的网站. 

至此, 就可以正常跳转了, 如果我们不希望某个APP通过URL Scheme的方式打开我们的应用, 我们可以在`- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation`方法中判断指定的Scheme然后返回`NO`, 如下所示, 只有`com.tencent.weixin`的Scheme才能打开我们的APP. 

```
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation {
    NSLog(@"sourceApplication: %@", sourceApplication);
    NSLog(@"URL scheme:%@", [url scheme]);
    NSLog(@"URL query: %@", [url query]);

    if ([sourceApplication isEqualToString:@"com.tencent.weixin"]){
        // 允许打开
        return YES;
    }else{
        return NO;
    }
}

```

### Universal Links方式跳转

使用这个功能可以使我们的App通过HTTP链接来启动App, 通用链接就是HTTP协议的普通URL, 通过在服务器上配置一些文件, 配合应用. 实现客户点击网页链接之后直接打开应用. 客户在微信\QQ中点击链接时不再需要点击右上'在Safari浏览器打开'才能打开软件, 实现客户操作的无缝跳转, 让客户体验更加连贯, 更顺畅. 

1. 如果安装过App, 不管在微信里面http链接还是在Safari浏览器, 还是其他第三方浏览器, 都可以打开App. 

2. 如果没有安装过App, 就会打开网页. 


具体设置需要3步: 

1. App需要开启Associated Domains服务, 并设置Domains, 注意必须要applinks：开头. 这里需要在APP使用的证书中设置这个选项, 否则在APP设置中看不到Associated Domains服务. 

2. 域名必须要支持HTTPS. 

3. 上传内容是Json格式的文件, 文件名为apple-app-site-association到自己域名的根目录下, 或者.well-known目录下. iOS自动会去读取这个文件. 具体的文件内容请查看[官方文档](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html). 

[参考文章一](http://www.jianshu.com/p/c83164c2aec2)

[参考文章二](http://www.jianshu.com/p/1970fd59de12)

## 应用内路由协议设计思路

*页面间跳转* 和 *组件间调用*, 是应用内路由协议要解决的两大问题, 举个例子来说明一下:

* *页面跳转*, 如果以传统的Push来说, 我们怎么才能从任一界面push到另外任一界面, 各个界面之间的跳转必然要相互`Import`, 这些怎么解决.
* *组件见调用*, 随着业务的模块化拆分, 各个模块之间有业务调用怎么办, 本身独立的模块, 为了相互调用必然又增加接口供外部调用, 本身相对独立的业务模块, 瞬间又变得相互耦合了.

而路由协议正是为了解决这一类问题, 现在比较流行的路由协议有如下几种:

* [JLRouts](https://github.com/joeldev/JLRoutes)
* [Routable](https://github.com/clayallsopp/routable-ios)
* [HHRouter](https://github.com/lightory/HHRouter)
* [MGJRouter](https://github.com/meili/MGJRouter)
* [CTMediator](https://github.com/casatwy/CTMediator)

![URL资源](https://ob6mci30g.qnssl.com/Blog/ArticleImage/40_15.png)

通过上面应用间的跳转, 我们可以发现iOS 系统里面使用的是URL Scheme方式. 对于一个资源的访问，苹果也是用URL的方式来访问的, 那么我们就可以想办法通过URL来统一三端的跳转. 一段标准的URL的格式, 每一部分都代表不同的含义. 我们可以按照规则来解析接受到的URL, 从而获得有用的信息. 下面说一下我的解决方案. 

大致的解决思路如下:

1. 在APP开始加载时设置Module的`Scheme`, 并且初始化Module的核心类`HCModuleCore`.

    `Scheme`是用来标记当前APP, 每个APP的`Scheme`不尽相同, 他和应用间跳转时设置的`Scheme`是一个东西. `HCModuleCore`是个单例, 它存在于整个APP的生命周期中. 
    
2. 有一个`HCModuleProtocol`协议, 里面有几个必须要实现的方法.
    
    如果某个类想要通过协议被跳转就必须实现该协议, 并且实现协议中的`@required`方法. `@optional`方法根据实际需要选择实现.
    
    ```
    HCModuleProtocol.h
    
    @required
    /**
    该方法返回当前类的标签, 该标签是当前类的唯一标识, 不可重复
    @return 字符串类型
    */
    + (NSString *)moduleName;

    @optional
    /**
    如果是通过push方式打开, 就实现该方法, 返回当前类的self
    
    @param params 传入的参数
    @param callback 传入的block回调
    @return 实现跳转协议类的self
    */
    - (id)open:(NSDictionary *)params callback:(void(^)(NSDictionary *))callback;
    
    /**
 如果是通过present方式打开, 就实现该方法, 返回当前类的self
       
    @param params 传入的参数
    @param callback 传入的block回调
    @return 实现跳转协议类的self
    */
    - (id)open_present:(NSDictionary *)params callback:(void(^)(NSDictionary *))callback;

    ```

3. `HCModuleCore`是个单例, 在初始化的时候, 通过runtime提供的方法把遵守``的类名缓存起来, 缓存信息存储在单例中, 存在于整个APP的生命周期中.

    ```
    HCModuleCore.m
    
    + (instancetype)moduleCore {
        static HCModuleCore *moduleCore;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            moduleCore = [[HCModuleCore alloc] init];
        });
        return moduleCore;
    }

    - (instancetype)init {
        self = [super init];
        if (self) {
            [self cacheModuleProrocolClasses];
        }
        return self;
    }
    
    /**
    把遵守HCModuleProtocol的类缓存起来
    */
    - (void)cacheModuleProrocolClasses {
        if (_cache.count != 0) {
            return;
        }
        NSMutableDictionary *tmpCache = [NSMutableDictionary dictionary];
        Class *classes;
        unsigned int outCount;
        classes = objc_copyClassList(&outCount);//获取全部类
        for (int i = 0; i < outCount; i++) {
            Class class = classes[i];
        
            //实现了HCModuleProtocol的类
            if (class_conformsToProtocol(class, @protocol(HCModuleProtocol))) {
                NSString *moduleName = [class moduleName];
                //重复检查
                NSCAssert([tmpCache objectForKey:moduleName] == nil, @"in class %@, module %@ has defined, please check!", NSStringFromClass(class), moduleName);
                [tmpCache setObject:NSStringFromClass(class) forKey:moduleName];
            }
        }
        free(classes);
        self.cache = [tmpCache copy];
}
    ```
4. 主要是通过传入的moduleName或者URl来获取到被打开页面的唯一标识, 再通过唯一标识从单例中缓存的遵守跳转协议的类中去找. 如果找到的话, `performSelector:withObject:withObject:`方法的返回值是响应的方法的返回值, 通过该函数获取到被跳转的类的实例. 
    
    ```
    HCModuleCore.m
    
    //根据moduleName返回对应注册的类
    - (id)moduleName:(NSString *)moduleName openWithParams:(NSDictionary *)params callback:(void(^)(NSDictionary *moduleInfo))callback {
        NSCAssert(moduleName != nil, @"moduleName can not be nil!");
        id module = [self moduleName:moduleName performSelectorName:@"open:callback:" withParams:params callback:callback];
        if (module == nil) {
            module = [self moduleName:moduleName performSelectorName:@"open_present:callback:" withParams:params callback:callback];
        }
        return module;
    }
    
    //获取缓存起来的响应相应协议方法的类
    - (id)moduleName:(NSString *)moduleName performSelectorName:(NSString *)selectorName withParams:(NSDictionary *)params callback:(void(^)(NSDictionary *moduleInfo))callback {
        NSCAssert(moduleName != nil && selectorName != nil, @"moduleName and selectorName can not be nil!");
        id module;
        NSString *clsName = self.cache[moduleName];
        if (clsName.length) {
            Class class = NSClassFromString(clsName);//根据缓存的类名字创建类
            SEL selec = NSSelectorFromString(selectorName);
            if (class) {
                id target = [[class alloc] init];//初始化一个类的对象
                if ([target respondsToSelector:selec]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
                    //performSelector:withObject:withObject:的返回值是响应的方法的返回值
                    module = [target performSelector:selec withObject:params withObject:callback];
#pragma clang diagnostic pop
                }
            }
        }
        return module;
    }
    ```
    
5. 然后有两个category, 分别是`UINavigationController+HCModuleCore`和`UIViewController+HCModuleCore`. 分别对应push和present的情况. 因为上面已经拿到了要跳转到的页面的实例, 这里就可以通过push或者present的方法跳转过去.   
    


-------
参考文章:
1.[iOS 组件化 —— 路由设计思路分析](https://halfrost.com/ios_router/)

2.[路由跳转的思考](http://awhisper.github.io/2016/06/12/%E8%B7%AF%E7%94%B1%E8%B7%B3%E8%BD%AC%E7%9A%84%E6%80%9D%E8%80%83/)

3.[iOS组件化方案探讨](http://wereadteam.github.io/2016/03/19/iOS-Component/)

4.[iOS应用架构谈-组件化方案](https://casatwy.com/iOS-Modulization.html)

5.[iOS10跳转系统设置的正确姿势](http://www.jianshu.com/p/bb3f42fdbc31)

6.[关于 iOS 系统功能的 URL 汇总列表](http://www.jianshu.com/p/32ca4bcda3d1)

7.[iOS应用间相互跳转](http://www.jianshu.com/p/b5e8ef8c76a3)

8.[从微信直接跳转到我们的APP](http://www.jianshu.com/p/c83164c2aec2)

9.[iOS Universal Links(通用链接)的使用](http://www.jianshu.com/p/1970fd59de12)


