# Provider组件使用及源码解析

version: 6.1.2

1. ### ChangeNotifierProvider

   **父类：**ListenableProvider，主要区别在于ListenableProvider监听的是Listenable模型，而ChangeNotifierProvider则监听ChangeNotifier模型（一种Listenable）。如果需要自定义Listenable或者监听其他Listenable例如Animation，则可以使用父类。

   **功能：**自己会监听一个ChangeNotifier模型，一旦模型改变后，会调用markNeedsNotifyDependents，通知所有依赖者rebuild

   

   这是使用最多的组件，基于inheritedWidget实现。该组件携带一个数据模型，共享给下面的所有child widget 访问(read)、监听(watch)。

   由此可知：

   1、如果下面的控件监听（watch）ChangeNotifierProvider携带的数据变化，数据改变后会通知所有监听者

   2、如果下面的控件读取（read）ChangeNotifierProvider携带的数据，获取的是当时的数据，后续变化后是不会得到通知

   3、数据模型必须继承ChangeNotifier，且需要手动调用notifyListeners方法才能通知到ChangeNotifierProvider rebuild，从而触发InheritedWidget的逻辑->rebuild all dependents

2. ### ListenableProvider

   **父类：**InheritedProvider

   **作用：**与ChangeNotifierProvider一样，只不过他可以监听更为原始的Listenable类型的模型，通过源码可知，其底层原理是收到消息后，直接调用InheritedContext的markNeedsNotifyDependents方法通知所以依赖更新，而不是通过rebuild InheritedWidget来触发。

   **QA**：下面两个构造函数：传create，和传value又何区别呢？

   ```dart
   ///作为父类，可以监听更为原始的Listenable类型数据变化
   class ListenableProvider<T extends Listenable?> extends InheritedProvider<T> {
     ListenableProvider({
       Key? key,
       required Create<T> create,
       Dispose<T>? dispose,
       bool? lazy,
       TransitionBuilder? builder,
       Widget? child,
     }) : super(
             key: key,
             startListening: _startListening,
             create: create,
             dispose: dispose,
             lazy: lazy,
             builder: builder,
             child: child,
           );
   
     /// Provides an existing [Listenable].
     ListenableProvider.value({
       Key? key,
       required T value,
       UpdateShouldNotify<T>? updateShouldNotify,
       TransitionBuilder? builder,
       Widget? child,
     }) : super.value(
             key: key,
             builder: builder,
             value: value,
             updateShouldNotify: updateShouldNotify,
             startListening: _startListening,///当调用_startListening时就已经添加了监听，当调用返回值则取消监听
             child: child,///作为优化使用
           );
   
     ///全局函数
     static VoidCallback _startListening(
       InheritedContext<Listenable?> e,
       Listenable? value,
     ) {
       value?.addListener(e.markNeedsNotifyDependents);///监听方法就是通知InheritedWidget更新其所有依赖
       return () => value?.removeListener(e.markNeedsNotifyDependents);
     }
   }
   ```

   

