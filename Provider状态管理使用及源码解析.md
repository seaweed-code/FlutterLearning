# Provider组件使用及源码解析

version: 6.1.2

- ### ChangeNotifierProvider

  这是使用最多的组件，基于inheritedWidget实现。该组件携带一个数据模型，共享给下面的所有child widget 访问(read)、监听(watch)。

  

- ### Provider

  该组件携带一个数据模型，共享给下面的所有child widget 访问(read)，但不能监听(watch)。你可能要问，那有何用？为何不直接使用全局变量？好处在于当Provider在树中被移除，数据模型也会被释放。

- ### ProxyProvider

  该组件可以依赖一个或多个其上的模型，并跟随变化，但是当数据变化时不会通知下面监听者更新，只能让下面的监听者读取

- 

### 