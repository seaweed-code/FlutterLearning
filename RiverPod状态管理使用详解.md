## RiverPod状态管理使用详解

Riverpod是由Provider的作者，在Provider的基础上演变而来的，把Provider字母顺序打乱组成“RiverPod”，这是一个带**有缓存**、且响应式的状态管理组件。可以帮你实现带有缓存的网络请求，并在需要的时候重新请求。

### Provider组件的局限性

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
    .... ///如果需要，可以依赖任意个，而Provider则需要使用一些变体类例如Select0,Select1等来实现
    return number * 2;
  }
  ```

- 多个页面共享Provider，容易出现BUG。例如，页面A中有一个Provider，从A页面进入B中，B可正常访问其共享数据，但如果B页面显示而A页面被释放了，则会出现BUG。

- 不安全，当重构代码，或大型项目时，容易抛出ProviderNotFoundException的异常（事实上这就是起初开发RiverPod的主要原因之一）

- Provider无法做到没有监听者时，自动移除其所持有的状态。其状态一直持有，不管是否有监听者，直到其在组件树中被移除。

  而riverpod可以轻松做到：

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

- Provider中数据变化后，依赖的child widget会被全部rebuild，如果只想要监听数据变化但不rebuild，则无法做到，但是riverpod中ref.listen方法可以轻松做到，参考：

  ```dart
  class DiceRollWidget extends ConsumerWidget {
    const DiceRollWidget({super.key});
  
    @override
    Widget build(BuildContext context, WidgetRef ref) {
      ref.listen(diceRollProvider, (previous, next) {///只是监听数据变化，去做指定的事，当前widget不会更新
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Dice roll! We got: $next')),
        );
      });
      return TextButton.icon(
        onPressed: () => ref.invalidate(diceRollProvider),
        icon: const Icon(Icons.casino),
        label: const Text('Roll a dice'),
      );
    }
  }
  ```

- 缺乏可依赖的参数化功能，而riverpod支持，代码如下：

  ```dart
  ///使用该Provider需要额外传递一个参数id，可增加任意数量、类型的参数
  final messagesFamily = FutureProvider.family<Message, String>((ref, id) async {
    return dio.get('http://my_api.dev/messages/$id');
  });
  
  ///由于我们的messagesFamilyProvider使用时必须多传递一个参数id，因而下面的代码将会失效了
    Widget build(BuildContext context, WidgetRef ref) {
    final response = ref.watch(messagesFamily);// Error – messagesFamily is not a provider
  }
  
  ///我们应该这样使用：
  Widget build(BuildContext context, WidgetRef ref) {
    final response = ref.watch(messagesFamily('id'));///传递一个字符串id='id'过去才行
  }
  ```

### Riverpod使用介绍

1. 简单网络请求

   步骤如下：

   1. 设置ProviderScope

      ```dart
      void main() {
        runApp(
          ProviderScope( // 放在最上面，整个APP都可以使用riverPod
            child: MyApp(),
          ),
        );
      }
      ```

   2. 用全局函数自动创建一个provider

      ```dart
      import 'dart:convert';
      import 'package:http/http.dart' as http;
      import 'package:riverpod_annotation/riverpod_annotation.dart';
      import 'activity.dart';
      
      part 'provider.g.dart';///自动生成的代码
      
      ///这将自动创建一个名为：`activityProvider`的provider
      ///下面方法只会在provider被首次读取时执行，后续直接使用缓存
      ///当没有人监听此provider时，数据会被释放，后续一旦又有人访问，则会重新执行此函数，生成缓存
      ///此次并没有处理错误，riverpod会自动处理错误
      @riverpod
      Future<Activity> activity(ActivityRef ref) async {
        ///网络请求，JSON转换模型
        final response = await http.get(Uri.https('boredapi.com', '/api/activity'));
        final json = jsonDecode(response.body) as Map<String, dynamic>;
        
        return Activity.fromJson(json);
      }
      ```

   3. 

2. 房

