# Provider组件使用及源码解析

version: 6.1.2

- ### ChangeNotifierProvider

  这是使用最多的组件，基于inheritedWidget实现。该组件携带一个数据模型，共享给下面的所有child widget 访问(read)、监听(watch)。

  

- ### Provider

  该组件携带一个数据模型，共享给下面的所有child widget 访问(read)，但不能监听(watch)。你可能要问，那有何用？为何不直接使用全局变量？好处在于当Provider在树中被移除，数据模型也会被释放。

  由此可知：

  1、如果下面的控件监听（watch）Provider携带的数据变化，是不会得到通知的

  2、如果下面的控件读取（read）Provider携带的数据，获取的是当时的数据，后续变化后是不会得到通知

- ### ProxyProvider

  该组件功能 = Provider的功能  +  依赖上面其他模型（携带一个数据模型，该数据依赖于上面一个或多个Parrent Provider的数据。当其上面依赖的数据发送变化，自己携带的数据也会自动更新）。

  由此可知，如果下面的控件监听ProxyProvider的变化，是不会得到通知的，只能读取其数据

- 

### 