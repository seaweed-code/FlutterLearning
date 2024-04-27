## RiverPod状态管理使用详解

Riverpod是由Provider的作者，在Provider的基础上演变而来的，把Provider字母顺序打乱组成“RiverPod”，这是一个带**有缓存**、且响应式的框架。可以帮你实现带有缓存的网络请求，并在需要的时候重新请求。

### Provider框架的问题

- 必须依赖BuildContext来获取组件树上面的Provider中携带的信息
- 由于基于继承式组件原理，同一个数据类型，不能被多个 Provider使用【只以最近的一个Provider为准】
- 当数据变化时，监听者【例如Cosumer】，只能收到最新数据，无法收到老数据。
- Provider之间依赖关系不好管理
- 多个页面共享Provider，容易出现BUG。例如，页面A中有一个Provider，从A页面进入B中，B可正常访问其共享数据，但如果B页面显示时，A页面被释放了，则会出现BUG。
- 不安全，当重构代码，或大型项目时，容易抛出ProviderNotFoundException的异常（事实上这就是起初开发RiverPod的主要原因之一）
- Provider无法做到没有监听者时，自动移除其所持有的状态。其状态一直持有，不管是否有监听者，直到其在组件树中被移除。

### Riverpod使用介绍

1. 

