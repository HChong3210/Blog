---
title: RN实践指南
date: 2017-12-01 23:45:59
tags:
    - RN
    - 基础知识
categories: 
    - 基础知识
    - RN
---

# RN实践指南

RN区别于传统项目三板斧(HTML, CSS, JS)我们可以在JS文件中写HTML的语法和CSS的布局, 这种语法称为JSX, 属于JS的语法拓展. RN使用的是虚拟DOM(存在于内存中的DOM), 他与DOM是一一对应的关系. 当界面发生变化时, 得益于DOM Diff算法, 我们就知道了虚拟DOM的变化, 从而高效的改动DOM, 避免了重新绘制DOM.

RN的本质就是JS与OC通过JavaScript Core的相互调用, 源码分析参考[这里](http://awhisper.github.io/2016/06/24/ReactNative%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/), JS与OC的通信机制参考[这里](http://blog.cnbang.net/tech/2698/).

## 生命周期

RN的生命周期, 大致可以用下面这张图来概括:

![RN声明周期](http://7rf9ir.com1.z0.glb.clouddn.com/3-3-component-lifecycle.jpg)

如图所示, RN的生命周期大致分为三个阶段:

* 第一阶段: 是组件第一次绘制阶段, 如图中的上面虚线框内, 在这里完成了组件的加载和初始化;

    * `getDefaultProps`该函数用于初始化一些默认属性, 全局仅调用一次, 通常会将固定的内容放在这个函数中进行初始化和赋值. *注意*这是ES5的写法, 在ES6中我们使用static成员来实现. 更多ES5与ES6使用对照参考[这里](http://bbs.reactnative.cn/topic/15/react-react-native-%E7%9A%84es5-es6%E5%86%99%E6%B3%95%E5%AF%B9%E7%85%A7%E8%A1%A8)
    
    * `getInitialState`该函数用于对组件一些状态进行初始化, 可以将控制控件状态的一些变量放在这里初始化. 通过get取值, set写值. *注意*这是ES5的写法, 在ES6中我们常使用下面这种写法:
    
          ```
          constructor(props){
              super(props);
              this.state = {
                  key: value,
              };
          }
          ```
    * `componentWillMount`, 该方法表示组件将要加载到虚拟DOM中, 在render()方法之前执行, 整个生命周期只执行一次.
    
    * `componentDidMount`, 该方法在组件第一次绘制之后调用, 通知组件已经加载完成, 这个函数整个生命周期只被调用一次, 这个函数被调用的时候就说虚拟DOM已经构建完成, 可以在这个函数中获取其中的元素和子组件. 这个函数之后, 就进入了稳定状态, 只有等到其他事件触发才会再次调用其他的函数.
    
* 第二阶段: 是组件在运行和交互阶段, 如图中左下角虚线框, 这个阶段组件可以处理用户交互, 或者接收事件更新界面;

    * `componentWillReceiveProps`, 在组件接收到其父组件传递的props的时候执行, 参数为父组件传递的props. 在组件的整个生命周期可以多次执行, 通常在此方法接收新的props值, 重新设置state. 
        
        ```
        componentWillReceiveProps(nextProps) {
             this.setState({
               //key : value
             });
        }
        ```
    * `shouldComponentUpdate`, 当组件接收到新的props或者state改变时, 都会调用该方法, 该函数的返回值直接决定是否要更新组件. 该方法包含两个参数, 分别是props和state. 该方法在组件的整个生命周期可以多次执行. 如果该方法返回false, 则componentWillUpdate(nextProps, nextState)及其之后执行的方法都不会执行, 组件则不会进行重新渲染.

        ```
        shouldComponentUpdate(nextProps, nextState) {
          return true;
        }
        ```
    * `componentWillUpdate`, 如果组件状态或者属性改变, 并且`shouldComponentUpdate`的返回值为true, 就会更新组件, 在组件更新之前会先调用该方法, 该方法在整个声明周期中可以被多次执行. 在该方法中可以做一些更新界面之前要做的事情. *注意*, 在这个函数中不能使用set来修改state的值. 该函数调用之后, 就会把 nextProps 和 nextState 分别设置到 this.props 和 this.state 中. 紧接着这个函数, 就会调用 `render()` 来更新界面.

        ```
        componentWillUpdate(nextProps, nextState) {
            
        }
        ```
        
    * `render`, 方法用于渲染组件, 在初始化阶段和运行期阶段都会执行.

        ```
        render() {
          return(
            <View/>
          );
        }
        ```
        
    * `componentDidUpdate`, 该方法在`render()`之后立刻调用, 包含两个参数, 分别是props和state. 该方法在组件的整个生命周期可以多次执行. 因为到这里已经完成了属性和状态的更新, 此时, 函数的参数就变成prevProps和prevState.

        ```
        componentDidUpdate(prevProps, prevState)(  
            
        )
        ```

* 第三阶段: 是组件卸载消亡的阶段, 如图中右下角的虚线框中, 这里做一些组件的清理工作;

    * `componentWillUnmount`, 当组件将要从界面上移除的时候, 就会调用该方法.

## 布局

RN布局采用的是前端的flex布局, 关于这种布局方式, 网上的教程一大堆, 强烈推荐阮阮一峰[这篇文章](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html).

## 组件间通信

RN的组件间通信主要分为父子组件和跨级组件间通信.

* 父子组件通信, 父组件设置属性参数, 子组件通过props取得参数.
* 子父组件通信, 也通过props.
    
    ```
    //子组件
    class Son extends Component {
        constructor(props) {
            super(props);
        }
        componentDidMount() {
            //子组件给父组件的方法传参
            this.props.onChange('newVal');
        }
        render() {
            return (
                <View />
            );
        }
    }
    
    //父组件
    class Father extends Component {
        constructor(props) {
            super(props);
            this.state = {
                key: 'defVal'
            };
        }
        
        //父组件接受子组件的参数，并改变 state
        handleChange(val) {
            this.setState({
                key: val 
            });
        }
        
        render() {
            return (
                <Son onChange={this.clickItem}/>
            );
        }
        
        clickItem = (item) => {
            this.handleChange(item);
        }
    }
    ```
* 跨级组件间通信 
    如果是兄弟组件可以先把值向上传递到父组件, 在向下传递到子组件. 如果是嵌套多层的传值可以使用context对象. 参考[React 中组件间通信的几种方式](https://www.jianshu.com/p/fb915d9c99c4).
    
* 观察者模式通信, 使用eventProxy对象. 该对象共有四个函数, 使用方法很简单.
    * on, one: on 与 one 函数用于订阅者监听相应的事件, 并将事件响应时的函数作为参数, on 与 one 的唯一区别就是, 使用 one 进行订阅的函数, 只会触发一次, 而使用 on 进行订阅的函数, 每次事件发生相应时都会被触发.
    * trigger: trigger 用于发布者发布事件, 将除第一参数(事件名)的其他参数, 作为新的参数, 触发使用 one 与 on 进行订阅的函数.
    * off: 用于解除所有订阅了某个事件的所有函数.
* 使用Refs, Refs的关键在于保存组件的实例, 实例代码如下(子组件向父组件传递).
    
    ```
    //子组件
    class Son extends Component {
        constructor(props) {
            super(props);
        }
        
        //开放的实例方法
        doIt() {
            //...做点什么
        }
        
        render() {
            return (
                <View />
            );
        }
    }
    
    //父组件
    class Father extends Component {
        constructor(props) {
            super(props);
        }
        
        render() {
            //this.yyy 保存组件的实例
            return (
                <Son ref={(xxx) => {this.yyy = xxx;}} />
            );
        }
        
        componentDidMount() {
            //调用组件的实例方法
            this.myCpt.doIt();
        }
    }
    ```
    
* 使用global
    global 类似浏览器里的 window 对象, 它是全局的, 一处定义, 所有组件都可以访问, 一般用于存储一些全局的配置参数或方法. 使用场景: 全局参数不想通过 props 层层组件传递, 有些组件对此参数并不关心, 只有嵌套的某个组件使用.
    
    ```
    global.isOnline = true;
    ```
    
## MobX与RN的结合使用

强烈推荐使用[MobX](https://mobx.js.org/index.html)以数据驱动视图, 通过数据绑定, 我们只需修改数据本身, 便可自动更新视图. [中文文档](http://cn.mobx.js.org/)

## MobX与RN的结合使用范式
我们尽量推荐一个模块由若干pages, 若干models和stores组成. 他们分别对应到iOS开发中的VC, M和VM三层. 如果使用store, 那么建议不要再使用state来控制UI状态, 建议都放进store里面.

* pages内就是一些组件, 业务逻辑
* store内可以定义一些跟数据相关的action, 对接口获取回来的数据做一层映射, 利用Mobx来实现数据变化自动分发渲染, 统一规划数据层.
* model类似于OC的瘦model, 也可以在里面做数据映射和数据校验

## MobX使用注意
关于MobX的使用还要注意下面这些: 
> MobX 会对在追踪函数执行过程中读取现存的可观察属性做出反应。
> “读取” 是对象属性的间接引用，可以用`.`(例如 user.name) 或者 `[]`(例如 user['name']) 的形式完成。
> “追踪函数” 是 computed 表达式、observer 组件的 render() 方法和 when、reaction 和 autorun 的第一个入参函数。
> “过程(during)” 意味着只追踪那些在函数执行时被读取的 observable 。这些值是否由追踪函数直接或间接使用并不重要。
> MobX 追踪属性访问, 而不是值. 也可以理解为c/c++当中的指针的概念. 例如一个数组新增一个元素, 是不会触发MobX.


## MobX与其他状态管理框架的对比
MobX与Redux, Flux总是要放在一起对比的, 他们的对比[看这里](http://zhenhua-lee.github.io/react/state-manage.html). 

------

参考资料:
1.[React Native组件的生命周期](https://www.jianshu.com/p/2a1571d23cf1)

2.[React Native 中组件的生命周期](https://www.race604.com/react-native-component-lifecycle/)

3.[React/React Native的ES5 ES6对照表](http://bbs.reactnative.cn/topic/15/react-react-native-%E7%9A%84es5-es6%E5%86%99%E6%B3%95%E5%AF%B9%E7%85%A7%E8%A1%A8)

4.[ReactNative iOS源码解析](http://awhisper.github.io/2016/06/24/ReactNative%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

5.[React Native 从入门到原理](https://bestswifter.com/react-native/)

6.[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)

7.[在原生和React Native间通信](https://reactnative.cn/docs/0.39/communication-ios.html)


