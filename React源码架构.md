# React源码架构

React16的架构分为三层：
* Scheduler（调度器）：调度任务的优先级
* Reconciler（协调器）：找出需要更改的操作，比如diff算法
* Renderer（渲染器）：负责将改变渲染到屏幕上

## 源码结构

react采用的是多包管理的方式，将各种包独立发布到npm进行维护。

### packages/react

定义了react所有的全局api，如：

* React.createElement
* React.Component
* React.Children

### shared

定义了react公用的变量和方法

### renderer

以下的包都和renderer相关，不同平台使用不同的renderer

* packages/react-art：canvas和svg专用
* packages/react-dom：浏览器和SSR（服务端渲染）的入口
* packages/react-native-renderer：RN入口
* packages/react-noop-renderer：用于debug fiber
* packages/react-test-renderer：用于测试，输出react纯js对象

### experimental

实验性的feature

* packages/react-server：创建自定义SSR流
* packages/react-client：创建自定义的流
* packages/react-fetch：用于数据请求
* packages/react-interactions：用于测试交互相关的内部特性，比如React的事件模型
* packages/react-reconciler：Reconciler的实现，你可以用他构建自己的Renderer

### 辅助类的包

* packages/react-is：用于判断react组件的类型
* packages/react-refresh：react对热重载的实现
* packages/react-devtool-*：和开发相关的各种包

### scheduler

scheduler的实现


