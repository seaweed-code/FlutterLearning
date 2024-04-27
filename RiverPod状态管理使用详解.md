## RiverPod状态管理使用详解

Riverpod是由Provider的作者，在Provider的基础上演变而来的，把Provider字母顺序打乱组成“RiverPod”，这是一个带**有缓存**、且响应式的框架。可以帮你实现带有缓存的网络请求，并在需要的时候重新请求。

### Provider框架的问题

- 必须依赖BuildContext来获取组件树上面的Provider中携带的信息

- 由于基于继承式组件原理，同一个数据类型，不能被多个 Provider使用【只以最近的一个Provider为准】

- 当数据变化时，监听者【例如Cosumer】，只能收到最新数据，无法收到老数据。

- Provider之间依赖关系不好管理

  ```dart
  @riverpod
  int number(NumberRef ref) {
    return Random().nextInt(10);
  }
  
  @riverpod
  int doubled(DoubledRef ref) {
    final number = ref.watch(numberProvider);///依赖上面的Provider
    .... ///如果需要，可以依赖任意个，而Provider则需要使用一些变体类Select0,Select1等来实现
    return number * 2;
  }
  ```

  

- 多个页面共享Provider，容易出现BUG。例如，页面A中有一个Provider，从A页面进入B中，B可正常访问其共享数据，但如果B页面显示而A页面被释放了，则会出现BUG。

  **而使用riverpod则不会出现此问题，参考代码：**

  ```dart
  @riverpod ///使用代码生成的provider,默认是.autoDispose
  int diceRoll(DiceRollRef ref) {///也就是说当无人监听时，state数据会被自动释放，一有人再使用，数据又会重新生成
    final dice = Random().nextInt(10);
    return dice;
  }
  
  @riverpod
  int cachedDiceRoll(CachedDiceRollRef ref) {
    final coin = Random().nextInt(10);
    if (coin > 5) throw Exception('Way too large.');
   
    ref.keepAlive();///强制保留缓存，即使无人监听
    return coin;
  }
  ```

- 不安全，当重构代码，或大型项目时，容易抛出ProviderNotFoundException的异常（事实上这就是起初开发RiverPod的主要原因之一）

- Provider无法做到没有监听者时，自动移除其所持有的状态。其状态一直持有，不管是否有监听者，直到其在组件树中被移除。

### Riverpod使用介绍

1. 

