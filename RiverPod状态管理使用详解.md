# RiverPod状态管理使用详解

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

1. #### 用全局函数自动创建的provider实现简单网络请求

   1. ###### 设置ProviderScope

      ```dart
      void main() {
        runApp(
          ProviderScope( // 放在最上面，整个APP都可以使用riverPod
            child: MyApp(),
          ),
        );
      }
      ```

   2. ###### 用全局函数自动创建一个provider

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

   3. ###### 通过Consumer读取、监听provider的数据变化

      ```dart
      import 'package:flutter/material.dart';
      import 'package:flutter_riverpod/flutter_riverpod.dart';
      
      import 'activity.dart';
      import 'provider.dart';
      
      /// The homepage of our application
      class Home extends StatelessWidget {
        const Home({super.key});
      
        @override
        Widget build(BuildContext context) {
          return Consumer(///使用Consumer主要就是为了使用ref参数
            builder: (context, ref, child) {///主要使用ref参数监听provider，而不是context
              // 通过ref监听 activityProvider. 首次读取会触发函数调用->请求数据
              // 使用ref.watch监听，当前Counsumer会自动rebuild当activityProvider更新时，这可能发生在：
              // the activityProvider updates. This can happen when:
              // - 状态从loading-->成功（data）/失败Error
              // - 刷新请求、provider状态数据被改变时
              final AsyncValue<Activity> activity = ref.watch(activityProvider);///可监听任意个provider
      
              return Center(
      /// 因为网络请求是异步的，而且可能失败，所以我们要处理可能的错误。也可以通过activity.isLoading判断是否在加载中
                child: switch (activity) {
                  AsyncData(:final value) => Text('Activity: ${value.activity}'),///请求成功，返回数据value
                  AsyncError() => const Text('Oops, something unexpected happened'),///请求失败
                  _ => const CircularProgressIndicator(),
                },
              );
            },
          );
        }
      }
      ```

   4. ###### 为了使用ref参数，我们上面使用了Consumer，事实上，以上代码完全等同于：

      ```dart
      class Home extends ConsumerWidget {///等同与上面的"Consumer" + "StatelessWidget"
        const Home({super.key});
      
        @override
        Widget build(BuildContext context, WidgetRef ref) {//注意 ：这里的build多了一个参数ref
          //这里可以通过ConsumerWidget的ref参数，监听、读取任何provider
          final AsyncValue<Activity> activity = ref.watch(activityProvider);
          // The rendering logic stays the same
          return Center(/* ... */);
        }
      }
      
      
      ///如果是StatefulWidget的话，就使用ConsumerStatefulWidget
      class Home extends ConsumerStatefulWidget {///等同于"Consumer" + "StatefulWidget".
        const Home({super.key});
      
        @override
        ConsumerState<ConsumerStatefulWidget> createState() => _HomeState();
      }
      
      // Notice how instead of "State", we are extending "ConsumerState".
      // This uses the same principle as "ConsumerWidget" vs "StatelessWidget".
      class _HomeState extends ConsumerState<Home> {
        @override
        void initState() {
          super.initState();
      
          //通过ref.watch方法监听，会自动触发rebuild，如果我们只是想单纯的监听变化而不rebuild，可使用手动监听
          ref.listenManual(activityProvider, (previous, next) {
            // TODO show a snackbar/dialog
          });
        }
      
        @override
        Widget build(BuildContext context) {///ref不是一个参数，而是ConsumerState的一个成员变量，所以直接使用
          // We can therefore keep using "ref.watch" inside "build".
          final AsyncValue<Activity> activity = ref.watch(activityProvider);
      
          return Center(/* ... */);
        }
      }
      ```

2. #### 用全局类自动创建的provider（也就是所谓的“notifier”）

   **问题**：第一步中使用函数创建的provivder，其数据是有函数计算结果返回的，后续读取该provider可以获取到该缓存，但是*<u>外部无法对该缓存进行修改</u>*，例如增加、删除等。因而我们可以用类来自动创建一个Provider，代码参考如下：

   ```dart
   //要使用riverpod代码自动生成provider，只需要类上加上注解：@riverpod，且继承一个类（名为：_$原类名）
   @riverpod
   class TodoList extends _$TodoList {///类TodoList将会自动生成一个provider
     @override
     Future<List<Todo>> build() async {///这个build函数，跟上面用函数创建Provider的函数意义完全一样
       return [
         Todo(description: 'Learn Flutter', completed: true),
         Todo(description: 'Learn Riverpod'),
       ];
     }
   }
   ```

   **注意：**

   - 当该类中只有Build方法时，其效果跟用函数生成Provider完全一样。
   - 不要在notifier的构造方法中写任何逻辑。因为ref和其他属性此时还没有初始化，应该在build方法中写逻辑。

   现在我们可以在TodoList类中增加其他方法，来**修改其内部持有的缓存State**，初始数据也就是build方法中返回的数据。

   ```dart
   @riverpod
   class TodoList extends _$TodoList {
     @override
     Future<List<Todo>> build() async => [/* ... */];
   
     ///1、增加一个方法,调用接口后，修改Provider所持有的缓存数据
     Future<void> addTodo(Todo todo) async {
       ///调用接口，
       final response = await http.post(
         Uri.https('your_api.com', '/todos'),
         headers: {'Content-Type': 'application/json'},
         body: jsonEncode(todo.toJson()),
       );
   
      ///2、将上面接口的返回数据转换成模型
       List<Todo> newTodos = (jsonDecode(response.body) as List)
           .cast<Map<String, Object?>>()
           .map(Todo.fromJson)
           .toList();
       //3、 使用最新数据，修改本地缓存（会自动通知所有监听者）
       state = AsyncData(newTodos);
     }
   }
   ```

   