3. ### InheritedProvider

   携带数据的Provider都是继承自此类

   ```dart
   class InheritedProvider<T> extends SingleChildStatelessWidget {
     ///携带一个，从create方法返回的数据作为共享数据
     InheritedProvider({
       Key? key,
       Create<T>? create,
       T Function(BuildContext context, T? value)? update,
       UpdateShouldNotify<T>? updateShouldNotify,
       void Function(T value)? debugCheckInvalidValueType,
       StartListening<T>? startListening,
       Dispose<T>? dispose,
       this.builder,
       bool? lazy,
       Widget? child,
     })  : _lazy = lazy,
           _delegate = _CreateInheritedProvider(///由此类完成
             create: create,
             update: update,
             updateShouldNotify: updateShouldNotify,
             debugCheckInvalidValueType: debugCheckInvalidValueType,
             startListening: startListening,
             dispose: dispose,
           ),
           super(key: key, child: child);
   
     /// 携带一个已经存在的数据，Provider从数中移除时不会自动dispose模型
     InheritedProvider.value({
       Key? key,
       required T value,
       UpdateShouldNotify<T>? updateShouldNotify,
       StartListening<T>? startListening,
       bool? lazy,
       this.builder,
       Widget? child,
     })  : _lazy = lazy,
           _delegate = _ValueInheritedProvider(////由此类完成
             value: value,
             updateShouldNotify: updateShouldNotify,
             startListening: startListening,
           ),
           super(key: key, child: child);
   
     InheritedProvider._constructor({
       Key? key,
       required _Delegate<T> delegate,
       bool? lazy,
       this.builder,
       Widget? child,
     })  : _lazy = lazy,
           _delegate = delegate,
           super(key: key, child: child);
   
     final _Delegate<T> _delegate;///实际数据保存在此_ValueInheritedProvider or _CreateInheritedProvider
     final bool? _lazy;///是否需要懒加载
   
     /// For an explanation on the `child` parameter that `builder` receives,
     /// see the "Performance optimizations" section of [AnimatedBuilder].
     final TransitionBuilder? builder;
   
     ///一旦调用下面的方法，才能通知所有的依赖者rebuild,build函数中直接调用本函数
     ///QA：此函数会被动态调用吗？
     @override
     Widget buildWithChild(BuildContext context, Widget? child) {
       ///这是最为核心的代码
       //当函数重新调用时，_InheritedProviderScope（InheritedWidget的子类）会重新创建，但是child参数不会。根据继承式组件原理，一旦InheritedWidget重新创建，所有依赖者depedents都会被标记为重建，从而更新所有依赖者
       return _InheritedProviderScope<T?>(///InheritedWidget子类，这是数据能共享的本质
         owner: this,//真正的数据保存在自己手里，而依赖者需要数据只能通过InheritedWidget来找，
         debugType: kDebugMode ? '$runtimeType' : '',
         child: builder != null
             ? Builder(
                 builder: (context) => builder!(context, child),
               )
             : child!,
       );
     }
   }
   ```

   

4. ### Provider

   父类：InheritedProvider

   没有核心内容，基本上就是对父类的简单封装

   该组件携带一个数据模型，共享给下面的所有child widget 访问(read)，但不能监听(watch)。你可能要问，那有何用？为何不直接使用全局变量？好处在于当Provider在树中被移除，数据模型也会被释放。

   由此可知：

   1、如果下面的控件监听（watch）Provider携带的数据变化，是不会得到通知的

   2、如果下面的控件读取（read）Provider携带的数据，获取的是当时的数据，后续变化后是不会得到通知

