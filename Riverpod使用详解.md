## Riverpod状态管理使用详解

### FutureProvider

相当于最基础的Provider + 异步处理

```dart
final configProvider = FutureProvider<Configuration>((ref) async {
  final content = json.decode(
    await rootBundle.loadString('assets/configurations.json'),
  ) as Map<String, Object?>;

  return Configuration.fromJson(content);
});

///UI监听方式
Widget build(BuildContext context, WidgetRef ref) {
  AsyncValue<Configuration> config = ref.watch(configProvider);

  return switch (config) {
    AsyncData(:final value) => Text(value.host),
    AsyncError(:final error) => Text('Error: $error'),
    _ => const CircularProgressIndicator(),
  };
}
```



### (Async)NotifierProvider

相当于最基础的Provider + 支持外部修改其缓存state

- 【优点】将所有的修改state的业务逻辑代码集中放置在Notifier类（如下的TodosNotifier）中，有利于代码维护
- NotifierProvider和AsyncNotifierProvider的区别主要是前者build方法是同步，后者是异步

```dart
class Todo {
  Todo(this.description, this.isCompleted);
  final bool isCompleted;
  final String description;
}

class TodosNotifier extends Notifier<List<Todo>> {
  @override
  List<Todo> build() {
    return [];
  }

  void addTodo(Todo todo) {
    state = [...state, todo];///修改缓存
  }
  // TODO add other methods, such as "removeTodo", ...
}

final todosProvider = NotifierProvider<TodosNotifier, List<Todo>>(() {
  return TodosNotifier();
});
```



### StateProvider

- 相比于最基础的Provider（不能外部修改其缓存state）,StateProvider提供一种简单方法让外部可以修改state
- 一种简化版本的[NotifierProvider](https://riverpod.dev/docs/providers/notifier_provider)，可以直接调用update方法更新state，而不用写notifier类进行修改。

```dart
final pageIndexProvider = StateProvider<int>((ref) => 0);

class PreviousButton extends ConsumerWidget {
  const PreviousButton({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // if not on first page, the previous button is active
    final canGoToPreviousPage = ref.watch(pageIndexProvider) != 0;

    void goToPreviousPage() {
      ref.read(pageIndexProvider.notifier).update((state) => state - 1);
    }

    return ElevatedButton(
      onPressed: canGoToPreviousPage ? goToPreviousPage : null,
      child: const Text('previous'),
    );
  }
}
```



### StateNotifierProvider

```dart
// The state of our StateNotifier should be immutable.
// We could also use packages like Freezed to help with the implementation.
@immutable
class Todo {
  const Todo({required this.id, required this.description, required this.completed});

  // All properties should be `final` on our class.
  final String id;
  final String description;
  final bool completed;

  // Since Todo is immutable, we implement a method that allows cloning the
  // Todo with slightly different content.
  Todo copyWith({String? id, String? description, bool? completed}) {
    return Todo(
      id: id ?? this.id,
      description: description ?? this.description,
      completed: completed ?? this.completed,
    );
  }
}

// The StateNotifier class that will be passed to our StateNotifierProvider.
// This class should not expose state outside of its "state" property, which means
// no public getters/properties!
// The public methods on this class will be what allow the UI to modify the state.
class TodosNotifier extends StateNotifier<List<Todo>> {
  // We initialize the list of todos to an empty list
  TodosNotifier(): super([]);

  // Let's allow the UI to add todos.
  void addTodo(Todo todo) {
    // Since our state is immutable, we are not allowed to do `state.add(todo)`.
    // Instead, we should create a new list of todos which contains the previous
    // items and the new one.
    // Using Dart's spread operator here is helpful!
    state = [...state, todo];
    // No need to call "notifyListeners" or anything similar. Calling "state ="
    // will automatically rebuild the UI when necessary.
  }

  // Let's allow removing todos
  void removeTodo(String todoId) {
    // Again, our state is immutable. So we're making a new list instead of
    // changing the existing list.
    state = [
      for (final todo in state)
        if (todo.id != todoId) todo,
    ];
  }

  // Let's mark a todo as completed
  void toggle(String todoId) {
    state = [
      for (final todo in state)
        // we're marking only the matching todo as completed
        if (todo.id == todoId)
          // Once more, since our state is immutable, we need to make a copy
          // of the todo. We're using our `copyWith` method implemented before
          // to help with that.
          todo.copyWith(completed: !todo.completed)
        else
          // other todos are not modified
          todo,
    ];
  }
}

// Finally, we are using StateNotifierProvider to allow the UI to interact with
// our TodosNotifier class.
final todosProvider = StateNotifierProvider<TodosNotifier, List<Todo>>((ref) {
  return TodosNotifier();
});
```