5. ### ProxyProvider

   该组件功能包括 ：

   1、 Provider的功能  

   2、 可依赖上面其他模型（依赖于上面一个或多个Parrent Provider的数据。当其上面依赖的数据发送变化，自己携带的数据也会自动更新）。

   ```dart
   // {@template provider.proxyprovider}
   /// A provider that builds a value based on other providers.
   ///
   /// The exposed value is built through either `create` or `update`, then passed
   /// to [InheritedProvider].
   ///
   /// As opposed to `create`, `update` may be called more than once.
   /// It will be called once the first time the value is obtained, then once
   /// whenever [ProxyProvider] rebuilds or when one of the providers it depends on
   /// updates.
   ///
   /// [ProxyProvider] comes in different variants such as [ProxyProvider2]. This
   /// is syntax sugar on the top of [ProxyProvider0].
   ///
   /// As such, `ProxyProvider<A, Result>` is equal to:
   /// ```dart
   /// ProxyProvider0<Result>(
   ///   update: (context, result) {
   ///     final a = Provider.of<A>(context);
   ///     return update(context, a, result);
   ///   }
   /// );
   /// ```
   ///
   /// Whereas `ProxyProvider2<A, B, Result>` is equal to:
   /// ```dart
   /// ProxyProvider0<Result>(
   ///   update: (context, result) {
   ///     final a = Provider.of<A>(context);
   ///     final b = Provider.of<B>(context);
   ///     return update(context, a, b, result);
   ///   }
   /// );
   /// ```
   ///
   /// This last parameter of `update` is the last value returned by either
   /// `create` or `update`.
   /// It is `null` by default.
   ///
   /// `update` must not be `null`.
   ///
   /// See also:
   ///
   ///  * [Provider], which matches the behavior of [ProxyProvider] but has only
   ///     a `create` callback.
   /// {@endtemplate}
   class ProxyProvider<T, R> extends ProxyProvider0<R> {
     /// Initializes [key] for subclasses.
     ProxyProvider({
       Key? key,
       Create<R>? create,
       required ProxyProviderBuilder<T, R> update,
       UpdateShouldNotify<R>? updateShouldNotify,
       Dispose<R>? dispose,
       bool? lazy,
       TransitionBuilder? builder,
       Widget? child,
     }) : super(
             key: key,
             lazy: lazy,
             builder: builder,
             create: create,
             update: (context, value) => update(
               context,
               Provider.of(context),///监听T的变化
               value,
             ),
             updateShouldNotify: updateShouldNotify,
             dispose: dispose,
             child: child,
           );
   }
   ```

   

6. ### ChangeNotifierProxyProvider

   该组件功能包括 ：

   1、 ChangeNotifierProvider的功能  

   2、 可依赖上面其他模型（依赖于上面一个或多个Parrent Provider的数据。当其上面依赖的数据发送变化，自己携带的数据也会自动更新）。

7. ### MultiProvider避免多层嵌套

   注意，由于继承自Nested类，会自动把providers数组中的Provider按序设置成父子关系，所以若数组元素中某个Provider含有child参数则会无效，因为Nested会自动把它下一个元素设置为它的child

   ```dart
   /// A provider that merges multiple providers into a single linear widget tree.
   /// It is used to improve readability and reduce boilerplate code of having to
   /// nest multiple layers of providers.
   ///
   /// As such, we're going from:
   ///
   /// ```dart
   /// Provider<Something>(
   ///   create: (_) => Something(),
   ///   child: Provider<SomethingElse>(
   ///     create: (_) => SomethingElse(),
   ///     child: Provider<AnotherThing>(
   ///       create: (_) => AnotherThing(),
   ///       child: someWidget,
   ///     ),
   ///   ),
   /// ),
   /// ```
   ///
   /// To:
   ///
   /// ```dart
   /// MultiProvider(
   ///   providers: [
   ///     Provider<Something>(create: (_) => Something()),
   ///     Provider<SomethingElse>(create: (_) => SomethingElse()),
   ///     Provider<AnotherThing>(create: (_) => AnotherThing()),
   ///   ],
   ///   child: someWidget,
   /// )
   /// ```
   class MultiProvider extends Nested {
     MultiProvider({
       Key? key,
       required List<SingleChildWidget> providers,
       Widget? child,///为了优化使用，当builder被多次调用时，child不会重新创建，会原封不动的传递给builder，然而此处builder并不会被多次调用，所以此处的优化其实并没有意义
       TransitionBuilder? builder,///动态构建child
     }) : super(
             key: key,
             children: providers,
             child: builder != null
                 ? Builder(///这里说明,builder和child参数都是可选的，优先使用builder，没有就使用child
                     builder: (context) => builder(context, child),
                   )
                 : child,
           );
   }
   ```

   

8. ### Consumer 监听所有数据变化

   一个Consumer就是一个监听者，可以监听上面一个或多个数据模型的变化。

   ```dart
   class Consumer<T> extends SingleChildStatelessWidget {
     Consumer({
       Key? key,
       required this.builder,
       Widget? child,
     }) : super(key: key, child: child);
   
     final Widget Function(
       BuildContext context,
       T value,
       Widget? child,
     ) builder;
   
     @override
     Widget buildWithChild(BuildContext context, Widget? child) {
       return builder(
         context,
         Provider.of<T>(context),///非常简单的监听者，代码简单，此处通过context监听了模型T的变化
         child,
       );
     }
   }
   ```

   

9. ### Selector 监听部分数据变化

   1、一个Selector也是一个监听者，与Consumer不同的是，可选择性监听部分变化。它本身也是全量监听了模型的变化，只不过其内部做了判断，是否需要部分刷新，底层逻辑没有用到InheritedWidget中的Aspect字段。源码如下：

   ```dart
   /// 在dart 3.0后要做到同时监听多个属性最简单的方式就是使用Dart语言中的`Records`类型（类似于swift中元组，C++结构体），而不用去重新定义一个新的类再重写==方法
   /// ```dart
   /// Selector<Foo, ({String item1, String item2})>(
   ///   selector: (_, foo) => (item1: foo.item1, item2: foo.item2),///监听item1和item2
   ///   builder: (_, data, __) {///只有item1或者item2发生变化，才会触发rebuild
   ///     return Text('${data.item1}  ${data.item2}');
   ///   },
   /// );
   /// ```
   class Selector<A, S> extends Selector0<S> {
     Selector({
       Key? key,
       required ValueWidgetBuilder<S> builder,
       required S Function(BuildContext, A) selector,
       ShouldRebuild<S>? shouldRebuild,
       Widget? child,
     }) : super(
             key: key,
             shouldRebuild: shouldRebuild,
             builder: builder,
             selector: (context) => selector(context, Provider.of(context)),///此处最为关键，这里的context相当于中间商，它会监听模型A的任何变化，A一旦有任何改变contenx代表widget会被rebuild，但是内部会根据shouldRebuild来判断是否需要重新rebuild其child
             child: child,
           );
   }
   
   class Selector0<T> extends SingleChildStatefulWidget {
     Selector0({
       Key? key,
       required this.builder,
       required this.selector,
       ShouldRebuild<T>? shouldRebuild,
       Widget? child,///作为缓存优化使用，自动传给builder方法
     })  : _shouldRebuild = shouldRebuild,
           super(key: key, child: child);
     
     final ValueWidgetBuilder<T> builder;///数据变化后，需要刷新则会调用重新构建子widget
     final T Function(BuildContext) selector;///对哪些字段感兴趣？就返回他们
     final ShouldRebuild<T>? _shouldRebuild;///如果感兴趣的内容没有发生变化则不需要更新
   
     @override
     _Selector0State<T> createState() => _Selector0State<T>();
   }
   
   class _Selector0State<T> extends SingleChildState<Selector0<T>> {
     T? value;
     Widget? cache;
     Widget? oldWidget;
   
     @override
     Widget buildWithChild(BuildContext context, Widget? child) {
       final selected = widget.selector(context);///用户感兴趣的数据
   
       final shouldInvalidateCache = oldWidget != widget ||
           (widget._shouldRebuild != null &&
               widget._shouldRebuild!(value as T, selected)) ||
           (widget._shouldRebuild == null &&
               !const DeepCollectionEquality().equals(value, selected));///数据有没有发生变化
       if (shouldInvalidateCache) {
         value = selected;
         oldWidget = widget;
         cache = Builder(///重新构建新的Widget
           builder: (context) => widget.builder(
             context,
             selected,
             child,
           ),
         );
       }
       return cache!;
     }
     
   }
   ```

   2、部分监听，也可以使用context的select方法，源码如下，可以看到这个方法不如上面的方式灵活。但是底层使用了InheritedWidget Aspect参数来保存获取感兴趣的部分数据，例如，如果我想监听模型的属性A变化 且 B 没有变化时刷新UI，或者我想监听多个模型的某些属性变化 等复杂逻辑，下面的方法则无法支持。

   ```dart
   extension SelectContext on BuildContext {
     ///如果一个widget只想监听 peson 模型的 name字段，但是不想监听age字段
     /// 你就不能使用`context.watch`/[Provider.of]. 等方法
     /// 我们可以使用 [select], 代码示例:
     ///
     /// ```dart
     /// Widget build(BuildContext context) {
     ///   final name = context.select((Person p) => p.name);
     ///   当且仅当name字段变化时会自动刷新
     ///   return Text(name);
     /// }
     /// ```
     /// It is fine to call `select` multiple times.
     R select<T, R>(R Function(T value) selector) {
       ///从当前context找到最近的Provider<T>
         final inheritedElement = Provider._inheritedElementOf<T>(this);
      ///取出里面的数据
         final value = inheritedElement?.value;
      ///调用selector得到用户感兴趣的字段内容
         final selected = selector(value);
   
         if (inheritedElement != null) {
           dependOnInheritedElement(////设置依赖，Aspect是一个函数
             inheritedElement,
             aspect: (T? newValue) {
               return !const DeepCollectionEquality()
                   .equals(selector(newValue), selected);
             },
           );
         } else {
           // tell Flutter to rebuild the widget when relocated using GlobalKey
           // if no provider were found before.
           dependOnInheritedWidgetOfExactType<_InheritedProviderScope<T?>>();
         }
         return selected;
     }
   }
   ```

   

10. 